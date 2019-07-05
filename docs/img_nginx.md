# Nginx

## 版本

v1.16.0

## 镜像制作

工作目录处于[images/image_nginx1.16.0](../images/image_nginx1.16.0)

```
#目录结构
.
├── conf.d
│   ├── denyip.conf
│   ├── xxxx_log.conf
├── Dockerfile
├── nginx.conf
└── op_lan.conf

#制作镜像
docker build -t Harbor_ip:Port/Group/nginx1.16.0 .
```

`conf.d` Nginx公共配置目录，此处的denyip.conf、xxxx_log.conf仅作为模版提供，此目录所有文件通过Configmap进行内部挂载

`Dockerfile` 制作镜像的基础文件

`nginx.conf` Nginx主配置文件

`op_lan.conf` 项目配置文件，此处的op_lan.conf仅作为模版提供，Container内使用Configmap进行挂载

`Harbor_ip:` Harbor私有仓库的地址

`Port:` Harbor的端口，通常Harbor为采用https时需要声明，默认为5000 `Group:` Harbor项目组，通常以环境或Namespace命名，Ex：(dev、test、gray、online)，需要预先在Harbor界面下创建对应的组



## Dockerfile

```
FROM centos:7.4.1708

# Install base soft
RUN yum install -y epel-release
RUN set -x \
    && yum install -y wget gcc make gcc-c++ tzdata \
    && yum clean all \
    && rm -rf /var/cache/yum/

WORKDIR /data/service

# Install pcre version=8.38
RUN set -x \
    && wget -c https://ftp.pcre.org/pub/pcre/pcre-8.38.tar.gz \
    && tar zxvf pcre-8.38.tar.gz \
    && cd pcre-8.38 \
    && ./configure \
    && make install \
    && rm -rf ../pcre-8.38.tar.gz

# Install perl version=5.28.0
RUN set -x \
    && wget -c https://www.cpan.org/src/5.0/perl-5.28.0.tar.gz \
    && tar -xzf perl-5.28.0.tar.gz \
    && cd perl-5.28.0 \
    && ./Configure -des \
    && make -j 4 \
    && make install \
    && rm -rf ../perl-5.28.0.tar.gz

# Install zlib version=1.2.11
RUN set -x \
    && wget http://www.zlib.net/zlib-1.2.11.tar.gz \
    && tar zxvf zlib-1.2.11.tar.gz \
    && cd zlib-1.2.11 \
    && ./configure \
    && make install \
    && rm -rf ../zlib-1.2.11.tar.gz

# Install openssl version=1.0.2
RUN set -x \
    && wget -c https://www.openssl.org/source/openssl-1.1.1b.tar.gz \
    && tar zxvf openssl-1.1.1b.tar.gz \
    && cd openssl-1.1.1b \
    && ./config \
    && make install \
    && rm -rf ../openssl-1.1.1b.tar.gz

# Install nginx version=1.15.9
RUN set -x \
    && wget -c http://nginx.org/download/nginx-1.16.0.tar.gz \
    && tar zxvf nginx-1.16.0.tar.gz \
    && cd nginx-1.16.0 \
    && ./configure --prefix=/data/service/nginx --with-zlib=/data/service/zlib-1.2.11 --with-openssl=/data/service/openssl-1.1.1b --with-pcre=/data/service/pcre-8.38 --user=www --group=www --with-http_gzip_static_module --with-http_stub_status_module --with-http_ssl_module \
    && make install \
    && rm -rf ../nginx-*

# Add user
RUN set x \
    && groupadd -g 500 www \
    && adduser -u 500 -g www www

# Create base directory
RUN mkdir -p /data/service_logs/nginx_log /data/service/nginx/var/run

# Copy config file
ADD nginx.conf nginx/conf/nginx.conf
#ADD op_lan.conf nginx/xxxx/op_lan.conf
#ADD conf.d nginx/conf.d

# Global enviroments
ENV PATH=/data/service/nginx/sbin:$PATH
ENV TZ=Asia/Shanghai

# Start service
CMD ["nginx", "-g", "daemon off;"]
```



## 命名规则

归纳为应用类镜像

10.16.200.119/xxxx/nginx1.16.0