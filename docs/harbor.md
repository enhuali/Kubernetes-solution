# Harbor

## 安装

1. 下载docker-compose二进制文件用于启动编排好的harbor

   ```
   wget https://github.com/docker/compose/releases/download/1.18.0/docker-compose-Linux-x86_64
   
   mv docker-compose-Linux-x86_64 /etc/ansible/bin/docker-compose
   ```

   

2. 下载Harbor离线包

   ```
   # 下载1.8.0版本
   wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz
   
   # 解压
   tar zxvf harbor-offline-installer-v1.8.0.tgz
   ```

   

3. 修改配置文件harbor.yml
   `hostname:` 改为部署机器的ip或对应解析的域名
   `port:` 端口
   `data_volume:` 数据存储目录 

   

4. 安装

   ```
   ./install.sh
   ```



## 使用

1. 节点首次使用harbor需进行身份认证

   ```
   docker login Harbor_ip:Port
   #会生成登陆Harbor登陆密钥在~/.kube/config文件
   
   #将~/.kube/config内容替换至dockerhub.config下
     
   # 将认证文件批量下发
   ansible all -m copy -a 'src=~/.docker/config.json dest=~/.docker/config.json'
   ```

   

2. 创建对应的组
   初始化的Harbor会存在一个library的公开组，我们需要根据需求创建对应的组

   - 公共镜像组：用于存放制作镜像时所依赖的镜像，或微调过的镜像（比如时间，所需工具）比如alpine/centos:7.4.1708
   - 业务基础镜像：在公共基础镜像之上，加入Nginx\php所需环境模块扩展
   - 代码镜像，包含各版本代码的镜像，用不同环境作为分组进行区分
     - test 存放测试环境代码
     - sim 存放预上线环境代码
     - online 存放上线环境代码
       

3. 上传镜像

   ```
   docker push Harbor:port/group/image_name:tag
   ```

   

4. 下载镜像

   ```
   docker pull Harbor:port/group/image_name:tag
   ```

   