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

# Global enviroments
ENV PATH=/data/service/nginx/sbin:$PATH
ENV TZ=Asia/Shanghai

# Start service
CMD ["nginx", "-g", "daemon off;"]
