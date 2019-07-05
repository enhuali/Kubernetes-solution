FROM centos:7.4.1708
RUN yum install -y kde-l10n-Chinese glibc-common tzdata
RUN localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
ENV LC_ALL zh_CN.UTF-8
ENV TZ=Asia/Shanghai
