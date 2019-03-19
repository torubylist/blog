---
layout:     post
title:      "详细解读Dockerfile CMD和ENTRYPOINT"
subtitle:   "Dockerfile CMD ENTRYPOINT"
date:       2019-03-19 21:00:00
author:     "倔强蜗牛"
header-img: "img/post-bg-docker.png"
tags:
    - Dockerfile
    - CMD
    - ENTRYPOINT
---

我们都知道CMD指令和ENTRYPOINT指令的作用都是为镜像指定容器启动后的命令，那他们的缺别大吗？为什么需要两个启动命令呢？

### CMD使用方式
- CMD  ["executable", "param1", "param2"]　使用exec执行，推荐方式
- CMD command param1 param2　在/bin/sh中执行，提供给需要交互的应用
- CMD ["param1", "param2"] 给ENTRYPOINT提供默认参数
每个Dockerfile只能有一条CMD命令生效，如果指定了多条，只有最后一条生效．

### ENTRYPOINT使用方式
- ENTRYPOINT ["executable", "param1", "param2"]
- ENTRYPOINT command param1 param2
每个Dockerfile只有一条ENTRYPOINT生效，当存在多个时，只有最后一条生效

#### 共同点
1. 都可以指定shell或者exec函数调用的方式执行命令
2. 当存在多个CMD或者ENTRYPOINT时候，只有最后一条生效

#### 差异
1. CMD指定的容器启动命令可以被docker run指定的命令覆盖，而ENTRYPOINT指定的命令则不会被覆盖．而是将docker run指定的参数当作是ENTRYPOINT指定命令的参数．
2. CMD指令可以为ENTRYPOINT指令设置默认参数，而且可以被docker run指定的参数覆盖．

###差异测试1
> CMD指令指定的容器启动时命令可以被docker run指定的命令覆盖，而ENTRYPOINT则不会，而是将docker run的命令当作entrypoint的参数．

#####通过CMD方式启动容器
```
cat startuparg.sh 
#!/bin/bash

echo "In Startup, args: $@"
```
```
cat Dockerfile
FROM ubuntu:14.04
MAINTAINER yongping@xxx.com

ADD startuparg.sh .
RUN chmod a+x ./startuparg.sh

CMD ["./startuparg.sh"]
```
build 镜像```docker built -t test:live -f Dockerfile .```
run: 
```
docker run -it -rm=true test:live
In Startup, args:

docker run -ti --rm=true test /bin/bash -c 'echo Hello'
hello
```
#####　通过ENTRYPOINT方式启动容器
```
cat ENTRYDockerfile
FROM ubuntu:14.04
MAINTAINER yongping@xxx.com

ADD startuparg.sh .
RUN chmod a+x ./startuparg.sh

ENTRYPOINT ["./startuparg.sh"]
```
build 镜像```docker built -t entrytest:live -f ENTRYDockerfile .```
run: 
```
docker run -it -rm=true entrytest:live
In Startup, args:

docker run -ti --rm=true entrytest:live /bin/bash -c 'echo Hello'
In Startup, args: /bin/bash -c echo hello
```
### 差异测试２
> CMD指令可以为ENTRYPOINT指令设置默认参数，而且可以被docker run指定的参数覆盖．

CMDENTRYDockerfile
```
FROM ubuntu:14.04
MAINTAINER yongping@xxx.com

ADD startuparg.sh .
RUN chmod a+x ./startuparg.sh

ENTRYPOINT ["./startuparg.sh", "arg1"]
CMD ["arg2"]
```
build 镜像```docker built -t cmdentrytest:live -f CMDENTRYDockerfile .```
run: 
```
docker run -it -rm=true cmdentrytest:live
In Startup, args: arg1 arg2

docker run -ti --rm=true cmdentrytest:live arg3
In Startup, args: arg1 arg3
```
### Note
> CMD指令为ENTRYPOINT指令提供默认参数是基于镜像层次结构生效的，而不是基于是否在同个Dockerfile文件中，换句话说，如果Dockerfile指定的基础镜像中是ENTRYPOINT指定的启动命令，那么该Dockerfile中的CMD依然是为基础镜像中的ENTRYPOINT设置默认参数．

CMDENTRYBASEDockerfile
```
FROM cmdentrytest:live
MAINTAINER yongping@xxx.com

CMD ["/bin/bash", "-c", "echo in cmd"]
```
build 镜像```docker built -t cmdentrybasetest:live -f CMDENTRYBASEDockerfile .```
run: 
```
//cmd变成了基础镜像的参数
docker run -it -rm=true cmdentrybasetest:live
In Startup, args: arg1 /bin/bash -c echo in cmd

//cmd参数被覆盖
docker run -ti --rm=true cmdentrybasetest:live arg3
In Startup, args: arg1 arg3
```