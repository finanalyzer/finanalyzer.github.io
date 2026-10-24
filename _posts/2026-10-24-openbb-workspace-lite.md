# OpenBB Workspace 开源了，不会部署？自己构建 Docker 镜像就行

![openbb_workspace_lite](https://finanalyzer.github.io/assets/img/openbb/openbb_workspace_lite.jpg)

2026 年 10 月，OpenBB 创始人兼 CEO Didier Lopes 宣布：整个 OpenBB 产品体系 —— OpenBB Workspace、开放数据平台（Open Data Platform）、OpenBB Copilot 以及 Excel 插件 —— 都将以宽松许可证开源，并得到团队和 OSS Capital 的支持。五年多的工程积累，从此成为个人、初创公司、数据供应商和机构都可以自由使用、修改并在其上继续构建的基础设施。

OpenBB 的技术现在由独立非营利组织 OpenBQ 托管。该计划由 BrightQuery 的 Jose Plehn 主导，他带入了大约三分之一的 OpenBB 原团队成员继续开发。Workspace 代码库将由 Linux 基金会旗下的 FINOS 托管，BrightQuery 也在运营一个用自己的数据集驱动的 BQ Workspace 线上版本。

这是战略层面的交接。但对任何真正想用上 Workspace 的人来说，现实层面的空白是：**目前并没有一个可以直接注册使用的托管版 OpenBB Workspace 服务。** 如果你想拥有自己的工作空间 —— 你的数据、你的工作流、你的 Agent，全部运行在你自己掌控的环境里 —— 就得自己构建镜像、自己部署实例。

上游的 OpenBB Workspace 源码位于 [github.com/OpenBB-finance/workspace](https://github.com/OpenBB-finance/workspace)，但这个仓库无法开箱即用地通过 GitHub Actions 工作流构建成可用的 Docker 镜像。自托管的建设者需要一条可复现的 CI 路径：把前端和后端组装起来，产出多架构镜像，并推送到镜像仓库。

所以我围绕这条流水线，把原来的 monorepo 拆成了三个各司其职的仓库：一个构建编排仓库、一个前端仓库、一个后端仓库。每个仓库都可以独立进行版本管理，构建仓库在 CI 里把它们串联起来。

### 拆分后的三个仓库

这三个仓库是由`OpenBB-finance/workspace`中的三个目录拆分的来。如果自己构建，需要对lite仓库进行相应的修改。

- **构建编排 —— [finanalyzer/lite](https://github.com/finanalyzer/lite)：** 包含 Dockerfile、容器配置，以及"构建并发布"的 GitHub Actions 工作流。该工作流按可配置的分支、标签、commit SHA 或 PR ref 检出两个源码仓库，构建多架构镜像，并发布到 GitHub Container Registry 和 Docker Hub。自己构建，需要做相应的配置。配置了GitHub Action 的环境变量后，构建成功后，镜像会推送到你自己的Docker Hub和GHCR。

- **前端 —— [finanalyzer/terminalpro](https://github.com/finanalyzer/terminalpro)：** 基于 React 18 + TypeScript 的单页应用，使用 Vite、Tailwind CSS 和 Radix UI 构建；状态管理采用 Zustand 和 React Query（TanStack Query），路由使用 React Router v6；测试框架为 Vitest、Playwright 和 MSW。

- **后端 —— [finanalyzer/openbb-hub](https://github.com/finanalyzer/openbb-hub)：** FastAPI 应用 —— API 服务、数据库模型、迁移脚本和后台 Worker —— 外加一个管理 CLI 和一个管理面板。

#### Lite 镜像里有什么

一个 Docker 镜像打包了 FastAPI 后端、terminalpro 单页应用、Redis、nginx 和 supervisord。默认使用 SQLite，首次启动时自动生成密钥，并自动创建管理员账号 —— 正因如此，快速上手只需要一条命令。

容器内部的进程布局：nginx 监听 3000 端口，提供单页应用并把 /api 反向代理到 127.0.0.1:8000 上的 uvicorn FastAPI 后端。

```bash
docker run -d -p 3000:3000 \
  -v openbb-data:/data \
  --name openbb \
  openbblite/lite:latest
```

打开 http://localhost:3000，使用 admin@openbb.co 登录。密码在首次启动时生成，会打印在启动日志里，并保存在数据卷中；随时可以用下面的命令取回：

```bash
docker exec openbb credentials
```

镜像自动生成的每一项配置都可以在运行时覆盖：自定义管理员邮箱和密码、用 Postgres 或 MySQL 替代 SQLite、使用外部 Redis、自定义 JWT 和 AES 密钥，以及前端运行时配置 —— Copilot、MCP 伴生服务、应用市场、图表生成等相关 UI 开关。而品牌信息、注册、条款和遥测这些字段在构建时就被锁定在严格的 Lite 配置档里，运行时配置无法改动它们。

#### 构建产物在哪里

构建好的镜像发布到了两个镜像仓库，任何一个都可以直接拉起来部署你自己的实例：

- [**GitHub Container Registry：**](https://github.com/finanalyzer/lite/pkgs/container/lite)[ github.com/finanalyzer/lite/pkgs/container/lite](https://github.com/finanalyzer/lite/pkgs/container/lite)

- [**Docker Hub：**](https://hub.docker.com/r/openbblite/lite)[ hub.docker.com/r/openbblite/lite](https://hub.docker.com/r/openbblite/lite)

#### 如何部署

**VPS 自托管。** 用发布好的镜像部署自己的实例。这条路径需要你自备 VPS 和域名；Cloudflare 可以简化 DNS、TLS 终结和反向代理，域名和流量处理不必从零搭建。

#### 未来开发方向

对于个人用户来说，需要自己准备VPS和域名，会是一个门槛。OpenBB Workspace Lite是有可能使用Cloudflare的pages和worker来部署的。这是计划中的下一步，而非今天就能用的方案。由于前端是单页应用，可以通过 Cloudflare Pages 部署；后端则可以改造为 Cloudflare Worker。移植工作完成后，个人实例可以完全运行在 Cloudflare 的边缘网络上，不再需要常驻 VPS。目前可用的部署路径仍是上面的 VPS 方案。


