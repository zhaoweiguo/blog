----
title: 尝鲜：如何搭建一个简单的webrtc服务器
date: 2022-03-08 11:46:32
categories:
- introduce
tags:
- introduce
- webrtc
----

# 前言

前几天我一朋友问我有关webrtc的事，简单了解了下相关知识，搭建了一个webrtc的服务，以及经历的各种踩坑事件，感觉踩坑主要是Python、Node、OpenSSL等版本问题和证书问题导致。本来以为很简单的搭建，但在搭建的过程中遇到各种阻碍，写一篇文章梳理一下。

# WebRTC 简介

WebRTC，名称源自网页即时通信（英语： Web Real-Time Communication ）的缩写，是一个支持网页浏览器进行实时语音对话或视频对话的 API。2010 年 5 月，Google 以 6820 万美元收购 VoIP 软件开发商 Global IP Solutions 的 GIPS 引擎，并改为名为 “WebRTC”。在 2011 年 6 月 1 日开源 WebRTC 并在 Google、Mozilla、Opera 支持下被纳入万维网联盟的 W3C 推荐标准。

WebRTC的Demo项目: https://github.com/webrtc/apprtc

<!--more-->


# 基于 WebRTC 的 1v1 视频实时通话架构

从大的方面可以分为 3 部分:
```
1. WebRTC 终端
    负责音视频采集、编解码、NAT 穿越、音视频数据传输
2. Signal 服务器
    负责信令处理，如加入房间、离开房间、媒体协商消息的传递等
3. STUN/TURN 服务器
    负责获取 WebRTC 终端在公网的 IP 地址，以及 NAT 穿越失败后的数据中转
```

架构图：
![WebRTC 1 对 1 音视频实时通话过程示意图](https://img.zhaoweiguo.com/blog/webrtcs/architect1.png)


# 安装 WebRTC 服务器


## 运行环境准备

* 操作系统：Ubuntu 16.04
* 语言环境：NodeJS、Python、Golang
* 其他软件：git、tar、wget、unzip

命令安装:
```
apt-get update 
apt-get upgrade
apt install -y nodejs-legacy npm python python-webtest golang-go unzip git-core tar wget
```

## 安装依赖服务

安装google_appengine：
```
cd /root/webrtc
wget https://storage.googleapis.com/appengine-sdks/featured/google_appengine_1.9.40.zip
unzip google_appengine_1.9.40.zip
echo "export PATH=$PATH:/root/webrtc/google_appengine" >> /etc/profile
```

安装libevent：
```
wget https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz
tar xvf libevent-2.0.21-stable.tar.gz
cd libevent-2.0.21-stable
./configure
make && make install
```

## 安装信令(Signal)服务
```
mkdir -p /root/webrtc/goWorkspace/src
echo "export GOPATH=/root/webrtc/goWorkspace" >> /etc/profile
source /etc/profile
cd /root/webrtc
git clone git://github.com/webrtc/apprtc.git
ln -s /root/webrtc/apprtc/src/collider/collider $GOPATH/src
ln -s /root/webrtc/apprtc/src/collider/collidermain $GOPATH/src
ln -s /root/webrtc/apprtc/src/collider/collidertest $GOPATH/src
go get collidermain
go install collidermain
```

## 安装STUN/TURN服务
```
cd /root/webrtc
wget http://coturn.net/turnserver/v4.5.0.7/turnserver-4.5.0.7.tar.gz
tar xvfz turnserver-4.5.0.7.tar.gz
cd turnserver-4.5.0.7
./configure
make install
```

## 安装WebRTC 终端服务

通过修改配置文件 ``/root/webrtc/apprtc/src/app_engine/constants.py``来设备好ICE配置（与STUN/TURN服务交互）和WebSocket配置（与信令服务交互）。然后再

### 配置好ICE
```
ICE_SERVER_OVERRIDE  = [
   {
     "urls": [
      # 这儿要改成你自己启动STUN/TURN服务的外网IP
       "turn:47.105.214.21:3478?transport=udp",
       "turn:47.105.214.21:3478?transport=tcp"
     ],
      # 这儿要改成你设备的STUN/TURN服务的用户名和凭证
     "username": "flyer",
     "credential": "123456"
   },
   {
      # 这儿要改成你自己启动STUN/TURN服务的外网IP
     "urls": [
       "stun:47.105.214.21:3478"
     ]
   }
 ]
# 这儿要改成WebRTC 终端服务的请求地址
ICE_SERVER_BASE_URL = 'https://webrtc.zhaoweiguo.com'
```

### 配置好信令服务
```
# 这儿建议设置和WebRTC 终端服务同样的域名，否则要申请两个证书
WSS_INSTANCE_HOST_KEY = 'webrtc.zhaoweiguo.com:8089'
WSS_INSTANCE_NAME_KEY = 'vm_name'
WSS_INSTANCE_ZONE_KEY = 'zone'
WSS_INSTANCES = [{
    WSS_INSTANCE_HOST_KEY: 'webrtc.zhaoweiguo.com:8089',
    WSS_INSTANCE_NAME_KEY: 'wsserver-std',
    WSS_INSTANCE_ZONE_KEY: 'us-central1-a'
}]

```

### 安装相关依赖

```
# 1. 升级npm
npm install -g n
n stable
hash -r  #   (for bash, zsh, ash, dash, and ksh)
# or rehash   (for csh and tcsh)

# 2. 安装node构建工具：grunt
npm -g install grunt-cli@1.3.2

# 3. 安装 coffeescript
npm install --dev coffeescript
```

### 安装apprtc
```
cd /root/webrtc/apprtc
npm install --no-fund
grunt build
```


# 证书生成

## 生成私有证书
```
openssl req -x509 -out /cert/cert.crt -keyout /cert/key.pem \
  -newkey rsa:2048 -nodes -sha256 \
  -subj '/CN=webrtc.zhaoweiguo.com' \
  && cat /cert/key.pem > /cert/cert.pem \
  && cat /cert/cert.crt >> /cert/cert.pem \
  && chmod 600 /cert/cert.pem /cert/key.pem /cert/cert.crt

# 注意：把-subj '/CN=webrtc.zhaoweiguo.com'中的域名改成你自己的域名
```

## 使用certbot生成免费的Let’s Encrypt证书

安装certbot：
```
# Debian
apt-get install certbot
# Docker
docker pull certbot/certbot
```
前提：
```
1. 执行此命令必须使用 root 用户获得文件夹的权限
2. 域名能访问并且有绑定的公网 IP
3. 必须在此域名绑定的服务器上运行
```
基于 Nginx 生成证书：
```
安装插件:
apt-get install python-certbot-nginx
执行命令:
$ sudo certbot certonly --nginx
  Saving debug log to /var/log/letsencrypt/letsencrypt.log
  Plugins selected: Authenticator nginx, Installer nginx

  Which names would you like to activate HTTPS for?
  - - - - - - - - - - - - - - - - - - - - - - - -
  1: zhaoweiguo.com
  2: knowledge.zhaoweiguo.com
  3: www.zhaoweiguo.com
  - - - - - - - - - - - - - - - - - - - - - - - -

选择你要生成证书的域名，根据提示一步步执行就好了
```

不基于webserver直接生成证书
```
1. 直接命令执行
certbot certonly --standalone --email admin@zhaoweiguo.com -d webrtc.zhaoweiguo.com
2. 使用Docker执行
docker run -it -p 80:80 --rm --name certbot \
    -v "/etc/letsencrypt:/etc/letsencrypt" \
    -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
    certbot/certbot certonly --standalone \
    --email admin@zhaoweiguo.com -d webrtc.zhaoweiguo.com

注意：80端口不可被占用，它会启动80端口的一个服务用于验证你的合法性
```
证书生成成功界面
```
- Congratulations! Your certificate and chain have been saved at:
 /etc/letsencrypt/live/webrtc.zhaoweiguo.com/fullchain.pem
 Your key file has been saved at:
 /etc/letsencrypt/live/webrtc.zhaoweiguo.com/privkey.pem
 Your cert will expire on 2023-03-19. To obtain a new or tweaked
 version of this certificate in the future, simply run certbot
 again. To non-interactively renew *all* of your certificates, run
 "certbot renew"
- If you like Certbot, please consider supporting our work by:
 Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 Donating to EFF:                    https://eff.org/donate-le
```

# 启动服务

## 设定环境变量
```
# 本地IP地址
export LOCAL_IP=xx.xx.xx.xx
# 外网IP地址
export SERVER_IP=xx.xx.xx.xx
```

## 启动STUN/TURN服务
```
nohup turnserver -L $LOCAL_IP -a -u flyer:123456 -v -f -r nort.gov &
```
检查是否启动成功：
```
lsof -i:3478
# 出现如下信息即为成功：
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
turnserve 14754 root   16u  IPv4 112232      0t0  UDP instance-5i00bk4l:3478 
turnserve 14754 root   17u  IPv4 112233      0t0  UDP instance-5i00bk4l:3478 
turnserve 14754 root   32u  IPv4 112263      0t0  TCP instance-5i00bk4l:3478 (LISTEN)
turnserve 14754 root   33u  IPv4 112267      0t0  TCP instance-5i00bk4l:3478 (LISTEN)
```
## 启动信令服务
启动命令：
```
cd /cert    # 这个目录里面有前面生成的域名证书和密钥
nohup $GOPATH/bin/collidermain -port=8089 -tls=true  -room-server="https://$SERVER_IP:8080" &
```
检查是否启动成功：
```
lsof -i:8089
# 出现如下信息即为成功：
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
colliderm 14817 root    3u  IPv6 112969      0t0  TCP *:8089 (LISTEN)
```
出现如下打印日志即为启动成功：
```
2022/03/11 02:09:00 Starting collider: tls = true, port = 8089, room-server=https://47.253.204.204:8080
```

## 启动
```
nohup dev_appserver.py --host=$LOCAL_IP /root/webrtc/apprtc/out/app_engine --skip_sdk_update_check &
```
这时候在你的浏览器输入 http://服务器公网IP:8080/ 如果能顺利打开，那么恭喜你环境算是基本搭建成功啦！！！

## 配置Nginx
```
upstream roomserver {
    server 本机ip:8080;
}
server {
    root /usr/share/nginx/html;
    index index.php index.html index.htm;
    listen      443 ssl;
    ssl_certificate /cert/cert.pem;
    ssl_certificate_key /cert/key.pem;
    server_name webrtc.zhaoweiguo.com;   // 改成你自己的域名
    location / {
        proxy_pass http://roomserver$request_uri;
        proxy_set_header Host $host;
    }
}

```
加载Nginx:
```
nginx -s reload
```

# 运行效果

打开浏览器，输入 ``https://webrtc.zhaoweiguo.com`` ，页面显示如下：

![首页](https://img.zhaoweiguo.com/blog/webrtcs/web-show1.png)

点击连接，后

![等待对方连接页面](https://img.zhaoweiguo.com/blog/webrtcs/web-show2.png)

可以看到下面等待其他人加入的链接，使用电脑（手机）浏览器打开这个链接，后会出现（如图我自己手机打开）

![准备连接页面](https://img.zhaoweiguo.com/blog/webrtcs/web-show4.jpg)

点击「Join」后，显示连接成功页面

![连接成功页面](https://img.zhaoweiguo.com/blog/webrtcs/web-show5.jpg)

# Docker版实现

如果你只想快速验证使用，最简单的方法是使用Docker方式，我已经把大部分的工作封装到 镜像``zhaowg/webrtc:new2`` 中了，可以直接使用，不过使用时注意网络要使用Docker的``宿主网络模式(host)``:
```
# 注：使用--net=host使用宿主网络模式，如果使用默认的bridge模式会多一层Nat转换
#   在使用STUN/TRUN时会有问题
docker run -e "SERVER_IP=xx.xx.xx.xx" -e "LOCAL_IP=172.17.0.2" --net=host  -it zhaowg/webrtc:new2 bash
```
按上面文档说明修改成你自己的的域名、证书、ip等信息，然后执行
```
./start.sh就可以了
```

# 踩坑记录

1. WebSocket connection failed
```
打开连接页面点击「JOIN」时报下面错
WebSocket connection to 'wss://webrtc.zhaoweiguo.com:8089/ws' failed: 
  Error in connection establishment: net::ERR_CERT_AUTHORITY_INVALID
这时，「信令服务」日志报：
2022/03/11 02:13:22 http: TLS handshake error from 111.205.43.248:15029: remote error: unknown certificate
解决方案：
修改AppRTC服务的配置中的WSS_INSTANCE_HOST_KEY选项
即修改文件/root/webrtc/apprtc/src/app_engine/constants.py
把WSS_INSTANCE_HOST_KEY选项的值设置为对应证书的域名（而不是用IP）
```

2. 各种OpenSSL问题
```
这种问题我的解决方案是使用指定的OS版本
比如这儿用Ubuntu 16.04
```

3. Failed to execute 'pushState' on 'History' 
```
运行都成功，但就是在点击连接时报错
解决方法是:
修改文件 ``./src/web_app/js/appcontroller.js`` 中的 ``AppController.prototype.pushCallNavigation_`` 函数
在此函数最开始增加一句代码：
roomLink = roomLink.replace("http", "https");
参考自：https://github.com/ISBX/apprtc-node-server/issues/7
```
4. grunt build报错
```
原因：
  主要是npm、grunt版本问题
解决方法：
  升级npm，使用指定的grunt版本
```
5. go get collidermain命令失败
```
原因：
collidermain依赖项目golang.org/x/net/websocket
而执行go get collidermain命令时，会请求golang.org/x/net/websocket
而这个请求在国内默认不可访问
解决方案：
1. 解决请求golang.org网站的问题
2. 执行如下命令
mkdir -p $GOPATH/src/golang.org/x/
cd $GOPATH/src/golang.org/x/
git clone git://github.com/golang/net.git net 
go install net
```

# 附录：Dockerfile文件内容

```
FROM ubuntu:16.04

RUN apt-get update && apt-get upgrade && apt install -y nodejs-legacy npm python python-webtest golang-go unzip wget git-core
RUN npm -g install grunt-cli@1.3.2
RUN mkdir -p /root/webrtc/goWorkspace/src && cd /root/webrtc && wget https://storage.googleapis.com/appengine-sdks/featured/google_appengine_1.9.40.zip && unzip google_appengine_1.9.40.zip
RUN wget https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz && tar xvf libevent-2.0.21-stable.tar.gz && cd libevent-2.0.21-stable && ./configure && make && make install

ENV GOPATH=/root/webrtc/goWorkspace
ENV PATH=$PATH:/root/webrtc/google_appengine

WORKDIR /root/webrtc
RUN git clone git://github.com/webrtc/apprtc.git && ln -s /root/webrtc/apprtc/src/collider/collider $GOPATH/src && ln -s /root/webrtc/apprtc/src/collider/collidermain $GOPATH/src && ln -s /root/webrtc/apprtc/src/collider/collidertest $GOPATH/src && go get collidermain && go install collidermain

RUN wget http://coturn.net/turnserver/v4.5.0.7/turnserver-4.5.0.7.tar.gz && tar xvfz turnserver-4.5.0.7.tar.gz && cd turnserver-4.5.0.7 && ./configure && make install

WORKDIR /root/webrtc/apprtc

RUN npm install -g n && n stable && 
RUN npm install --no-fund && npm install --dev coffeescript &&  grunt build

WORKDIR /root/webrtc

RUN echo "nohup turnserver -L $LOCAL_IP -a -u flyer:123456 -v -f -r nort.gov &" >> ./start.sh && echo 'nohup $GOPATH/bin/collidermain -port=8089 -tls=true  -room-server="https://$SERVER_IP:8080" &' >> ./start.sh && echo "nohup dev_appserver.py --host=$LOCAL_IP /root/webrtc/apprtc/out/app_engine --skip_sdk_update_check &" >> start.sh && chmod +x start.sh
```

# 参考

- 极客时间-从 0 打造音视频直播系统: https://zh.wikipedia.org/wiki/WebRTC
- 博客-WebRTC 之服务器搭建: https://www.jianshu.com/p/df300071d8d6






