---
title: "使用 Tushare 作为 OpenBB 的数据源"
date: 2025-08-01 09:00:00 +0800
categories: [教程, 数据源]
tags: [OpenBB, Tushare, A股, 港股, 财务数据]
pin: true
---

**OpenBB** 作为一个开源金融数据平台，致力于为投资者、分析师和开发者提供免费、透明的金融数据接口。

在查询 A 股和港股数据时，OpenBB 迫切需要改进本地化金融数据源的接入。主流付费方案包括 Wind、东方财富 Choice 和同花顺 iFind（主要面向机构客户）；而开源数据源主要使用 **Tushare** 或 **AKShare** 作为替代方案。

## Tushare vs AKShare 对比

| 特性 | Tushare | AKShare |
|------|---------|---------|
| 是否免费 | 免费版功能有限，高级功能需要积分 | 完全免费 |
| 数据质量 | 高，标准化 | 中等，依赖源网站，需要清洗 |
| 数据覆盖 | 广泛，侧重 A 股和宏观数据 | 非常广泛，包括加密货币、海外市场等 |
| 接口稳定性 | 高（Pro API） | 中等（受源网站影响） |
| 文档完整性 | 高，详细 | 中等，部分需要查看源码 |
| 适合用户 | 企业、专业研究者 | 学生、个人开发者、初学者 |

## 💻 环境搭建与安装流程

### 1. 创建 Python 虚拟环境

```bash
python3 -m venv .venv
source .venv/bin/activate  # Linux/Mac
```

### 2. 安装 OpenBB Platform CLI

```bash
pip install openbb-cli
```

### 3. 安装 openbb_tushare

```bash
pip install openbb_tushare
python -c "import openbb; openbb.build()"
```

## 🔧 配置 Tushare Token

由于 Tushare 需要令牌才能访问其数据源，因此在使用前需要完成令牌配置。在 `~/.openbb_platform/user_settings.json` 文件中添加：

```json
{
  "credentials": {
    "tushare_api_key": "你的Tushare Token"
  }
}
```

> 前往 [Tushare 官网](https://tushare.pro/register) 注册获取 Token。

## 🚀 使用 Tushare 数据源

### 获取股票历史数据（以中国银行为例）

**Jupyter Notebook**：

```python
import pandas as pd
from openbb import obb

symbol = "601988.SH"
data = obb.equity.price.historical(
    symbol=symbol,
    start_date="2025-06-01",
    end_date="2025-07-31",
    provider="tushare"
)
daily = data.to_dataframe()
daily.head()
```

**使用 OpenBB CLI**：

```bash
/equity/price/ $ historical --symbol 601988.SH --start_date 2025-06-01 --end_date 2025-07-31 --provider tushare
```

### 获取股票财务数据（以中国银行为例）

股票基本面分析通常涵盖三大核心财务报表：利润表、资产负债表和现金流量表。

```python
income_obj = obb.equity.fundamental.income(symbol=symbol, provider="tushare")
income_df = income_obj.to_dataframe()
income_df.head()
```

**使用 OpenBB CLI**：

```bash
/equity/fundamental/ $ income --symbol 601988.SH --provider tushare
```

## 🌟 项目生态

`openbb_tushare` 项目目前处于活跃开发阶段，欢迎开源社区的贡献：

- **GitHub**: <https://github.com/finanalyzer/openbb_tushare>
- **GitCode**: <https://gitcode.com/finanalyzer/openbb_tushare>

### 贡献方式

1. 提交 Issues 报告数据需求或 Bug
2. 贡献 PR 优化数据源接口
3. 完善文档帮助更多用户

通过 Tushare 与 OpenBB 的集成，中国用户可以更便捷地获取 A 股和港股等市场的高质量数据，为量化分析和投资研究提供强有力的数据支持。
