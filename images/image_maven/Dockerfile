FROM 10.16.200.119/base/jdk1.8.0:144

WORKDIR /app

RUN set -x \
    && wget -c https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.1/binaries/apache-maven-3.6.1-bin.zip \
    && unzip apache-maven-3.6.1-bin.zip


#ARG user=jenkins
#ARG group=jenkins
#ARG uid=1000
#ARG gid=1000
#
#RUN groupadd -g ${gid} ${group}
#RUN useradd -c "Jenkins user" -d $HOME -u ${uid} -g ${gid} -m ${user}

#RUN set -x \
#    && mkdir -p /data/maven-nexus/data \
#    && chown -R jenkins:jenkins /data/maven-nexus/data

#USER ${user}



COPY settings.xml /app/apache-maven-3.6.1/conf/settings.xml
ENV MAVEN_HOME=/app/apache-maven-3.6.1
ENV PATH=/app/apache-maven-3.6.1/bin:$PATH

