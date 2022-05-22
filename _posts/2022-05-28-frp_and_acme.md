---
layout       : post
title        : "FRP&ACME——内网穿透+证书签发"
subtitle     : "Keep your data on your own machine"
date         : 2022-05-28 10:03:29
author       : "Manford Fan"
catalog      : false
header-img   : "img/post-bg-universe.jpg"
tags         :
               - FRP
               - ACME
---

很久之前听说过花生壳可以让内网的设备在公网上访问，当时还不知道什么叫域名，什么是代理，只是觉得很神奇。最近一年由于工作原因，接触了很多跟web相关的东西，恰巧自己也跟着搭建了一些网络服务器，更巧的是，家里有一台配置还行的内存不是很大的老笔记本——i7 6500u/4G DDR3 1600MHz/256G Nvme SSD，就想着把搭建在腾讯云的服务挪到自己的笔记本上，用FRP实现内网穿透（因为花生壳收费【穷.jpg】），而腾讯云的机器只做流量转发，优点是真正实现了所有数据保存在自己的机器上，安全性高，缺点也很明显，增加了访问链路长度，带宽的损耗比可避免，具体是多少还在验证中，另外如果VPS是上下行计费的话，流量消耗将是双倍，再就是自己的笔记本不能断网断电，不然服务就彻底嗝屁了，这就涉及到本地网络带宽以及笔记本的功耗问题。

## 一、FRP —— A fast reverse proxy

FRP采用C/S模式，将服务端部署在具有公网IP的机器上，客户端部署在内网或防火墙内的机器上，通过访问暴露在服务器上的端口，反向代理到处于内网的服务。在此基础上，frp支持 TCP/UDP/HTTP/HTTPS等多种协议，提供了加密、压缩，身份认证，代理限速，负载均衡等众多能力。

#### 1. 原理

frp主要由客户端(frpc)和服务端(frps)组成，服务端通常部署在具有公网IP的机器上，客户端通常部署在需要穿透的内网服务所在的机器上。内网服务由于没有公网IP，不能被非局域网内的其他用户访问。用户通过访问服务端的frps，由frp负责根据请求的端口或其他信息将请求路由到对应的内网机器，从而实现通信。在frp中一个代理对应一个需要暴露的内网服务，一个客户端支持同时配置多个代理。其中支持的类型如下：

|类型|描述|
|:-|:-|
|tcp|单纯的TCP端口映射，服务端会根据不同的端口路由到不同的内网服务|
|udp|单纯的UDP端口映射，服务端会根据不同的端口路由到不同的内网服务|
|http|针对HTTP应用定制了一些额外的功能，例如修改Host Header，增加鉴权|
|https|针对HTTPS应用定制了一些额外的功能|
|stcp|安全的TCP内网代理，需要在被访问者和访问者的机器上都部署frpc，不需要在服务端暴露端口|
|sudp|安全的UDP内网代理，需要在被访问者和访问者的机器上都部署frpc，不需要在服务端暴露端口|
|xtcp|点对点内网穿透代理，功能同stcp，但是流量不需要经过服务器中转|
|tcpmux|支持服务端TCP端口的多路复用，通过同一个端口访问不同的内网服务|
|kcp|基于udp，比tcp多浪费10%~20%的带宽，换取30%~40%的延时性能提升|


#### 2. 配置

frp由客户端和服务端构成，通过客户端主动与服务端发起链接从而建立通信，其配置文件分别是frpc.ini和frps.ini，可以通过命令启动，也可以配置systemd服务。

> **服务端**

```text
[common]
# FRP监听端口
bind_port = 8765
kcp_bind_port = 8765 # 一定要开启该端口的udp协议
max_pool_count = 100
tcp_mux = on

# http监听端口
vhost_http_port = 8888

# https监听端口
vhost_https_port = 54321

# domain域
subdomain_host = rustle.cc


# 授权码
token = rcAhsh$ub1lh

# frp管理后台端口
dashboard_port = 12345

# frp管理后台用户名和密码
dashboard_user = admin
dashboard_pwd = XXXXXXX
enable_prometheus = true

# frp日志配置，需要提前创建日志文件，并修改所有者和所属组为nobody与nogroup
log_file = /opt/logs/frps.log
log_level = trace
log_max_days = 3
```

> **客户端**

```text
# 客户端配置
[common]
server_addr = 1.23.4.56
server_port = 8765  # 对应bind_port
pool_count = 20
use_encryption = true
use_compression = true
protocol = kcp

# 授权码
token = rcAhsh$ub1lh

# 配置ssh服务
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 8888  # ssh服务必须要指定，http(s)服务可以在nginx中指定

[homepage]
type = http        # http协议
local_port = 8001  # 本地监听8001端口
subdomain = www    # 域名是www.rsutle.cc 
custom_domains = rustle.cc  # 或者是 rustle.cc 

# frp日志配置，要求同服务端，但是好像客户端不需要配置...
log_file = /opt/logs/frps.log
log_level = trace
log_max_days = 3
```

#### 3. 隐藏端口&开启https

通过frp转发的http服务都必须带端口访问，比如"http://rustle.cc:8080/"，看起来很丑，还要记端口号，很不友好。可以通过nginx进行反向代理把请求转发给frps监听的端口，到达将服务端端口号隐藏掉的效果。另外，现在很多浏览器都强制要求https协议访问，更有些网站要求HSTS，所以当用frp转发内网的web服务的时候，也支持开启https协议。目前有两种方式，一个是在服务端的nginx服务器配置证书，第二个就是通过frp本身的https2http插件在客户端实现。

> **服务端实现**

原理很简单，就是用nginx服务监听80和443端口，同时配置ssl证书即可，LAN客户端无需做任何https相关的配置。

```nginx
server {
    server_name *.rustle.cc;
    listen 80;
    listen 443 ssl http2;
    ssl_certificate /opt/cert/server.crt;
    ssl_certificate_key /opt/cert/server.key;

    proxy_connect_timeout 60;
    proxy_send_timeout 60;
    proxy_read_timeout 60;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

    location / {
        proxy_pass http://localhost:8080;
    }
}
```

> **客户端实现**

说是客户端实现，其实服务端也需要做稍微的配置，即开启https监听端口`vhost_https_port = 54321`，其他的就交给LAN客户端即可。如下是客户端frpc.ini文件的相关配置：

```text
[test_https2http]
type = https
custom_domains = test.example.com

plugin = https2http
plugin_local_addr = 127.0.0.1:8001
plugin_crt_path = ./server.crt
plugin_key_path = ./server.key
plugin_host_header_rewrite = 127.0.0.1
plugin_header_X-From-Where = frp
```

个人更喜欢第一种方式，在服务端配置，同时监听80和443端口，客户端实现方法也能达到同样的效果，但是需要在服务端同时配置两个server块，分别监听80和443，并使用nginx分别转发到不同的frp端口，稍微复杂一点，对于这两种方法，暂时没有对比转发效率的优劣。

#### 4. 优化

经过frp转发之后，带宽一定会有所损耗，可以通过开启kcp协议减缓，在弱网环境效果尤其明显；另外由于网络的复杂性，frp服务经常会断开链接，可以通过systemd服务，让frp服务断开一定时间之后重新链接。以内网客户端为例：

```text
[Unit]
Description=Frp Client Service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
User=www-data
Restart=always
RestartSec=5s
ExecStart=/usr/bin/frpc -c /etc/frp/frpc.ini
ExecReload=/usr/bin/frpc reload -c /etc/frp/frpc.ini
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

另外，当我们做frp性能调优的时候，经常会尝试一些不同的配置，很有可能会把客户端搞挂掉，但是自己又不在家，于是家里的服务器就失联了。这是很悲催的事情，不过可以通过`crontab`服务来规避，将验证过的有效的配置放在备份文件夹，设置每30分钟或1个小时将该备份文件拷贝到frp配置文件夹，这样就有30分钟或1个小时的时间进行调试实验，即使调试过程中内网服务器挂掉了也不怕，一段时间之后就自动恢复了，**调优成功之后记得及时把备份文件更新**。

## 二、ACME —— 自动签发证书

既然要配置https访问，肯定要申请相应的证书，之前用`certbot`只能申请单域名证书，无法申请泛域名证书。所以这次使用`acme.sh`来申请泛域名证书，同时也**大大简化了**服务端nginx的配置。`acme.sh`支持多个CA机构签发免费证书，但是只支持DV证书的签发，也就是通过验证域名所有权，然后签发该域名的证书。它支持两种验证方式，一种是通过HTTP的方式验证，另一种是通过DNS的方式验证，只有第二种方式支持签发泛域名证书，所以如下也是记录的该方式。

#### 1. 准备工作

ACME支持cloudflare, dnspod, aliyun, cloudxns, godaddy以及ovh等数十种[DNS解析商](https://github.com/Neilpang/acme.sh/tree/master/dnsapi)的自动集成，脚本会根据提供的API的AK和SK自动的在解析中添加一条TXT解析，并验证你对该域名具有所有权，处理完之后在调用API自动的将该TXT解析删除。所以要先获取AK和SK并作相应的配置，以阿里云为例：https://usercenter.console.aliyun.com/#/manage/ak，点击该链接申请创建AK，SK，并将其导入服务端环境变量定义如下：

```bash
export Ali_Key="sddiwjedfasSDFSFsdaf"
export Ali_Secret="jlsdsddiwjedfasSDFSFkljlfdsaklkjflsa"
```

#### 2. 开始安装

执行如下命令进行安装，默认情况下，acme.sh以隐藏文件夹的形式安装在用户的主目录（ACME不要求root权限）下，并且crontab会自动创建一条定时任务，每天凌晨检查证书是否过期，是否需要续签等。

```bash
$ cd ~
$ git clone https://github.com/acmesh-official/acme.sh.git
$ cd acme.sh
$ ./acme.sh --install -m  my@example.com
```

安装完成之后就开始申请证书，因为是签发，所以要使用`–issue`参数；指明使用`dns_ali`作为验证方式；后面跟着的`-d`为指定证书中的域名，这里有一点需要注意的：**如果证书中只包含泛域名，那么签发出来的证书是没有根域的。所以需要额外添加一个根域。**等待2~3分钟，证书会自动申请成功，并存放在~/.acme/rustle.cc/目录下。

```
$ acme.sh --issue --dns dns_ali -d rustle.cc -d *.rustle.cc
```


#### 3. 配置证书

证书主要用在nginx服务中，所以要使用包含中级CA的域名证书与私钥，配置在NG服务配置文件即可。

```nginx
ssl_buffer_size            16k;
ssl_protocols              TLSv1.1 TLSv1.2 TLSv1.3;
ssl_ecdh_curve             X25519:P-256:P-384:P-224:P-521;
ssl_ciphers                TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256;
ssl_prefer_server_ciphers  on;
ssl_session_timeout        3h;
ssl_stapling               on;
ssl_session_tickets        on;

ssl_certificate /opt/cert/fullchain.cer;
ssl_certificate_key /opt/cert/rustle.cc.key;
```


## 三、参考文档

- [FRP on Github](https://github.com/fatedier/frp)
- [FRP中文文档](https://gofrp.org/)
- [ACME on Github](https://github.com/acmesh-official/acme.sh)
- [acme.sh与阿里云DNS自动签发](https://www.orcy.net.cn/1337.html)
