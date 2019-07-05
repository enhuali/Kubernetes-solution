### 日志落地和切割方案

#### 日志落地方案

1、日志映射到虚拟机可以使用hostpath模式的volume，然后将日志映射至宿主机上,代码1 

2、创建日志映射的时候注意需要将宿主机上需要映射的目录给予其他用户的读写权限，也就是需要执行chmod 757 /dir 

3、ingress-controller默认将日志输出和error日志重定向到容器进程的stdout和stderr上，这样只能看容器的日志（docker logs docker_id）才能看到，但是这个会混合docker其他相关的日志，因此需要需改这两个日志路径，通过configmap来修改-代码2 

4、后续可以将宿主上的目录按照现有的结构区分业务日志、负载均衡日志、买点日志目录来进行区分，每个业务只挂载自己的日志内容 

5、目前ingress-controller的acess日志里会有init的日志，属于stream日志，这个日志在ingress启动时日志的format定义的值不能为空，否则启动不了，这一块可以修改ingress-controller镜像里的/etc/nginx/template/nginx.tmpl模版文件，然后生成新的镜像来修改那些官方不支持的修改选项，可以算作是定制化 

6、在宿主机上nginx日志目录：/data/service_logs/nginx_log/里加上业务方目录比如：saas、loadbbanece、maidian、之类的，宿主机和容器提前将这些目录创建好，目录权限给757，php也是同理，在业务目录里面区分php7和5，每个业务创建deployment时指定不同的挂载路径即可 

代码1

```yaml
containers:
        - name: nginx-ingress-controller
          #image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          #使用以下镜像，方便国内下载加速
          image: jmgao1983/nginx-ingress-controller:0.21.0
          volumeMounts:
          - mountPath: /etc/localtime
            name: time
          - mountPath: /var/log/nginx
            name: service-logs
...
...
...
 volumes:
      - name: time
        hostPath:
          path: /etc/localtime
      - name: service-logs
        hostPath:
          path: /data/service_logs/nginx_log/loadbalance_log
          type: Directory


```

代码2

```yaml
kind: ConfigMap
data:
  access-log-path: "/var/log/nginx/loadbalance_access.log"
  log-format-escape-json: "true"
  error-log-path: "/var/log/nginx/loadbalance_error.log"

```



#### 日志切割方案

ingress-controller日志切割方案使用logrotate工具进行切割，让其切割日志之后在宿主机上执行相关容器的reopen命令 ，下面的为ingress-nginx(业务日志的nginx同理)的日志切割脚本方案，其他的如php之类的大同小异，可以参考nginx。

1、创建切割脚本文件/etc/logrotate.d/nginx&/etc/logrotate.d/php

/etc/logrotate.d/nginx

```
/data/service_logs/nginx_log/loadbalance_log/loadbalance_access.log
/data/service_logs/nginx_log/loadbalance_log/loadbalance_error.log
/data/service_logs/nginx_log/access.log
{
    rotate 7
    daily
    compress
    sharedscripts
    postrotate
       docker ps |grep -v pause|grep nginx-ingress-controller|awk '{print $1}'|xargs -I {} docker exec {}  /usr/sbin/nginx -s reopen
       docker ps|grep nginx[0-9]\.[0-9]|awk '{print $1}'|xargs -I {} docker exec {}  /data/service/nginx/sbin/nginx -s reopen
    endscript
    missingok
    notifempty
    su root www
}
#前几行为需要切割的日志文件，可以用通配符来匹配，中间执行的脚本根据容器的不同而不同

```

/etc/logrotate.d/php

```
/data/service_logs/php_log/*.log
{
    rotate 7
    daily
    compress
    sharedscripts
    postrotate
     docker ps |grep php[0-9]\.[0-9] | awk '{print $1}' | xargs -I {} docker exec {} kill -USR2 $(cat /data/service/php7.0.16/var/run/php-fpm.pid)
    endscript
    missingok
    notifempty
    su root www
}
#根据相关的日志路径来修改响应的日志文件

```



2、修改/etc/cron.daily/logrotate 脚本文件

```shell
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status -f  /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
#这个脚本加了一个-f参数，用来强制执行，避免日志文件过小而忽略切割

```



3、可以直接执行 bash /etc/cron.daily/logrotate来看看切割的情况



### ingress-controller定制化

   官方默认通过configmap和annotations去使用nginx的高级功能和定制nginx配置，configmap是指的全局的配置，比如日志格式，性能调优等，annotations主要是每个ingress来指定相关配置。

ingress-inginx里的主配置文件是通过go的template来渲染的，如果官方支持的功能或配置不能满足需求，可以修改这个模版，通过重做ingress-controller镜像的方式将其放在容器中替换原来模版，然后使用新的镜像重新启动ingress-controller

下面为使用configmap来修改ingress-controller的日志格式和相关性能调优

1、编写configmap.yaml文件

2、创建configmap

```shell
kubectl apply -f configmap.yaml

```

3、此时应该已经生效，查看ingress-nginx 命名空间里的configmap配置

```shell
kubectl describe  configmap nginx-configuration  -n ingress-nginx

```

4、验证ingress-controller的日志文件或配置

注意：必须将data的内容用引号引起来,数字类型也一样

configmap.yaml

```yaml
apiVersion: v1
data:
  log-format-upstream: '"timestamp": "$time_iso8601","http_x_forwarded_for": "$http_x_forwarded_for","remote_addr": "$remote_addr","remote_user": "$remote_user","domain": "$host","server_addr": "$server_addr","http_referer": "$http_referer","request_method": "$request_method","request_uri": "$request_uri","http_version": "$server_protocol","request_time": "$request_time","upstream_response_time": "$upstream_response_time","status": "$status","body_bytes_sent": "$body_bytes_sent","http_user_agent": "$http_user_agent"'
  ssl-ciphers: "ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS"
 # access-log-path: '/var/log/nginx/access_1.log'
  ssl-protocols: "TLSv1 TLSv1.1 TLSv1.2 TLSv1.3"
  enable-vts-status: "true"
  keep-alive: "60"
  client-header-buffer-size: "32k"
  client-body-buffer-size: "8m"
  large-client-header-buffers: "4 32k"
  max-worker-connections: "51200"
  max-worker-open-files: "51200"
  server-name-hash-bucket-size: "128"
  ssl-session-cache: "true"
  ssl-session-cache-size: "100m"
  ssl-session-timeout: "10m"
  use-gzip: "true"
  use-gzip: "true"
  gzip-level: "2"
  gzip-types: "text/plain application/x-javascript text/css application/xml"
  worker-processes: "16"
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx


```

