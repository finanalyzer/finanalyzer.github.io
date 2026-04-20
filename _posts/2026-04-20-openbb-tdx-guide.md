---
title: "openbb_tdx：打通通达信与 OpenBB 的 A 股港股量化数据新桥梁"
date: 2026-04-20 09:00:00 +0800
categories: [教程, 数据源]
tags: [OpenBB, Tdx, 通达信, A股, 港股, 实时行情, 量化]
pin: true
---

## 导读

在金融量化分析领域，数据获取始终是核心环节。OpenBB 作为全球领先的开源金融数据平台，为全球投资者提供了统一的数据访问层，但其对中国 A 股市场的原生支持一直较为有限。此前，FinAnalyzer 团队已成功开发了 openbb_akshare 和 openbb_tushare 两个扩展插件。

如今，随着通达信官方推出 TdxQuant Python 量化接口，一条全新的数据通道已经打开。**openbb_tdx** 项目应运而生——它将通达信近三十年的金融数据积累与 OpenBB 的开放架构深度融合，为中国量化投资者带来前所未有的数据接入体验。

## 一、项目背景

### OpenBB 与中国市场的数据困境

尽管 OpenBB 支持多源数据接口，但其对中国市场的金融数据获取主要依赖 Yahoo Finance，存在明显不足：

- A 股市场数据覆盖不全，缺乏详细的财务指标和板块数据
- 国内用户访问 Yahoo Finance 需要使用 VPN
- 实时行情数据延迟较大，无法满足日内交易需求

### 通达信 TdxQuant：新的数据通道

2025 年底，通达信正式推出了 **TdxQuant**——一套基于通达信金融终端构建的 Python 量化策略运行框架。其核心特点包括：

- **高质量数据**：所有历史与实时数据均使用通达信客户端提供的数据
- **实时行情**：支持实时快照、K 线、分笔（Tick）数据的订阅与获取
- **全面覆盖**：包含 A 股、港股、ETF、可转债、基金等多种证券品种
- **本地运行**：策略在本地 IDE 环境中开发与运行，保障代码安全与私密性

## 二、核心能力

### 数据接入能力

openbb_tdx 支持获取以下数据类型：

| 类别 | 数据内容 |
|------|----------|
| **行情数据** | 历史 K 线（日线/周线/月线/分钟线）、实时行情快照、分笔 Tick 数据、复权数据 |
| **基本面数据** | 除权除息信息、财务报表、涨跌停/换手率/市盈率、板块交易数据 |
| **分类与标的** | 市场类型分类、行业分类、自定义板块、标的基础信息 |

### 技术特点

- **标准化接口设计**：遵循 OpenBB Provider Framework，切换 provider 即可无缝切换数据源
- **高性能数据传输**：本地客户端-服务端架构，无网络延迟
- **灵活的部署方式**：支持 CLI、Python API、REST API 及 OpenBB Workspace

### 快速开始

```bash
pip install openbb_tdx
python -c "import openbb; openbb.build()"
```

```python
from openbb import obb
# 获取中国石化（600028）的历史股价
df = obb.equity.price.historical(
    symbol="600028",
    start_date="2024-01-01",
    end_date="2025-12-31",
    provider="tdxquant"
).to_dataframe()
print(df.tail())
```

## 三、三大扩展插件对比

| 对比维度 | openbb_akshare | openbb_tushare | openbb_tdx |
|----------|---------------|---------------|-----------|
| **底层数据源** | AKShare（聚合多平台） | Tushare Pro | 通达信 TdxQuant |
| **数据获取方式** | 网络爬虫 | API 调用 | 本地客户端 |
| **实时数据支持** | 部分支持 | 需高级权限 | **原生支持，低延迟** |
| **数据质量** | 中等 | 高 | 高 |
| **使用成本** | 完全免费 | 免费版功能有限 | **完全免费** |
| **调用频率限制** | 有（反爬） | 有（积分限制） | **无限制** |
| **跨平台** | Win/Linux/macOS | Win/Linux/macOS | 仅 Windows |
| **最佳场景** | 数据探索、学术研究 | 基本面分析、量化回测 | **日内交易、实时监控** |

### 选型建议

- **openbb_akshare**：需要覆盖多个市场的综合数据分析、学术研究、跨平台需求
- **openbb_tushare**：需要高质量标准化基本面数据、专业量化研究、跨平台需求
- **openbb_tdx**：日内交易策略、高频数据分析、实时监控预警、已是通达信用户

> 三个扩展插件并非互斥关系，用户可根据不同模块的数据需求灵活组合使用。

## 四、环境要求

- **Python 版本**：3.13+（推荐 3.13）
- **OpenBB Platform CLI**：最新版本
- **通达信金融终端**：支持 TQ 策略功能的版本（V7.72 及以上）

> ⚠️ 运行 openbb_tdx 前需预先启动通达信金融终端。

## 五、总结

openbb_tdx 项目的诞生，标志着 OpenBB 在中国市场数据生态布局中迈出了重要一步。与 openbb_akshare 和 openbb_tushare 相比，openbb_tdx 在**实时行情数据获取**方面具有独特的优势。三个扩展插件各有侧重、互为补充，共同构成了 OpenBB 在中国市场的完整数据解决方案。

- **GitHub**：<https://github.com/finanalyzer/tdx>
- **TdxQuant**：<https://github.com/finanalyzer/tdxquant>
