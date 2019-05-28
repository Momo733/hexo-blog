---
title: nginx部署golang服务
date: 2019-05-28 16:06:33
tags: nginx
---

第一点Tomcat的主要是对于动态的请求的支持更加的好些，但是对于静态文件就不是很好，主要是用于配置一些javaweb的项目，而我的这个blog方便前后端分离，便于编写实现分工，实现各自的内容，利于代码的观赏和整洁，防止代码的臃肿。

第二点在于Tomcat的配置对于我来说比较繁琐，没有nginx更加利于更新修复。

nginx的优点主要是在于轻量，对于这种纯粹的RESTful服务可以隐藏服务的端口，使用反向代理，对于静态文件的支持很好。
 
nginx的所有HTTP请求都是在HTTP模块下面，可以在这里面配置server模块实现拦截请求然后转移到对应的资源或者端口服务，

### 1.使用源安装
nginx的配置文件是nginx.conf,需要看安装的方式，对于不同的源安装，nginx的目录文件是位于不同的目录下面的。我使用的是ubuntu使用过两种安装方式一种是用:
```
apt-get install nginx
```
这种方式安装是从ubuntu的软件源进行安装，这里面的安装的配置文件是位于/etc/nginx/目录下面，而配置的静态文件html的root目录一般都是位于/usr/share/nginx/目录下面。

对于nginx的文件位于的具体目录在于你使用安装的时候，安装的信息最好是详细的看一下，不然会找对应的文件位置很久，浪费时间，具体的安装方式，还有安装依赖库我就不详细解说了。

### 2.使用官网文件安装
第一种的安装方式很方便，但是我为什么还是使用了第二种的安装方式呢，主要是在于第一种的安装是会把nginx相关的配置文件更改了，并不是最原始的方式，对于新手来说很不好理解具体的对应的模块的功能和含义。所以才有了第二种的安装方式:
```
wget http://nginx.org/download/nginx-1.6.2.tar.gz
```
使用这种的安装方式最后所有的配置文件和nginx程序都是位于/usr/lcoal/nginx目录下面，如果没有这个目录，那么也是位于你的linux的用户目录下面（不是你解压这个包的目录下面，解压这个包只是为了安装），这样使用官网的原始版本利于自己的理解各个模块的意义也利于自己的配置，具体的安装细节在这里不细讲。

### 3.配置nginx的nginx.conf文件
安装完成nginx之后，在浏览器可以直接使用自己的ip进行访问自己的一个网站模型了。

默认的nginx的配置是server监听80的端口，location是指向nginx默认的html里面的index.html文件，在进行动静分离的时候，只需要把来时客户端的请求进行识别匹配，让这个请求访问相应的资源路径就可以了。

对于静态的html文件，我们需要使用正则表达式来匹配，因为一个网页需要加载的内容种类实在很多，就需要使用正则了，方便准确。

location的后面加上~就是使用正则开始来匹配请求的资源路径，在这之后就直接使用正则来匹配了。

而对于动态的api访问接口，我们就需要配置通用的路由路径，这样的话包含这个路径的资源自然都会走这个nginx定义的路径来访问。

例如在下面我就是使用了/api/来控制api的访问路径，包含api的路径就会走location里面的代理设置,使用proxy_pass来配置自己的需要代理的资源地址和端口，如果需要配置多个资源的路径可以使用：
```
upstream blog {
    server http://127.0.0.1:7333  weight=1;
    server http://127.0.0.1:7334  weight=2;
}
location / {
    proxy_pass http://blog;
}
```
weight是代表的权值，值越大就nginx就会把对于资源的访问请求多给这个server，这也是实现负载均衡的一个很好的办法，减少对应的服务器的资源访问压力。

下面是一个配置nginx实现动静分离的文件配置，nginx直接反向代理到本地的服务器端口7333，实现对web的动态访问。

其余的一些参数是对于请求头的转换给对应的资源服务，和直接访问服务的请求相同的效果，但是实现上更加的安全。
```
server {
    listen 80;
    server_name localhost;
    location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css|mp3|mp4|svg|eot|ttf|woff|woff2|JPG)$ {
		root html;
	}
	
    location /api/ {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_pass http://127.0.0.1:7333;
      }
}
```