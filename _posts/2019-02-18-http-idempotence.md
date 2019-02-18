---
layout:     post
title:      "HTTP Idempotence"
subtitle:   "http restapi"
date:       2019-02-18 21:00:00
author:     "倔强蜗牛"
header-img: "img/http-restapi.jpg"
tags:
    - http
    - idempotent
    - restapi
---

---

### http restapi幂等性研究
#### What is idempotence
> Idempotence is the property of certain operations in mathematics and computer science whereby they can be applied multiple times without changing the result beyond the initial application. From [Wiki](https://en.wikipedia.org/wiki/Idempotence)

大意就是幂等性指的是，数学或者计算科学领域里面某些操作，除初次操作以外的相同操作并不会改变被操作对象的状态．简而言之，就是多次操作和一次操作的结果是一致的．例如数学里面的加0或者乘１就是幂等的．

#### HTTP Verbs
RESTAPI操作包括GET, POST, HEAD, PUT, OPTIONS, DELETE, TRACE.
1. POST 不是幂等的．
2. GET,HEAD, PUT, OPTIONS, DEDETE, TRACE是幂等的．

##### POST
POST是指新增实例，每次操作都会新增一个或多个实例，改变了对象的数据，所以不是幂等的．

##### GET, HEAD, PUT, OPTIONS, DELETE, TRACE
1. GET，HEAD, TRACE, OPTIONS, 多次GET跟一次GET对被操作的对象不会有改变．故幂等．
2. DELETE，一次DELETE操作就删除了该实例，再次操作的时候，该实例已经被删除，故不会产生其他影响，虽然这两次操作的返回值一个是200或204,一个是404，但是该操作是幂等的，因为最终的对象并没有因为多次delete而删除其他实例．
3. PUT，PUT操作是对实例的更新，如果实例不存在，则新增实例，如果实例已存在，则更新该实例到目标状态．这个目标状态跟新增的实例状态是一致的．故是幂等的．

###　综述
我们讨论幂等性的时候只要抓住一个要点，就是目标对象的状态是否因为多次操作和一次操作会有不同．如果不同，则非幂等，反之，则幂等．



