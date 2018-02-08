---
layout: post
#标题配置
title:  LiveVideo[1]--Thinkphp下配置使用nginx
#时间配置
date:   2018-01-05 11:37:00 +0800
#大类配置
categories:
#小类配置
tag: 教程
---

* content
{:toc}

# 1. Backgroud
使用ThinkPHP重构demo后，访问http://xxx/public/index.php成功，但是对于我自定义的Controller访问失败。
是由于thinkPHP对pathinfo不支持，需要对nginx进行配置

# 2. Solution
配置如下，解决了入口文件隐藏，及自定义的controller访问不到的问题
```
server {
        #        listen 443 ssl ; #E0000 ssl backlog=2048 so_keepalive=300:300:3;
                listen 80;
                server_name             localhost;

        root /data/video/public;
        error_log logs/livedemo.tencentyun.com-error_log;
        access_log logs/livedemo.tencentyun.com-access_log;

        location / {
            index index.htm index.html index.php;
            #使其支持Thinkphp的rewrite模式
            if (!-e $request_filename) {
                rewrite  ^(.*)$  /index.php?s=$1  last;
                    break;
            }
        }

        location ~ .+\.php($|/) {
            fastcgi_index index.php;
            #使其支持thinkphp的pathinfo模式
            fastcgi_split_path_info ^(.+\.php)(.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;            fastcgi_param SCRIPT_NAME $fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_connect_timeout 600s;
            fastcgi_send_timeout 600s;
            fastcgi_read_timeout 600s;
        }
        include fastcgi.conf;

        error_page  404              /404.html;
        location = /40x.html {
        }
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
        }
}
```


