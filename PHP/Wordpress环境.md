## 安装MySQL

版本：5.7.23

#### 数据初始化

```shell
./mysqld --initialize --user=${USER_NAME} --basedir=${MYSQL_ROOT} --datadir=${DATA_ROOT}
```

#### 增加配置文件

添加`my.cnf`到`${MYSQL_ROOT}`

内容：

```shell
[mysqld]
basedir = ${MYSQL_ROOT}
datadir = ${DATA_ROOT}
port = ${MYSQL_PORT}
socket = ${SOCK_FILE}
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

[mysql ] 
socket = ${SOCK_FILE}
```

#### 添加sock的软连接

```shell
ln ${SOCK_FILE} /tmp/mysql.sock
```

## 安装PHP

版本：7.2.11

#### 前置环境

##### cURL

```shell
sudo apt-get install curl
```

##### curl-config

```shell
sudo apt-get install libcurl4-gnutls-dev librtmp-dev
```

#### 编译配置

```shell
./configure --prefix=${PHP_ROOT} --enable-fpm --with-mysqli --with-pdo_mysql --with-zlib --with-curl
```

#### 编译安装

```sehll
make && make install
```

## 安装Wordpress

版本：4.9.4-zh_CN

解压就可以了

## 安装nginx

版本：1.15.5

#### 编译配置

```shell
./configure --prefix=${NGINX_ROOT}
```

#### 编译安装

```sehll
make && make install
```

#### 配置

修改`${NGINX_ROOT}/conf/nginx.cnf`

```shell
server {
        listen       5300;
        server_name  localhost;

        root /home/zhangyd/Software/wordpress;
        index index.php index.html index.htm;

        server_name your_domain.com;

        location / {
                try_files $uri $uri/ /index.php?q=$uri&$args;
        }

        error_page 404 /404.html;

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
                root /usr/share/nginx/html;
        }

        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
```

## 启动

1. #### MySQL

```shell
${MYSQL_ROOT}/support-files/./mysql.server start 
```

2. #### php-fpm

```shell
${PHP_ROOT}/sbin/php-fpm
```

3. ### nginx

```shell
${NGINX_ROOT}/sbin/nginx
```

## Wordpress配置

### 主题

#### WordStart

### 插件

#### SyntaxHighlighter Evolved

#### Markdown Editor

##### 修改代码

这个插件的代码块解析有点问题，会将HTML字符转化成实体，例如`"`会转化成实体`&quot;`，与上面那个语法高亮的插件冲突，最后代码显示出来就是`&quot;`，修改源码可以解决这个问题。

180行左右的`do_codeblock_preserve`方法

```php
public function do_codeblock_preserve( $matches ) {
	$block = stripslashes( $matches[3] );
 	// $block = esc_html( $block ); // 注释这一行
	$block = str_replace( '\\', '\\\\', $block );
	$open = $matches[1] . $matches[2] . "\n";
	return $open . $block . $matches[4];
}
```