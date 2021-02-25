# 编译docker实例


以webapi为例，webapi 是web后台接口。负责任务的增删改查、将任务分发到真实运行的节点、提供统计查询和相关告警信息查询。如下是打包以后可运行的安装包目录

```bash
[root@localhost webapi-0.1.1]$ tree -a
.
├── bin             #可执行文件bin目录
│   └── webapi
├── conf            #配置目录
│   └── conf.yml
├── docker-entrypoint.sh  #docker entrypoint
├── Dockerfile
├── .dockerignore        #忽略文件声明 (忽略安装脚本，更新脚本，Dockerfile)
├── install.sh           #非docker版本 安装脚本
├── ssl                  #自带证书
│   ├── key.crt
│   └── tls.crt
└── update.sh            #非docker版本 更新脚本
```

其中Dockerfile 内容如下：

**debian 版本**

```Dockerfile
FROM debian:buster-slim

LABEL author="tjuliuyou@gmail.com" \
    description="my web api app"
# 注意使用 .dockerignore 忽略其他文件
ADD ./ / 

ENV CONFIG_FILE=/conf/conf.yml
ENV LOG_LEVEL=INFO
ENV VERBOSE=2

EXPOSE 80

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["webapi"]

```

**alpine 版本**

```Dockerfile
FROM alpine:3.10

LABEL author="tjuliuyou@gmail.com" \
    description="my web api app"
# 注意使用 .dockerignore 忽略其他文件
ADD ./ / 

ENV CONFIG_FILE=/conf/conf.yml
ENV LOG_LEVEL=INFO
ENV VERBOSE=2

EXPOSE 80

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["webapi"]
```

其中 `docker-entrypoint.sh`文件如下：

```bash
#!/usr/bin/env bash
#
# Created by tjuliuyou on 21/02/25.
#
set -e

# if command starts with an option, prepend webapi
if [ "${1:0:1}" = '-' ]; then
    set -- webapi "$@"
fi
# cd workspace

# if command webapi only, add use default args
if [ "$1" = 'webapi' ] && [ "$#" -eq 1 ]; then
    exec webapi -conf ${CONFIG_FILE} -log_level ${LOG_LEVEL} -verbose ${VERBOSE} -logtostderr true
fi

exec "$@"

```

k8s 部署实例(关键部分)

```yaml
#...
  template:
    metadata:
      labels:
        app: webapi
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: app-tls
      - name: conf-volume
        configMap:
          name: webapi-config
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - node1.skynetcloud.com 
      containers:
      - name: webapi
        image: webapi:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: secret-volume
          mountPath: /ssl
        - name: conf-volume
          mountPath: /conf/conf.yml
          subPath: conf.yml
        env:
        - name: VERBOSE
          value: 2
```

**alpine 版本需要添加额外DNS选项**

```yaml
  template:
    metadata:
      labels:
        app: webapi
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: app-tls
      containers:
      - name: webapi
        image: webapi:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: conf-volume
          mountPath: /conf/conf.yml
          subPath: conf.yml
        env:
        - name: VERBOSE
          value: 2
        dnsConfig:
          options:
            - name: ndots
              value: "2"
```

解释：域名中不大于两个点将会从搜索域中搜索该域名解析，这种写法对于主域名可能还是会存在问题 举例而言：例如（magic.ac.cn)

```bash
magic.ac.cn        -> 异常解析
www.magic.ac.cn    -> 正常解析
````

参考 [kubernetes容器中域名解析优化](https://ieevee.com/tech/2019/06/22/ndots.html)

### FAQ

#### 1. ENTRYPOINT 与 CMD 使用？

ENTRYPOINT不会被直接覆盖：使用场景为入口点；CMD 可以被覆盖：使用场景一般填写入口参数。具体而已参考上一节webapi实例

**当不输入任何参数时，使用默认参数相当于运行**
`bash /docker-entrypoint.sh webapi`



```
[root@localhost webapi]$ docker run -it webapi:0.1.1
I1028-15:48:25.878 main.go:43] 当前WebAPI版本:  0.1.1
I1028-15:48:25.878 main.go:44] Git Commit Hash : 115827ab9e8696e141d58b47f7170f21f932f08d
I1028-15:48:25.878 main.go:45] UTC Build Time : 2021-02-25_07:15:12上午
...

```
> 可以通过-e 传递环境变量 如：`docker run -it -e CONFIG_FILE=/app/config.yaml webapi:0.9.2-643`

**当输入`--help`参数时，使用附带参数，相当于运行**
`bash /docker-entrypoint.sh webapi --help`

```
[root@localhost webapi]$ docker run -it webapi:0.1.1 --help
Usage of webapi:
  -conf string
        The configure file (default "conf.yml")
  -log_dir string
        If non-empty, write log files in this directory
  -log_level value
        logs at or above this threshold go to stderr
  -logtostderr
        log to standard error instead of files
  -pprof string
        [localhost:6060]start debug page.
  -verbose value
        log level for V logs
  -version
        show build version.
failed to resize tty, using default size
...

```

**当输入`bash`参数时，使用附带参数，相当于运行**
`bash /docker-entrypoint.sh bash`

```bash
[root@localhost webapi]$ docker run -it webapi:0.1.1 bash
root@09b00aa0f1bc:/# ls
bin  boot  conf  dev  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  ssl  sys  tmp  usr  var
root@09b00aa0f1bc:/# exit
```


#### 2. 为什么使用 `debian:buster-slim`  替换 `alpine`

优点:

* Alpine Linux 使用了 musl，可能和其他 Linux 发行版使用的 glibc 实现会有所不同。在容器化中最可能遇到的是 [DNS 问题](https://github.com/gliderlabs/docker-alpine/issues/8)，即 musl 实现的 DNS 服务不会使用 resolv.conf 文件中的 search 和 domain 两个配置，这对于一些通过 DNS 来进行服务发现的框架可能会遇到问题
* `debian:buster-slim`编译脚本如下：添加了默认UTC+8时区配置；添加了智为默认RootCA证书

```Dockerfile
FROM debian:buster-slim
LABEL author="tjuliuyou@gmail.com" \
      version="1.0.0" \
      description="my k8s base container"

COPY sources.list /etc/apt/sources.list

RUN apt-get update && \
  apt-get install -y ca-certificates && \
  rm -rf /var/lib/apt/lists/* && \
  update-ca-certificates && \
  cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
CMD ["/bin/bash"]

```

缺点：

体积变大 5M ->50M



