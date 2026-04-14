---
title: "使用 AKShare 作为 OpenBB 的数据源"
date: 2025-06-26 09:00:00 +0800
categories: [教程, 数据源]
tags: [OpenBB, AKShare, A股, 港股]
pin: true
---

**OpenBB** 作为一个开源金融数据平台，致力于为投资者、分析师和开发者提供免费、透明的金融数据接口。

虽然 OpenBB 支持多源数据接口，但中国（含香港）金融数据的获取主要依赖 Yahoo Finance。作为免费基础数据源，它可以满足基本需求，但对中国和香港市场的覆盖深度仍然不足。更重要的是，中国大陆用户需要使用 VPN 才能访问该服务，造成了显著的使用障碍。

为了增强在大中华地区的服务能力，OpenBB 迫切需要改进本地化金融数据源的实现。主流的付费方案包括 Wind、东方财富 Choice 和同花顺 iFind（主要面向机构客户），而免费方案则以开源工具 **AKShare** 为核心方案。通过整合东方财富、同花顺、腾讯、新浪、雪球等平台的数据接口，AKShare 已成为国内 Python 生态中最全面、最新、对开发者最友好的金融数据聚合库之一。

`openbb_akshare` 项目作为 OpenBB 的数据源扩展，实现了 AKShare 数据与 OpenBB 平台的无缝集成。

## 💻 环境搭建与安装流程

作为开发者，我们主要通过 OpenBB Platform CLI 与平台交互。要集成 AKShare 数据源，请按以下步骤配置开发环境：

### 1. 创建 Python 虚拟环境

可以使用 `venv`、`uv` 或 `poetry` 创建虚拟环境。这里使用 venv（Python 内置）：

```bash
# 创建虚拟环境
python3 -m venv .venv

# 激活虚拟环境（Linux/Mac）
source .venv/bin/activate

# Windows 系统
.venv\Scripts\activate
```

### 2. 安装 OpenBB Platform CLI

通过 pip 安装核心 OpenBB CLI。中国大陆用户可以配置镜像以加速安装：

```bash
# （可选）设置 pip 国内镜像
# Linux/Mac
export PIP_INDEX_URL=https://mirrors.aliyun.com/pypi/simple

# 安装 OpenBB CLI
pip install openbb-cli
```

### 3. 安装 openbb_akshare

接下来安装 `openbb_akshare` 扩展以使用 AKShare 数据源：

```bash
# 安装 AKShare 数据源扩展
pip install openbb_akshare

# 重建 OpenBB 资源以激活插件
python -c "import openbb; openbb.build()"
```

## 🚀 使用 AKShare 数据源

### 案例 1：查询 A 股公司新闻（以万科为例）

**Jupyter Notebook**：

```python
from openbb import obb
messages = obb.news.company("000002", provider="akshare")
for result in messages.results:
    print(f"{result.title}")
```

**使用 OpenBB CLI**：

```bash
openbb
/news/ $ company --symbol 000002 --provider akshare
```

### 案例 2：获取港股历史股价（以香港电信为例）

**Jupyter Notebook**：

```python
from openbb import obb
prices = obb.equity.price.historical(
    symbol='06823',
    start_date="2025-06-01",
    end_date="2025-06-10",
    provider="akshare"
)
prices.results[0].date, prices.results[0].open, prices.results[0].close
```

**使用 OpenBB CLI**：

```bash
/equity/price/ $ historical --symbol 06823 --provider akshare
```

## 🌟 项目生态

`openbb_akshare` 项目目前处于活跃开发阶段，欢迎开源社区的贡献：

- **GitHub**: <https://github.com/finanalyzer/openbb_akshare>
- **GitCode（国内镜像）**: <https://gitcode.com/finanalyzer/openbb_akshare>

### 贡献方式

1. 提交 Issues 报告数据需求或 Bug
2. 贡献 PR 优化数据源接口
3. 完善文档帮助更多用户

通过 AKShare 与 OpenBB 的集成，中国用户可以更便捷地获取 A 股和港股等市场的实时和历史数据，为量化分析和投资研究提供强有力的数据支持。
