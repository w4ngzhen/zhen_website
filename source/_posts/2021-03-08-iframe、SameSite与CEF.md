---
title: iframe、SameSite与CEF
date: 2021-03-08
tags:
 - CEF
---



# iframe、SameSite与CEF

## 背景

本人使用CEF（或是Chrome）来加载开发的前端页面，其中使用iframe嵌入了第三方页面，在第三方页面中需要发送cookie到后端，然而加载会报错，第三方页面后端无法接受到Cookie。

<!-- more -->

## 原因

由于CEF（Chrome内核）的安全策略，在51版本以前、80版本以后，绝大多数情况下是禁止嵌入的iframe提交Cookie的（下文会列出哪些禁止），所以需要浏览器配置策略来允许iframe提交Cookie，这个策略就是SameSite。

SameSite 属性可以让 Cookie 在跨站请求时不会被发送，从而可以阻止跨站请求伪造攻击（CSRF）。
SameSite 可以有下面三种值：

- **Strict**（严格的）。仅允许一方请求携带 Cookie，即浏览器将只发送相同站点请求的 Cookie，即当前网页 URL 与请求目标 URL 完全一致。
- **Lax**（松懈的）。允许部分第三方请求携带 Cookie。

| 请求类型  |                 示例                 |    正常情况 | Lax         |
| :-------- | :----------------------------------: | ----------: | :---------- |
| 链接      |         `<a href="..."></a>`         | 发送 Cookie | 发送 Cookie |
| 预加载    | `<link rel="prerender" href="..."/>` | 发送 Cookie | 发送 Cookie |
| GET 表单  |  `<form method="GET" action="...">`  | 发送 Cookie | 发送 Cookie |
| POST 表单 | `<form method="POST" action="...">`  | 发送 Cookie | 不发送      |
| iframe    |    `<iframe src="..."></iframe>`     | 发送 Cookie | 不发送      |
| AJAX      |            `$.get("...")`            | 发送 Cookie | 不发送      |
| Image     |          `<img src="...">`           | 发送 Cookie | 不发送      |

- **None**（无）。无论是否跨站都会发送 Cookie。

## 解决方案

### Chrome（或是基于Chromium的Edge）

在基于Chrome中，可以进入如下的页面进行配置：

- 地址栏输入：`chrome://flags/`（Edge中会自动转为`edge://`）
- 找到`SameSite by default cookies`和`Cookies without SameSite must be secure`
- 将上面两项设置为 `Disable`

### CEF

上面的方法很通用，不过，对于CEF项目来说，并没有这个页面供我们配置。我们可以通过命令行形式传入：

```
cef-app.exe（你的cef应用程序） --disable-features=SameSiteByDefaultCookies
```

## 参考

http://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html