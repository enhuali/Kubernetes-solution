# Jdk-1.8.0_144

## 用途

提供java基础环境，可用于代码运行及maven环境运行

## 镜像制作

工作目录处于[images/image_jdk1.8.0_144](../images/image_jdk)

```
#工作目录
.
├── Dockerfile
└── jdk1.8.0_144

#制作镜像
docker build -t Harbor_ip:Port/Group/jdk1.8.0:144 .
```

`Dockerfile` 制作镜像的基础文件

`jdk1.8.0_144` jdk程序包由于没有找到可用的url直接下载，所以需手动将此项目[下载](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-javase8-2177648.html)至当前目录 



## Dockerfile

```
# centos:7.4.1708 ---> 10.16.200.119/base/centos:7.4
FROM centos:7.4.1708

# workspace
WORKDIR /app

# Install base soft
RUN yum install -y jq zip python git wget unzip

# Import jdk package
COPY jdk1.8.0_144 jdk1.8.0_144

# Env
ENV JAVA_HOME=/app/jdk1.8.0_144
ENV JRE_HOME=$JAVA_HOME/jre
ENV PATH=/app/jdk1.8.0_144/bin:$PATH
```



## 命名规则

归纳为基础类镜像

10.16.200.119/base/jdk1.8.0:144