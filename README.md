cat /etc/os-release
PRETTY_NAME="Astra Linux"
NAME="Astra Linux"
ID=astra
ID_LIKE=debian
ANSI_COLOR="1;31"
HOME_URL="https://astralinux.ru"
SUPPORT_URL="https://astralinux.ru/support"
LOGO=astra
VERSION_ID=1.7_x86-64
VERSION_CODENAME=1.7_x86-64
VARIANT_ID=se

uname -m
x86_64

command -v php
/usr/bin/php

 php -v
PHP 8.1.12-1ubuntu4.3.astra2 (cli) (built: Sep 26 2025 01:43:38) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.12, Copyright (c) Zend Technologies
    with Zend OPcache v8.1.12-1ubuntu4.3.astra2, Copyright (c), by Zend Technologies

php --ini
Configuration File (php.ini) Path: /etc/php/8.1/cli
Loaded Configuration File:         /etc/php/8.1/cli/php.ini
Scan for additional .ini files in: /etc/php/8.1/cli/conf.d
Additional .ini files parsed:      /etc/php/8.1/cli/conf.d/10-mysqlnd.ini,
/etc/php/8.1/cli/conf.d/10-opcache.ini,
/etc/php/8.1/cli/conf.d/10-pdo.ini,
/etc/php/8.1/cli/conf.d/15-xml.ini,
/etc/php/8.1/cli/conf.d/20-calendar.ini,
/etc/php/8.1/cli/conf.d/20-ctype.ini,
/etc/php/8.1/cli/conf.d/20-curl.ini,
/etc/php/8.1/cli/conf.d/20-dom.ini,
/etc/php/8.1/cli/conf.d/20-exif.ini,
/etc/php/8.1/cli/conf.d/20-ffi.ini,
/etc/php/8.1/cli/conf.d/20-fileinfo.ini,
/etc/php/8.1/cli/conf.d/20-ftp.ini,
/etc/php/8.1/cli/conf.d/20-gd.ini,
/etc/php/8.1/cli/conf.d/20-gettext.ini,
/etc/php/8.1/cli/conf.d/20-iconv.ini,
/etc/php/8.1/cli/conf.d/20-ldap.ini,
/etc/php/8.1/cli/conf.d/20-mbstring.ini,
/etc/php/8.1/cli/conf.d/20-mysqli.ini,
/etc/php/8.1/cli/conf.d/20-pdo_mysql.ini,
/etc/php/8.1/cli/conf.d/20-pdo_pgsql.ini,
/etc/php/8.1/cli/conf.d/20-pgsql.ini,
/etc/php/8.1/cli/conf.d/20-phar.ini,
/etc/php/8.1/cli/conf.d/20-posix.ini,
/etc/php/8.1/cli/conf.d/20-readline.ini,
/etc/php/8.1/cli/conf.d/20-shmop.ini,
/etc/php/8.1/cli/conf.d/20-simplexml.ini,
/etc/php/8.1/cli/conf.d/20-sockets.ini,
/etc/php/8.1/cli/conf.d/20-sysvmsg.ini,
/etc/php/8.1/cli/conf.d/20-sysvsem.ini,
/etc/php/8.1/cli/conf.d/20-sysvshm.ini,
/etc/php/8.1/cli/conf.d/20-tokenizer.ini,
/etc/php/8.1/cli/conf.d/20-xmlreader.ini,
/etc/php/8.1/cli/conf.d/20-xmlwriter.ini,
/etc/php/8.1/cli/conf.d/20-xsl.ini,
/etc/php/8.1/cli/conf.d/20-zip.ini,
/etc/php/8.1/cli/conf.d/~bx.ini

php -m
[PHP Modules]
calendar
Core
ctype
curl
date
dom
exif
FFI
fileinfo
filter
ftp
gd
gettext
hash
iconv
json
ldap
libxml
mbstring
mysqli
mysqlnd
openssl
pcntl
pcre
PDO
pdo_mysql
pdo_pgsql
pgsql
Phar
posix
readline
Reflection
session
shmop
SimpleXML
sockets
sodium
SPL
standard
sysvmsg
sysvsem
sysvshm
tokenizer
xml
xmlreader
xmlwriter
xsl
Zend OPcache
zip
zlib

[Zend Modules]
Zend OPcache

 free -h
              total        used        free      shared  buff/cache   available
Mem:          7,7Gi       1,1Gi       618Mi       247Mi       6,1Gi       6,2Gi
Swap:         1,0Gi       3,0Mi       1,0Gi

 df -h
Файловая система Размер Использовано  Дост Использовано% Cмонтировано в
udev               3,9G            0  3,9G            0% /dev
tmpfs              794M          79M  716M           10% /run
/dev/sda1          147G          22G  118G           16% /
tmpfs              3,9G          16K  3,9G            1% /dev/shm
tmpfs              5,0M            0  5,0M            0% /run/lock
tmpfs              794M            0  794M            0% /run/user/999
tmpfs              794M            0  794M            0% /run/user/1000

systemctl list-units --type=service --all --no-pager | grep -Ei 'php|nginx|angie|apache|httpd|mysql|maryadb|postgres|redis|memcached'
  angie.service                               loaded    active   running Angie - high performance web server
  php8.1-fpm.service                          loaded    active   running The PHP 8.1 FastCGI Process Manager
  postgresql.service                          loaded    active   exited  PostgreSQL RDBMS
  postgresql@11-main.service                  loaded    active   running PostgreSQL Cluster 11-main
  redis-server.service                        loaded    active   running Advanced key-value store

