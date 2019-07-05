# Maven3.6.1

## 用途

编译java项目，用于jenkins-slave

## 镜像制作

工作目录处于[images/image_maven](images/image_maven)

```
#工作目录
.
├── deploy
│   └── pv.yaml
├── Dockerfile
└── settings.xml

#制作镜像
docker build -t Harbor_ip:Port/Group/maven:3.6.1 .
```

`Dockerfile` 制作镜像的基础文件

`settings.xml` maven配置文件，主要配置nexus库连接信息

`deploy` 目录下`pv.yaml`为创建动态pv,作用于存放maven编译时下载的包



## Dockerfile

```
# centos:7.4.1708 ---> 10.16.200.119/base/centos:7.4
FROM 10.16.200.119/base/centos:7.4

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

归纳为应用类镜像

10.16.200.119/xxxx/maven:3.6.1