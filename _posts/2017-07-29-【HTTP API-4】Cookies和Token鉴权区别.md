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
2. 服务端校验登录凭证，成功后创建session并存储到内存或Mysql，同时在HTTP response header Set-Cookie字段中返回session给客户端
3. 客户端本地设置Cookie保存下session
4. 针对每个请求，客户端通过HTTP header Cookie字段把session传给服务端，服务端根据session进行身份认证
5. 当用户登出服务之后，session会过期无效

Golang项目使用Cookies鉴权举例
1. 前后端分离，服务端代码使用Golang进行开发，常用的HTTP Server框架选用 [Gin](https://github.com/gin-gonic/gin)
2. 前端一般会需要提供一个登录页面，或者使用统一登录页面，服务端需要提供一个登录接口（一般大公司都会有自己的大账号统一登录系统）
3. 当用户登录之后，服务端会根据用户id生成一个session，并保存在服务端内存中，同时返回给客户端（可以使用github.com/gin-contrib/sessions）
4. 之后客户端发起的任何一个HTTP请求都需要在header里面设置 `Cookie` 字段，值设置为session，服务端接收到之后解析出session，查询内存或数据库获取session对应用户信息

# 三. Token鉴权
基于`Token`鉴权方式在最近几年由于单体应用、Web API、物联网的兴起越来越流行，有很多种实现Token的方式，但是`JSON Web Tokens (JWTs)`已经成为标准。基于`Token`鉴权方式是无状态的，服务端不再需要保存用户登录的session，同时客户端只需要在每个HTTP header请求带上Token即可，基于`Token`鉴权常见的流程如下。

<img src="https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-token-auth.png?raw=true" width="640" />

1. 用户请求登录服务，输入登录凭证
2. 服务端校验登录凭证，成功后返回一个签名的Token
3. 签名Token只需要保存在客户端本地
4. 针对每个请求，客户端通过HTTP Header把Token传给服务端，服务端根据Token进行身份认证
5. 当用户登出服务之后，Token会过期无效

目前用的最多的Token实现方式是 [JSON Web Tokens](https://jwt.io/introduction/)

Golang项目使用Token鉴权举例
1. 前后端分离，服务端代码使用Golang进行开发，常用的HTTP Server框架选用 [Gin](https://github.com/gin-gonic/gin)
2. 前端一般会需要提供一个登录页面，或者使用统一登录页面，服务端需要提供一个登录接口（一般大公司都会有自己的大账号统一登录系统，同时会有统一Token服务）
3. 当用户登录之后，服务端从统一Token服务获取一个token，并把token下发给客户端
4. 之后客户端发起的任何一个HTTP请求都需要在header里面设置 `Authorization` 字段，值为对应的token，服务端接收到之后解析出token，请求token服务进行校验并获取对应用户信息

# 四. Cookies vs Token
1. Token需要经过加密签名，Cookie session则不需要
2. Cookie session是有状态的，需要在客户端和服务端分别存储一份，而Token是无状态的只需要客户端本地存储即可
3. Cookie session只支持某个域名下所有子域名，对跨域不是很方便，而Token则不存在这个问题，只要Token有效就可以


