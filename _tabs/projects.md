---
icon: fas fa-rocket
order: 1
---

# 项目总览

Finanalyzer 提供四个核心开源项目，构成从数据获取到可视化分析的完整工具链。

---

## 📦 openbb_akshare

> OpenBB 的 AKShare 数据源扩展插件 —— 免费、无需 API Key

[GitHub 仓库](https://github.com/finanalyzer/openbb_akshare) · `pip install openbb_akshare`

### 主要功能

- **A 股公司新闻查询**：获取个股最新新闻资讯
- **港股历史股价数据**：支持港股历史行情查询
- **多数据源聚合**：整合东方财富、同花顺、腾讯、新浪、雪球等平台数据
- **完全免费**：无需注册、无需付费 API Key
- **双模式使用**：支持 Python API 和 OpenBB CLI

### 快速开始

```bash
pip install openbb-cli
pip install openbb_akshare
python -c "import openbb; openbb.build()"
```

```python
from openbb import obb
# 查询公司新闻
messages = obb.news.company("000002", provider="akshare")
# 查询历史股价
prices = obb.equity.price.historical(
    symbol='06823',
    start_date="2025-06-01",
    end_date="2025-06-10",
    provider="akshare"
)
```

---

## 📊 openbb_tushare

> OpenBB 的 Tushare 数据源扩展插件 —— 高质量、专业级金融数据

[GitHub 仓库](https://github.com/finanalyzer/openbb_tushare) · `pip install openbb_tushare`

### 主要功能

- **A 股历史股价**：标准化高质量行情数据
- **三大财务报表**：利润表、资产负债表、现金流量表
- **数据质量高**：标准化程度好、接口稳定性强
- **专业级接口**：适合研究者和企业用户
- **双模式使用**：支持 Python API 和 OpenBB CLI

### 快速开始

```bash
pip install openbb-cli
pip install openbb_tushare
python -c "import openbb; openbb.build()"
```

```python
from openbb import obb
# 获取历史股价
data = obb.equity.price.historical(
    symbol="601988.SH",
    start_date="2025-06-01",
    end_date="2025-07-31",
    provider="tushare"
)
# 获取利润表
income = obb.equity.fundamental.income(
    symbol="601988.SH",
    provider="tushare"
)
```

> ⚠️ 使用 Tushare 需要 [注册获取 Token](https://tushare.pro/register)

---

## 🔌 openbb_tdx

> OpenBB 的通达信 TdxQuant 数据源扩展插件 —— 实时行情、低延迟、零成本

[GitHub 仓库](https://github.com/finanalyzer/tdx) · `pip install openbb_tdx`

### 主要功能

- **实时行情数据**：支持实时快照、K 线、分笔（Tick）数据
- **高质量历史数据**：通达信近三十年金融数据积累
- **全面覆盖**：A 股、港股、ETF、可转债、基金等
- **零频率限制**：本地通信，无调用频率限制
- **完全免费**：需安装通达信终端（V7.72+）

### 快速开始

```bash
pip install openbb_tdx
python -c "import openbb; openbb.build()"
```

```python
from openbb import obb
# 获取历史股价
df = obb.equity.price.historical(
    symbol="600028",
    start_date="2024-01-01",
    end_date="2025-12-31",
    provider="tdxquant"
).to_dataframe()
```

> ⚠️ 使用 openbb_tdx 需要预先启动通达信金融终端，目前仅支持 Windows

---

## 🖥️ openbb-hka

> A 股 & 港股分析仪表盘 —— 可视化 + AI 赋能

[GitHub 仓库](https://github.com/finanalyzer/openbb-hka)

### 主要功能

- **预配置仪表盘**：A 股和港股分析模板，开箱即用
- **三大分析模块**：
  - **Profile**（公司概况）：公司基本信息总览
  - **Financials**（财务分析）：三大财务报表可视化
  - **Stock Price**（历史股价）：交互式股价图表
- **TradingView 图表**：专业级金融图表
- **AI 集成**：支持 ChatGLM 等国内 AI 工具辅助分析
- **自选股功能**：自定义关注列表
- **多种部署方式**：本地运行、Docker、Google Firebase Studio

### 技术栈

- 后端：Python (FastAPI)
- 前端：TypeScript
- 数据源：openbb_akshare + openbb_tushare
- 容器化：Docker 支持

### 快速开始

```bash
# 方式一：本地运行
uv sync
uv run uvicorn main:app --reload

# 方式二：Docker 部署
docker compose up
```

---

## 架构关系

```
┌─────────────────────────────────┐
│         openbb-hka              │
│    (可视化分析仪表盘)            │
├──────────┬──────────┬───────────┤
│ openbb_  │ openbb_  │ openbb_   │
│ akshare  │ tushare  │ tdx       │
│ (免费数据)│ (专业数据)│ (实时行情)│
├──────────┴──────────┴───────────┤
│       OpenBB Platform           │
├──────────┬──────────┬───────────┤
│ AKShare  │ Tushare  │ TdxQuant  │
│ (免费接口)│(专业 API)│ (官方接口)│
└──────────┴──────────┴───────────┘
```
