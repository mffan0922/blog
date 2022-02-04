## [诲之的博客](https://blog.rustle.cc)

本博客是从[黄玄的博客](https://github.com/Huxpro/huxpro.github.io)借鉴修改而来，最原始的博客模板是[clean blog jekyll](https://github.com/StartBootstrap/startbootstrap-clean-blog-jekyll)，个人认为，博客整体风格是很棒的，对于并不是前端的小白来讲，也比较容易上手，适合折腾一下，不过根绝博主自己的经验还是先打好基础再去折腾，会比较不走弯路，不会浪费过多的时间。

## User Manual 👉

构建一个[Jekyll](https://jekyllrb.com/)博客网站，需要的东西有，一台有公网IP的服务器，一个有所属权的域名，以及必要的软件环境，比如jekyll所依赖的ruby环境，jekyll主程序，以及web服务器（Nginx或者Apache），下面我们一步一步来构造，let's start~

#### 一个VPS服务器

按照步骤，首先需要有一个VPS服务器，根据域名是否已经备案可以选择国内或者国外的，知名的国内VPS服务器厂商有[阿里云](https://www.aliyun.com/)的ECS，[腾讯云](https://cloud.tencent.com/)的CVM，[天翼云](https://www.ctyun.cn/portal/)的EVM，还有[UCloud](https://www.ucloud.cn/)，加拿大的[BandwagonHost](https://bwh81.net/)，美国的[vultr](https://www.vultr.com/)以及[Linode](https://www.linode.com/)等等，具体如何开通不在此记录。

#### 一个用有所属权的域名

域名（Domain Name）又称网域，是由一串用点分隔的名字组成的Internet上某一台计算机或计算机组的名称，用于在数据传输时对计算机的定位标识（有时也指地理位置）。域名的出现是为了解决IP不好记忆的问题，使用域名的另一个好处是，可以在同一个IP上构建不同的网站。和VPS一样，域名是需要购买的，国内国外均有，基于维护的成本，国内是比较方面的，域名提供商也是参考VPS提供商即可。

购买完域名之后，需要设置解析，即哪一个域名要解析到哪一个IP上

![](https://blog.rustle.cc/img/domain.png)

#### Jekyll环境

接下来配置Jekyll环境，Jekyll是一个静态网页生成器，诞生之初，网络上也充斥着关于静态网页博客的各种讨论和争执，不过现在看来，似乎没有什么意义了。他就是一个工具，内容才是核心。Jekyll是依赖ruby环境的，大多数人可能没有听说过这个编程语言，我也仅仅是听过而已。不过这并不影响我们使用，因为只要安装好环境，其他时间我只需要与Markdown打交道。如下是安装环境的主要命令（Ubuntu环境下），在Debian和Redhat下安装的时候所需要的依赖是不同的，这里不详细的记录，主要是看报错日志，缺少哪个依赖，安装上即可。

```bash
[root@rustle ~]$ apt install ruby ruby-dev
[root@rustle ~]$ gem install jekyll bundler
[root@rustle ~]$ bundle install
[root@rustle ~]$ jekyll build
```

#### web 服务器环境

目前主流的web服务器有nginx和apache两种，这里使用nginx为例来做演示，如下是安装配置过程

```bash
# install nginx
[root@rustle ~]$ wget http://nginx.org/download/nginx-1.20.2.tar.gz
[root@rustle ~]$ tar -xzf nginx-1.20.2.tar.gz && cd nginx-1.20.2
[root@rustle nginx-1.20.2]$ ./configure --with-http_ssl_module && make && make install
...
[root@rustle nginx-1.20.2]$ ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx

# configure sercice
[root@rustle ~]$ systemctl enable nginx.service
[root@rustle ~]$ systemctl disable nginx.service
[root@rustle ~]$ systemctl start nginx.service
[root@rustle ~]$ systemctl stop nginx.service
[root@rustle ~]$ systemctl restart nginx.service
[root@rustle ~]$ systemctl status nginx.service
```

如下是nginx.service的把内容，将该文件放在/lib/systemd/system/，可执行如上的nginx的启动，停止，重载等命令：

```text
---------------------------------------------------------------------------------------
[Unit]
Description=nginx
After=network.target
  
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx -c /opt/websites/conf/rustle.cc.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
  
[Install]
WantedBy=multi-user.target
---------------------------------------------------------------------------------------
```

最后是修改nginx域名的配置文件，位置一般在/usr/local/nginx/conf/nginx.conf，如果是使用apt或者yum这种在线工具安装的nginx，那么配置文件在/etc/nginx/conf/nginx.conf，修改好重新运行nginx服务即可。

#### 配置证书（可选）

如下是申请Let's Encrypt免费证书的命令行方法，只适用于申请单证书。当域名增加的时候只需要重新执行最后一行即可，Let's Encrypt证书有效期是3个月，可以使用`certbot renew --dry-run`命令实现自动续期：

```bash
[ 0 root@rustle ~]$ yum install epel-release
[ 0 root@rustle ~]$ yum upgrade
[ 0 root@rustle ~]$ yum install snapd
[ 0 root@rustle ~]$ systemctl enable --now snapd.socket
[ 0 root@rustle ~]$ ln -s /var/lib/snapd/snap /snap
[ 0 root@rustle ~]$ snap install core
[ 0 root@rustle ~]$ snap refresh core
[ 0 root@rustle ~]$ snap install --classic certbot
[ 0 root@rustle ~]$ ln -s /snap/bin/certbot /usr/bin/certbot
[ 0 root@rustle ~]$ certbot --nginx-server-root /usr/local/nginx/conf
```

#### 效果图

![](https://blog.rustle.cc/img/blog_screenshot.jpg)

## Reference

 * Jekyll 官方站点：https://jekyllrb.com/
 * 黄玄的博客：https://github.com/Huxpro/huxpro.github.io

## License

<img src="https://img.shields.io/github/license/mffan0922/wiki" data-origin="https://img.shields.io/github/license/mffan0922/wiki" alt=""> Apache License 2.0
Copyright (c) 2022-present Manford Fan

本博客是从[黄玄的博客](https://github.com/Huxpro/huxpro.github.io)借鉴修改而来，最原始的博客模板是[clean blog jekyll](https://github.com/StartBootstrap/startbootstrap-clean-blog-jekyll)，引用借鉴请注明原始出处。

