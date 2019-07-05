# Code

## 用途

代码的镜像版本控制、更新、回滚所用

## 镜像制作

工作目录处于[images/image_codes](../images/image_codes)

```
#工作目录
.
├── config
├── Dockerfile
├── id_rsa
└── id_rsa.pub

#制作镜像
docker build -t Harbor_ip:Port/Group/code .
```

`config` Ssh-key免认证文件

`Dockerfile` 制作镜像的基础文件

`id_rsa` 具有xxxx_dek扩展库pull权限的私钥

`id_rsa.pub` 具有xxxx_dek扩展库pull权限的公钥



## Dockerfile

- 安装git\ssh-client服务用于镜像中clone项目
- 创建宿主机上对应web项目用户及uid.gid

```
# alpine ---> 10.16.200.119/base/alpine
FROM alpine

# Update repo
RUN echo "http://mirrors.aliyun.com/alpine/latest-stable/main/" > /etc/apk/repositories
RUN echo "http://mirrors.aliyun.com/alpine/latest-stable/community/" >> /etc/apk/repositories

# Install base soft and add user
RUN set -x \
    && apk update \
    && apk add --no-cache openssh-client git \
    && adduser -D -u 500 www \
    && mkdir /git \
    && chown -R 500:500 /git

# set ssh-key
COPY config id_rsa id_rsa.pub /home/www/.ssh/
RUN set -x \
    && chmod 0600 /home/www/.ssh/id_rsa* \
    && chown -R 500:500 /home/www/.ssh/

# Global
USER 500
WORKDIR /git
```

## 

## 命名规则

归纳为应用类镜像

10.16.200.119/xxxx/code