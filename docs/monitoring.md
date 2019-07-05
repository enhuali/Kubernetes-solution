

# Prometheus-operator

工作文件处于 [monitoring](./monitoring)

# 安装

通过helm安装

## 下载项目

```
helm fetch stable/prometheus-operator --version 5.10.2 --untar
```



## 启动服务

```
# 常规启动（daemon）
helm install --name promethues-operator --namespace monitoring prometheus-operator

# 高级启动，我们环境真正用到的启动方式，但需先进行如下配置文件的创建
helm install --name promethues-operator --namespace monitoring -f prome-settings.yaml prometheus-operator
```

没错以上就安装完了，可以进行单纯的访问使用啦。但由于此项目更贴近于rancher部署的k8s集群，所以初始化中对etcd/kube-schuduler/kube-controller-manager监控不了，除此之外还有一些基本参数需根据需求调整。



# 参数配置

## 定义配置文件

这个文件（prom-seting.yaml）的作用是根据charts预留的变量进行赋值，没在此文件中配置的均使用[官方默认参数](https://hub.helm.sh/charts/stable/prometheus-operator) 对应的参数格式为

```
[] 								// 多个时换行并用-开头
endpoints: []

endpoints:
- xxx
- xxx


""								// 多个时用逗号隔开
caFile: ""

caFile: xxx,xxx


{}								// 代表是层级关系，需要换行
storageSpec: {}

storageSpec:
  volumeClaimTemplate:
    spec:
      storageClassName: managed-nfs-storage
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```



```
#下载prometheus-operator的helm包
#helm fetch stable/prometheus-operator --version 5.10.2 --untar
#
# helm安装监控
# helm install --name promethues-operator --namespace monitoring -f prome-settings.yaml prometheus-operator
#
# 修改配置后更新
# helm upgrade  promethues-operator --namespace monitoring -f prome-settings.yaml prometheus-operator
#
#
#
# 非容器部署时需要声明其ip，并确保端口10252监听在0.0.0.0上，如若不是修改Unit文件中--dadress
kubeControllerManager:
  endpoints:
  - 10.16.200.101
  - 10.16.200.102
  - 10.16.200.103

# 非容器部署时需要声明其ip，并确保端口10251监听在0.0.0.0上，如若不是修改Unit文件中--dadress
kubeScheduler:
  endpoints:
  - 10.16.200.101
  - 10.16.200.102
  - 10.16.200.103

# 非容器部署时需要声明其ip，并创建能虐狗提供给prometheus的secret
# kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/kubernetes/ssl/ca.pem  --from-file=/etc/etcd/ssl/etcd.pem --from-file=/etc/etcd/ssl/etcd-key.pem
kubeEtcd:
  endpoints:
  - 10.16.200.101
  - 10.16.200.102
  - 10.16.200.103

# 开启证书访问,其中etcd-certs目录为上文中定义的secret名称
  serviceMonitor:
    scheme: https
    insecureSkipVerify: true
    serverName: etcd
    caFile: /etc/prometheus/secrets/etcd-certs/ca.pem
    certFile: /etc/prometheus/secrets/etcd-certs/etcd.pem
    keyFile: /etc/prometheus/secrets/etcd-certs/etcd-key.pem

# prometheus中添加etcd-certs,名称为上文中创建secret的名称,并开启nodeport方式端口为39000
prometheus:
  ingress:
    enabled: true
    hosts:
    -  prometheus.ingress.com
  service:
    type: NodePort
    nodePort: 39000
  prometheusSpec:
    secrets:
    - etcd-certs 
    #nodeSelector:
    #  kubernetes.io/hostname: 10.16.200.108
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-nfs-storage
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    replicas: 1
    

# Alertmanager配置
alertmanager:
  ingress:
    enabled: true
    hosts:
    -  alertmanager.ingress.com
  service:
    type: NodePort
    nodePort: 39001
  alertmanagerSpec:
    #nodeSelector:
    #  kubernetes.io/hostname: 10.16.200.108
    storage:
     volumeClaimTemplate:
       spec:
         storageClassName: managed-nfs-storage
         accessModes: ["ReadWriteOnce"]
         resources:
           requests:
             storage: 50Gi

# 开启grafananodeport方式端口为39002
grafana:
  ingress:
    enabled: true
    hosts:
    -  grafana.ingress.com
  service:
    type: NodePort
    nodePort: 39002

#prometheusOperator:
#  nodeSelector:
#    kubernetes.io/hostname: 10.16.200.108
#此chart没引出对应的变量，此处定义几个能够下载的镜像地址，修改prometheus-operator/charts/kube-state-metrics/values.yaml中image字段
#kubeStateMetrics:
#  image:
#      repository: mirrorgooglecontainers/kube-state-metrics
#      repository: quay.io/coreos/kube-state-metrics
```

`kubeControllerManager` `kubeScheduler` `kubeEtcd` 前面有提到当这三个组件不是通过Container部署时，需要申明其部署的主机ip

- etcd使用安全（既证书）部署时，需创建secret资源并提供给prometheus内部识别

```
# 创建命名为etcd-certs的secret资源
kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/kubernetes/ssl/ca.pem  --from-file=/etc/etcd/ssl/etcd.pem --from-file=/etc/etcd/ssl/etcd-key.pem
```

需要注意的是namespace需要和部署prometheus的相同，对应密钥位置和最重要的*命名*（它会在prometheus中的/etc/prometheus/secrets下创建"命名"目录）用于上文中填写证书存放识别



- kubeControllerManager  kubeScheduler 这两个服务端口10252/10251一定是监听的0.0.0.0，否则prometheus在获取其metric时会抛出connection x.x.x.x 10251 timeout；需要调整Unit文件中--address配置

  

`prometheus` `alertmanager` `grafana` 下的配置采用配置Ingress地址/NodePort方式/绑定到固定的节点上（当使用hostPath作为持久化方案时需要配置）/使用动态pv/控制实例数量

- 其中需要注意的是在prometheus中申明使用刚才创建的secret资源，名字一定是刚才的命名"etcd-certs"



`kubeStateMetrics` 在此项目中未预留且镜像用的国外的不易下载，我们需要修改其中的镜像名

```
vim prometheus-operator/charts/kube-state-metrics/values.yaml
# 将k8s.gcr.io/kube-state-metrics 改为quay.io/coreos/kube-state-metrics
```



# 持久化

此项目默认是没有将数据进行持久化，Volume方式均采用的EmptyDir,意味着期间运行的数据随着Pod的周期而丢失，真正生产环境我们对数据的持久化是必不可少的；持久化的方式有两种，一种是采用hostPath本地落盘，另一种就是采用Persistent Volume即共享存储，官方推荐是采用hostPath方式，因为毕竟一个Prometheus-operator具备处理千万数据的处理能力，但如果涉及其高可用那就只能采用共享存储来做，接下来介绍一下两者的配置方式及需要注意的点

### HostPath

1. 首先获取相关应用的yaml

   ```
   kubectl get pod xxxx -n monitoring -o yaml > /tmp/xxx.yaml
   ```

2. 查找其yaml中的volumeMounts、volumes字段对应挂载的位置信息，同样可以快速定位到empty的位置，主要获取有效挂载路径或名称

   ```
     - emptyDir: {}
       name: sc-dashboard-volume
     - configMap:
         defaultMode: 420
         name: promethues-operator-grafana-config-dashboards
       name: sc-dashboard-provider
     - emptyDir: {}
   ```

3. 用上一步骤获取到的有效挂载路径或名称在charts中进行搜索

   ```
   grep -r 'sc-dashboard-provider' prometheus-operator/*
   # 搜索到结果prometheus-operator/charts/grafana/templates/deployment.yaml
   ```

4. 将empty挂载方式改为hostPath

   ```
             persistentVolumeClaim:
               claimName: {{ .Values.persistence.existingClaim | default (include "grafana.fullname" .) }}
         {{- else }}
             #emptyDir: {}
             hostPath:
               path: /data/grafana
         {{- end -}}
         {{- if .Values.sidecar.dashboards.enabled }}
           - name: sc-dashboard-volume
             #emptyDir: {}
             hostPath:
               path: /data/dashboard-provider          
           - name: sc-dashboard-provider
             configMap:
               name: {{ template "grafana.fullname" . }}-config-dashboards
         {{- end }}
         {{- if .Values.sidecar.datasources.enabled }}
           - name: sc-datasources-volume
             #emptyDir: {}
             hostPath:
               path: /data/datasources
         {{- end -}}
   
   ```

5. 上文有提到，使用hostPath作为持久化存储时需要使用nodeSelector参数进行Pod分配绑定到固定节点

6. 在节点下创建步骤4中hostPath目录，并将权限给定为777或者进入容器内确定启动容器的用户uid:gid并赋予给这几个目录

### Persistent Volume Claim

1. 已存在nfs文件存储

2. 使用动态pv

   - `prometheus` `alertmanager` 配置在项目中预留出相关变量，我们将对应的配置定义在prom-seting.yaml

     ```
     prometheus:
         storageSpec:
           volumeClaimTemplate:
             spec:
               storageClassName: managed-nfs-storage
               accessModes: ["ReadWriteOnce"]
               resources:
                 requests:
                   storage: 10Gi
     
     
     alertmanager:
         storage:
          volumeClaimTemplate:
            spec:
              storageClassName: managed-nfs-storage
              accessModes: ["ReadWriteOnce"]
              resources:
                requests:
                  storage: 50Gi
     ```

     由于prometheus和alertmanager采用的是statusetful类型进行的部署，所以如果使用pvc进行持久化数据，一定要确保replicas的数量是大于等于之前的数值，否则会存在部署数据不可读，通常这两个值保持为1即可，毕竟它们具备处理千万级别数据，如果你的集群足够大或者为了提高其高可用性可以将其初始定义为2或者更多

     

   - `grafana` 需在其charts（prometheus-operator/charts/grafana）下定义，并且有两种方式可以定义，分别说明一下其配置及相关注意事项

     1. 定义templates/deployment.yaml

        ```
        # 预先创建pv(grafana.yaml)
        ---
        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          name: grafana-storage-claim
          namespace: monitoring
          annotations:
            #volume.beta.kubernetes.io/storage-class: "my-nfs-storage-class"
            volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
        spec:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
              
        # 创建pv
        kubectl create -f grafana.yaml
        
        # 定义persistentVolumeClaim
        persistentVolumeClaim:
          claimName: grafana-storage-claim
          
        # 启动grafana
        
        ```

        此种配置方法需在grafana创建之前预先做一些工作，好处是数据能够一直存在固定的pv上，下面介绍如何通过项目进行定义

        

     2. 定义values.yaml

        ```
        persistence:
          enabled: true
          storageClassName: managed-nfs-storage
          accessModes:
            - ReadWriteOnce
          size: 10Gi
        
        ```

     按此类方法配置后有一个问题，一定要确保动态pv创建时定义的垃圾回收为`Retain`,否则一但Pod对应的资源（Deployment）被删除后则无法再使用之前的PV，就算后台achived了这份数据也无法直接通过迁移到新的PV上而实现服务正常，那么是完全不能恢复嘛？那当然不会，那么也需要改为上面介绍的那种方式进行修复（其实应该还有挺多方法，就没有去验证）；不过即便采用了Retain方式后操作了Pod的删除同样需要额外的操作才能恢复

     ```
     # 查看pv，处于Bound
     kubectl get pv
     pvc-96e431ee-7ec9-11e9-8f83-00505693fec2   10Gi       RWO            Retain           Bound    monitoring/promethues-operator-grafana                                                                                      managed-nfs-storage            11m
     
     # 手动删除grafana的deployment
     kubectl delete deployment grafana -n monitoring
     
     # 查看pv，处于Released
     kubectl get pv
     pvc-96e431ee-7ec9-11e9-8f83-00505693fec2   10Gi       RWO            Retain           Released   monitoring/promethues-operator-grafana                                                                                      managed-nfs-storage            14m
     
     # 此时我们需要将其之前服务的Pod信息进行擦除
     kubectl edit pv pvc-96e431ee-7ec9-11e9-8f83-00505693fec2 #将spec.claimRef{}下的内容删除
     
     # 查看pv，处于可用
     pvc-96e431ee-7ec9-11e9-8f83-00505693fec2   10Gi       RWO            Retain           Avaliable   monitoring/promethues-operator-grafana                                                                                      managed-nfs-storage            14m
     
     # 启动grafana即可恢复数据使用
     
     ```

     不管何种方式其实都是存在一系列的操作成本，所以在日常维护中不到不得已的情况不要对Pod进行删除

     

# 更新配置

修改完配置后，需要重新进行加载

```
# 更新配置
helm upgrade  promethues-operator --namespace monitoring -f prome-settings.yaml prometheus-operator

```



最后来张效果图

![image-20190610115829460](pics/monitoring_01.png)