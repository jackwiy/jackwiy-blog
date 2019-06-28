# Docker部署LNMP

在Docker中部署LNMP环境可以分为以下几个步骤：

1. 安装Docker
2. 创建镜像
   1. 创建Dockerfile
   2. build Docerfile
   3. 复制/修改配置文件
3. 运行镜像，并映射端口

为了方便分布式部署，Nginx、PHP、MySQL和Web目录会分别放在4个不同的容器中，最后我们会打包成4个镜像。

### 1 安装docker和docker-compose

具体安装步骤不作说明，详细步骤请参考：[https://docs.docker.com/engine/installation/](https://docs.docker.com/engine/installation/)。

Docker安装要求Linux 3.10以上版本，注意这一点就可以了。

安装之后，用这个命令查看版本：

```text
$ yum install docker-io docker-compose
$ docker -v
```

#### 1.1 镜像仓库

Nginx、PHP、MySQL在Docker官方都提供现成的镜像仓库，这些我们是可以直接使用的，各个仓库地址如下。

* Nginx仓库：[https://hub.docker.com/\_/nginx/](https://hub.docker.com/_/nginx/)
* PHP仓库：[https://hub.docker.com/\_/php/](https://hub.docker.com/_/php/)
* MySQL仓库：[https://hub.docker.com/\_/mysql/](https://hub.docker.com/_/mysql/)

#### 1.2 Dockerfile文件

Dockerfile 文件类似 Makefile，只是前者根据规则部署容器，后者根据规则编译项目。

Dockerfile规则请参考：[https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)。

接下来每个程序的部署都会用到Dockerfile，所以一定要提前了解每个命令的功能。

### 2 部署MySQL

因为安装MySQL相对简单，所以我们从MySQL开始。

从Docker的公共仓库 Dockerhub 下载 MySQL 镜像：

```text
$ docker pull mysql
```

然后启动数据库：

```text
$ docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql
```

就完成了MySQL的安装，接下来我们再在PHP-FPM容器中使用。

### 3 部署Nginx

#### 安装Nginx

首先，从Docker的公共仓库 Dockerhub 下载nginx镜像：

```text
$ docker pull nginx
```

这个命令会下载容器需要的所有部件，并且Docker会对它们进行缓存，所以在运行容器的时候，不需要每次都下载容器镜像。

然后，启动Nginx容器：

```text
$ docker run --name nginx -p 80:80 -d nginx
```

运行成功后，终端会返回容器的ID号，上面的命令中，

* `run`：创建一个新的容器
* `--name`：指定容器的名称（如果留空，docker会自动分配一个名称）
* `-p`：导出容器端口到本地服务器，格式：`-p <local-port>:<container-port>`。在本例中，我们映射容器的80端口到本地服务器的80端口。
* `nginx`：是 Dockerhub 上下载nginx镜像名称（如果本地没有可用的镜像，Docker会自动下载一个）
* `-d`：后台启动。

通过浏览器浏览：http://localhost 就会看到Nginx欢迎界面。接下来可能会用到这几个命令：

```text
$ docker ps -a                       # 查看正在运行的容器
$ docker stop nginx                  # 停止正在运行的容器
$ docker start nginx                 # 启动一个已经停止的容器
$ docker rm nginx                    # 删除容器
```

#### 映射HTML路径

默认情况下，Docker nginx服务器的HTML路径（网站根目录）在容器 **/usr/share/nginx/html** 目录下，现在需要把这个目录映射到本地服务器的 **~/www/html** 目录。在上面命令的基础上加上`-v`参数，具体如下：

```text
$ docker run --name nginx -p 80:80 -d -v ~/www/html:/usr/share/nginx/html nginx
```

 `-v`的参数格式为：`<local-volumes>:<container-volumes>`。

在~/www/html下创建一个 index.html 文件，内容

```text
here is ~/www/html/index.html
```

在浏览器上访问 http://localhost，刷新一下就可以看到新的内容了。

#### 配置 Nginx

Nginx 的强大很大部分体现在配置文件上，对于一些高级的应用来说，自定义 Nginx 非常重要。所以，我们需要把 Nginx 的配置文件复制到本地服务器目录：

```text
$ cd ~/www
$ docker cp nginx:/etc/nginx/conf.d/default.conf default.conf
```

再加一个`-v`参数，把本地的配置文件映射到容器上，在重启容器：

```text
$ docker stop nginx
$ docker rm nginx
$ docker run --name nginx -p 80:80 -v ~/www/html:/usr/share/nginx/html -v ~/www/default.conf:/etc/nginx/conf.d/default.conf -d nginx
```

如果配置文件有修改，需要重启容器生效：

```text
$ docker restart nginx
```

这样就可以直接在本地修改配置文件了。

### 4 部署PHP-FPM

下载 PHP-FPM 镜像：

```text
$ docker pull php:fpm
$ docker run --name php-fpm -p 9000:9000 -d php:fpm
$ cd ~/www
$ docker cp php-fpm:/usr/local/etc/php-fpm.d/www.conf www.conf
$ docker cp php-fpm:/usr/local/etc/php/php.ini-production php.ini
```

在本地服务器修改 php.ini 的内容，设置cgi.fix\_pathinfo=0（要先删除前面的;注释符）：

```text
cgi.fix_pathinfo=0
```

然后重新启动容器：

```text
$ docker stop php-fpm
$ docker rm php-fpm
$ docker run --name php-fpm --link mysql:mysql -v ~/www/html:/var/www/html -v ~/www/www.conf:/usr/local/etc/php-fpm.d/www.conf -v ~/www/php.ini:/usr/local/etc/php/php.ini -d php:fpm
```

这样PHP-FPM的容器就部署完成了。注意，要保证配置文件 php.ini 和 www.conf 没有错误，否则会无法启动容器。

### 5 让Nginx容器支持FPM

打开 nginx 的配置文件，修改内容如下：

```text
server {
    listen       80;
    server_name  _;
    root /usr/share/nginx/html;
    index index.php index.html index.htm;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        #root   /usr/share/nginx/html;
        #index  index.php index.html index.htm;
	try_files $uri $uri/ =404;
    }

    error_page  404  /404.html;
    location = /40x.html {
        root    /user/share/nginx/html;     
    }

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        fastcgi_pass   php-fpm:9000;
        fastcgi_index  index.php;
    #   fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
	    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    location ~ /\.ht {
        deny  all;
    }
}
```

然后启动 nginx：

```text
$ docker run --name nginx -p 80:80 --link php-fpm -v ~/www/html:/usr/share/nginx/html -v ~/www/default.conf:/etc/nginx/conf.d/default.conf -d nginx
```

在 ~/www/html 下创建 index.php 文件，内容：

```text
<?php
    phpinfo();
?>
```

完成！

