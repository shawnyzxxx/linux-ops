# WEB服务

## 文章信息

- 原文发布于：2025-10-31 19:43:52
- 标签：1024程序员节、运维、linux、服务器
- 原阅读量：1051

## 文章导读

本文面向刚接触 Linux 运维与 Web 服务部署的读者，从「Web 服务如何工作」的底层逻辑讲起，系统梳理 HTTP/HTTPS、URL、请求与响应、状态码、数据交互格式以及 B/S 架构、前后端分离等基础概念；随后深入 Nginx，讲解其安装启动、配置文件结构、server_name / root / alias / location 等核心指令与匹配规则，以及静态服务、反向代理、负载均衡、动静分离等实战能力；最后通过五个实验（一般实现、修改端口与目录、虚拟主机、访问控制、HTTPS 加密认证）把理论落到命令行。读完后，读者应能独立完成基于 Nginx 的站点搭建、虚拟主机配置与基础安全加固，并掌握常见 Web 服务问题的定位思路。

## 一、Web 服务基本概念

这部分是理解 Web「如何工作」的底层逻辑，是所有后续知识的前提。

### 1. Web 服务核心基础概念

**Web 服务（Web Service）**

本质是基于网络的、可交互的应用程序组件，通过标准化协议（如 HTTP）为其他应用（或前端）提供数据 / 功能接口，实现跨语言、跨平台的资源共享（例如：天气 API、支付接口都属于 Web 服务）。

**客户端（Client）与服务器（Server）**

- 客户端：发起请求的一方，包括浏览器（Chrome、Safari）、手机 App、其他 Web 服务等（例如：用浏览器访问百度，浏览器就是客户端）。
- 服务器：接收并处理请求的一方，是运行 Web 服务的计算机 / 程序（例如：百度的后端服务器集群，负责接收「搜索关键词」请求并返回结果）。

核心关系：遵循「请求 - 响应（Request-Response）模型」——客户端主动发请求，服务器被动处理并返回响应。

**URL（Uniform Resource Locator，统一资源定位符）**

俗称「网址」，是定位 Web 资源（如网页、接口、图片）的唯一地址。格式拆解：

`协议://域名:端口/路径?查询参数#锚点`

示例各部分含义：

- 协议：如 HTTP/HTTPS；
- 域名：服务器的「网络名字」（如 www.baidu.com，对应背后的 IP 地址）；
- 端口：服务器上区分不同服务的「门牌号」（如 HTTP 默认 80，HTTPS 默认 443）；
- 路径：指定服务器上的具体资源位置；
- 查询参数：向服务器传递的额外信息（如 `wd=Web服务` 表示「搜索关键词是 Web 服务」）。

**IP 地址与域名（Domain Name）**

- IP 地址：服务器在网络中的「唯一数字标识」（如 14.215.177.38，是百度的一个 IP），但人类难记。
- 域名：IP 地址的「易记别名」（如 www.baidu.com），通过 DNS（域名系统）将域名解析为 IP，实现「记名字访服务器」。

**静态资源与动态资源**

- 静态资源：内容固定、无需服务器动态生成的文件，如 HTML、CSS、JavaScript、图片（.jpg/.png）、视频等；服务器接收请求后直接返回文件，无需额外处理。
- 动态资源：内容需服务器根据请求参数 / 业务逻辑实时生成，如用户订单页面、搜索结果页；依赖后端语言（Java、Python、Node.js）和数据库处理，返回动态生成的 HTML 或数据。

### 2. Web 通信核心协议

协议是客户端与服务器的「沟通规则」，Web 服务最核心的协议是 HTTP/HTTPS，此外还需了解相关辅助协议。

**HTTP（HyperText Transfer Protocol，超文本传输协议）**

定义客户端与服务器之间「传输超文本（如 HTML、JSON）」的规则，是 Web 服务的「灵魂协议」，基于 TCP/IP 工作（确保数据可靠传输）。

核心特点：

- 无状态：服务器不记住前一次请求的信息，需通过 Cookie/Session 补充；
- 无连接：HTTP/1.1 前，一次请求对应一次连接；HTTP/1.1 后支持「长连接」，减少连接开销。

**HTTPS（HTTP Secure，安全的 HTTP）**

在 HTTP 基础上增加 SSL/TLS 加密层，解决 HTTP「明文传输」的安全问题（防止数据被窃取、篡改）。

核心作用：验证服务器身份（通过 SSL 证书）、加密传输数据（如用户密码、支付信息），现在主流网站（淘宝、微信）均强制使用 HTTPS。

**HTTP 请求（Request）与响应（Response）**

是客户端与服务器交互的「具体内容」，两者均有固定格式。

HTTP 请求：客户端发给服务器的「指令」，包含 3 部分：

- 请求行：请求方法 URL 协议版本；
- 请求头：键值对形式的附加信息（如 User-Agent 表示客户端类型，Cookie 传递用户身份）；
- 请求体：仅 POST/PUT 等方法使用，传递复杂数据（如表单提交的用户名密码、JSON 格式的参数）。

HTTP 响应：服务器返回给客户端的「结果」，包含 3 部分：

- 响应行：协议版本 状态码 状态描述（如 `HTTP/1.1 200 OK`）；
- 响应头：服务器的附加信息（如 Content-Type 表示返回数据的格式，Set-Cookie 设置 Cookie）；
- 响应体：实际返回的数据（如 HTML 页面、JSON 数据、图片二进制流）。

**HTTP 请求方法（Method）**

定义请求的「目的」，常用方法及含义：

| 方法 | 核心作用 |
| --- | --- |
| GET | 获取资源（如查询数据、访问网页） |
| POST | 提交资源（如表单提交、创建数据） |
| PUT | 更新资源（全量更新，如修改用户信息） |
| DELETE | 删除资源（如删除订单） |

GET/POST 区别是高频考点，需重点理解：GET 是「读」，POST 是「写」；GET 参数可见，POST 参数隐藏（但仍需 HTTPS 加密，否则可被抓包）。

**HTTP 状态码**

服务器用数字代码告知客户端「请求处理结果」，按首位数字分为 5 类，常用状态码：

- 2xx（成功）：200 OK（请求成功）、201 Created（资源创建成功）；
- 3xx（重定向）：301 永久重定向（如旧域名跳新域名）、302 临时重定向；
- 4xx（客户端错误）：400 Bad Request（请求参数错误）、401 未授权（需登录）、403 禁止访问（权限不足）、404 Not Found（资源不存在）；
- 5xx（服务器错误）：500 Internal Server Error（服务器代码报错）、503 Service Unavailable（服务器过载 / 维护）。

常见状态码的详细说明：

| 状态码 | 说明 |
| --- | --- |
| 200 OK | 最常见的成功状态码，表示请求已成功，服务器返回了所请求的数据 |
| 204 No Content | 请求成功处理，但服务器不会返回任何实体内容。常用于需要执行操作但无需更新当前页面的场景 |
| 206 Partial Content | 表示服务器成功处理了客户端的部分 GET 请求（即范围请求）。该响应报文中包含由 Content-Range 指定范围的实体内容，常用于大文件下载或断点续传 |
| 301 Moved Permanently | 永久重定向。请求的资源已被永久分配了新的 URI，后续所有请求都应使用响应头中 Location 字段提供的新 URI |
| 302 Found | 临时重定向。请求的资源临时位于另一个 URI 下，客户端本次应使用新的 URI 访问，但后续请求建议仍使用原始 URI |
| 304 Not Modified | 当客户端发起条件 GET 请求（例如请求头中包含 If-Modified-Since 等信息）时，如果资源自指定时间后未被修改，服务器会返回此状态码，告知客户端可继续使用缓存的版本。这有助于节省带宽 |
| 400 Bad Request | 表示请求报文存在语法错误，服务器无法理解。可能是参数格式错误、请求消息体无效等 |
| 401 Unauthorized | 请求需要用户认证。通常意味着需要登录或提供的认证凭证（如用户名密码）无效 |
| 403 Forbidden | 服务器理解了请求，但拒绝执行。与 401 不同，403 通常发生在已认证的用户尝试访问其权限不足的资源时 |
| 404 Not Found | 服务器找不到请求的资源。可能是 URI 错误、资源已被删除等原因造成 |
| 500 Internal Server Error | 最常见的服务器端错误码，表示服务器内部发生了未知错误，无法完成请求。通常是服务器端程序或配置出现问题 |
| 502 Bad Gateway | 当服务器作为网关或代理，试图执行请求时，从上游服务器收到了一个无效的响应 |
| 503 Service Unavailable | 服务器当前由于超载或正在进行停机维护，无法处理请求。这通常是临时状态，服务器返回的响应头中可能包含 Retry-After 字段，提示客户端何时可重试 |
| 504 Gateway Timeout | 服务器作为网关或代理，未能及时从上游服务器收到请求。通常意味着上游服务器无响应 |

> 实际应用提示
>
> 在实际开发和运维中，理解这些状态码有助于快速定位和解决问题。遇到 4xx 错误，应重点检查客户端发出的请求信息；遇到 5xx 错误，则需要排查服务器端的应用服务、数据库连接、负载等状况。

### 3. Web 服务数据交互格式

HTML、XML、JSON、CSS、JavaScript 等语言格式编写的代码或数据，就是 HTTP 协议响应包里面的数据。

| 内容类型（HTTP 运输的「货物」） | 在 HTTP 协议中的角色（如何「打包」和「标识」） | 主要作用 |
| --- | --- | --- |
| HTML | 服务器在 HTTP 响应体中放置 HTML 代码，并在响应头中设置 `Content-Type: text/html`，告诉浏览器「这是一份 HTML 文档，请解析渲染」。显示数据，专注于数据在浏览器中的外观和呈现 | 构建网页的结构和内容，是页面的骨架 |
| CSS | 同样通过响应体传输，响应头为 `Content-Type: text/css`。浏览器会将这些样式规则应用到对应的 HTML 元素上 | 控制网页的视觉表现，如布局、颜色、字体等，负责网页的「颜值」 |
| JavaScript | 获取 JSON 数据，然后执行类似 `document.getElementById("city").innerText = data.city;` 的操作，将数据填充到 HTML 标签中 | 为网页添加交互行为和动态功能，让网页能「动」起来，响应用户操作 |
| JSON | 通过响应体传输，响应头为 `Content-Type: application/json`。负责携带数据：服务器返回 `{"city": "北京", "temp": 25, "weather": "晴"}` | 在现代 Web 开发中，尤其用于 API 接口，在客户端和服务器之间传递结构化的数据 |
| XML | 通过响应体传输，响应头为 `Content-Type: application/xml` 或 `text/xml`。它和 JSON 类似，也是一种用于存储和传输数据的标记语言。传输和存储数据，专注于数据的内容和结构 | 在过去常作为 API 和数据配置文件的数据格式，现在虽然使用减少，但在某些领域仍很常见 |

### 4. Web 服务架构与开发相关概念

理解 Web 服务的「分层结构」和开发中常用的技术组件，是后续学习框架的基础。

**B/S 架构与 C/S 架构**

Web 服务主要基于 B/S 架构（Browser/Server，浏览器 / 服务器），无需安装客户端，通过浏览器即可访问；与之对应的是 C/S 架构（Client/Server，客户端 / 服务器），需安装专用客户端（如 QQ、微信 App）。

核心区别：B/S 架构「轻客户端、重服务器」，维护成本低；C/S 架构体验更好，但需更新客户端。

**前后端分离架构**

现代 Web 服务的主流架构：

- 前端：负责「页面展示」和「用户交互」，使用 HTML/CSS/JavaScript（或 Vue、React 框架）开发，通过 HTTP 请求调用后端接口获取数据；
- 后端：负责「业务逻辑处理」和「数据存储」，仅返回 JSON 数据（不渲染页面），使用 Java（Spring Boot）、Python（Django/Flask）、Node.js（Express）等开发。

优势：前后端独立开发、部署，提高效率（如前端改页面无需动后端，后端升级不影响前端）。

**服务器软件（Web Server）**

接收 HTTP 请求、转发给后端程序的「中间件」，常用软件：

- Nginx：高性能的反向代理服务器，支持负载均衡（将请求分发到多个后端服务器，避免单点故障）、静态资源缓存（加速 HTML/CSS/图片的访问），是生产环境的首选；
- Apache：早期的 Web 服务器，功能稳定但性能不如 Nginx，现在逐渐被 Nginx 替代；
- Tomcat：Java 生态的 Web 服务器，支持运行 Java Web 程序（如 WAR 包），常与 Nginx 配合使用（Nginx 负责转发请求，Tomcat 负责处理 Java 业务）。

## 二、Nginx

### 1. Nginx 基础认知

**什么是 Nginx**

- 定义：一款高性能的 HTTP 和反向代理服务器，也可作为负载均衡器、邮件代理服务器等。
- 特点：轻量级、高并发（支持数万并发连接）、低内存占用、模块化设计、跨平台（Linux/Windows 等）。
- 应用场景：静态资源服务（如 HTML/CSS/JS）、反向代理、负载均衡、动静分离等。

### 2. Nginx 安装与启动

**安装方式**

- 源码编译安装（推荐，可自定义模块）：需掌握 configure 配置、make 编译、make install 安装流程。
- 包管理工具安装（简单，适合快速部署）：如 `yum install nginx`（CentOS）、`apt install nginx`（Ubuntu）。

**基本操作**

启动 / 停止 / 重启、检测配置与查看版本：

```bash
nginx                 # 启动
nginx -s stop         # 强制停止
nginx -s quit         # 优雅停止
nginx -s reload       # 重载配置
nginx -t              # 检测配置文件语法是否错误

systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl enable --now nginx
systemctl disable --now nginx

nginx -v              # 版本号
nginx -V              # 详细版本 + 编译参数
```

### 3. 核心配置文件解析 ⭐

Nginx 的配置文件（默认 /etc/nginx/nginx.conf）是核心，需理解结构和常用指令。

**配置文件的组成部分**

- 主配置文件：nginx.conf；
- 辅助配置文件：`include conf.d/*.conf`。

`/usr/share/nginx/modules/*.conf`：配置影响 Nginx 的核心功能和模块行为。生效范围：这些配置文件中的设置将在 Nginx 启动时加载，并且对整个 Nginx 服务器的操作都有效。它们通常用于定义 Nginx 模块的行为，而不是特定服务或 location 的配置。

`/etc/nginx/conf.d/*.conf`：配置影响特定的虚拟主机和服务配置。这个 include 指令位于 http 块内部，通常用于包含定义了虚拟主机（server blocks）和服务配置的文件。每个 .conf 文件可以定义一个或多个服务器块（server blocks），这些服务器块可以监听不同的端口、处理不同的域名以及提供不同的网站内容。生效范围：这些配置文件中的设置仅对它们各自定义的虚拟主机或服务有效。例如，如果在 /etc/nginx/conf.d/ 目录下有一个名为 example.com.conf 的文件，那么该文件中的配置只会应用于 example.com 这个域名。

**主配置文件结构**

- 全局块：配置影响 Nginx 整体运行的参数（如工作进程数 worker_processes、错误日志路径 error_log）。
- events 块：配置网络连接相关参数（如最大连接数 worker_connections、事件模型 use epoll）。
- http 块：配置 HTTP 服务器的核心参数，包含 server 块（虚拟主机）、upstream 块（负载均衡池）等。
- server 块：定义虚拟主机，可通过 listen（端口）、server_name（域名）区分不同网站。
- location 块：根据 URL 路径匹配请求，配置路由规则、缓存、反向代理等（核心重点）。

```nginx
# main 配置段（全局配置）
user nginx;              # 运行的账户，默认即是 nginx，可以不进行设置
worker_processes auto;   # worker 进程数，根据硬件调整，通常等于 CPU 数量或者 2 倍于 CPU
error_log /var/log/nginx/error.log;   # 错误日志存放目录
pid /run/nginx.pid;     # 指定运行 Nginx master 主进程的 pid 文件存放路径

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;  # 导入的外部文件位置

# events 配置段，性能模块设置，其中可以设置时间处理模式等
events {
    use epoll;     # 使用 epoll 事件驱动模型，但不推荐配置它，让 nginx 自己选择
    worker_connections 1024;   # 每个进程的最大连接数量（并发数）
    accept_mutex on # 默认是 off 关闭的，这里推荐打开
}

# http 配置段，包含全局块和 server 块，使用最频繁的部分，代理、缓存、日志定义等绝大多数功能和第三方模块的配置都在这里设置
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # 以上定义日志格式
    access_log  /var/log/nginx/access.log  main;   # 访问日志存储位置

    sendfile            on;   # 高效文件传输模式，默认开启
    tcp_nopush          on;   # 性能优化参数，数据是否立刻发送
    tcp_nodelay         on;   # 性能优化参数，小数据包是否延迟发送
    keepalive_timeout   65;   # 超时时间
    types_hash_max_size 4096; # 性能优化参数，影响散列表的冲突率

    include             /etc/nginx/mime.types;      # 可解析的静态资源类型
    default_type        application/octet-stream;   # 默认文件类型

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;   # 加载子配置项

    # server 配置段，虚拟主机设置
    server {
        listen       80;        # 监听 IPV4 端口
        listen       [::]:80;   # 监听 IPV6 端口
        server_name  _;         # 访问的域名
        root         /usr/share/nginx/html;    # 网页默认目录

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;   # 子配置文件存储路径

        # location 配置段，一般用来匹配不同的 URI 请求
        location / {
            root   /usr/share/nginx/html;     # 相对路径网站根目录
            alias  /usr/share/nginx/html/;    # 绝对路径网站根目录
            index  index.html index.htm;      # 默认首页文件
            deny 172.168.22.11;       # 禁止访问的 ip 地址，可以为 all
            allow 172.168.33.44;      # 允许访问的 ip 地址，可以为 all
            autoindex  on;           # 开启目录文件列表
            autoindex_exact_size on;   # 显示出文件的确切大小，单位是 bytes
            autoindex_localtime on;    # 显示的文件时间为文件的服务器时间
            charset utf-8,gbk;         # 避免中文乱码
            auth_basic "xxxx";   # 加密网页验证时的提示信息
            auth_basic_user_file   /路径/文件名;   # 加密网页使用的密码验证文件
        }
        error_page 404 /404.html;        # 404 时返回给客户端的页面
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;  # 50x 错误返回给客户端的页面
           location = /50x.html {
        }
    }

# Settings for a TLS enabled server.   # https 虚拟主机定义
#
#    server {
#        listen       443 ssl http2;    # 监听的 IPV4 端口
#        listen       [::]:443 ssl http2;
#        server_name  _;
#        root         /usr/share/nginx/html;   # 网页默认目录
#
#        ssl_certificate "/etc/pki/nginx/server.crt";   # 证书存储路径
#        ssl_certificate_key "/etc/pki/nginx/private/server.key"; # 密钥存储
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers PROFILE=SYSTEM;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }

}
```

**server_name 指令的匹配写法**

server_name 指令一般用于指定虚拟主机的域名，在匹配时有以下四种写法：

- 精确匹配：`server_name http://www.nginx.com ;`
- 左侧匹配：`server_name *.http://nginx.com ;`
- 右侧匹配：`server_name www.nginx.* ;`
- 正则匹配：`server_name ~^www.nginx.*$ ;`

> 注：匹配优先级：精确匹配 > 左侧通配符匹配 > 右侧通配符匹配 > 正则表达式匹配。

**root 指令与 alias 指令的区别**

root 指令用于静态资源目录位置，它可以写在 http、server、location 等配置中，root 指令定义的路径会与 URI 叠加产生最终访问路径，如：

```nginx
location /image {
  root /opt/nginx/static;
}
```

当用户访问 `www.test.com/image/1.png` 时，实际在服务器找的路径是 `/opt/nginx/static/image/1.png`。

alias 也是指定静态资源目录位置，但只能访问指定的绝对路径，不会叠加目录，只能写在 location 中且末尾一定要添加 `/`，如：

```nginx
location /image {
  alias /opt/nginx/static/image/;
}
```

当用户访问 `www.test.com/image/1.png` 时，实际在服务器找的路径是 `/opt/nginx/static/image/1.png`。

**location 匹配路径**

格式：

```nginx
location  [ = | ~ | ~* | ^~ ] uri {
  ...
}
```

匹配规则：

- `=` 精确匹配；
- `~` 正则匹配，区分大小写；
- `~*` 正则匹配，不区分大小写；
- `^~` 匹配以某个字符串开头，不是正则匹配；
- `/` 通用匹配，如果没有其它匹配，任何请求都会匹配到。

> 注：匹配优先级：`=` > `^~` > `~` > `~*` > 不带任何字符。

示例：

```nginx
server {
  listen  80;
  server_name  www.nginx-test.com;

  # 只有当访问 www.nginx-test.com/match_all/ 时才会匹配到 /usr/share/nginx/html/match_all/index.html
  location = /match_all/ {
      root  /usr/share/nginx/html
      index index.html
  }

  # 当访问 www.nginx-test.com/1.jpg 等路径时会去 /usr/share/nginx/images/1.jpg 找对应的资源
  location ~ \.(jpeg|jpg|png|svg)$ {
    root /usr/share/nginx/images;
  }

  # 当访问 www.nginx-test.com/bbs/ 时会匹配上 /usr/share/nginx/html/bbs/index.html
  location ^~ /bbs/ {
    root /usr/share/nginx/html;
    index index.html index.htm;
  }
}
```

注意：

```nginx
location / {
    root html;
    index index.html index.htm;
  }
```

其 `/` 不是根目录，而是统统都匹配到指定路径，而指定路径为 html，即 nginx 的默认网页目录 `/usr/share/nginx/html`。

**URI 中斜线（`/test` 与 `/test/`）的差异**

```nginx
location /test {      # 尾部无 /
    ...
}

location /test/ {
  ...
}
```

- 不带 `/`：当访问 `http://www.nginx-test.com/test` 时，Nginx 先找是否有 test 目录，如果有则找 test 目录下的 index.html；如果没有 test 目录，Nginx 则会找是否有 test 文件。
- 带 `/`：当访问 `http://www.nginx-test.com/test` 时，Nginx 先找是否有 test 目录，如果有则找 test 目录下的 index.html，如果没有它也不会去找是否存在 test 文件。

**常用指令**

- 路径相关：root（静态资源根目录）、index（默认首页）。
- 反向代理：proxy_pass（代理目标地址）、proxy_set_header（修改请求头）。
- 负载均衡：upstream（定义后端服务器池）、weight（权重）、ip_hash（会话保持）。
- 静态资源优化：expires（缓存过期时间）、gzip（压缩开关）。

### 4. 核心功能实践

**静态资源服务**

配置 root 或 alias 指定资源目录，通过 location 匹配 URL 路径，实现 HTML、图片、JS 等文件的访问。

示例：通过 `location /static/ { root /data/; }`，访问 `http://域名/static/1.jpg` 对应 `/data/static/1.jpg`。

**反向代理**

概念：客户端请求 Nginx，Nginx 转发到后端服务器（如 Tomcat、Node.js），再将结果返回客户端（隐藏后端服务器信息）。

配置：`location /api/ { proxy_pass http://127.0.0.1:8080/; }`，将 /api/* 请求代理到本地 8080 端口服务。

**负载均衡**

场景：多台后端服务器分摊请求压力，提高可用性。配置：通过 upstream 定义服务器池，结合 proxy_pass 转发。

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=2;  # 权重 2，接收更多请求
    server 192.168.1.102:8080;            # 默认权重 1
}
server {
    location / { proxy_pass http://backend; }
}
```

常用策略：轮询（默认）、weight（权重）、ip_hash（同一 IP 固定到某台服务器）、fair（按响应时间分配）。

**动静分离**

思路：静态资源（CSS/JS/图片）由 Nginx 直接处理，动态请求（如 Java/Python 接口）代理到后端应用服务器，提高效率。

### 5. 实战与问题排查

**实战练习**

搭建静态网站服务、配置反向代理到本地应用、实现多服务器负载均衡。模拟高并发场景（如用 ab 工具压测：`ab -n 1000 -c 100 http://localhost/`），观察 Nginx 表现。

**问题排查**

- 配置错误：通过 `nginx -t` 检查配置文件语法，查看错误日志定位问题。
- 端口占用：用 `netstat -tulpn | grep 80` / `ps -aux | grep 80` 检查端口是否被占用。
- 访问异常：检查 location 匹配规则、防火墙策略（如 firewalld 是否开放 80/443 端口）。

## 三、实验

> 说明：以下实验多在「关闭防火墙和 setenforce 0」的环境下进行，以便本机与客户端能直接访问服务。

### 实验 1：一般实现（关闭防火墙和 setenforce 0）

```bash
[root@server ~]# yum install nginx -y
[root@server ~]# systemctl start nginx.service
[root@server ~]# systemctl --now disable firewalld.service
[root@server ~]# setenforce 0
# 查看默认目录
[root@server ~]# cat /etc/nginx/nginx.conf | grep root
root /usr/share/nginx/html;
```

### 实验 2：修改端口、默认目录、默认文件访问 web 页面（关闭防火墙和 setenforce 0）

**通过不同的端口访问同一 web 页面**

```bash
[root@server ~]# grep listen /etc/nginx/nginx.conf
listen 80;
listen 666;
[root@server ~]# systemctl restart nginx.service
[root@server ~]# systemctl --now disable firewalld.service
[root@server ~]# setenforce 0
[root@server ~]# netstat -lntup | grep nginx
tcp 0 0 0.0.0.0:666  0.0.0.0:*  LISTEN  5630/nginx: master
tcp 0 0 0.0.0.0:80 0.0.0.0:* LISTEN  5630/nginx: master
tcp6 0 0 :::80 :::* LISTEN 5630/nginx: master

# 测试主机 129
[root@client ~]# curl 192.168.60.128:666
welcome to my test page
[root@client ~]# curl 192.168.60.128:80
welcome to my test page
[root@client ~]# curl 192.168.60.128:123
curl: (7) Failed to connect to 192.168.60.128 port 123: 拒绝连接
```

结论：666 端口和 80 端口可以连接，123 端口不能连接。

**自定义默认目录**

```bash
[root@server ~]# vim /etc/nginx/nginx.conf
[root@server ~]# grep root /etc/nginx/nginx.conf
# root /usr/share/nginx/html;
root /web;
[root@server ~]# mkdir /web
[root@server ~]# systemctl restart nginx.service
[root@server ~]# echo "my new dir is /web" > /web/index.html

# 测试主机 133
[root@client ~]# curl 192.168.60.128:80
my new dir is /web
```

**通过不同的文件访问 web 页面**

```bash
[root@server ~]# vim /etc/nginx/default.d/web.conf
location /haha {
    root /web; # 如果 root 在 location 里面，那么会覆盖 server 模块里的 root
    index haha.html;
    access_log /var/log/nginx/web.log;
    error_log /var/log/nginx/web_erro.log;
}
[root@server ~]# echo "my location is /haha" > /web/haha/haha.html
[root@server ~]# mkdir /web/haha/
[root@server ~]# systemctl restart nginx.service

# 测试主机 129
[root@client ~]# curl 192.168.60.128/haha/
my location is /haha
```

### 实验 3：虚拟主机——基于不同端口、不同目录、不同 IP、不同域名访问不同页面

Nginx 的虚拟主机功能在多种场景下非常有用，典型使用场景：

1. 共享主机服务：服务提供商可以使用 Nginx 的虚拟主机功能在同一台服务器上为多个客户托管不同的网站，每个客户拥有独立的域名和资源，互不干扰。
2. 开发与生产环境分离：在同一台服务器上配置不同的虚拟主机，分别用于开发、测试和生产环境，以便于管理和维护。
3. 多域名单站点：有时一个站点可能需要多个域名访问，例如国际化的网站可能根据不同地区有不同的域名，但指向同一站点。
4. 多站点托管：企业或个人可能需要在同一台服务器上托管多个独立的网站，每个网站有自己独特的配置和资源需求。
5. 子域名管理：通过虚拟主机可以为不同的子域名配置不同的网站或服务，例如 `blog.example.com` 和 `store.example.com` 可以是两个完全独立的网站。

此外还有基于路径的虚拟主机（Nginx 根据请求的 URL 路径将请求转发到不同的应用程序或服务）、HTTPS 和 HTTP 分离（在同一服务器上为同一个站点配置 HTTP 和 HTTPS 的虚拟主机，便于管理安全连接和非安全连接）。

> 注意：配置文件的写法不唯一。`root /data2/up2;` 表示在这个 location 块中，root 指令被设置为 /data2/up2，这将覆盖 server 块中的 root 设置。

```nginx
server {
    listen 192.168.5.123:8080;
    root /data/up1;
    location / {
        root /data2/up2;
    }
}
```

**基于不同端口访问不同页面**

配置 nginx/conf.d/port.conf 文件，重启 nginx 服务：

```bash
[rooteshawn ~]# cd /usr/share/nginx/html
[rooteshawn html]# echo html/ xiade port89.html > port89.html
[rooteshawn html]# curl 192.168.60.128:89
html/ xiade port89.html
[rootashawn html]# echo html/ xiade port99.html > port99.html
[rootashawn html]# curl 192.168.60.128:99
html/ xiade port99.html
```

或者也可以把配置文件写成多个 server 块，同样实现端口访问不同页面：

```bash
[root@server ~]# cat /etc/nginx/conf.d/port.conf
```

```nginx
server {
    listen 80;
    location / {
        root /web/port80;
        index index.html;
    }
}
server {
    listen 666;
    location / {
        root /web/port666;
        index index.html;
    }
}
```

```bash
[root@server ~]# mkdir /web/port{666,80}
[root@server ~]# echo my port is 666 > /web/port666/index.html
[root@server ~]# echo my port is 80 > /web/port80/index.html
[root@server ~]# vim /etc/nginx/nginx.conf
[root@server ~]# systemctl restart nginx.service
```

**基于不同目录访问不同页面（关闭防火墙和 setenforce 0）**

```bash
[root@server ~]# vim /etc/nginx/conf.d/dir.conf
```

```nginx
server {
    listen 80;

    location /dir1/ {
        root /;
        index index.html;
    }

    location /dir2/ {
        root /;
        index index.html;
    }
}
```

```bash
[root@rhce html]# mkdir /dir1
[root@rhce html]# echo "/dir1/  xia de index.h" > /dir1/index.html
[root@rhce html]# mkdir /dir2
[root@rhce html]# echo "/dir2/  xia de index.h" > /dir2/index.html

[root@client ~]# curl 192.168.78.132/dir1/
my location is /dir1
[root@client ~]# curl 192.168.78.132/dir2/
my location is /dir2
```

**基于不同 IP 访问不同页面（关闭防火墙和 setenforce 0）**

服务端 ens128 网卡拥有两个 IP：128 和 129。

```bash
[root@server ~]# nmcli connection modify ens160 +ipv4.addresses 192.168.60.129/24
[root@server ~]# nmcli connection up ens160
连接已成功激活（D-Bus 活动路径：/org/freedesktop/NetworkManager/ActiveConnection/3）
[root@server ~]# mkdir /web/{128,129}
[root@server ~]# echo my ip is 128 > /web/128/index.html
[root@server ~]# echo my ip is 129 > /web/129/index.html
[root@server ~]# vim /etc/nginx/conf.d/ip.conf
```

```nginx
server {
    listen 192.168.60.129:80;
    root /web/129;
    location / {
        index index.html;
    }
}
server {
    listen 192.168.60.128:80;
    root /web/128;
    location / {
        index index.html;
    }
}
```

测试：

```bash
[root@client ~]# curl 192.168.60.129
my ip is 129
[root@client ~]# curl 192.168.60.128
my ip is 128
```

基于不同 IP 访问不同页面的配置文件也可以这样写，效果一样：

```bash
[root@server ~]# vim /etc/nginx/conf.d/ip.conf
```

```nginx
server {
    listen 192.168.78.132:80;
    root /web/132;
}
server {
    listen 192.168.78.123:80;
    root /web/123;
}
```

**基于不同域名访问不同页面（关闭防火墙和 setenforce 0）**

```bash
[root@server ~]# vim /etc/nginx/conf.d/ip.conf
```

```nginx
server {
    listen 80;
    root /web/129;
    server_name xixi.com;
    index   xixi.html;
}
server {
    listen 80;
    server_name haha.com;
    root /web/128;
    index   haha.html;
}
```

```bash
[root@server ~]# systemctl restart nginx.service

[root@rhce ~]# echo "/web/128/haha.html  page"  > /web/128/haha.html
[root@rhce ~]# echo "/web/129/xixi.html  page"  > /web/129/xixi.html

# 测试
[root@client ~]# vim /etc/hosts
192.168.60.129 xixi.com haha.com
[root@client ~]# curl xixi.com
/web/129/xixi.html  page
[root@client ~]# curl haha.com
/web/128/haha.html  page
```

### 实验 4：访问控制

**基于不同用户的访问控制访问同一页面**

```bash
[root@server ~]# yum install httpd-tools.x86_64
[root@server ~]# htpasswd -cb /etc/nginx/conf.d/auth_passwd li4 redhat123
Adding password for user li4
[root@server ~]# htpasswd -c /etc/nginx/conf.d/auth_passwd li4   # 修改密码
[root@server ~]# htpasswd -c /etc/nginx/conf.d/auth_passwd zhang3
[root@server ~]# vim /etc/nginx/conf.d/control.conf
```

```nginx
server {
    listen 192.168.60.129:88;   # 80 端口连接服务过多，建议更换端口
    root /web/129;
    location / {
        auth_basic on;
        auth_basic_user_file /etc/nginx/conf.d/auth_passwd;
    }
}
server {
    listen 192.168.60.128:88;   # 80 端口连接服务过多，建议更换端口
    root /web/128;
    location / {
        auth_basic on;
        auth_basic_user_file /etc/nginx/conf.d/auth_passwd;
    }
}
```

```bash
[root@server ~]# systemctl restart nginx.service
[root@client ~]# curl 192.168.60.129 -u li4
Enter host password for user 'li4':
my ip is 129
[root@client ~]# curl 192.168.60.128 -u li4
Enter host password for user 'li4':
my ip is 128
```

**基于不同的用户认证访问不同的页面**

```bash
[root@server ~]# htpasswd -cb /etc/nginx/conf.d/auth_passwd li4 redhat123
[root@server ~]# htpasswd -cb /etc/nginx/conf.d/auth_passwd2 tom redhat123
[root@server ~]# vim /etc/nginx/conf.d/control.conf
```

```nginx
server {
    listen 192.168.60.128:80
    location / {
        root /web/128;
        auth_basic on;
        auth_basic_user_file /etc/nginx/conf.d/auth_passwd;
    }
}
server {
    listen 192.168.60.129:80;
    location / {
        root /web/129;
        auth_basic on;
        auth_basic_user_file /etc/nginx/conf.d/auth_passwd2;
    }
}
```

```bash
[root@server ~]# systemctl restart nginx.service
```

**基于源 IP 的访问控制（可加在前一个实验）**

```nginx
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    # 基于源端口的访问控制
    location /test.html {
        root /usr/share/nginx/html;
        allow 192.168.1.0/24;   # 允许来自 192.168.1.0/24 网段的请求
        deny all;               # 拒绝其他所有请求
    }
}
```

关于 allow / deny 的说明：

- 如果只有 `allow 192.168.1.0/24;`，表示允许所有 IP，包括 192.168.1.0/24；
- 如果要实现允许 192.168.1.0/24，并且拒绝其他的 IP 访问服务器，需要同时写：

```nginx
allow 192.168.1.0/24;   # 允许来自 192.168.1.0/24 网段的请求
deny all;
```

- 表示拒绝所有 IP 访问服务器，此时 `allow 192.168.5.1;` 配置不会生效：

```nginx
deny all;
allow 192.168.5.1;
```

### 实验 5：使用 nginx 的 http_ssl 模块建立加密认证网站

**查看模块**

```bash
[root@server ~]# nginx -V
# 看是否有 --with-http_ssl_module 模块，否则需要安装 mod_ssl
```

**配置文件与证书文件**

- 证书文件：/……/xxxx.crt
- 私钥文件：/……/xxxx.key

**ssl 配置文件的主要参数**

```bash
[root@server ~]# vim /etc/nginx/nginx.conf   # 主配置文件 https 段，默认为注释，可以取消注释
```

```nginx
# Settings for a TLS enabled server.
server {
    listen       443 ssl http2;    # 监听 443 端口
    listen       [::]:443 ssl http2;
    server_name  _;                  # 域名
    root         /usr/share/nginx/html;    # 网页默认

    ssl_certificate "/etc/pki/nginx/server.crt";   # 证书路径
    ssl_certificate_key "/etc/pki/nginx/private/server.key";  # 私钥文件路径
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

#### 实验 5-1：搭建 nginx + ssl 的加密认证 web 服务器

（https 功能由 ngx_http_module 模块提供）

**第一步：准备工作**

```bash
[root@server ~]# setenforce 0

[root@server ~]# systemctl stop firewalld

[root@server ~]# systemctl disable firewalld

[root@server ~]# yum install nginx mod_ssl -y

[root@server ~]# systemctl start nginx   # 启动

[root@server ~]# systemctl enable nginx  # 设置开机启动
```

**第二步：生成私钥**

```bash
[root@server certs]# cd /etc/pki/tls/certs/
[root@server certs]# openssl genrsa -out miyao.key
[root@server certs]# ll
-rw-------. 1 root root 1704 8月 28 14:33 miyao.key
```

**第三步：制作证书**

```bash
[root@server ~]# openssl req -utf8 -new -key /etc/pki/tls/certs/miyao.key -x509 \
-days 100 -out /zhengshu.crt
```

**第四步：制作配置 nginx**

```bash
[root@server ~]# echo i am port 443 > /web/index.html
[root@localhost ~]# vim /etc/nginx/conf.d/https.conf
```

```nginx
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    root         /web/https;
    ssl_certificate "/etc/nginx/conf.d/zhengshu.crt";
    ssl_certificate_key "/etc/nginx/conf.d/miyao.key";
}
```

```bash
[root@server certs]# systemctl restart nginx.service
```

**证书申请与 CA 制作过程**

生成私钥 → 生成 CSR（包含公钥和身份信息） → 将 CSR 提交给 CA → CA 验证后用自己的私钥签名，生成最终证书。

**HTTPS 证书的申请过程**

- 选择证书服务商：挑选可信赖的证书服务商，如 DigiCert、GlobalSign、Sectigo 等国际知名品牌，以及安信证书等国产品牌。
- 确定证书类型：根据需求选择，如 DV 证书适合个人网站，OV 证书适合企业官网，EV 证书适合金融等高安全要求网站。
- 生成 CSR 和私钥：使用 OpenSSL 等工具在服务器上生成 CSR 文件和私钥。例如，在 Apache 服务器上可通过命令「`openssl req -new -newkey rsa:2048 -nodes -keyout yourdomain.key -out yourdomain.csr`」生成。
- 提交申请与验证：将 CSR 文件提交给 CA 机构。DV 证书可通过邮箱验证、DNS 验证或文件验证等方式确认域名所有权；OV 和 EV 证书则需提交企业营业执照等资料进行人工审核。
- 等待审核并下载证书：DV 证书验证通常较快，可能几分钟内完成；OV 和 EV 证书一般需要 1-5 个工作日。审核通过后，从 CA 机构获取证书文件。
- 证书安装与部署：根据服务器类型，如 Apache、Nginx、IIS 等，按照相应的步骤安装证书，确保证书正确配置。

**CA 机构制作证书的过程**

- 创建 CA 私钥和根证书：CA 机构首先使用 OpenSSL 等工具生成 CA 私钥，如通过命令「`openssl genpkey -algorithm RSA -out ca.key -aes256`」生成加密的 RSA 私钥。然后基于该私钥创建自签名的根证书，命令为「`openssl req -key ca.key -new -x509 -out ca.crt -days 3650`」，此根证书具有签发其他证书的能力。
- 接收并验证 CSR：CA 机构接收来自申请者的 CSR 文件，其中包含申请者的公钥和相关信息。根据证书类型，对申请者进行身份验证，如 DV 证书验证域名所有权，OV 和 EV 证书验证企业信息。
- 签发证书：验证通过后，CA 机构使用自己的根证书私钥对申请者的 CSR 进行签名。例如，使用命令「`openssl ca -in client.csr -out client.crt -cert ca.crt -keyfile ca.key -days 365`」生成客户端证书，该证书包含了申请者的公钥、CA 的签名等信息，至此证书制作完成。

> 关键参数 `-x509` 的作用：直接生成「自签名证书」，跳过「生成 CSR → CA 签名」的步骤。在自签名场景中，你自己就是「CA」，不需要向第三方申请，因此无需生成 CSR。这条命令会直接基于私钥（zy.key）生成包含公钥和身份信息的证书（zy.crt），并使用你的私钥对证书进行签名（即「自签名」）。
