FROM centos:centos7.9.2009
COPY apisix-2.9-0.el7.x86_64.rpm /apisix.rpm
RUN yum -y install yum-utils && yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo \
    && yum install -y pcre openresty which tzdata \
    && yum install -y /apisix.rpm \
    && yum clean all \
    && sed -i 's/PASS_MAX_DAYS\t99999/PASS_MAX_DAYS\t60/g' /etc/login.defs
WORKDIR /usr/local/apisix
RUN ln -sf /dev/stdout /usr/local/apisix/logs/access.log \
    && ln -sf /dev/stderr /usr/local/apisix/logs/error.log
EXPOSE 9080 9443
CMD ["sh" "-c" "/usr/bin/apisix init && /usr/bin/apisix init_etcd && /usr/local/openresty/bin/openresty -p /usr/local/apisix -g 'daemon off;'"]