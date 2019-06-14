---
layout:     post
title:      "xcrun错误"
subtitle:   " \"xcrun错误\""
date:       2019-06-14 20:00:00
author:     "会飞的蜗牛"
header-img: "img/post-bg-2015.jpg"
tags:
    - xcode
    - mac

    
---

今天升级了macOS High Sierra，终端里面make build编译operator的时候，弹出一行莫名其妙的错误：


> xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun

解决方法，重装xcode command line,会比较慢，不要着急。

```
xcode-select --install
```