---
title: "使用 OpenBB Workspace 构建 A 股和港股分析仪表盘"
date: 2025-09-15 09:00:00 +0800
categories: [教程, 仪表盘]
tags: [OpenBB, Dashboard, A股, 港股, Docker, AI]
pin: true
---

**openbb-hka** 是一个 OpenBB Workspace 应用，提供用于构建 A 股和港股分析仪表盘的模板和组件。它使用 openbb-akshare 和 openbb-tushare 作为数据源。

## OpenBB Workspace Apps vs Dashboards

| **Apps** | **Dashboards** |
|----------|----------------|
| 预配置模板，具有特定分析目标 | 空白画布，自定义配置 |
| Widget 自带预链接参数 | 需要手动配置参数 |
| 包含精选的提示库 | 无预定义提示 |
| 由领域专家为特定工作流设计 | 通用分析工作区 |

## 环境搭建

### API Key 配置

虽然 AKShare 使用免费数据源，但从雪球获取数据时仍需要 API Key。在 `$HOME/.openbb_platform/user_settings.json` 中配置：

```json
{
    "credentials": {
        "akshare_api_key": "你的雪球 xq_a_token"
    }
}
```

### 环境变量

创建 `.env` 文件，参考 `env.example` 进行配置。

### 安装依赖

本项目使用 `uv` 进行依赖管理：

```bash
uv sync
```

### 启动应用

```bash
uv run uvicorn main:app --reload
```

## 使用 Docker 部署

### 构建镜像

```bash
docker build -t openbb-hka:0.2.4 .
```

### 启动容器

```bash
docker compose up
```

## Google Firebase Studio

openbb-hka 也可以在 Google Firebase Studio 中运行，实现云端一键部署。

## 在 OpenBB Workspace 中使用

应用运行后，在 OpenBB Workspace 中点击 **"Connect Backend"** 按钮添加应用。成功添加后，工作区中将出现 **"A-Shares"** 和 **"H-Shares"** 两个新应用。

### 三大核心分析模块

- **Profile**（公司概况）：公司基本信息总览
- **Financials**（财务分析）：三大财务报表可视化
- **Stock Price**（历史股价）：交互式股价图表

你可以通过添加、删除或调整 Widget 来自定义分析仪表盘。

## AI 集成

OpenBB Workspace 中的所有 Widget 都可以添加到右侧的 **Copilot** 面板进行 AI 驱动的分析。OpenBB Copilot 也是可扩展的——你可以将常用的 AI 工具集成到平台中。

## 总结

OpenBB Workspace 的组件和 Copilot 功能都支持基于用户需求的深度定制，通过开发 OpenBB Workspace 应用实现个性化扩展。

- **GitHub**: <https://github.com/finanalyzer/openbb-hka>

下一步将推进 Copilot 功能的本地化。一旦实现，中国用户将能够在 OpenBB Workspace 中无缝集成通义千问、豆包等本地 AI 工具，大幅提升用户体验和操作效率。
