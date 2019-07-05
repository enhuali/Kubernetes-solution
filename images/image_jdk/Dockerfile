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
