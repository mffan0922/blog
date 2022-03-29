---
layout       : post
title        : "Debian Server部署全过程记录"
subtitle     : "Make it "
date         : 2022-03-29 08:54:06
author       : "Manford Fan"
catalog      : false
header-img   : "img/post-bg-universe.jpg"
tags         :
               - OS
               - Tools
               - Server
---

当一个念头萌生的时候，便一发不可收拾。昨天下午loading稍微轻了一点，我就又开始想要不做个自己的网盘吧，有好多东西没有统一的地方放，主要是坚果云不能再Debian下使用非root用户登录；然后又想要不把Server换成Debian吧，这样服务端和客户端都统一了，感觉也挺不错的；再然后就是想着完整的记录下部署过程，云服务器也快到期了，方便下次部署。于是，就在备份数据之后，把阿里云的服务器重置成Debian10.5了......

## 1、重装系统

登录阿里云的控制台，VPS安装系统Debian10.5，倒不是不想安装11.2，阿里的镜像只支持10.5。因为重置的时候并不会给设定root密码的界面，重新安装完成之后，通过web console进入系统，第一件事就是重置root密码。可以通过`sudo passwd root`来设定，或者通过控制台界面来设定也可。此时，就可以通过远程登录工具（xshell或者SecureCRT等）使用密码登录系统了。

## 2、配置远程登录

在使用xshell进行远程登陆之前首先要设置下sshd服务，因为默认root用户是不允许远程登录进来了的。可通过修改/etc/ssh/sshd_config文件的配置，如下参数按照对应的值设置，保存退出后重启sshd服务：

```txt
LoginGraceTime 2m
PermitRootLogin yes
StrictModes yes
MaxAuthTries 6
MaxSessions 10
PubkeyAuthentication yes
PermitEmptyPasswords no
PasswordAuthentication yes
```

通常这样设置就可以了，如果追求更高的安全性，可以考虑使用密钥登陆：

 1. 生成密钥对 `ssh-keygen -t rsa -b 4096 -f ECX -C "key for ECX VPS"`
 2. 配置.ssh文件夹下的文件内容及权限
	```bash
	cat ECX.pub >> ~/.ssh/authorized_keys
	chmod 700 ~/.ssh
	chmod 600 ~/.ssh/authorized_keys
	```
 3. 修改服务器ssh配置，取消密码登录
	PasswordAuthentication yes → PasswordAuthentication no
 4. 重启服务 `systemctl restart sshd`

记得把服务器上的密钥对删除，并把私钥拷贝出来，放在xshell或者跳板机的某个地方，然后配置登录。

## 3、系统基本配置

可以远程获取系统管理权之后，第一时间需要对系统做一些基础配置，一来是为了适应自己的使用习惯，二来也是为了系统使用起来更加顺畅。主要包括命令行显示，系统安全与更新，以及其他一些常用的软件安装配置等。

```bash
# 1. 修改PS1以及hostname
echo 'PS1="\e[1;33m[ $? \u@\h \W]\$ \e[0m"' >> .bashrc
hostnamectl set-hostname rustle

# 2. 更新软件源并更新系统
apt update && apt upgrade

# 3. 安装配置防火墙，可以通过ufw -h获取帮助(如下顺序一定不能颠倒，不然你将永远失去你的机器...)
apt install ufw
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable

# 4. 一些其他的软件
apt install lrzsz curl wget gnupg2 ca-certificates lsb-release debian-archive-keyring
```

## 4、编译安装NGINX

Nginx是一款高性能的反向代理服务器，支持高并发是它极具优势的一个方面。可以通过apt/yum等工具直接安装二进制文件，也可以通过源码编译安装，区别在于源码安装可以通过编译定制所需要的模块，升级更加灵活，但是后续配置稍显麻烦，因工作需要，这里选择通过源码安装。

```bash
# 准备工作，安装一些依赖模块
apt install libpcre3 libpcre3-dev openssl libssl-dev zlib1g-dev
# 从nginx官网下载安装包并安装
wget https://nginx.org/download/nginx-1.20.2.zip
unzip -q nginx-1.20.2.zip && cd nginx-1.20.2
./configure --with-http_ssl_module \
--with-http_realip_module \
--with-http_v2_module \
--with-http_gzip_static_module \
--with-stream \
--with-stream_ssl_module
make && make install
ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx

#-------------------------------------------------------------------
Configuration summary
  + using system PCRE library
  + using system OpenSSL library
  + using system zlib library

  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx modules path: "/usr/local/nginx/modules"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"
```

如上是默认安装之后，各模块所在路径，可以通过编译选项指定，详见[官方指导文档](https://nginx.org/en/docs/configure.html)。安装好nginx之后，需要制作一个nginx.service文件，放在/lib/systemd/system/目录下，然后就可以使用`systemctl`命令来管理服务了。

```bash
[Unit]
Description=nginx
After=network.target
  
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
  
[Install]
WantedBy=multi-user.target
```

## 5、开启https

开启https有很多方法，可以直接通过直接购买，也可以通过免费的证书颁发机构申请，还可以自己做个CA，做自签名证书。对于钱包并不是那么鼓的同学来说，第一种方法可能会直接pass掉，做自签名证书也不麻烦，但是可能会出现浏览器依然判定为不安全的风险，因为CA并不是权威的。这里使用的是申请免费的证书，只不过有效期比较短，只有3个月，之后要续订，可以通过crontab服务自动化续订。

- 复杂的方法
	```bash
	apt install snapd
	systemctl enable --now snapd.socket
	ln -s /var/lib/snapd/snap /snap
	snap install core
	snap refresh core
	snap install --classic certbot
	ln -s /snap/bin/certbot /usr/bin/certbot
	# 生成证书
	certbot --nginx-server-root /usr/local/nginx/conf
	# 续订证书
	certbot renew --dry-run
	```
- 简单的方法
	```bash
	apt install python3-certbot-nginx certbot
    # 之所以重新link一遍，是因为如上命令会破坏掉之前的/usr/bin/nginx的软链接，放置一个标准的nginx程序在这个地方
    ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx
	certbot --nginx-server-root /usr/local/nginx/conf
	certbot renew --dry-run
	```

## 6、配置Jekell blog

jekyll blog是一种静态网页生成器，通过markdown文件编译成html文件，极大地简化了blog发布流程，环境配置步骤如下：

```bash
apt install ruby ruby-dev
gem install jekyll bundler
gem source -l
gem source -r https://rubygems.org
gem source --add https://gems.ruby-china.com
# 如下命令有问题，原因可能是Debian ruby版本是2.5，比较低，升级有各种问题，可以通过rvm等管理工具解决
# 但是通过删除Gemfile解决
# gem update --system 3.2.3  
# bundle install
gem install jekyll-paginate
jekyll build
```

## 7、安装nextcloud网盘

最近在自己家中的台式机安装了Debian11.2，之前在Windows下是使用[坚果云](https://www.jianguoyun.com/)来做数据同步盘的，Debian确实也可以安装，只不过只能使用root用户登录。另外数据还是存放在别人服务器的，虽然出于商业道德，我们有理由坚信，坚果云不会窃取我们的数据信息。但是还是想把自己的隐私信息存放在自己的服务器上，所以，我决定再次折腾一下nextcloud网盘。

#### 1. 下载nextcloud服务器版

安装nextcloud网盘也有很多方式，仅仅是官网就提供了三种，也可以使用docker安装，不过这里还是使用最普通的方式构建的，大概还是为了配置灵活，也最能够理解为什么某些配置生效或者不生效吧。安装网盘第一步，下载服务端文件，解压到指定目录：

```bash
wget https://download.nextcloud.com/server/releases/nextcloud-23.0.3.zip
unzip -q nextcloud-23.0.3.zip -d /opt
chown -R www-data:www-data /opt/nextcloud
```

#### 2. 安装并配置mariaDB

```bash
wget -O /usr/share/keyrings/mariadb-archive-keyring.asc https://mariadb.org/mariadb_release_signing_key.asc
echo "deb [signed-by=/usr/share/keyrings/mariadb-archive-keyring.asc] https://mirror-cdn.xtom.com/mariadb/repo/10.6/debian $(lsb_release -sc) main" > /etc/apt/sources.list.d/mariadb.list
apt update && apt upgrade
apt install mariadb-server
mysql_secure_installation
mysql -u root -p
> create database nextcloud;
> create user 'nextcloud'@'localhost' identified by '<new_password>';
> grant all privileges on nextcloud.* to 'nextcloud'@'localhost';
> flush privileges;
> exit
```

#### 3. 安装并配置PHP8

nextcloud推荐使用php8，但是不支持php8.1，只能使用8.0版本！！！由于Debian官方源的php版本不符合要求，所以要使用第三方源，具体的规则可以参考[Debian使用第三方源](https://wiki.debian.org/DebianRepository/UseThirdParty)的规则。

```bash
wget -O /usr/share/keyrings/php-archive-keyring.gpg https://packages.sury.org/php/apt.gpg
echo "deb [signed-by=/usr/share/keyrings/php-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update && apt upgrade
apt install php8.0-fpm php8.0-cli php8.0-mysql php8.0-curl php8.0-gd php8.0-mbstring php8.0-xml php8.0-zip php8.0-imap php8.0-opcache php8.0-soap php8.0-gmp php8.0-bcmath php8.0-intl php8.0-imagick  -y

# 修改/etc/php/8.0/fpm/pool.d/www.conf
clear_env = no  # 取消该条注释
listen.allowed_clients = 127.0.0.1   # 取消该条注释
将listen = /run/php/php8.0-fpm.sock → listen = 127.0.0.1:9000
```

#### 4. 配置NGINX文件并登录使用安装导向

nextcloud默认是支持Apache web服务器的，官方指导文档也是推荐使用Apache作为默认，但是工作需要，使用的是nginx web server，配置上有很大的不同，好在官方给出了一个非官方的[nginx配置示例](https://dev.to/yparam98/nextcloud-setup-with-nginx-2cm1)，可以参考下。配置完了之后重启nginx服务即可，输入指定域名（以及端口）就可以看到安装导向

#### 5. nextcloud问题以及调优

- 上传速度慢
```bash
sudo -u www-data php8.0 /opt/nextcloud/occ config:app:set files max_chunk_size --value 0
```
- 访问速度慢-1
```bash
apt install php8.0-memcache* memcached
'memcache.distributed' => '\OC\Memcache\Memcached',
'memcached_servers' => array(
  array('localhost', 11211),
)
```
- 访问速度慢-2
```bash
apt install php8.0-redis*
'memcache.distributed' => '\OC\Memcache\Redis',
'redis' => array(
'host' => 'localhost',
'port' => 6379,
)
```
- 开启内存访问
```bash
apt install php8.0-apcu
echo 'apc.enable_cli=1' >> /etc/php/8.0/mods-available/apcu.ini
echo 'apc.enable_cli=1' >> /etc/php/8.0/cli/php.ini
echo 'apc.enable_cli=1' >> /etc/php/8.0/fpm/php.ini
```
- /etc/php/8.0/fpm/php.ini 文件调优
```bash
memory_limit = 1024M
post_max_size = 4096M
upload_max_filesize = 4096M
max_execution_time = 3600;
max_input_time = 3600;
```
- /opt/nextcloud/config/config.php 文件调优
```bash
'memcache.local' => '\OC\Memcache\APCu',  # 本地缓存优化
'filelocking.enabled' => true  # 解决网盘文件(夹)无法删除时设置为false，但是检测会报错，建议删除后在改为true
'appstoreenabled' => true,  # 解决appstore无法打开
'appstoreurl' => 'https://www.orcy.net/ncapps/v1/', 
'allow_local_remote_servers' => true,
```
- nginx的配置文件调优
```bash
client_max_body_size 500M;        
client_header_timeout 3600s;
client_body_timeout 3600s;
fastcgi_connect_timeout 3600s;
fastcgi_send_timeout 3600s;
fastcgi_read_timeout 3600s;
# 关闭文件压缩
```
- /etc/php/8.0/fpm/pool.d/www.conf文件调优
```bash
pm.max_spare_servers = 18
pm.min_spare_servers = 6
pm.start_servers = 12
pm.max_children = 120
pm = dynamic
```
- cron定时任务：同时也需要在界面上修改
```bash
crontab -u www-data -e
*/5  *  *  *  * php -f /opt/nextcloud/cron.php
```
- root用户指定www-data执行命令报错
```bash
# sudo: unable to resolve host aliyun: Name or service not known
# 修改/etc/hosts，注意localhost后面的aliyun这个hostname是自定义的
127.0.0.1	localhost aliyun
```
- 手动重新构建用户文件结构
```bash
sudo -u www-data php8.0 occ files:scan --all
```
- 此实例中的 php-imagick 模块不支持 SVG。
```bash
apt install libmagickcore-6.q16-6-extra
```


> **建议所有配置设置完毕之后，重启nginx服务和php-fpm服务**

## 8、参考链接

- [NGINX官网](https://nginx.org/)
- [Nextcloud Setup with Nginx](https://dev.to/yparam98/nextcloud-setup-with-nginx-2cm1)
- [Debian 11 / Ubuntu 20.04 使用源安装 LAMP 教程](https://u.sb/debian-install-apache-php-mysql/)
- [Nextcloud - NGINX configuration](https://docs.nextcloud.com/server/19/admin_manual/installation/nginx.html)
- [Let's Encrypt官网](https://letsencrypt.org/)
- [Certbot 指南](https://certbot.eff.org/)
- [Debian 第三方源使用](https://wiki.debian.org/DebianRepository/UseThirdParty)

