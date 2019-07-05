# Ceph

本文使用ceph-deploy工具进行部署，接下来在部署机器上进行如下操作

# 创建用户

推荐使用非root但具有root权限的用户进行部署同时此用户名不能为ceph

```
// 创建cephdeploy用户
adduser cephdeploy					

// 设置用户密码
passwd cephdeploy						

// 给用户添加root权限
chmod -v u+w /etc/sudoers
cephdeploy ALL=(ALL)    NOPASSWD: ALL
chmod -v u-w /etc/sudoers
```



# 配置主机名解析

将集群所有机器上的hosts信息进行添加

```
vim /etc/hosts
10.16.200.119   g1-k8s-harbor-v01
......
```



# 免密钥操作

```
// 创建公私钥
ssh-keygen

// 分发公钥
ssh-copy-id cephdeploy@g1-k8s-harbor-v01
......
```



```
vim ~/.ssh/config
Host g1-k8s-harbor-v01     
  Hostname g1-k8s-harbor-v01
  User cephdeploy
......				//	多个时填写多个  
```



# 安装Ntp时间服务器

```
// 安装epel源
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

// 安装ntp
yum install  -y  ntp ntpdate ntp-doc


// 开启服务
systemctl start ntpd.service
systemctl enable ntpd.service
```



# 关闭防火墙

```
systemctl stop firewalld
systemctl disable firewalld

// 关闭Selinux
setenforce 0
vim /etc/selinux/config
SELINUX=disabled
```



# 安装Ceph集群

## 添加yum源(所有server)

```
# cat /etc/yum.repos.d/ceph.repo 
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/$basearch
enabled=1
priority=1
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/noarch
enabled=1
priority=1
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/SRPMS
enabled=0
priority=1
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
```



## 安装ceph-deploy

```
 // 安装 2.0.1版本
 yum install -y ceph-deploy
 
```



## 安装ceph相关包

```
// 创建部署目录
mkdir ceph-deploy
cd ceph-deploy

// 安装l版本（12.2.5）的ceph相关包
ceph-deploy install --release=luminous g1-k8s-harbor-v01  // 多节点时在末端添加即可用空格隔开
```



## 准备工作

```
ceph-deploy new g1-k8s-harbor-v01	// 在当前目录会生成ceph.conf等相关文件

// 由于是单节点需要在ceph.conf添加内容
osd pool default size = 1

// 声明pubulic网段
public network = 10.16.200.0/24

// 推送配置文件
ceph-deploy --overwrite-conf config push g1-k8s-harbor-v01 // 多节点时在末端添加即可用空格隔开
```



## 安装Mon

```
ceph-deploy mon create g1-k8s-harbor-v01
ceph-deploy gatherkeys g1-k8s-harbor-v01
ceph-deploy admin g1-k8s-harbor-v01
ceph-deploy mgr create g1-k8s-harbor-v01
```



## 安装Osd

- 格式化磁盘（所有OSD节点）

  ```
  ceph-volume lvm zap /dev/sdx				// 根据盘符进行格式化
  ```

- 创建osd

  ```
  ceph-deploy osd create g1-k8s-harbor-v01 --data /dev/sdb --block-db /dev/sda2 --block-wal /dev/sda3			// db wal最好用ssd分区
  ```

  

## 创建Mds

```
// 创建mds
ceph-deploy mds create g1-k8s-harbor-v01

// 创建元数据和数据池
ceph osd pool create cephfs_data 64 64
ceph osd pool create cephfs_metadata 64 64

// 创建cephfs
ceph fs new cephfs cephfs_metadata cephfs_data // 需要注意元数据池在前 数据池在后
```



# 挂载

- 本地挂载

  ```
  // 查看key
  cat /etc/ceph/ceph.client.admin.keyring
  
  // 挂载
  mount -t ceph 10.16.200.119:/ /mnt -o name=admin,secret=key // key为上文内容
  ```

- k8s静态pv挂载

  ```
  # cat ceph-secret.yml
  apiVersion: v1
  kind: Secret
  metadata:
    name: ceph-secret
    namespace: autocnn
  data:
  // 需要注意的是此处的key为 key|base64值
    key: QVFBdlZnWmJoQ3NhRWhBQU95SlZGWWJadVJESnBRR3BKRERhc3c9PQo=
  
  # cat ceph-pv.yml
  ---
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: test-pv
  spec:
    capacity:
      storage: 512Mi
    accessModes:
      - ReadWriteMany
    cephfs:
      monitors:
        - 10.16.200.119:6789
      path: /
      user: admin
      secretRef:
        name: ceph-secret
  #    secretFile: "/etc/ceph/admin.secret"
      readOnly: false
    persistentVolumeReclaimPolicy: Recycle
  
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: test-claim
  spec:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 512Mi
  ```

- 动态PV
  [工作目录](./ceph_provisioner)