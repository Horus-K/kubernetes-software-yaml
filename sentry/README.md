# 注意事项
- 内部有跨域问题，所以将一些服务塞到一个pod里了
- 很多服务没有暴露端口，无法加入健康检查，web服务只要配置http健康检查就会500
- kafka需要在同命名空间，一些代码吧地址写死了
# 所作优化
## 代码本身
- 这个版本worker会调用github facebook的一些模板，但是地址写死了master，会导致worker报错，将其地址更换，worker服务很重要，但有性能瓶颈，如果worker与kafka调度到同一台node上，会产生大量连接数，要优化服务器连接数
    ```
    # 表示系统级别的能够打开的文件句柄的数量， 一般如果遇到文件句柄达到上限时，会碰到"Too many open files"或者Socket/File: Can’t open so many files等错误。
    # fs.file-max = 2097152
    fs.file-max=52706963
    # fs.nr_open = 1048576
    fs.nr_open = 52706963
    #-----------------------
    # 最大文件句柄
    # vm.max_map_count = 262144
    vm.max_map_count = 262144
    #-----------------------
    # 最大跟踪连接数，默认 nf_conntrack_buckets * 4
    # net.nf_conntrack_max = 131072
    net.nf_conntrack_max = 1048576
    # 允许的最大跟踪连接条目，是在内核内存中netfilter可以同时处理的“任务”（连接跟踪条目）
    # net.netfilter.nf_conntrack_max = 131072
    net.netfilter.nf_conntrack_max=10485760
    #-----------------------
    ```
- relay服务会向web进行注册，无法启动多个副本(注册生成token),修改了其启动脚本，让其启动后只向pod内的web服务注册，重新打镜像
    - config.yml
        ```
        relay:
        upstream: "http://MY_POD_IP:9000/"
        host: 0.0.0.0
        port: 3000
        logging:
        level: DEBUG
        processing:
        enabled: true
        kafka_config:
            - {name: "bootstrap.servers", value: "kafka:9092"}
            - {name: "message.max.bytes", value: 5000000000} 
        redis: redis://redis:6379
        ```
    - credentials.json
        ```
        {"secret_key":"T54-TgzbtoLO02iCxysVNno5hORLARnRbYCjuvueWHg","public_key":"GegncsiiQXC2VEdo9mFX695CQ-T0-mCMfpXunN2kOaQ","id":"a1dc0a56-3342-4f73-874d-111111111112"}
        ```
    - docker-entrypoint.sh
        ```
        #!/usr/bin/env bash
        set -e
        #sed -i "s/111111111111/$HOSTNAME/" /work/.relay/credentials.json
        sed -i "s/MY_POD_IP/$MY_POD_IP/" .relay/config.yml
        # Enable core dumps. Requires privileged mode.
        if [[ "${RELAY_ENABLE_COREDUMPS:-}" == "1" ]]; then
        mkdir -p /var/dumps
        chmod a+rwx /var/dumps
        echo '/var/dumps/core.%h.%e.%t' > /proc/sys/kernel/core_pattern
        ulimit -c unlimited
        fi

        # For compatibility with older images
        if [ "$1" == "bash" ]; then
        set -- bash
        elif [ "$(id -u)" == "0" ]; then
        set -- gosu relay /bin/relay "$@"
        else
        set -- /bin/relay "$@"
        fi

        exec "$@"
        ```
    - Dockerfile
        ```
        FROM getsentry/relay:20.8.0
        RUN sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
            && sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
            && apt update \
            && apt -y install \
                    vim \
                    iproute2 \
                    iotop \
                    net-tools \
                    lrzsz \
                    tree \
                    telnet \
                    lsof \
                    tcpdump \
                    wget \
                    traceroute \
                    telnet \
                    findutils \
                    unzip \
                    coreutils \
                    tzdata \
                    cronolog \
                    inetutils-ping \
                    curl \
        && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
        && apt-get clean
        RUN rm /docker-entrypoint.sh
        COPY docker-entrypoint.sh /docker-entrypoint.sh
        COPY credentials.json /work/.relay/
        COPY config.yml /work/.relay/
        ```
## 参数优化
- sentry.conf.py
    ```
    urllib3.disable_warnings()

    DEFAULT_KAFKA_OPTIONS = {
        "bootstrap.servers": "kafka:9092",
        #"bootstrap.servers": "kafka.kafka.svc.nbugs.local:9092",
        "message.max.bytes": 50000000,
        "socket.timeout.ms": 100000,  # 增大timeout很重要
    }

    SENTRY_WEB_OPTIONS = {
        "http": "%s:%s" % (SENTRY_WEB_HOST, SENTRY_WEB_PORT),
        "protocol": "uwsgi",
        # This is needed in order to prevent https://git.io/fj7Lw
        "uwsgi-socket": None,
        "so-keepalive": True,
        # Keep this between 15s-75s as that's what Relay supports
        #"http-keepalive": 15,
        "http-keepalive": 30,
        "http-chunked-input": True,
        # the number of web workers
        "workers": 3,
        "threads": 4,
        "memory-report": False,
        # Some stuff so uwsgi will cycle workers sensibly
        #"max-requests": 100000,
        "max-requests": 10000000,
        #"max-requests-delta": 500,
        "max-requests-delta": 50000,
        #"max-worker-lifetime": 86400,
        "max-worker-lifetime": 864000000,
        # Duplicate options from sentry default just so we don't get
        # bit by sentry changing a default value that we depend on.
        "thunder-lock": True,
        "log-x-forwarded-for": False,
        "buffer-size": 32768,
        "limit-post": 209715200,
        "disable-logging": True,
        "reload-on-rss": 600,
        "ignore-sigpipe": True,
        "ignore-write-errors": True,
        "disable-write-exception": True,
    }

    ####自定义变量
    SENTRY_RELAY_OPEN_REGISTRATION = True #开启远程注册
    SENTRY_DEFAULT_TIME_ZONE = 'CST' #时区
    #TIME_ZONE = 'CST'


    SENTRY_SOURCE_FETCH_SOCKET_TIMEOUT = 20
    SENTRY_SOURCE_FETCH_TIMEOUT = 200
    SYMBOLICATOR_POLL_TIMEOUT = 10
    SYMBOLICATOR_PROCESS_EVENT_HARD_TIMEOUT = 1200
    SYMBOLICATOR_PROCESS_EVENT_WARN_TIMEOUT = 240
    DATA_UPLOAD_MAX_NUMBER_FIELDS = 10000
    SENTRY_DEFAULT_MAX_EVENTS_PER_MINUTE = "100%"
    SENTRY_MAX_AVATAR_SIZE = 50000000
    SENTRY_MAX_DICTIONARY_ITEMS = 100
    SENTRY_MAX_HTTP_BODY_SIZE = 1638400
    SENTRY_MAX_MESSAGE_LENGTH = 819200
    SENTRY_MAX_VARIABLE_SIZE = 51200
    ```