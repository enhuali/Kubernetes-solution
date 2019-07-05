# Jenkins-slave

## 用途

包含项目触发时所需运行Jenkinsfile逻辑的环境

## jnlp镜像制作

工作目录处于[jenkins-slave](images/image_jenkins-slave)

- 需拷贝k8s集群环境下的kubectl、docker对应版本的二进制文件至当前目录
- 需将各个集群的kubeconfig合并至config文件并保存至当前目录

```
  kubectl config set-cluster $Cluster-name --certificate-authority=$Ssl_dir/ca.pem   --embed-certs=true --server=https://$Master_ip:Port

  kubectl config set-credentials $User-name --client-certificate=$Ssl_dir/admin.pem   --embed-certs=true   --client-key=$Ssl_dir/admin-key.pem

  kubectl config set-context $Context-name  --cluster=$Cluster-name   --user=$User-name
```

`Cluster-name:` 集群名称，通常以环境+Kubernetes命名，集群内唯一

`Ssl_dir:` 存放Kubernets证书路径，默认为/etc/kubernetes/ssl `Master_ip:` Kubernetes Master Api 对外提供服务的ip，如VIP

`Port:` Kubernetes Master Api 端口，默认6443

`Context-name:` Kubernetes集群识别名称，通常以环境+用户命名，集群内唯一

`User-name:` Kubernetes集群用户，通常以用户名+环境命名，集群内唯一

- 镜像制作

```
  docker build -t Harbor_ip:Port/Group/jnlp .
  docker push Harbor_ip:Port/Group/jnlp
```

`Harbor_ip:` Harbor私有仓库的地址

`Port:` Harbor的端口，通常Harbor为采用https时需要声明，默认为5000 `Group:` Harbor项目组，通常以环境或Namespace命名，Ex：(dev、test、gray、online)，需要预先在Harbor界面下创建对应的组



## Dockerfile

```
FROM debian:stretch-slim

# A few reasons for installing distribution-provided OpenJDK:
#
#  1. Oracle.  Licensing prevents us from redistributing the official JDK.
#
#  2. Compiling OpenJDK also requires the JDK to be installed, and it gets
#     really hairy.
#
#     For some sample build times, see Debian's buildd logs:
#       https://buildd.debian.org/status/logs.php?pkg=openjdk-11

RUN apt-get update && apt-get install -y --no-install-recommends \
		bzip2 \
		unzip \
		xz-utils \
                curl \
		git \
		openssh-client \
	&& rm -rf /var/lib/apt/lists/*

RUN echo 'deb http://deb.debian.org/debian stretch-backports main' > /etc/apt/sources.list.d/stretch-backports.list

# Default to UTF-8 file.encoding
ENV LANG C.UTF-8

# add a simple script that can auto-detect the appropriate JAVA_HOME value
# based on whether the JDK or only the JRE is installed
RUN { \
		echo '#!/bin/sh'; \
		echo 'set -e'; \
		echo; \
		echo 'dirname "$(dirname "$(readlink -f "$(which javac || which java)")")"'; \
	} > /usr/local/bin/docker-java-home \
	&& chmod +x /usr/local/bin/docker-java-home

# do some fancy footwork to create a JAVA_HOME that's cross-architecture-safe
RUN ln -svT "/usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)" /docker-java-home
ENV JAVA_HOME /docker-java-home

ENV JAVA_VERSION 11.0.3
ENV JAVA_DEBIAN_VERSION 11.0.3+1-1~bpo9+1

RUN set -ex; \
	\
# deal with slim variants not having man page directories (which causes "update-alternatives" to fail)
	if [ ! -d /usr/share/man/man1 ]; then \
		mkdir -p /usr/share/man/man1; \
	fi; \
	\
# ca-certificates-java does not work on src:openjdk-11 with no-install-recommends: (https://bugs.debian.org/914860, https://bugs.debian.org/775775)
# /var/lib/dpkg/info/ca-certificates-java.postinst: line 56: java: command not found
	ln -svT /docker-java-home/bin/java /usr/local/bin/java; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
		openjdk-11-jdk-headless="$JAVA_DEBIAN_VERSION" \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	rm -v /usr/local/bin/java; \
	\
# ca-certificates-java does not work on src:openjdk-11: (https://bugs.debian.org/914424, https://bugs.debian.org/894979, https://salsa.debian.org/java-team/ca-certificates-java/commit/813b8c4973e6c4bb273d5d02f8d4e0aa0b226c50#d4b95d176f05e34cd0b718357c532dc5a6d66cd7_54_56)
	keytool -importkeystore -srckeystore /etc/ssl/certs/java/cacerts -destkeystore /etc/ssl/certs/java/cacerts.jks -deststoretype JKS -srcstorepass changeit -deststorepass changeit -noprompt; \
	mv /etc/ssl/certs/java/cacerts.jks /etc/ssl/certs/java/cacerts; \
	/var/lib/dpkg/info/ca-certificates-java.postinst configure; \
	\
# verify that "docker-java-home" returns what we expect
	[ "$(readlink -f "$JAVA_HOME")" = "$(docker-java-home)" ]; \
	\
# update-alternatives so that future installs of other OpenJDK versions don't change /usr/bin/java
	update-alternatives --get-selections | awk -v home="$(readlink -f "$JAVA_HOME")" 'index($3, home) == 1 { $2 = "manual"; print | "update-alternatives --set-selections" }'; \
# ... and verify that it actually worked for one of the alternatives we care about
	update-alternatives --query java | grep -q 'Status: manual'

#RUN set -x \
#    && apt-get update \
#    && apt-get install -y docker libltdl-dev librados-dev gcc wget \
#    && wget https://dl.google.com/go/go1.10.4.linux-amd64.tar.gz -P /root \
#    && tar zxvf /root/go1.10.4.linux-amd64.tar.gz -C /usr/local \
#    && rm -rf /root/go1.10.4.linux-amd64.tar.gz /var/lib/apt/lists/*
#
#RUN set -x \
#    && mkdir /opt/workspace \
#    && chown -R 999:999 /opt/workspace
#
#ENV PATH=$PATH:$GOPATH:/usr/local/go/bin:/autocnn/bin \
#    GOROOT=/usr/local/go \
#    GOPATH=/opt/workspace

RUN set -x \
    && rm -rf /bin/sh \
    && cp /bin/bash /bin/sh

# see CA_CERTIFICATES_JAVA_VERSION notes above
RUN /var/lib/dpkg/info/ca-certificates-java.postinst configure

# If you're reading this and have any feedback on how this image could be
# improved, please open an issue or a pull request so we can discuss it!
#
#   https://github.com/docker-library/openjdk/issues

###
# build slave image
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000

ENV HOME /home/${user}
RUN groupadd -g ${gid} ${group}
RUN useradd -c "Jenkins user" -d $HOME -u ${uid} -g ${gid} -m ${user}
LABEL Description="This is a base image, which provides the Jenkins agent executable (slave.jar)" Vendor="Jenkins project" Version="3.27"

ARG VERSION=3.27
ARG AGENT_WORKDIR=/home/${user}/agent

RUN curl --create-dirs -sSLo /usr/share/jenkins/slave.jar https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/${VERSION}/remoting-${VERSION}.jar \
  && chmod 755 /usr/share/jenkins \
  && chmod 644 /usr/share/jenkins/slave.jar

USER ${user}
ENV AGENT_WORKDIR=${AGENT_WORKDIR}
RUN mkdir /home/${user}/.jenkins && mkdir -p ${AGENT_WORKDIR}

VOLUME /home/${user}/.jenkins
VOLUME ${AGENT_WORKDIR}
WORKDIR /home/${user}


###
# build jnlp-slave image
COPY --chown=jenkins:jenkins config /tmp/config
COPY --chown=jenkins:jenkins kubectl /bin/kubectl
COPY --chown=jenkins:jenkins docker /bin/docker
COPY --chown=jenkins:jenkins jenkins-slave /bin/jenkins-slave
RUN chmod +x /bin/jenkins-slave

ENV KUBECONFIG=/tmp/config
#ENV DOCKER_HOST="tcp://192.168.1.154:2376/"

ENTRYPOINT ["/bin/jenkins-slave"]
```



## 命名规则

归纳为应用类镜像

10.16.200.119/xxxx/jnlp