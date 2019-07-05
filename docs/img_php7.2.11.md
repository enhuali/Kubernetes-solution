# Php

## 版本

V7.2.11

## 镜像制作

工作目录处于[images/image_php7.2.11](../images/image_php7.2.11)

```
#目录结构
.
├── config
├── Dockerfile
├── id_rsa
├── id_rsa.pub
├── php-fpm.conf
├── php.ini
└── www.conf

#制作镜像
docker build -t Harbor_ip:Port/Group/php7.2.11 .
```

`config` Ssh-key免认证文件

`Dockerfile` 制作镜像的基础文件

`id_rsa` 具有xxxx_dek扩展库pull权限的私钥

`id_rsa.pub` 具有xxxx_dek扩展库pull权限的公钥

`php-fpm.conf` Php-fpm全局配置文件

`php.ini` Php扩展配置文件

`www.conf` Php启动配置文件

## 扩展版本

```
[PHP Modules]
apcu												Version => 5.1.8
bcmath
calendar
Core												PHP Version => 7.2.0
ctype
curl												cURL Information => 7.29.0
date												timelib version => 2017.05beta9
dom													DOM/XML API Version => 20031129	libxml Version => 2.9.1
exif												EXIF Version => 7.2.0
filter
ftp
gd													GD Version => bundled (2.1.0 compatible)
gettext
grpc												grpc module version => 1.20.0
hash
iconv												iconv library version => 2.17
imagick											imagick module version => 3.4.4
intl												version => 1.1.0
json												json version => 1.6.0
ldap
libxml											libXML Compiled Version => 2.9.1
mbstring										libmbfl version => 1.3.2	oniguruma version => 6.3.0
memcache										Version => 4.0.3
memcached										Version => 3.1.3	libmemcached version => 1.0.18
mongodb											MongoDB extension version => 1.3.4
mysqli											Client API library version => 5.6.27
mysqlnd											Version => mysqlnd 5.0.12-dev
openssl											OpenSSL Library Version => OpenSSL 1.0.2k-fips
pcntl
pcre												PCRE Library Version => 8.41
PDO
pdo_mysql										Client API version => 5.6.27
pdo_sqlite									SQLite Library => 3.20.1
Phar												Phar EXT version => 2.0.2	Phar API version => 1.1.1
posix
rdkafka											version => 3.0.1
redis												Redis Version => 3.1.1
Reflection
session
shmop
SimpleXML
soap
sockets
SPL
sqlite3											SQLite3 module version => 7.2.0	SQLite Library => 3.20.1
standard
sysvsem											Version => 7.2.0
thrift_protocol							Version => 1.0
tideways_xhprof							Version => 4.1.6
tidy												libTidy Version => 5.4.0
tokenizer
wddx
xhprof											xhprof => 0.9.5
xml													libxml2 Version => 2.9.1
xmlreader
xmlrpc
xmlwriter
Zend OPcache
zip													Zip version => 1.15.1	Libzip version => 1.1.2
zlib												Compiled Version => 1.2.7	Linked Version => 1.2.7

[Zend Modules]
Zend OPcache
```



## Dockerfile

```
FROM centos:7.4.1708

# Install base soft
RUN yum install -y epel-release
RUN set -x \
    && yum install -y wget git gcc make cmake gcc-c++ bison-devel ncurses-devel zlib-devel gd-devel curl-devel libxml2-devel libicu-devel libcurl-devel libicu-devel libevent-devel bzip2-devel libtidy-devel libjpeg-turbo-devel libpng-devel openssl-devel openldap-devel freetype-devel libmcrypt-devel tzdata ImageMagick-devel m4 autoconf librdkafka-devel \
    && yum clean all \
    && rm -rf /var/cache/yum/

# Work directory
WORKDIR /data/service

# Install Mysql module version=5.6.27
RUN set -x \
    && wget -c http://mirror.neu.edu.cn/mysql/Downloads/MySQL-5.6/mysql-5.6.27.tar.gz \
    && tar zxvf mysql-5.6.27.tar.gz \
    && cd mysql-5.6.27 \
    && cmake -DCMAKE_INSTALL_PREFIX=/data/service/mysql \
    && make -j 4 \
    && make install \
    && rm -rf ../mysql-*

# Global enviroment
ENV LD_LIBRARY_PATH=/data/service/mysql/lib:/lib/:/usr/lib/:/usr/local/lib
ENV PATH=/data/service/php7.2.11/sbin:/data/service/php7.2.11/bin:$PATH
ENV TZ=Asia/Shanghai

# Install Php version=7.2.11
RUN set -x \
    && wget -c http://ftp.ntu.edu.tw/php/distributions/php-7.2.11.tar.gz \
    && tar zxvf php-7.2.11.tar.gz \
    && cd php-7.2.11 \
    && ./configure --prefix=/data/service/php7.2.11 --with-config-file-path=/data/service/php7.2.11/etc --enable-fpm --with-fpm-user=www --with-fpm-group=www --enable-mysqlnd --enable-mysqlnd-compression-support --enable-calendar --enable-exif --with-iconv-dir --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir --enable-xml --disable-rpath --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --with-curl --enable-mbregex --enable-mbstring --enable-intl  --with-libmbfl --enable-ftp --with-gd --enable-gd-jis-conv --with-openssl --with-mhash --enable-pcntl --enable-sockets --enable-wddx --with-tidy --with-xmlrpc --with-kerberos --with-libdir=lib64 --enable-zip --enable-soap --with-gettext --with-mcrypt --enable-opcache --with-pear --enable-maintainer-zts --with-ldap=shared --without-gdbm --with-pcre-dir --with-pdo-mysql=/data/service/mysql --with-mysqli=/data/service/mysql/bin/mysql_config \
   && make -j 4 \
   && make install \
   && rm -rf ../php-*

# Install libmemcached version=1.0.18
RUN set -x \
    && wget -c https://launchpad.net/libmemcached/1.0/1.0.18/+download/libmemcached-1.0.18.tar.gz \
    && tar zxvf libmemcached-1.0.18.tar.gz \
    && cd libmemcached-1.0.18 \
    && ./configure --prefix=/data/service/libmemcached \
    && make -j 4 \
    && make install \
    && rm -rf ../libmemcached-*

# Install memcached moudule version=3.1.3
RUN set -x \
    && curl -O https://pecl.php.net/get/memcached-3.1.3.tgz \
    && tar zxvf memcached-3.1.3.tgz \
    && cd memcached-3.1.3 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config  -with-libmemcached-dir=/data/service/libmemcached  --disable-memcached-sasl \
    && make -j 4 \
    && make install \
    && rm -rf ../memcached-*

# Install memcache module version=4.0.3
RUN set -x \
    && git clone https://github.com/websupport-sk/pecl-memcache \
    && cd pecl-memcache \
    && git checkout 4.0.3 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../pecl-memcache

# Install Redis module version=3.1.1
RUN set -x \
    && wget -c https://pecl.php.net/get/redis-3.1.1RC2.tgz \
    && tar zxvf redis-3.1.1RC2.tgz \
    && cd redis-3.1.1RC2 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../redis-*

# Install apcu module version=5.1.8
RUN set -x \
    && wget -c https://pecl.php.net/get/apcu-5.1.8.tgz \
    && tar zxvf apcu-5.1.8.tgz \
    && cd apcu-5.1.8 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../apcu-*

#Install imagick module version=3.4.4
RUN set -x \
    && wget -c https://pecl.php.net/get/imagick-3.4.4.tgz \
    && tar zxvf imagick-3.4.4.tgz \
    && cd imagick-3.4.4 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../imagick-*

# Install mongodb module version=1.3.4
RUN set -x \
    && wget -c https://pecl.php.net/get/mongodb-1.3.4.tgz \
    && tar zxvf mongodb-1.3.4.tgz \
    && cd mongodb-1.3.4 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../mongodb-*

# Install rdkafka module version=3.0.1
RUN set -x \
    && wget -c https://pecl.php.net/get/rdkafka-3.0.1.tgz \
    && tar zxvf rdkafka-3.0.1.tgz \
    && cd rdkafka-3.0.1 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../rdkafka-*

# Install grpc module version=1.20.0
RUN set -x \
    && wget -c https://pecl.php.net/get/grpc-1.16.0.tgz \
    && tar zxvf grpc-1.16.0.tgz \
    && cd grpc-1.16.0 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../grpc-*

# Install phpng-xhprof module version=0.9.5
RUN set -x \
    && git clone http://www.github.com/yaoguais/phpng-xhprof \
    && cd phpng-xhprof \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../phpng-xhprof

# Install thrift_protocol module version=1.0
RUN set -x \
    && wget -c https://github.com/apache/thrift/archive/0.10.0.tar.gz \
    && tar zxvf 0.10.0.tar.gz \
    && cd thrift-0.10.0/lib/php/src/ext/thrift_protocol \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf /data/service/{0.10.0.tar.gz,thrift-0.10.0}

# Install tideways module version=4.1.6
RUN set -x \
    && git clone https://github.com/tideways/php-xhprof-extension.git \
    && cd php-xhprof-extension \
    && git checkout v4.1.6 \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf /data/service/php-xhprof-extension

# Install  doumo_dek
# set ssh-key
COPY config id_rsa id_rsa.pub /root/.ssh/
RUN chmod 0600 /root/.ssh/id_rsa*

RUN set -x \
    && git clone git@git.corp.xxxx.com:core/dek_xxxx.git \
    && cd dek_xxxx \
    && git checkout php_7.0.14_ext_dek \
    && cd dek \
    && phpize \
    && ./configure --with-php-config=/data/service/php7.2.11/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf /data/service/dek_xxxx

# Add user uid=gid=500
RUN set x \
    && groupadd -g 500 www \
    && adduser -u 500 -g www www

# Create base directory
RUN mkdir -p /data/service_logs/php_log

# Copy config file
COPY php.ini php-fpm.conf /data/service/php7.2.11/etc/
COPY www.conf /data/service/php7.2.11/etc/php-fpm.d/

# Run service
CMD ["php-fpm","-F"]
```

## 命名规则

归纳为应用类镜像

10.16.200.119/xxxx/php7.2.11

## 注意事项

- mongo扩展需要用大于等于1.3.0版本，否则编译不过去