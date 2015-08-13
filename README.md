使用php官方image為底，多安裝composer和laravel, php extension需安裝zip
http://logdown.com/account/posts/290718-docker-php-laravel/edit

```
docker build -t image-name .
```
Dockerfile:
```
FROM php:5.6-fpm
# Install modules
RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng12-dev \
        wget \ 
        vim \
    && docker-php-ext-install iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd \
    && docker-php-ext-install zip \
    && docker-php-ext-install mbstring \

    # ........................
    && apt-get clean \
    && apt-get autoclean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \

    # .. Composer.... PHP ...........
    && curl -sS https://getcomposer.org/installer \
        | php -- --install-dir=/usr/local/bin --filename=composer \

    # install laravel
    && composer global require "laravel/installer=~1.1"

#COPY profile /etc/profile

ENV PATH ~/.composer/vendor/bin:$PATH
```

__nginx和php分別在不同的container中執行，不使用--link而透過本機串連，方便抽換php版本__
run container時分別掛入nginx的*.conf和php.ini
```
docker run --restart=always --name nginx -v ~/nginx_setting/:/etc/nginx/conf.d/:ro -v /etc/localtime:/etc/localtime:ro -p 80:80 -e NGINX_SITE_ROOT=/usr/share/nginx/html -v /var/www:/usr/share/nginx/html -d nginx
```
```
docker run --restart=always --name php56fpm -v /etc/localtime:/etc/localtime:ro -v /var/www:/works/www -v ~/php_setting/php.ini:/usr/local/etc/php/php.ini -p 9000:9000 -d php56_laravel
```
進入php container建立新的laravel project
掛source code路徑時沒有加:ro(readonly)，所以在container中先建立laravel專案後會同步到本機的/var/www內，之後在外部編輯程式即可
```
docker exec -ti php56_laravel bash
```
```
cd /works/www
laravel new project_name
```

備註:附上掛入的設定檔
nginx default.conf
```
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;

        # try to serve file directly, fallback to app.php
        #try_files $uri /index.php$is_args$args;
    }

    #error_page  404              /404.html;

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
    location ~ \.php$ {
        root           /usr/share/nginx/html;  #nginx container的路徑
        fastcgi_pass   172.17.42.1:9000;       #本機的ip
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /works/www/$fastcgi_script_name; #/var/www為php container的路徑
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

若要讓整個domain都指到laravel，使用下述nginx config
若有需要可以使用多個virtualhost切換 server_name取不同名稱，並修改自己本機的/etc/hosts讓電腦可以辨識
```
server {
    listen       80;
    server_name  laravel-host;

    root   /usr/share/nginx/html/bien_testing/public;
    index  index.php;

    location / {
        # try to serve file directly, fallback to app.php
        try_files $uri $uri/ /index.php?$query_string;
    }

    #error_page  404              /404.html;

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
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   172.17.42.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /works/www/bien_testing/public/$fastcgi_script_name;
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```
