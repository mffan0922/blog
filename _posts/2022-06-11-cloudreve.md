---
layout       : post
title        : "Cloudreve——超棒的网盘存储应用"
subtitle     : "Wow!!!"
date         : 2022-06-11 21:16:37
author       : "Manford Fan"
catalog      : false
header-img   : "img/post-bg-universe.jpg"
tags         :
               - Cloudreve
               - Aria2
               - Onedrive
               - OD
---

5月28号晚上7点左右，酒足饭饱之后准备刷《毛骗》，突然接到值班的电话，说有个线上客户COC过来一个故障单，想让看下。其实我内心是拒绝的，但是想到这个问题处理能换来以后好多个Jira单，emmmm，忍了。拉了个群看了下问题，是一个自建网盘使用CDN无法正常删除文件的问题，一番沟通+测试下来，问题解决了，就是边缘不支持`DELETE`方法，52预部署了一下，配置了边缘NG直接`DELETE`回源即可解决问题。问题解决了，然后我开始审视这个客户的应用，用的是Cloudreve搭建，倒是很简单，引起我注意的是它下载的url域名并不是源站，而是世纪互联的OD存储的域名.......

然后就是跟这个客户的深入交谈，以及自己的实践了。

## 一、Cloudreve安装

Cloudreve是完全用Go语言开发的，目前作者已经将其开源，官方文档也提供了三种安装方式，基本和之前的posts中提到的一样——源码，二进制以及Docker。为了不在VPS安装过多的编译软件，这里选择的是二进制文件，用作者的话来说，就是开箱即用！

```bash
$ wget 'https://github.com/cloudreve/Cloudreve/releases/download/3.5.3/cloudreve_3.5.3_linux_amd64.tar.gz'
$ tar -xzf cloudreve_3.5.3_linux_amd64.tar.gz
$ ./cloudreve
```

运行`cloudreve`之后会出现如下截图所示的内容，显示已经启动成功，初始化的账号和密码也有给出，该界面只显示一次，如果没有记下来，需要删除同目录文件夹下的cloudreve.db以及conf.ini文件，重新运行。首次运行程序的时候，数据库使用的是内置的SQLite，如果想要修改数据库，可以在删除cloudreve之后，修改conf.ini文件，添加自定义的mysql，更换数据库之后之前的数据就都没有了，需要重新运行`cloudreve`以获得初始账号密码。

![cloudreve-install](/img/posts/cloudreve-install.png 'install cloudreve')

## 二、优化配置

虽说开箱即用，也只是说该应用设计的非常人性化，可以让使用者很容易上手，想要更好的使用，还需要做一些配置。

#### 1. NG反向代理

总不希望一直使用IP+port或者Domain+port的方式访问网盘，所以还是和之前一样，做一个反向代理：

```nginx
# =====================================================================
# cloudreve configuration
# =====================================================================
server {
    server_name pan.rustle.cc;
    listen 80;
    listen [::]:80;
    listen [::]:443 ssl http2;
    listen 443 ssl http2;
    charset utf-8;
    access_log logs/pan.access.log main;
    
    if ($scheme = http) {
        return 302 https://$host$request_uri;
    }

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://localhost:5212;

        client_max_body_size 20000m;
    }
}
```

#### 2. 更换数据库

内置的SQLite数据库并不适用于生产环境，虽然小破站算不上多重要，也不愿意崩溃的时候无法恢复，[使用mysql](https://docs.cloudreve.org/getting-started/config)，以后崩溃或者换站还可以备份恢复一下。

#### 3. 配置Aria2离线下载

到目前为止，我举得这个离线下载功能挺鸡肋的，可能是我没有找到合适的下载源吧，因为我尝试下载PT以及迅雷链接都不能生效，不过还是记录一下配置过程：

- step1: 安装aria2
- step2: 根据[文档配置](https://docs.cloudreve.org/use/aria2)在/etc/aria2/aria2c.conf文件中
- step3: 配置cron服务开机启动——`/usr/bin/aria2c --conf-path=/etc/aria2/aria2c.conf -D`

#### 4. 更换世纪互联Onedrive存储

和nextcloud遇到了一样的问题，因为服务器是人家的，带宽是买的，钱包不够鼓呀，上传和下载速度都非常捉急，上传其实还好，因为是BGP的线路从本地上传基本能达到1~3M，如果是从其他VPS传输的话，基本可以到15M左右，但是VPS的硬盘容量有限；下载就非常不能接受了，6M的带宽跑满也就700K左右.....

最初是想用鹅厂家的COS来着，速度也挺快，新用户50G存储能白嫖6个月，28号晚上试了一下上传下载，都很快，第二天醒来就受到了鹅厂的账单——0.03元，拍了下脑门突然想起来，下行流量是收费的！虽然腾讯的COS和CVM（或者CDN，还是要花钱...）如果在同一个区域比如上海地区，就可以通过内网传输，上下行不计费，太麻烦了，劝退。

这个时候想起来昨天晚上那位兄弟说的他用的是世纪互联，下行不计费，相当于白嫖。我当时还没反应过来，联想到之前onenote做笔记的痛苦经历以及想要解决问题时所做的努力，终于想起来这个世纪互联是个什么东东了——微软在中国地区的存储代理商。于是悄咪咪的去某宝买了5T的空间，根据Cloudreve配置页的提示信息配置上了，虽然是非正规渠道，本着用一年不亏，用两年血赚的心态，真香~

#### 5. 配置systemd服务

为了更好的管理应用的启动和关闭，systemd当然是最好的选择，不得不说，作者真的很贴心，[服务配置文件](https://docs.cloudreve.org/getting-started/install)也给写好了，稍微修改一下就能用。

## 三、参考文档

- [Cloudreve官网](https://cloudreve.org/)
- [Cloudreve on Github](https://github.com/cloudreve/Cloudreve)
- [Cloudreve官方文档](https://docs.cloudreve.org/)
- [Cloudreve官方论坛](https://forum.cloudreve.org/)
- []()
