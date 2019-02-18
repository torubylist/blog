---
layout:     post
title:      "工作中遇到的问题--git/docker"
subtitle:   "issues in work"
date:       2019-01-29 21:00:00
author:     "倔强蜗牛"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Git
    - Go
    - Docker
---
# 工作中遇到的问题--git/docker


---

### go项目依赖包不能正确的commit,并push到远端gitlab
将go项目里vendor里面的内容push到远端，发现里面并没有具体的文件，只有一个类似链接的字符串，类似 api@abdalkx这样一个链接，里面没有真实的文件．

解决方案：
删除vendor项目里面的.git/目录．先找出所有的.git/文件．然后删掉．
```cd vendor && find . -name .git | xargs rm -rf ```
再重新添加所有的vendor文件．再push到远端．关注.gitignore,.git文件．


### docker容器报错，standard_init_linux.go:195: exec user process caused "no such file or directory

解决方案：
安装dos2unix
sudo apt install dos2unix

将相关文件转换一下字符即可：
dos2unix Dockerfile

重新build可以正常运行了
