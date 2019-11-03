---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

# 一. 背景介绍
绝大部分的互联网应用都在使用HTTP协议，我们知道HTTP协议是一个无状态的协议，所谓无状态指的是多次HTTP请求之间是没有任何关联的，也就是每个请求完全不知道之前请求做了哪些操作。

但是绝大部分情况下，我们都需要Web应用是有状态的，举个例子当我们登录某个Web应用之后我们希望再浏览其它页面的时候就不需要再登录了，否则按照HTTP协议无状态特征，我们需要在每个页面都进行一次登录操作，根本就不可能。

所以，为了保证客户端和服务端之间有状态，每个请求必须带上相关的"标识"。这个标识的目的主要是用于鉴权认证，常见的鉴权方式有以下2种 `基于Cookies` 和 `基于Token`。

# 二. Cookies鉴权
长期以来基于`Cookies`鉴权方式一直是身份认证的默认方式，基于`Cookies`鉴权方式是有状态的，这就意味着我们需要在客户端和服务端保存session记录。服务端需要从内存或者Mysql查找指定的session，同时客户端浏览器需要保存对应的session，基于`Cookies`鉴权常见的流程如下。

<img src="https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-cookie-auth.png?raw=true" width="640" />

1. 用户请求登录服务，输入登录凭证
2. 服务端校验登录凭证，成功后创建session并存储到内存或Mysql，同时在HTTP response Set-Cookie中返回session给客户端
3. 客户端本地设置Cookie保存下session
4. 针对每个请求，客户端通过HTTP Cookie字段把session传给服务端，服务端根据session进行身份认证
5. 当用户登出服务之后，session会过期无效

# 三. Token鉴权
基于`Token`鉴权方式在最近几年由于单体应用、Web API、物联网的兴起越来越流行，有很多种实现Token的方式，但是`JSON Web Tokens (JWTs)`已经成为标准。基于`Token`鉴权方式是无状态的，服务端不再需要保存用户登录的session，同时客户端只需要在每个HTTP header请求带上Token即可，基于`Token`鉴权常见的流程如下。

<img src="https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-token-auth.png?raw=true" width="640" />

# 四. Cookies vs Token
