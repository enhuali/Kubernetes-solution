# Php

## 版本

V5.5.9

## 镜像制作

工作目录处于[images/image_php5.5.9](../images/image_php5.5.9)

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
docker build -t Harbor_ip:Port/Group/php5.5.9 .
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
apc
apcu											Version => 4.0.8
bcmath
calendar
Core											PHP Version => 5.5.9
ctype
curl
date
dom
xxxx_dek
ereg
exif											EXIF Version => 1.4 $Id$
fileinfo									version => 1.0.5-dev
filter
ftp
gd												GD Version => 2.1.0-alpha
grpc
hash
iconv											iconv library version => 1.14
imagick										imagick module version => 3.1.2
json											json version => 1.2.1
ldap
libxml										libXML Compiled Version => 2.7.6
mbstring									libmbfl version => 1.3.2
mcrypt										Version => 2.5.8
memcache									Version => 2.2.7
memcached									libmemcached version => 1.0.18 Version => 2.1.0
mhash
mmhash64
mongo											Version => 1.6.11
mysql											Client API version => 5.6.27
mysqli										Client API version => 5.6.27
mysqlnd										Version => mysqlnd 5.0.11-dev
openssl										OpenSSL Library Version => OpenSSL 1.0.1e-fips 11 Feb 2013
pcntl
pcre											PCRE Library Version => 8.32 2012-11-30
PDO
pdo_mysql									Client API version => 5.6.27
pdo_sqlite								SQLite Library => 3.7.7.1
Phar											Phar EXT version => 2.0.2	Phar API version => 1.1.1
posix
rdkafka										version => 0.9.1
redis											Redis Version => 2.2.7
Reflection
session
shmop
SimpleXML
soap
sockets
SPL
sqlite3										SQLite3 module version => 0.7-dev	SQLite Library => 3.7.7.1
standard
sysvsem
thrift_protocol						Version => 1.0
tidy											Extension Version => 2.0 ($Id$)
tokenizer
uprofiler									uprofiler => 0.11.0
wddx
xhprof										xhprof => 0.9.2
xml												libxml2 Version => 2.7.6
xmlreader
xmlrpc										php extension version => 0.51	
xmlwriter
Zend OPcache
zip												Zip version => 1.11.0	Libzip version => 0.10.1
zlib											Compiled Version => 1.2.8	Linked Version => 1.2.8

[Zend Modules]
Zend OPcache
```



## Dockerfile

```
FROM centos

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
ENV PATH=/data/service/php5.5.9/sbin:/data/service/php5.5.9/bin:$PATH
ENV TZ=Asia/Shanghai

# Install Php version=5.5.9
RUN set -x \
    && wget -c http://ftp.ntu.edu.tw/php/distributions/php-5.5.9.tar.gz \
    && tar zxvf php-5.5.9.tar.gz \
    && cd php-5.5.9 \
    && ./configure --prefix=/data/service/php5.5.9 --with-config-file-path=/data/service/php5.5.9/etc --enable-fpm --with-fpm-user=www --with-fpm-group=www --enable-mysqlnd --enable-mysqlnd-compression-support --enable-calendar --enable-exif --with-iconv-dir --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir --enable-xml --disable-rpath --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --with-curl --enable-mbregex --enable-mbstring --enable-intl  --with-libmbfl --enable-ftp --with-gd --enable-gd-jis-conv --with-openssl --with-mhash --enable-pcntl --enable-sockets --enable-wddx --with-tidy --with-xmlrpc --with-kerberos --with-libdir=lib64 --enable-zip --enable-soap --with-gettext --with-mcrypt --enable-opcache --with-pear --enable-maintainer-zts --with-ldap=shared --without-gdbm --with-pcre-dir --with-pdo-mysql=/data/service/mysql --with-mysqli=/data/service/mysql/bin/mysql_config \
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

# Install memcached moudule version=2.1.0
RUN set -x \
    && curl -O http://pecl.php.net/get/memcached-2.1.0.tgz \
    && tar zxvf memcached-2.1.0.tgz \
    && cd memcached-2.1.0 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config  -with-libmemcached-dir=/data/service/libmemcached  --disable-memcached-sasl \
    && make -j 4 \
    && make install \
    && rm -rf ../memcached-*

# Install memcache module version=2.2.7
RUN set -x \
    && wget -c https://pecl.php.net/get/memcache-2.2.7.tgz \
    && tar zxvf memcache-2.2.7.tgz \
    && cd memcache-2.2.7 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../memcache-2.2.7*

# Install Redis module version=2.2.7
RUN set -x \
    && wget -c https://pecl.php.net/get/redis-2.2.7.tgz \
    && tar zxvf redis-2.2.7.tgz \
    && cd redis-2.2.7 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../redis-*

# Install apcu module version=4.0.8
RUN set -x \
    && wget -c https://pecl.php.net/get/apcu-4.0.8.tgz \
    && tar zxvf apcu-4.0.8.tgz \
    && cd apcu-4.0.8 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../apcu-*

#Install imagick module version=3.1.2
RUN set -x \
    && wget -c https://pecl.php.net/get/imagick-3.1.2.tgz \
    && tar zxvf imagick-3.1.2.tgz \
    && cd imagick-3.1.2 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../imagick-*

# Install mongodb module version=1.2.5
RUN set -x \
    && wget -c https://pecl.php.net/get/mongodb-1.2.5.tgz \
    && tar zxvf mongodb-1.2.5.tgz \
    && cd mongodb-1.2.5 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../mongodb-*

# Install rdkafka module version=0.9.1
RUN set -x \
    && wget -c https://pecl.php.net/get/rdkafka-0.9.1.tgz \
    && tar zxvf rdkafka-0.9.1.tgz \
    && cd rdkafka-0.9.1 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../rdkafka-*

# Install grpc module version=1.16.0
RUN set -x \
    && wget -c https://pecl.php.net/get/grpc-1.16.0.tgz \
    && tar zxvf grpc-1.16.0.tgz \
    && cd grpc-1.16.0 \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf ../grpc-*

# Install thrift_protocol module version=1.0
RUN set -x \
    && wget -c https://github.com/apache/thrift/archive/0.10.0.tar.gz \
    && tar zxvf 0.10.0.tar.gz \
    && cd thrift-0.10.0/lib/php/src/ext/thrift_protocol \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf /data/service/{0.10.0.*,thrift-0.10.0}

# Install  doumo_dek
# set ssh-key
COPY config id_rsa id_rsa.pub /root/.ssh/
RUN chmod 0600 /root/.ssh/id_rsa*

RUN set -x \
    && git clone git@git.corp.xxxx.com:core/dek_xxxx.git \
    && cd dek_xxxx \
    && git checkout php_5.5.9_ext_dek \
    && cd dek \
    && phpize \
    && ./configure --with-php-config=/data/service/php5.5.9/bin/php-config \
    && make -j 4 \
    && make install \
    && rm -rf /data/service/dek_xxxx

# Add user uid=gid=6234
RUN set x \
    && groupadd -g 6234 www \
    && adduser -u 6234 -g www www

# Create base directory
RUN mkdir -p /data/service_logs/php_log

# Copy config file
COPY php.ini php-fpm.conf /data/service/php5.5.9/etc/
COPY www.conf /data/service/php5.5.9/etc/php-fpm.d/

# Run service
CMD ["php-fpm","-F"]
```



## 命名规则

归纳为应用类镜像

10.16.200.119/xxxx/php5.5.9