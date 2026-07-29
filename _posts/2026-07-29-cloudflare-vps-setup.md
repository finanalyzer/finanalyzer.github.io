---
layout: post
title: "用自己的小主机和Cloudflare搭建自己的私人云服"
date: 2026-07-29 12:00:00 +0800
categories: [文档]
tags: [fintech, cloudflare]
---

> **摘要**：本文以 PassXYZ、Finanalyzer 和 Vibe-Trading 三个真实开源项目为例，完整演示如何利用 Cloudflare Tunnel + Cloudflare Access，在无公网 IP、不开放任何端口的前提下，将跑在 Mac mini 或树莓派上的多个私有服务安全地部署为带零信任认证的私有云。文章涵盖 Nginx 路径路由配置、Cloudflare Tunnel 创建与路由规则、Cloudflare Access 零信任应用配置（人类访问走邮箱/PIN、机器调用走 Service Token），以及完整的测试验证步骤。通过这套方案，你只需一台常开电脑 + 一个域名 + 免费 Cloudflare 账号，即可拥有一个安全、免费、无需暴露源站 IP 的个人云服务。


## 前言：为什么会需要这套方案

如果你也维护过自己的个人云服务——密码库、仪表盘、交易机器人、笔记系统——大概都卡在同样几个问题上：

- 家里宽带没有公网 IP，或者运营商封了 80/443 端口；
- 在路由器上做端口转发，等于把源站 IP 直接暴露给全世界，担心被扫描、被攻击；
- HTTPS 证书麻烦，续期更麻烦；
- 多个应用各有各的端口，对外一长串 `:8081`、`:8899`，既不优雅也不安全。

我的开源项目 PassXYZ / Finanalyzer 和港大的开源项目 Vibe-Trading 正好就是这样一组服务：一个 .NET 后端、两个前端、还有一个交易 Agent。最终我选了 Cloudflare 全家桶，原因很简单：**免费、不用动路由器、源站 IP 完全隐藏、还能在边缘做身份认证。**

这篇文章把整套流程拆成 7 步，你照抄即可。

---

## 一、你需要准备的三样东西

| 准备项 | 说明 |
| --- | --- |
| 一台常开的电脑 | Mac mini、树莓派（Raspberry Pi 4/5，ARM64）、或者任意一台 7×24 开机的 Linux 小主机都可以 |
| 一个域名 | 在任意注册商处购买（如 `yourdomain.com`），并把 DNS 解析托管到 Cloudflare |
| 一个 Cloudflare 账号 | 免费版（Free plan）就够用，零成本 |

把域名的 Nameserver 改成 Cloudflare 提供的地址后，这个域名下的所有解析就由 Cloudflare 接管了。后面我们既不在这里做 A 记录指向家庭 IP，也不开放任何端口——这正是关键。

---

## 二、我们要部署的 5 个服务

下面是本文要上线的服务清单。所有请求都先经过 Cloudflare Tunnel 到达本机的 **Nginx（端口 80）**，再由 Nginx 根据路径把流量分发到各个后端服务。这种架构让 Cloudflare Tunnel 的配置变得极为简洁——只需一条指向 Nginx 的映射规则。

| 外部路径                    | Nginx 转发目标                 | 应用                  | 内部端口  | 作用                           |
| ---------------------- | ------------------------- | ------------------- | ---- | ---------------------------- |
| `/api/*`              | `proxy_pass` → localhost:5182 | **PassXYZ.Server**  | 5182 | 统一后端 API（密码库 + 金融数据）         |
| `/vault/*`            | 静态文件 `/usr/share/nginx/html/vault` | **passxyz-web**     | — | 兼容 KeePass 的网页密码管理器          |
| `/app/*`              | 静态文件 `/usr/share/nginx/html/app` | **finanalyzer-app** | — | 基于 OpenBB Workspace 的金融分析工作台 |
| `/`（根路径）          | `proxy_pass` → localhost:8899 | **Vibe-Trading**    | 8899 | 金融交易 / Agent 服务（网站入口）        |
| `agent.yourdomain.com` | `proxy_pass` → localhost:8899 | **Vibe-Trading**    | 8899 | 同一服务的 API 出口，供其它应用调用         |

### PassXYZ.Server — 后端 API

这是整个系统的核心后端，基于 ASP.NET Core 10 构建，监听 5182 端口。它承担三类职责：密码库管理（兼容 KeePass 的加密数据库读写）、用户认证（JWT 签发与校验）、以及 Dashboard 和 Portfolio 的数据存储。数据层用 SQLite，按用户隔离——每个用户的密码库文件和仪表盘数据库都独立存储在 `data/` 目录下，用户名经过 Base58 编码后作为文件名，避免特殊字符在不同操作系统上的兼容问题。

认证体系是三层的：Cloudflare Access 负责身份验证（确认你是谁），本地 JWT 负责会话管理（保持登录状态），主密码负责数据解密（打开 KeePass 数据库）。这意味着即使 JWT 被窃取，没有主密码也无法解密密码库中的数据。

### passxyz-web — 密码管理前端

React 18 SPA应用，构建后的静态文件部署到 Nginx 的 `/vault` 路径下。它是 KeePass 密码库的 Web 前端，支持条目的增删改查、自定义字段、Markdown 笔记、附件管理，以及实时 TOTP 验证码生成。登录成功后，JWT 会写入 `localStorage` 的 `passxyz-token` 键，同域下的其他应用可以直接读取这个 token 来发送认证请求。

### finanalyzer-app — 金融分析仪表盘

基于 OpenBB Workspace 架构的金融分析平台，构建后的静态文件部署到 Nginx 的 `/app` 路径下。它采用"核心应用 + 后端注册 Widget"的架构：核心只提供运行环境，所有业务功能（投资组合管理、交易记录）都通过后端注册的 Widget 组件实现。由于和 passxyz-web 共用同一个域名，它能自动读取 `localStorage` 中的 `passxyz-token`，无需用户二次登录。

### Vibe-Trading — AI 交易助手

运行在 8899 端口的 AI 金融分析，回测和交易服务。它通过根路径（`yourdomain`）提供 Web 界面，同时通过 `agent.yourdomain` 子域名暴露 API 接口。这个子域名配置了独立的 client id 和 secret，允许 OpenBB Workspace 等外部应用通过 OAuth 方式接入，复用 Vibe-Trading 的 AI 分析能力。这种设计把"面向用户的 Web 界面"和"面向程序的 API 接口"分到了不同的域名上，便于分别控制访问策略。

---

## 三、安装 Nginx 和 Cloudflare Tunnel

### 安装 Nginx

Nginx 是本机的统一入口，负责路径路由和静态文件托管。它把所有请求收口到一个端口，再分发到各个后端服务。

**在 Mac 上安装：**

```bash
brew install nginx
# 配置文件位于 /opt/homebrew/etc/nginx/nginx.conf（Apple Silicon）或 /usr/local/etc/nginx/nginx.conf（Intel）
```

**在树莓派 / Linux 上安装：**

```bash
sudo apt install nginx
# 配置文件位于 /etc/nginx/nginx.conf
```

### 安装 cloudflared

Cloudflared 是跑在你那台电脑上的代理，它的唯一职责是：**主动向外连接 Cloudflare，建立一条加密隧道**。因为连接是"由内向外"发起的，所以你不需要在防火墙上开任何入站端口。

**在 Mac 上安装：**

```bash
brew install cloudflared
```

**在树莓派 / Linux（ARM64）上安装：**

```bash
# 下载官方二进制
wget -O cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64
sudo install -m 755 cloudflared /usr/local/bin/cloudflared
```

---

## 四、配置 Nginx：用路径路由，而不是用端口

这是整套方案里最巧妙的一步。我们把"不同的服务"映射到"同一个域名的不同的路径"，全部由 Nginx 统一分发。好处有两个：

1. **对外只有一个域名、标准 443 端口**，浏览器和调用方都觉得很干净；
2. **所有服务同源（same-origin）**。前端调 `/api`、调 `/app` 都是相对路径，浏览器不会触发跨域（CORS）限制——你再也不用手忙脚乱地配 `Access-Control-Allow-Origin` 了。

以下是一份完整的 Nginx 配置示例（`nginx.conf` 中的 server 块）：

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # API 请求 → 转发到 PassXYZ.Server 后端
    location /api/ {
        proxy_pass http://127.0.0.1:5182/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 支持 WebSocket（用于 SSE 和实时推送）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 300s;
    }

    # 网页密码管理器（静态文件 SPA）
    location /vault/ {
        alias /usr/share/nginx/html/vault/;
        try_files $uri $uri/ /vault/index.html;

        # 静态资源长期缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2|woff|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # 金融分析工作台（静态文件 SPA）
    location /app/ {
        alias /usr/share/nginx/html/app/;
        try_files $uri $uri/ /app/index.html;

        # 静态资源长期缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2|woff|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # 根路径 → Vibe-Trading 网站
    location / {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> **说明**：`passxyz-web` 与 `finanalyzer-app` 的构建产物分别放到 `/usr/share/nginx/html/vault/` 和 `/usr/share/nginx/html/app/` 目录下，Nginx 直接用 `alias` 指令做静态文件托管。每个 SPA 的 `try_files` 回退到各自路径下的 `index.html`，保证前端路由正常工作。`proxy_pass` 末尾的 `/` 会自动剥离 location 前缀（例如 `/api/users` 转发到后端变成 `/users`），无需额外 rewrite。

**关于 `agent.yourdomain.com` 的处理：**

`agent.yourdomain.com` 是给**机器**用的 API 出口——我们稍后会为它单独签发一组 Cloudflare Access **Service Token（Client ID + Client Secret）**，让 OpenBB Workspace 这类外部应用能带着这两个值来调用，而不走人类登录流程。一套服务，两种入口，互不干扰。

**测试 Nginx 配置并启动：**

```bash
# 检查配置语法
nginx -t

# 重新加载配置（无需停机）
nginx -s reload
```

本地的 `curl http://localhost/vault/`、`curl http://localhost/app/` 能正常返回，就说明 Nginx 路由没问题。

---

## 五、创建 Cloudflare Tunnel 并配置路由

### 在 Cloudflare Dashboard 创建 Tunnel

现在我们通过 Cloudflare 控制台来创建和配置 Tunnel，**无需编辑任何本地的 `config.yml` 文件**。

1. 登录 [Cloudflare Zero Trust 控制台](https://one.dash.cloudflare.com/)
2. 进入 **Networks → Tunnels → Create a tunnel**
3. 给 Tunnel 起个名字（如 `home-server`），选择 **Cloudflared** 类型
4. 点击 **Save tunnel**

### 添加公网路由规则

创建 Tunnel 后，在 **Public Hostname** 标签页中添加路由规则。由于 Nginx 已经做了路径分发，Tunnel 只需要把域名流量转发到 Nginx 即可：

| 公网主机名（Hostname）              | 路径（Path） | Service                  |
| --------------------------------- | ---------- | ----------------------- |
| `yourdomain.com`                  | 留空（所有路径）  | `http://localhost:80`     |
| `agent.yourdomain.com`            | 留空（所有路径）  | `http://localhost:8899`   |

就这么简单——只有两条规则。所有的路径分发工作都交给了 Nginx，Cloudflare Tunnel 只负责"把外面的流量安全地送进来"。

> **为什么 `agent.yourdomain.com` 不经过 Nginx 而是直接指向 8899？** 因为这个子域名专门给机器调用使用，不需要 Nginx 的路径路由功能，直连更简洁。如果你希望它也经过 Nginx，可以改为 `http://localhost:80`，然后在 Nginx 中为 `agent.yourdomain.com` 配置独立的 server 块。

### 在本机安装并启动 cloudflared

回到 Cloudflare Dashboard 的 Tunnel 详情页，会看到安装 cloudflared 的命令。**在 Mac 上执行：**

```bash
brew install cloudflared && sudo cloudflared service install <your-token>
```

`<your-token>` 是 Dashboard 上显示的一串 Base64 编码的认证令牌，cloudflared 会用它自动完成 Tunnel 的关联和认证——不需要手动创建隧道、不需要编辑 config.yml、不需要处理证书文件。

**在 Linux 上安装并启动：**

```bash
# 先安装 cloudflared 二进制
sudo cloudflared service install <your-token>
```

安装完成后，cloudflared 会作为系统服务自动运行。可以在 Tunnel 详情页的 **Health** 标签页看到连接状态变为 **HEALTHY**，说明隧道已经通了。

---

## 六、创建 Cloudflare Access 应用，给所有服务加一道"门"

Tunnel 解决了"能连进来"，但还没解决"谁能进来"。如果到此为止，任何知道你域名的人都能摸到后端。所以我们要在 Cloudflare 边缘加一层**零信任身份认证（Cloudflare Access）**：未通过认证的人，流量根本到不了你的电脑。

进入 **Cloudflare Zero Trust 控制台 → Access → Applications → Add an application → Self-hosted**，创建两个应用：

### 应用 A：保护人类访问的主域名

- **Application host**：`yourdomain.com`
- **Path**：留空（保护该域名下所有路径），或按需填写 `/vault`、`/app`、`/api`
- **Policies → Add a policy**：
  - **Action**：Allow
  - **Rule**：个人使用最方便的是 **One-time PIN**（邮箱收一次性验证码）；也可以选 Emails / Email domain，只允许你自己的邮箱通过
- 保存后，访问 `https://yourdomain.com/vault/` 会先跳到 Cloudflare 登录页，验证通过才放行。

### 应用 B：保护机器调用的 agent 子域名

- **Application host**：`agent.yourdomain.com`
- 这里**不要**用邮箱/ PIN 策略，而是切到 **Service Tokens** 标签页，点击 **Generate service token**，生成一对：
  - `CF-Access-Client-Id`
  - `CF-Access-Client-Secret`
- 再到 Policies 里加一条规则：**Allow → Service Token → 选择刚生成的 token**。

这样，只有携带正确 `CF-Access-Client-Id` / `CF-Access-Client-Secret` 的请求才能访问 `agent.yourdomain.com`。在 OpenBB Workspace 或 Finanalyzer 的"连接配置"里填入这两个值（对应 `CF-Access-Client-Id` / `CF-Access-Client-Secret` 请求头）即可完成对接。

> 后端如何信任 Cloudflare 的身份？以 PassXYZ.Server 为例，它在 `CloudflareAccessMiddleware` 中读取 Cloudflare 注入的请求头 `Cf-Access-Identity-Email`，把邮箱写入上下文；随后 `JwtAuthenticationMiddleware` 在校验该身份后签发应用自身的本地 JWT。也就是说，认证分两层：**Cloudflare Access 回答"你是谁"，本地 JWT 回答"本次会话是否合法"**。开发环境下可通过 `appsettings.Development.json` 的 `Cloudflare:AccessEnabled:false` 临时关闭，方便本地调试。

---

## 七、测试你的服务

隧道和 Access 都就位后，逐项验证。

**1）Nginx 本地路由（先确认本机没问题）**

```bash
# 密码管理前端
curl -I http://localhost/vault/

# 金融分析工作台
curl -I http://localhost/app/

# 后端 API
curl -I http://localhost/api/

# Vibe-Trading
curl -I http://localhost/
```

本机全部正常返回后，再测试 Cloudflare Tunnel 链路。

**2）隧道连通性（命令行）**

```bash
# 根域名应返回 Vibe-Trading
curl -I https://yourdomain.com

# 后端 API（未带 Access 凭证会被拦在 401）
curl -I https://yourdomain.com/api
```

**3）浏览器访问**

打开 `https://yourdomain.com/vault/`，应当先看到 Cloudflare 的一次性验证码 / 邮箱登录页；登录成功后进入 PassXYZ 密码库登录界面。切换到 `https://yourdomain.com/app/` 应能打开 Finanalyzer 分析工作台——由于同源，它调用 `/api` 时浏览器不会报 CORS 错误。

**4）机器调用 agent 子域名**

```bash
curl https://agent.yourdomain.com/some-api \
  -H "CF-Access-Client-Id: <你的 Client ID>" \
  -H "CF-Access-Client-Secret: <你的 Client Secret>"
```

返回正常 JSON 即说明 Service Token 鉴权链路打通。

**5）一个极易踩的坑：清缓存**

前端是静态文件，部署后如果浏览器或 Cloudflare CDN 还缓存着旧的 404，你会以为是配置错了。每次更新发布后，**务必到 Cloudflare 控制台 → Caching → 点一次 "Purge Everything"**（或按 `https://yourdomain.com/vault/*`、`/app/*` 前缀精确清理），再 `nginx -s reload`。

---

## 八、小结与收获

回顾一下，我们用三样东西（一台常开电脑 + 一个域名 + 一个免费 Cloudflare 账号），达成了：

- **零公网 IP、零开放端口**：源站 IP 对互联网不可见，家庭宽带也能安心托管；
- **Nginx 统一路由 + Cloudflare Tunnel 极简配置**：Tunnel 只需两条规则，所有路径分发交给 Nginx，配置清晰、维护简单；
- **统一域名 + 路径路由**：对外只有一个 `yourdomain.com`，彻底告别 CORS 烦恼；
- **边缘零信任认证**：人类访问走邮箱/PIN，机器访问走 Service Token，**两层防护**让私有服务不再"裸奔"；
- **一份服务，两种出口**：`Vibe-Trading` 同时作为网站与 API 出口，轻松对接 OpenBB Workspace 等外部生态；
- **免费**：Cloudflare Free 计划覆盖 Tunnel + Access，个人项目零成本。

### 整体架构一览

```
互联网请求
    │
    ▼
Cloudflare CDN（HTTPS 443）
    │
    ├── Cloudflare Access（零信任认证）
    │
    ▼
Cloudflare Tunnel（加密隧道，无需开放端口）
    │
    ▼
Nginx（localhost:80，路径路由）
    │
    ├── /api/*     → PassXYZ.Server（localhost:5182）
    ├── /vault/*   → passxyz-web（静态文件）
    ├── /app/*     → finanalyzer-app（静态文件）
    └── /*         → Vibe-Trading（localhost:8899）

agent.yourdomain.com（子域名，直连）
    │
    ▼
Vibe-Trading（localhost:8899，Service Token 鉴权）
```

如果你也维护着一堆私有云服务，不妨照这套流程把家里的 Mac mini 或树莓派"升级"成一个属于你自己的安全云。PassXYZ 与 Finanalyzer 的全部源码已在 GitHub 开源，文中的 Nginx 路由配置、Cloudflare Dashboard 操作步骤、Access 中间件实现，都能在仓库文档里找到对应参考。

欢迎在评论区交流你的部署方案，或告诉我你最想上云的服务是什么。
