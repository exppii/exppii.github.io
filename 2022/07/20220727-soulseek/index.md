# Soulseek Docker的中文支持




## 引

一直想找个快乐的下载电影原声配乐、和其他音乐的方式。PT网站是一种选择，但是还需要自行的去下载种子，落雪也是很好的选择，但是没法下载无损音质。其他的就更麻烦。搜来搜去SoulSeek是一个很棒的选择。

## docker 版本
SoulSeekQT是提供mac，windows，linux三个平台的版本。但是为了能24小时下（分）载（享）数据。同时要将下载的数据快速的存储在nas服务器上，docker版本是一个很好的选择 github上也确实找到了一个不错的版本（https://github.com/realies/soulseek-docker）。大概命令如下：

```bash
docker run -d --name soulseek --restart=unless-stopped \
-v "/persistent/appdata":"/data/.SoulseekQt" \
-v "/persistent/downloads":"/data/Soulseek Downloads" \
-v "/persistent/logs":"/data/Soulseek Chat Logs" \
-v "/persistent/shared":"/data/Soulseek Shared Folder" \
-e PGID=1000 \
-e PUID=1000 \
-p 6080:6080 \
realies/soulseek
```
不过有个致命的问题，**不支持中文**。这是一个让人恼火的问题。

## 解决中文问题

简单看了一下为什么不支持，是因为原来镜像包使用的是标准的ubuntu。毕竟不是国人打包的软件。
可以尝试按照如下方式重新打包：

```Dockerfile
# 添加中文输入法，并设置中文
FROM realies/soulseek
RUN set -eux && \
    apt-get update && \
    apt-get install -y xfonts-wqy && \
    locale-gen zh_CN.UTF-8 && \
    update-locale LANG=zh_CN.UTF-8 LANGUAGE=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8 && \
    ln -fs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    find /var/lib/apt/lists -type f -delete && \
    find /var/cache -type f -delete
ENV LANG=zh_CN.UTF-8 LANGUAGE=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8
```
重新启动，饿居然报错了，怎么回事。不慌，看看日志：

```bash
root@home:/home/liuyou/soulseek-docker# docker logs soulseek
2022-07-27 05:23:47,737 INFO Set uid to user 1000 succeeded
2022-07-27 05:23:47,740 INFO supervisord started with pid 21
2022-07-27 05:23:48,743 INFO spawned: 'tigervnc' with pid 22
2022-07-27 05:23:48,745 INFO spawned: 'openbox' with pid 23
2022-07-27 05:23:48,747 INFO spawned: 'novnc' with pid 24
2022-07-27 05:23:48,749 INFO spawned: 'soulseek' with pid 25
2022-07-27 05:23:49,770 INFO success: tigervnc entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-07-27 05:23:49,770 INFO success: openbox entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-07-27 05:23:49,770 INFO success: novnc entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-07-27 05:23:49,770 INFO success: soulseek entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-07-27 05:23:49,770 INFO exited: soulseek (exit status 127; not expected)
2022-07-27 05:23:49,774 INFO spawned: 'soulseek' with pid 59
2022-07-27 05:23:50,833 INFO success: soulseek entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-07-27 05:23:50,833 INFO exited: soulseek (exit status 127; not expected)
```
不可能呀就是添加一个语音包而已。看不出什么呀，继续查看启动命令
cat init.sh
```
[program:soulseek]
user=$username
environment=HOME="/data",DISPLAY=":1",USER="$username"
command=/app/SoulseekQt
autorestart=true
```
依葫芦画瓢我们收到启动试试：

```bash
echo "
su soulseek
export HOME=/data
export DISPLAY=":1"
export USER=soulseek
/app/SoulseekQt --help
" > t.sh

root@81c5257a7174:/app# bash t.sh 
QStandardPaths: wrong ownership on runtime directory /data, 1000 instead of 0
/app/SoulseekQt: symbol lookup error: /lib/x86_64-linux-gnu/libfontconfig.so.1: undefined symbol: FT_Done_MM_Var
```

露出原型了吧，赶紧谷歌一把，得到两个相关的链接：
https://blog.51cto.com/u_15127633/3855600
https://askubuntu.com/questions/1140921/wolfram-mathematica-after-upgrade-to-ubuntu-19-04-symbol-lookup-error-usr-lib
使用系统的libfreetype.so.6代替游戏目录中的libfreetype.so.6

饿貌似`libfreetype` 和报错的 `libfontconfig`没有直接关系吧。不管了，再次修改Dockerfile文件。

```Dockerfile
# 添加中文输入法，并设置中文
FROM realies/soulseek
RUN set -eux && \
    apt-get update && \
    apt-get install -y xfonts-wqy && \
    locale-gen zh_CN.UTF-8 && \
    update-locale LANG=zh_CN.UTF-8 LANGUAGE=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8 && \
    ln -fs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    mv /app/lib/libfreetype.so.6 /app/lib/libfreetype.so.6.bak && \
    find /var/lib/apt/lists -type f -delete && \
    find /var/cache -type f -delete
ENV LANG=zh_CN.UTF-8 LANGUAGE=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8
```
最后运行启动命令：
```
docker run -it -d --restart=unless-stopped \
 --name soulseek \
 --net home6 \
-e PGID=1000 \
-e PUID=1000 \
-e XMODIFIERS="@im=fcitx" \
-e QT_IM_MODULE="fcitx"  \
-e GTK_IM_MODULE="fcitx" \
-p 6080:6080 \
 -v /data/music:/data/music \
soulseek:0.0.4
```


好了 这下舒服了

![](/images/soulseek_final_worked.png)

## 其他参考链接

http://www.soulseekqt.net/news/node/1
https://containerization-automation.readthedocs.io/zh_CN/latest/docker/basic/[Docker][Ubuntu%2018.04]%E4%B8%AD%E6%96%87%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE/
