# 部署笔记

## 约定

- 系统 CentOS 6.5
- 软件目录 /usr/local/soft
- 软件源码包目录 /usr/local/src
- 数据目录 /data
- 预安装软件 Nginx MySQL php Redis Memcache Sphinx Git Svn Node.js

## 初始

更新系统

`sudo yum update`

安装zsh和oh-my-zsh

```bash
sudo yum install zsh
chsh -s /bin/zsh
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

配置alias(zshconfig)

```bash
alias soft="cd /usr/local/soft"
alias src="cd /usr/local/src"
alias data="cd /data"
```

修改主机名为 dev.com  👌

```bash
sudo vim /etc/sysconfig/hostname
```

DNS配置

```bash
sudo vim /etc/resolv.conf
```

禁用SELinux

```bash
setenforce 0
sed -i 's/^SELINUX=.*$/SELINUX=disabled/' /etc/selinux/config
```

禁用IPv6

```bash
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

## 安装前准备

编译工具及相关库  👌

```bash
sudo -s
LANG=C
yum -y install gcc gcc-c++ autoconf automake cmake zlib zlib-devel compat-libstdc++-33 glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel unzip zip nmap ncurses ncurses-devel sysstat ntp curl curl-devel libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libtiff-devel gd gd-devel libxml2 libxml2-devel libXpm libXpm-devel libmcrypt libmcrypt-devel krb5 krb5-devel libidn libidn-devel openssl openssl-devel openldap openldap-devel nss_ldap openldap-clients openldap-servers pam-devel libicu libicu-devel
```

安装常用工具  👌

```bash
yum -y install man wget iptraf iotop vim-enhanced lrzsz

alias vi='vim'
echo "alias vi='vim'" >> ~/.bashrc
echo "set fencs=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936" >> ~/.vimrc
echo "set fileformats=unix,dos" >> ~/.vimrc

```

安装相关依赖库 👌

```bash
cd /usr/local/src
tar zxvf libiconv-1.14.tar.gz
cd libiconv-1.14
./configure --prefix=/usr/local
make && make install
cd ../

tar zxvf libmcrypt-2.5.8.tar.gz
cd libmcrypt-2.5.8/
./configure && make && make install
/sbin/ldconfig
cd libltdl/
./configure --enable-ltdl-install
make && make install
cd ../../

tar zxvf mhash-0.9.9.9.tar.gz
cd mhash-0.9.9.9/
./configure && make && make install
cd ../

ln -s /usr/local/lib/libmcrypt.la /usr/lib/libmcrypt.la
ln -s /usr/local/lib/libmcrypt.so /usr/lib/libmcrypt.so
ln -s /usr/local/lib/libmcrypt.so.4 /usr/lib/libmcrypt.so.4
ln -s /usr/local/lib/libmcrypt.so.4.4.8 /usr/lib/libmcrypt.so.4.4.8
ln -s /usr/local/lib/libmhash.a /usr/lib/libmhash.a
ln -s /usr/local/lib/libmhash.la /usr/lib/libmhash.la
ln -s /usr/local/lib/libmhash.so /usr/lib/libmhash.so
ln -s /usr/local/lib/libmhash.so.2 /usr/lib/libmhash.so.2
ln -s /usr/local/lib/libmhash.so.2.0.1 /usr/lib/libmhash.so.2.0.1
ln -s /usr/local/bin/libmcrypt-config /usr/bin/libmcrypt-config

tar zxvf mcrypt-2.6.8.tar.gz
cd mcrypt-2.6.8/
/sbin/ldconfig
./configure && make && make install
cd ../

echo "/usr/local/lib" >> /etc/ld.so.conf
/sbin/ldconfig
```

建立www:www用户 👌

```bash
/usr/sbin/groupadd www
/usr/sbin/useradd -g www -s /sbin/nologin www
mkdir -p /data/www
chmod +w /data/www
chown -R www:www /data/www
```



## 安装Nginx

安装PCRE 👌

```bash
cd /usr/src/
tar zxvf pcre-8.40.tar.gz
cd pcre-8.40/
./configure && make && make install
cd ../
```

配置并安装Nginx 👌

```bash
tar zxvf nginx-1.13.1.tar.gz
cd nginx-1.13.1/
/sbin/ldconfig
./configure \
--prefix=/usr/local/soft/nginx \
--user=www \
--group=www \
--with-http_v2_module \
--with-http_sub_module \
--with-http_ssl_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_realip_module \
--with-http_flv_module \
--with-http_stub_status_module \
--with-http_addition_module
make && make install
cd ../
```

启动脚本 👌

```bash
cat > /etc/init.d/nginx <<\EOF
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# nginx:       /usr/local/soft/nginx/sbin/nginx
# config:      /usr/local/soft/nginx/conf/nginx.conf
# pidfile:     /usr/local/soft/nginx/logs/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/local/soft/nginx/sbin/nginx"
prog=$(basename $nginx)

NGINX_CONF_FILE="/usr/local/soft/nginx/conf/nginx.conf"

lockfile=/var/lock/subsys/nginx

make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -z "`grep $user /etc/passwd`" ]; then
       useradd -M -s /bin/nologin $user
   fi
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
       if [ `echo $opt | grep '.*-temp-path'` ]; then
           value=`echo $opt | cut -d "=" -f 2`
           if [ ! -d "$value" ]; then
               # echo "creating" $value
               mkdir -p $value && chown -R $user $value
           fi
       fi
   done
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
EOF
```

开机自启 👌

```bash
chmod +x /etc/init.d/nginx
chkconfig --add nginx
chkconfig nginx on
```

## 安装MySQL

建立mysql用户

```bash
/usr/sbin/groupadd mysql
/usr/sbin/useradd -g mysql -s /sbin/nologin mysql
```

安装mysql

```bash
tar zxf mysql.tar.gz
cd mysql
```

```bash
cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/soft/mysql \
-DMYSQL_DATADIR=/data/mysql \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DMYSQL_TCP_PORT=3306 \
-DMYSQL_USER=mysql \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_ARCHIVE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DENABLE_DOWNLOADS=1

make 
make install
cp /usr/local/soft/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
```

配置文件

```bash
cat > /etc/my.cnf <<\EOF
[client]
port = 3306
socket = /tmp/mysql.sock
#default-character-set = utf8mb4

[mysqld]
port = 3306
socket = /tmp/mysql.sock
basedir = /usr/local/soft/mysql
datadir = /data/mysql
pid-file = /data/mysql/mysql.pid
user = mysql
bind-address = 0.0.0.0
server-id = 1
#init-connect = 'SET NAMES utf8mb4'
#character-set-server = utf8mb4
#skip-name-resolve
#skip-networking

back_log = 300
max_connections = 1000
max_connect_errors = 6000
open_files_limit = 65535
table_open_cache = 128
max_allowed_packet = 4M
binlog_cache_size = 1M
max_heap_table_size = 8M
tmp_table_size = 16M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
sort_buffer_size = 8M
join_buffer_size = 8M
key_buffer_size = 4M
thread_cache_size = 8
query_cache_type = 1
query_cache_size = 8M
query_cache_limit = 2M
ft_min_word_len = 4
log_bin = mysql-bin
binlog_format = mixed
expire_logs_days = 30
log_error = /data/mysql/mysql-error.log
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /data/mysql/mysql-slow.log
performance_schema = 0
explicit_defaults_for_timestamp
#lower_case_table_names = 1
skip-external-locking

default_storage_engine = InnoDB
#default-storage-engine = MyISAM
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 64M
innodb_write_io_threads = 4
innodb_read_io_threads = 4
innodb_thread_concurrency = 0
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 2M
innodb_log_file_size = 32M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120

bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 8M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1
interactive_timeout = 28800
wait_timeout = 28800

[mysqldump]
quick
max_allowed_packet = 16M

[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M
EOF
```

存储目录

```bash
mkdir -p /data/mysql
chown mysql:mysql /data/mysql/
/usr/local/soft/mysql/scripts/mysql_install_db  --user=mysql  --basedir=/usr/local/soft/mysql  --datadir=/data/mysql
```

将MySQL数据库的动态链接库共享至系统链接库

```bash
echo "/usr/local/soft/mysql/lib/" > /etc/ld.so.conf.d/mysql.conf
/sbin/ldconfig
/sbin/ldconfig -v | grep mysql
```

启动mysql

```bash
/etc/init.d/mysqld start
```

root添加密码

```bash
dbrootpwd=123456
/usr/local/soft/mysql/bin/mysql -e "grant all privileges on *.* to root@'127.0.0.1' identified by \"$dbrootpwd\" with grant option;"
/usr/local/soft/mysql/bin/mysql -e "grant all privileges on *.* to root@'localhost' identified by \"$dbrootpwd\" with grant option;"
```

## 安装PHP

创建htdocs目录

```bash
mkdir -p /data/www
chmod +w /data/www
chown -R www:www /data/www
```

安装PHP

```bash
tar zxvf php-5.6.*.tar.gz
cd php-5.6.*/
./configure --prefix=/usr/local/soft/php56 \
--with-config-file-path=/usr/local/soft/php56/etc \
--with-libdir=lib64 \
--enable-fpm \
--with-fpm-user=www \
--with-fpm-group=www \
--enable-mysqlnd \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--enable-opcache \
--enable-pcntl \
--enable-mbstring \
--enable-soap \
--enable-zip \
--enable-calendar \
--enable-bcmath \
--enable-exif \
--enable-ftp \
--enable-intl \
--enable-xml \
--enable-sockets \
--with-xmlrpc \
--with-openssl \
--with-zlib \
--enable-mbregex \
--with-curl \
--with-gd \
--with-zlib-dir=/usr/lib \
--with-png-dir=/usr/lib \
--with-jpeg-dir=/usr/lib \
--with-freetype-dir=/usr/include/freetype2/ \
--with-gettext \
--with-mhash \
--with-mcrypt \
--with-ldap \
--disable-ipv6

make ZEND_EXTRA_LIBS='-liconv' -j `nproc`
make install
```

配置文件

```bash
cp php.ini-production /usr/local/soft/php56/etc/php.ini
cp /usr/local/soft/php56/etc/php-fpm.conf.default /usr/local/soft/php56/etc/php-fpm.conf
```

php-fpm.conf 配置 `vi /usr/local/soft/php56/etc/php-fpm.conf`

```
[global]
pid = run/php-fpm.pid
error_log = log/php-fpm.log
emergency_restart_threshold = 60
emergency_restart_interval = 60
process_control_timeout = 5s
daemonize = yes
rlimit_files = 65535

[www]
user = www
group = www
pm = static
pm.max_children = 384
pm.start_servers = 20
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.process_idle_timeout = 10s
pm.max_requests = 51200
request_terminate_timeout = 0
```

php-fpm 启动脚本

```
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
chkconfig --add php-fpm
chkconfig php-fpm on
```

php扩展安装 //todo

```
wget https://pecl.php.net/get/redis-3.0.0.tgz
tar zxvf redis-3.0.0.tgz
cd redis-3.0.0/
/usr/local/soft/php56/bin/phpize
./configure --with-php-config=/usr/local/soft/php56/bin/php-config
make && make install
```

添加模块  vi /usr/local/webserver/php/etc/php.ini

```
; extension_dir = "ext"
; 在该行下添加如下配置：
extension_dir = "/usr/local/soft/php56/lib/php/extensions/no-debug-non-zts-20131226/"
extension=redis.so
```

安装composer

```Bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
composer config -g repo.packagist composer https://packagist.phpcomposer.com  (中国镜像)
```
