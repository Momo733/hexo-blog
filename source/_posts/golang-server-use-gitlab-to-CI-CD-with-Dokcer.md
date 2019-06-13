---
title: golang服务集成docker实现DevOps
date: 2019-05-28 15:39:23
tags: 
 - golang 
 - docker 
 - gitlab CI/CD
 - DevOps
---

本文主要是在对于我们的golang项目，使用官方包管理构建二进制文件，然后docker构建程序那个运行的镜像，最后部署到远程服务器运行的一些心得。
## 1.配置运行CI/CD的运行环境
这是针对于使用Gitlab的CI/CD服务，主要是方便避免重复性的部署和编译工作，使用Gitlab只需要在自己的文件目录下面添加``.gitlab-ci.yml``文件即可，然后配置``gitlab-runner``，按照相应的官方文档步骤进行配置即可。小提示，在配置runner的时候需要配置``runner tag``我们可以直接回车填空，就可用让这个runner运行多个项目，否则是需要在自己的项目中添加tags,或者在yml文件中添加tags,如果您的要求专属允许一个项目添加tag是一个很好的选择。
```
deploy:
 stage: deploy
 tags:
  -mk_service
 script: 
  - echo "Running deploy docker image"
 only:
  - dev
```
当我们代码提交的时候，gitlab会把我们的代码copy到一个临时的文件夹里面，我们在script里面执行响应的shell命令就达到我们想要的效果。

## 2.构建golang服务的docker镜像
现在说一下我的具体执行构建golang的项目自动化集成和部署。
构建golang的二进制文件，我们可以使用在runner服务器上安装golang包，这样是可以直接在script里面直接``go build ``构建二进制文件，但是如果golang的版本更换亦或者我们只需要编译某个版本的的文件，那么我们就需要安装各种版本的golang包，这样会极其的麻烦，我想这时候docker会是一个很好的选择。

使用doker镜像来构建，我们只需要指定对应的golang版本，最后使用scratch或者是alpine来包裹我们的二进制文件，这样我们的程序是可以在任何安装docker镜像的机器上允许，简直完美。

这里说一下scratch和alpine两个docker镜像的一些优缺点，以及我所遇到的坑。

scratch是一个空瓶镜像，这是官方的说明，任何安装docker的机器上都是带有这个镜像，这个镜像的大小是0B，其实就是一个能够运行二进制的空瓶子，里面bash什么的都是没有的，如果您不是对于镜像大小有着极致的追求，非常不建议您使用这个镜像，不经需要自己手动安装一堆的依赖包，而且后续的调试也是非常的不方便。

alpine是一个轻型的Linux发行版，基础的命令是有的，包含基础的目录结构，使用这个镜像来构建程序的运行环境是一个不错的选择，毕竟也不是很大，当然如果您没有要求的话，我想Ubuntu等热门的发行版也许是一个更好的选择。

```dockerfile
FROM golang:1.12 as builder
WORKDIR /web/cache
ADD  go.mod .
ADD  go.sum .
ENV GOPROXY https://goproxy.io
ENV GO111MODULE on
RUN go mod download
WORKDIR /web/maoka
COPY . .
ENV GOCACHE /web/buildcache
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64
RUN go build -ldflags "-s -w" -o mk_service main.go

FROM alpine:latest as product
ENV TIME_ZONE Asia/Shanghai
RUN apk --no-cache add ca-certificates
RUN apk add tzdata
RUN apk add bash
RUN apk add libc6-compat
RUN apk add libstdc++
WORKDIR  /maoka
COPY --from=builder /web/maoka/mk_service /maoka
ADD authorityinit.json .
RUN chmod +x mk_service
ENTRYPOINT ./mk_service
EXPOSE 7333

```
使用上例子的dockerfile您会构建您的程序镜像。
因为官方的包管理实在太香，我在自己的项目中使用的module来进行包管理，去掉了大量的目录依赖。

但是去掉本地的包，因为墙的原因导致一些包不能下载，或者速度太慢，这时候就需要使用使用最新的GOPROXY环境变量来作为``go get ``代理了，``https://goproxy.io``使用上手的时候速度非常快，其他的还有一些go的代理，使用过一些，效果并不太好。

这样我们就能过愉快的使用golang的包管理来构建我们的程序了，但是使用的时候你会发现，每次都需要取下载依赖，时间也是非常之长，一个小项目差不多四五分钟的下载时间，对于我们构建程序来说，简直是不能忍受，如果能够把我们的依赖缓存下来就好了，对的我们可以设置在使用``go mod ``情况下的缓存，直接设置``WORKDIR /web/cache``会自动创建目录在这个镜像中，如果go.mod文件没有修改，则是会启动缓存来构建我们的程序，速度很快。

那么设置了依赖的缓存，会加速我们的依赖，但是我的项目非常大，我build一个二进制文件的时间还是非常长怎么办，有没有什么办法能够加速构建？

其实在golang中就已经给我定义了一个环境变量GOCACHE，会把我们build的缓存放在里面，我们只需要指定对应的路径就可以了。

使用dockerfile多级构建镜像的时候，会存在一个拷贝镜像的问题，我们可以使用``COPY --from``来指定目录，这样就不存在各种文件找不到，丢失的问题了。

另外还要说一下，在运行64位二进制的文件的时候，执行的时候会报错not found，我们使用file 命令去查看我们的二进制，就会发现在linux上运行的是需要lib64这个依赖包的，而在Alpine中这个是没有的所以需要我们自己去添加依赖``RUN apk add libc6-compat``和`` RUN apk add libstdc++``，这就是之前我建议构建包裹镜像，如果要求不严格，最好使用Ubuntu等发行版的原因。

## 3.添加私密文件
经常能够听到某项目中的数据库密码泄露，其实是因为直接把私密信息写到代码中，是最快捷的方式，因为各种原因最后导致泄漏，为了以防万一，我们必须得保证项目代码泄露的可能性，所以一个统一的密钥管理是非常有必要的，下面将给出几种参考方法。
### 1.构建统一的敏感数据服务
专门为所有的服务提供密钥，这种服务系统一般构建在内网之中，登录的时候有一个统一的权限认证，可以说这是一种非常系统的解决方案，针对特别大的系统有很好的帮助，但是这种系统通常情况下也是需要大量的时间去构建，特别小的项目，时间紧，就不是一个很好的选择。
### 2.使用非对称加密
非对称加密的算法有许多，经典的是RSA，公钥加密，私钥解密，只要私钥不泄露，可以说很安全。简陋的方法是服务器存储私钥，本地使用公钥，最后运行代码的过程中解密，得到想要的数据。
### 3.配合Gitlab的变量
Gitlab本身有一套权限的代码管理，我们把私密文件放置在runner的环境变量中，在运行的时候，直接构建到我们的项目中，这种是我所采用的方法，比较简单方便。

**注意**：在使用docker构建镜像的过程中，不能直接使用``$KEY``来作为参数，
```
docker build -t myapp --build-arg ISDEV=true --build-arg DATAKEY=$KEY .
```
最后会直接报错``docker build" requires exactly 1 argument``，解决方法是把DATAKEY作为环境变量直接赋值。
```
export  DATAKEY=$KEY
docker build -t myapp --build-arg ISDEV=true --build-arg DATAKEY .
```


## 4.部署镜像到多个服务器
构建完成docker镜像，当然是需要发布到我们私人的docker hub，相当于是一次数据的备份。

使用``docker login -u $CI_DEPLOY_USER -p $CI_DEPLOY_PASSWORD $REGISTRY ``来登录到我们私人的docker hub ,当然是先需要在gitlab中配置以上的一些私有的信息变量的。

在我们push 镜像的时候，我们是需要给我们的镜像构建一个tag，这样很容易我们去区别镜像，如果不构建tag那么就会直接把我们之前的镜像给覆盖，如果还想要恢复到之前的版本的话，那么就欲哭无泪了。

那么在这种的情况下，我们就需要每次构建一个不一样的参数去作为tag，那么我们有什么选择：

1.使用``git commit``的sha来作为提交的tag，这肯定是不相同的，但是这一长串，很难得到具体的信息，还得去查``git log``的具体commitID。
2.使用``git tag`` 我们可以为程序提交git tag 这样gitlab会把这个作为变量设置成参数``$CI_COMMIT_TAG``,但是这样你就需要每次自己去打tag，这样一个小提交修改，操作会有点繁琐。
3.使用``export COMMIT_TIME=$(git show -s --format=%ct $CI_COMMIT_SHA)``，把当前提交的时间戳作为tag,这样非常便于我们清晰的知道这个镜像的一些信息。

如果在runner的本地部署我们的镜像，那就非常的简单，我们只需要使用``docker run``来运行镜像就可以了。当然为了最好第一次是手动启动镜像，这样我们提交代码时候触发的时候，直接执行``docker stop &&docker rm``就不会报错了。

部署到远程的服务器，就需要我们首先登陆远程的服务器。[Using SSH keys with GitLab CI/CD](https://docs.gitlab.com/ee/ci/ssh_keys/README.html)这篇官方文章引导如何使用ssh登陆远程服务器。

```
deploy_prod:
  stage: deploy
  before_script:
    - export COMMIT_TIME=$(git show -s --format=%ct $CI_COMMIT_SHA)
    - eval $(ssh-agent -s)
    - echo "$CD_HOSTS_KEY" | tr -d '\r' | ssh-add - > /dev/null
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan $CD_HOSTS >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - chmod +x deploy.sh
  script:
    - ssh  root@$CD_HOSTS "bash -s" < ./deploy.sh $CI_DEPLOY_USER $CI_DEPLOY_PASSWORD $COMMIT_TIME
    - echo "Running deploy success"
  only:
    - master
```
上述的一些关键的密钥信息最好配置在gitlab的项目runner里面。使用ssh 的``bash -s``用于接收输入流的命令。

因为在deploy.sh 有commit_time是runner环境中才有的，所以必须使用这种输入流的方式，如果你没有这种需求，只需要使用下面的这种方式就好。
```
ssh  root@$CD_HOSTS "docker pull image && docker run image && exit" 
```

接下来的当然是和在runner本地部署镜像相同的操作了，只是需要提前登陆，然后拉取镜像。