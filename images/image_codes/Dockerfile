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

