# 场外基金 AI 辅助决策系统 —— 工程化方案 SPEC

> **文档目标**：交付给实现 Agent（Claude Code CLI / Hermes / OpenClaw）的可直接落地工程方案。
>
> **版本**：v2.0
> **形态**：本地 Python 包，通过 MCP Server(stdio) / CLI / Python import 三种方式对外暴露
> **运行环境**：Claude Code CLI、Hermes Agent、OpenClaw Skill/工作流
> **目标用户**：通过支付宝/天天基金持有公募场外基金的个人投资者

---

## 1. 背景与目标

### 1.1 业务背景

用户通过支付宝持有若干公募场外基金。场外基金的核心特性：

- **未知价交易**：交易日 15:00 前提交，按当晚收盘后净值成交；过点顺延下一交易日。
- **T+1 / T+2 确认**：申请日 ≠ 确认日，**持有天数从确认日开始计算**。
- **阶梯赎回费**：典型 `<7天 ≥1.5%` / `7-30天 0.5%` / `30天-1年 0.25%` / `>1年 0%`，每只基金不同。
- **A/C 类差异**：A 类有申购费无销售服务费；C 类无申购费有销售服务费，通常 7/30 天后免赎回费。

适合"下午 14:30–15:00 决策一次"的低频场景，但对**规则正确性**和**事实数据**要求极高。

### 1.2 当前痛点

1. **数据缺失**：通用 LLM 拿不到基金实时估值、指数、政策日历等结构化数据。
2. **规则幻觉**：靠记忆回忆赎回费阶梯，容易编造。
3. **注意力漂移**：多轮对话中补充常识信息导致决策反复变化。
4. **合规边界**：持牌投顾只做长期配置，不给短线指令。


### 1.3 系统定位

把大模型从"决策者"降级为 **"纪律检查员 + 事实核对员 + 方案解释员"**：

- **代码**负责：数据采集、规则计算、费用计算、持仓识别
- **大模型**负责：综合事实、解释方案、按硬约束输出建议
- **人**负责：最终拍板与执行

### 1.4 非目标

- ❌ 不预测涨跌
- ❌ 不自动下单（支付宝无开放 API）
- ❌ 不做 7 天内高频择时
- ❌ 不替代持牌投顾

### 1.5 验收标准

- ✅ 上传支付宝持仓截图，10 秒内得到结构化持仓 JSON
- ✅ 决策中所有费率、持有天数、估值数字 100% 来自工具调用，无幻觉
- ✅ 持有 <7 天的份额永远不会被建议赎回（除非用户显式覆盖）
- ✅ 单次完整决策端到端 ≤ 60 秒

---

## 2. 总体架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│               调用方（三选一，均支持）                             │
│                                                                  │
│  ① Claude Code CLI  ── MCP stdio ──┐                            │
│  ② Hermes Agent     ── MCP stdio ──┼──→  fund-advisor 本地包    │
│  ③ OpenClaw Skill   ── python import ──→  fund-advisor 本地包   │
│     (或 shell 调 CLI)               │                            │
└─────────────────────────────────────────────────────────────────┘
                                      │
                          ┌───────────┴────────────┐
                          ▼                        ▼
                   ┌────────────┐          ┌─────────────┐
                   │  AKShare   │          │ holdings.json│
                   │  (数据源)  │          │ (持仓文件)   │
                   └────────────┘          └─────────────┘
```


### 2.2 对外暴露方式

| 调用方 | 接入方式 | 配置方法 |
|---|---|---|
| **Claude Code CLI** | MCP Server (stdio) | `claude mcp add fund-advisor -- uv run fund-advisor-server` |
| **Hermes** | MCP Server (stdio) | 在 Hermes config 里加 MCP server 定义 |
| **OpenClaw** | Python import / Shell CLI | Skill 里 `from fund_advisor.api import get_decision_brief` 或 `fund-advisor brief` |
| **手动/其他 LLM** | CLI 输出 JSON | `fund-advisor brief` 输出 → 复制到任意对话框 |

### 2.3 技术栈

| 层 | 选型 | 备注 |
|---|---|---|
| 语言 | Python 3.11+ | AKShare 生态 |
| MCP | `mcp` 官方 Python SDK | stdio 模式 |
| 数据 | AKShare（免费无 token） | 备选 Tushare |
| 持久化 | **JSON 文件** (`holdings.json`) | 零依赖，够用 |
| 规则 | YAML 配置 + Python 函数 | 不引重型引擎 |
| CLI | `click` | 简单 |
| 包管理 | `uv` | |

### 2.4 数据流

```
[首次] 用户上传持仓截图
  → LLM vision 提取 → 调 parse_holdings_image 工具
  → 补全基金代码/费率 → 写入 holdings.json
  → 用户确认/修正

[每日 14:30]
  调用 generate_decision_brief:
    1. 读 holdings.json
    2. AKShare 拉实时估值 + 指数 + 北向
    3. AKShare 拉新闻
    4. 计算每笔 lot 持有天数
    5. 跑规则引擎
    6. 组装 brief JSON 返回
  → LLM 基于 brief + 系统 prompt 输出决策矩阵
  → 人执行
```

---

## 3. 数据模型（轻量版）

### 3.1 设计原则

- **一个 JSON 文件搞定全部持仓**，不用数据库
- 文件路径：`~/.fund-advisor/holdings.json`（可配置）
- 程序启动时 load，修改后 save
- 后续如需历史回溯，再加 SQLite 不迟


### 3.2 holdings.json Schema

```json
{
  "$schema": "holdings-v1",
  "updated_at": "2026-05-21T14:30:00+08:00",
  "available_cash_cny": 50000,
  "funds": {
    "005827": {
      "name": "易方达蓝筹精选混合A",
      "share_class": "A",
      "fund_type": "混合",
      "is_qdii": false,
      "fee_table": [
        {"min_days": 0, "max_days": 7, "rate": 0.015},
        {"min_days": 7, "max_days": 365, "rate": 0.005},
        {"min_days": 365, "max_days": null, "rate": 0.0}
      ],
      "lots": [
        {
          "confirm_date": "2026-05-10",
          "amount_cny": 5000,
          "shares": 1234.56,
          "nav_at_buy": 4.05
        },
        {
          "confirm_date": "2026-05-19",
          "amount_cny": 3000,
          "shares": 740.12,
          "nav_at_buy": 4.05
        }
      ],
      "target_hold_days": 180,
      "take_profit_rate": 0.30,
      "stop_loss_rate": -0.15
    }
  }
}
```

### 3.3 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `available_cash_cny` | float | 可投资现金池 |
| `funds.{code}` | object | 按基金代码索引 |
| `fee_table` | array | 赎回费阶梯，`max_days=null` 表示无上限 |
| `lots` | array | 每笔申购记录（先进先出赎回时按此顺序） |
| `lots[].confirm_date` | string | 确认日（持有天数从此算） |
| `lots[].amount_cny` | float | 买入金额 |
| `lots[].shares` | float | 确认份额 |
| `lots[].nav_at_buy` | float | 买入时净值 |
| `target_hold_days` | int | 该基金计划持有天数 |
| `take_profit_rate` | float | 止盈线 |
| `stop_loss_rate` | float | 止损线（负数） |

### 3.4 为什么不用数据库

- 你持有的基金通常 5-15 只，每只 1-5 笔 lot → 整个文件 < 5KB
- JSON 可以直接用任何编辑器手动改（Git 友好）
- 不需要 SQL 查询能力
- 部署零依赖（不需要装 SQLite 驱动）
- 如果以后需要历史，加一个 `decision_log.jsonl`（每行一条记录）即可

---

## 4. 模块详细设计

### 4.1 模块 A：持仓识别（图片 → JSON）

#### 工作流

```
用户上传支付宝截图（"我的基金"页 + "交易记录"页）
  ↓
LLM 用 vision 能力提取文字信息（基金名、金额、收益率等）
  ↓
LLM 调用 parse_holdings_image(extracted_data) 工具
  ↓
Server 端：
  1. 用基金名 → AKShare fund_name_em() 查基金代码
  2. 用代码 → fund_fee_em() 查费率阶梯
  3. 合并到 holdings.json（标记 needs_confirm=true）
  ↓
返回给 LLM → LLM 让用户确认/补全缺失字段（确认日、份额等）
  ↓
用户确认后 → 调 confirm_holdings() → 写入正式版
```


#### 截图能读到 vs 需要补全

| 字段 | 支付宝"我的基金"页 | "交易记录"页 | 需要补全？ |
|---|---|---|---|
| 基金名称 | ✅ | ✅ | 不需要 |
| 基金代码 | 部分显示 | ✅ | 可能 |
| 持仓金额 | ✅ | - | 不需要 |
| 每笔买入金额 | - | ✅ | 需交易记录截图 |
| 确认日期 | - | ✅ | 需交易记录截图 |
| 份额 | 详情页 | ✅ | 需交易记录截图 |
| A/C 类 | 名称含 A/C | - | 可自动判断 |

> **重要**：单张"我的基金"截图不够算赎回费。必须同时要"交易记录"截图获取每笔的确认日和金额。

### 4.2 模块 B：实时数据采集

#### 数据源映射

| 信息 | AKShare 函数 | 用途 |
|---|---|---|
| 基金盘中估值 | `fund_value_estimation_em(symbol="全部")` | 判断今日涨跌 |
| 基金昨收净值 | `fund_open_fund_info_em(symbol, indicator="单位净值走势")` | 计算盈亏 |
| 基金费率 | `fund_fee_em(symbol, indicator="赎回费率")` | 初始化时拉取 |
| 基金重仓股 | `fund_portfolio_hold_em(symbol, date)` | 关联新闻 |
| 实时指数 | `stock_zh_index_spot_em()` | 大盘快照 |
| 北向资金 | `stock_hsgt_north_net_flow_in_em(symbol="北上")` | 情绪指标 |
| 行业板块 | `stock_board_industry_name_em()` | 板块轮动 |

#### 缓存（内存即可）

| 数据 | TTL |
|---|---|
| 基金基础信息/费率 | 30 天（首次拉取后缓存到 holdings.json） |
| 盘中估值 | 60 秒 |
| 指数/北向 | 30 秒 |
| 重仓股 | 季度 |

### 4.3 模块 C：新闻、政策、事件

#### 事件分类

| 类别 | 例子 | 影响范围 |
|---|---|---|
| 宏观政策 | 降准降息、财政政策 | 全市场 |
| 行业政策 | 集采、补贴、监管 | 行业基金 |
| 公司事件 | 财报、分红 | 重仓股相关基金 |
| 海外联动 | 美联储议息、美股大跌 | QDII/港股基金 |
| 基金自身 | 经理变更、限购 | 单基金 |

#### 数据源

| 源 | 接入方式 | AKShare 函数 |
|---|---|---|
| 财联社电报 | AKShare | `news_cls()` |
| 东财个股新闻 | AKShare | `stock_news_em(symbol)` |
| 业绩预报 | AKShare | `stock_yjbb_em(date)` |
| 美联储日历 | YAML 手动维护 | `config/events_calendar.yaml` |

#### 关联性打分（规则式，不用 embedding）

```python
score = 0
if any(stock in news_title for stock in fund_top10_stocks):
    score += 0.5
if any(industry in news_title for industry in fund_industries):
    score += 0.3
if any(kw in news_title for kw in ["降息", "降准", "美联储", "财报"]):
    score += 0.2
```


### 4.4 模块 D：规则引擎

#### 规则配置 `config/rules.yaml`

见仓库中 `config/rules.yaml`，含硬规则（H1-H4）和软规则（S1-S6）。

#### 核心逻辑

```python
def check_rules(holdings: dict, market: dict, events: dict, config: dict) -> list[dict]:
    """
    遍历 hard_rules + soft_rules，逐条检查是否触发。
    返回 [{rule_id, triggered, severity, message, affected_funds, evidence}]
    """
    results = []
    for fund_code, fund in holdings["funds"].items():
        for lot in fund["lots"]:
            holding_days = (today - parse_date(lot["confirm_date"])).days
            # H1: < 7 天禁止赎回
            if holding_days < 7:
                results.append({
                    "rule_id": "H1_min_holding_days",
                    "severity": "blocker",
                    "message": f"{fund['name']} 有份额仅持有{holding_days}天，禁止赎回",
                    "affected_funds": [fund_code],
                    "evidence": {"holding_days": holding_days, "confirm_date": lot["confirm_date"]}
                })
    # ... 其余规则类似
    return results
```

#### 规则触发后的行为

| severity | LLM 行为 |
|---|---|
| `blocker` | **绝对不允许**建议对应操作 |
| `warning` | 必须在输出中**显式警告**，但可以给出建议 |
| `info` | 作为参考信息附在建议后 |

### 4.5 模块 E：赎回费精确计算

```python
def calc_redemption_fee(
    fund_code: str,
    shares_to_sell: float,
    holdings: dict,
    today: date = None,
    lot_order: str = "FIFO"  # FIFO/LIFO/MIN_FEE_FIRST
) -> dict:
    """
    按 lot 逐笔计算赎回费。
    返回：
    {
      "total_fee_cny": 12.5,
      "effective_rate": 0.005,
      "lot_breakdown": [
        {"confirm_date": "2026-05-10", "shares": 500, "held_days": 11, "rate": 0.005, "fee": 10.1},
        ...
      ],
      "optimization_hint": "再等5天费率从0.5%降到0.25%，可省X元"
    }
    """
```

### 4.6 模块 F：MCP Server + CLI

#### MCP 工具列表

| 工具名 | 描述 | 何时调用 |
|---|---|---|
| `parse_holdings_image` | 接收 LLM 提取的持仓 JSON，补全代码/费率 | 用户上传截图后 |
| `add_lot` | 添加单笔申购记录 | 补全交易记录时 |
| `list_holdings` | 列出持仓（含持有天数、当前估值、盈亏） | 决策开始 |
| `get_fund_realtime` | 实时估值/昨收净值 | 决策时 |
| `get_market_snapshot` | 大盘/板块/北向 | 决策时 |
| `get_news_and_events` | 新闻+事件 | 决策时 |
| `calc_redemption_fee` | 精确赎回费（按 lot 拆分） | 涉及赎回时 |
| `check_rules` | 跑规则引擎 | 推理前必调 |
| `generate_decision_brief` | **聚合工具**：一次性返回全部数据+规则结果 | 推荐入口 |

#### CLI 命令

```bash
fund-advisor brief              # 输出完整决策 brief JSON
fund-advisor holdings           # 查看持仓
fund-advisor fee 005827 500     # 计算赎回费
fund-advisor rules              # 查看当前规则触发情况
fund-advisor market             # 大盘快照
fund-advisor news               # 相关新闻
fund-advisor add-lot 005827 --confirm-date 2026-05-19 --amount 3000 --shares 740
```


### 4.7 模块 G：决策输出（LLM 系统 Prompt）

#### 系统 Prompt（硬编码在 MCP server，作为 tool description 的一部分传递）

```
你是用户的场外基金"纪律检查员"。你不预测涨跌，只做规则核对和方案解释。

【绝对禁止】
- 编造任何费率、净值、持有天数（必须全部来自工具返回值）
- 在持有天数 < 7 天时建议赎回（blocker 级规则）
- 对 QDII 基金基于 A 股当天表现做决策
- 在没调 calc_redemption_fee 的情况下给出赎回金额

【强制工作流】
1. 调 generate_decision_brief() 获取全部数据
2. 对每只基金，先看 rule_results，blocker 一票否决
3. 涉及赎回 → 必调 calc_redemption_fee()
4. 输出决策矩阵

【输出格式】
对每只基金：
- 建议：保持 / 加仓 X 元 / 减仓 X 份 / 观察
- 触发规则：规则编号
- 决策依据：引用工具返回的具体数字
- 若赎回：精确费用（来自 calc_redemption_fee）
- 持有天数优化：如"再等 N 天费率从 X% 降到 Y%，省 Z 元"
- 最大不确定性：
- 备选方案：

【末尾必须输出】
- 数据时效说明（哪些是 T-1 净值、哪些是盘中估值）
- 免责声明："本输出非投资建议，最终决策由用户负责"
```

#### generate_decision_brief 返回结构

```json
{
  "as_of": "2026-05-21T14:32:00+08:00",
  "minutes_to_cutoff": 28,
  "holdings_summary": [
    {
      "fund_code": "005827",
      "fund_name": "易方达蓝筹精选混合A",
      "is_qdii": false,
      "total_value_cny": 8050,
      "total_profit_rate": 0.019,
      "lots": [
        {"confirm_date": "2026-05-10", "holding_days": 11, "shares": 1234.56, "fee_rate_now": 0.005, "next_breakpoint": {"days_until": 19, "rate_after": 0.0025}},
        {"confirm_date": "2026-05-19", "holding_days": 2, "shares": 740.12, "fee_rate_now": 0.015, "next_breakpoint": {"days_until": 5, "rate_after": 0.005}}
      ],
      "estimate_change_pct": 0.0052,
      "estimate_time": "14:30"
    }
  ],
  "market": {
    "sh000300": {"name": "沪深300", "change_pct": 0.42},
    "sz399006": {"name": "创业板指", "change_pct": -0.78},
    "north_flow_cny": -2100000000
  },
  "events_next_7d": [
    {"date": "2026-05-22", "type": "earnings", "desc": "宁德时代一季报", "related_funds": ["005827"]}
  ],
  "news_top5": [
    {"title": "...", "source": "财联社", "time": "14:20", "relevance": 0.7, "related_funds": ["005827"]}
  ],
  "rule_results": [
    {"rule_id": "H1_min_holding_days", "severity": "blocker", "message": "..."}
  ],
  "data_quality": [
    "005827 盘中估值时间戳 14:30，时效正常",
    "000834(QDII) 估值不可信，已标记"
  ]
}
```

---

## 5. 项目结构

```
fund-ai-advisor/
├── README.md
├── SPEC.md                         # 本文档
├── pyproject.toml
├── .env.example
├── config/
│   ├── rules.yaml                  # 规则配置
│   ├── targets.yaml                # 用户目标仓位
│   └── events_calendar.yaml        # 手动维护的重大事件（议息日等）
├── src/
│   └── fund_advisor/
│       ├── __init__.py
│       ├── server.py               # MCP server 入口 (stdio)
│       ├── cli.py                  # Click CLI 入口
│       ├── api.py                  # 对外 Python API（供 OpenClaw import）
│       ├── config.py               # 配置加载
│       ├── holdings.py             # holdings.json 读写 + lot 计算
│       ├── data/
│       │   ├── __init__.py
│       │   ├── akshare_client.py   # AKShare 封装+内存缓存
│       │   ├── realtime.py         # 盘中估值
│       │   ├── market.py           # 指数/北向/板块
│       │   └── news.py            # 新闻+事件
│       ├── rules/
│       │   ├── __init__.py
│       │   └── engine.py           # 规则引擎
│       ├── fee/
│       │   ├── __init__.py
│       │   └── calculator.py       # 赎回费计算
│       └── decision/
│           ├── __init__.py
│           ├── brief.py            # generate_decision_brief
│           └── prompts.py          # 系统 prompt 常量
└── tests/
    ├── test_fee.py
    ├── test_rules.py
    ├── test_holdings.py
    └── fixtures/
        └── sample_holdings.json
```


---

## 6. 部署与接入

### 6.1 安装

```bash
git clone <repo>
cd fund-ai-advisor
uv venv && source .venv/bin/activate
uv pip install -e .
# 初始化空的 holdings.json
fund-advisor init
```

### 6.2 接入 Claude Code CLI

```bash
# 注册 MCP server
claude mcp add fund-advisor -- uv --directory /path/to/fund-ai-advisor run fund-advisor-server

# 使用
claude
> 帮我看看今天基金怎么操作
# Claude 会自动调用 generate_decision_brief 等工具
```

### 6.3 接入 Hermes

在 Hermes 的配置中添加 MCP server：

```yaml
# hermes config
mcp_servers:
  - name: fund-advisor
    command: uv
    args: ["--directory", "/path/to/fund-ai-advisor", "run", "fund-advisor-server"]
```

Hermes Agent 即可通过 MCP 协议调用所有工具。

### 6.4 接入 OpenClaw

**方式一：作为 Skill（Python import）**

```python
# openclaw skill
from fund_advisor.api import get_decision_brief, list_holdings, calc_fee

def daily_check():
    brief = get_decision_brief()
    return brief  # 返回给 OpenClaw 的 LLM 层
```

**方式二：作为工作流中的 Shell 步骤**

```yaml
# openclaw workflow
steps:
  - name: fetch_brief
    type: shell
    command: fund-advisor brief
    output: brief_json

  - name: analyze
    type: llm
    prompt: |
      基于以下数据给出操作建议：
      {{brief_json}}
    system: (系统 prompt，见 4.7 节)
```

**方式三：MCP（如果 OpenClaw 支持）**

与 Hermes 相同配置方式。

### 6.5 纯 CLI 手动使用（不依赖任何 Agent）

```bash
# 每天 14:30 跑一次
fund-advisor brief | jq .

# 拿到 JSON 后复制到任意大模型对话框
# 配合固定提示词模板即可
```

### 6.6 配置文件位置

```
~/.fund-advisor/
├── holdings.json        # 持仓数据
├── config.yaml          # 覆盖默认配置（可选）
└── decision_log.jsonl   # 决策日志（可选，每行一条）
```

或者通过环境变量指定：

```bash
export FUND_ADVISOR_HOME=/path/to/data
```

---

## 7. 关键代码骨架

### 7.1 MCP Server 入口

```python
# src/fund_advisor/server.py
from mcp.server.fastmcp import FastMCP
from .api import (
    parse_holdings_image, add_lot, list_holdings,
    get_fund_realtime, get_market_snapshot,
    get_news_and_events, calc_redemption_fee,
    check_rules, generate_decision_brief
)

mcp = FastMCP("fund-advisor")

@mcp.tool()
def parse_holdings_image_tool(extracted: dict) -> dict:
    """接收 LLM vision 提取的持仓信息，补全基金代码和费率，写入 holdings.json"""
    return parse_holdings_image(extracted)

@mcp.tool()
def add_lot_tool(fund_code: str, confirm_date: str, amount_cny: float,
                 shares: float | None = None, nav_at_buy: float | None = None) -> dict:
    """添加一笔申购记录到指定基金"""
    return add_lot(fund_code, confirm_date, amount_cny, shares, nav_at_buy)

@mcp.tool()
def list_holdings_tool() -> dict:
    """列出所有持仓，含每笔 lot 的持有天数、当前估值、盈亏"""
    return list_holdings()

@mcp.tool()
def get_fund_realtime_tool(fund_codes: list[str]) -> dict:
    """获取指定基金的盘中估值和昨收净值"""
    return get_fund_realtime(fund_codes)

@mcp.tool()
def get_market_snapshot_tool() -> dict:
    """获取大盘指数、北向资金、行业板块涨跌"""
    return get_market_snapshot()

@mcp.tool()
def get_news_and_events_tool(horizon_days: int = 7) -> dict:
    """获取与持仓相关的新闻、宏观事件、财报日历"""
    return get_news_and_events(horizon_days)

@mcp.tool()
def calc_redemption_fee_tool(fund_code: str, shares_to_sell: float,
                              lot_order: str = "FIFO") -> dict:
    """精确计算赎回费，按 lot 逐笔拆分，含优化建议"""
    return calc_redemption_fee(fund_code, shares_to_sell, lot_order)

@mcp.tool()
def check_rules_tool() -> dict:
    """跑规则引擎，返回所有触发的规则（含 blocker/warning/info）"""
    return check_rules()

@mcp.tool()
def generate_decision_brief_tool() -> dict:
    """【推荐入口】一次性返回：持仓+实时数据+大盘+新闻+规则结果"""
    return generate_decision_brief()

def main():
    mcp.run()

if __name__ == "__main__":
    main()
```


### 7.2 CLI 入口

```python
# src/fund_advisor/cli.py
import click
import json
from .api import (
    generate_decision_brief, list_holdings, calc_redemption_fee,
    check_rules, get_market_snapshot, get_news_and_events, add_lot
)

@click.group()
def cli():
    """场外基金 AI 辅助决策工具"""
    pass

@cli.command()
def brief():
    """输出完整决策 brief（JSON）"""
    result = generate_decision_brief()
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
def holdings():
    """查看当前持仓"""
    result = list_holdings()
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
@click.argument("fund_code")
@click.argument("shares", type=float)
@click.option("--order", default="FIFO", help="FIFO/LIFO/MIN_FEE_FIRST")
def fee(fund_code, shares, order):
    """计算赎回费"""
    result = calc_redemption_fee(fund_code, shares, order)
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
def rules():
    """查看当前规则触发情况"""
    result = check_rules()
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
def market():
    """大盘快照"""
    result = get_market_snapshot()
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
@click.option("--days", default=7)
def news(days):
    """相关新闻与事件"""
    result = get_news_and_events(days)
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command("add-lot")
@click.argument("fund_code")
@click.option("--confirm-date", required=True)
@click.option("--amount", type=float, required=True)
@click.option("--shares", type=float, default=None)
@click.option("--nav", type=float, default=None)
def add_lot_cmd(fund_code, confirm_date, amount, shares, nav):
    """添加申购记录"""
    result = add_lot(fund_code, confirm_date, amount, shares, nav)
    click.echo(json.dumps(result, ensure_ascii=False, indent=2))

@cli.command()
def init():
    """初始化 ~/.fund-advisor/ 目录和空 holdings.json"""
    import os
    home = os.path.expanduser("~/.fund-advisor")
    os.makedirs(home, exist_ok=True)
    hf = os.path.join(home, "holdings.json")
    if not os.path.exists(hf):
        with open(hf, "w") as f:
            json.dump({"$schema": "holdings-v1", "available_cash_cny": 0, "funds": {}}, f, indent=2)
        click.echo(f"已创建 {hf}")
    else:
        click.echo(f"{hf} 已存在，跳过")

def main():
    cli()
```

### 7.3 Python API（供 OpenClaw import）

```python
# src/fund_advisor/api.py
"""
对外统一 API。MCP server / CLI / OpenClaw Skill 都调这里。
"""
from datetime import date
from .holdings import HoldingsStore
from .data.realtime import fetch_fund_realtime
from .data.market import fetch_market_snapshot
from .data.news import fetch_news_and_events
from .rules.engine import run_rules
from .fee.calculator import compute_redemption_fee
from .config import load_config

_store = None

def _get_store() -> HoldingsStore:
    global _store
    if _store is None:
        _store = HoldingsStore()
    return _store

def list_holdings() -> dict:
    store = _get_store()
    return store.list_with_computed_fields(today=date.today())

def add_lot(fund_code: str, confirm_date: str, amount_cny: float,
            shares: float | None = None, nav_at_buy: float | None = None) -> dict:
    store = _get_store()
    store.add_lot(fund_code, confirm_date, amount_cny, shares, nav_at_buy)
    return {"ok": True}

def parse_holdings_image(extracted: dict) -> dict:
    store = _get_store()
    # 补全逻辑：用 AKShare 查代码和费率
    from .data.akshare_client import lookup_fund_code, fetch_fee_table
    results = []
    for item in extracted.get("items", []):
        code = lookup_fund_code(item["fund_name_zh"])
        fee_table = fetch_fee_table(code) if code else []
        store.upsert_fund(code, item, fee_table)
        results.append({"fund_code": code, "name": item["fund_name_zh"], "status": "added"})
    return {"results": results, "next": "请补全每笔申购的确认日期和金额"}

def get_fund_realtime(fund_codes: list[str]) -> dict:
    return fetch_fund_realtime(fund_codes)

def get_market_snapshot() -> dict:
    return fetch_market_snapshot()

def get_news_and_events(horizon_days: int = 7) -> dict:
    store = _get_store()
    fund_codes = list(store.data["funds"].keys())
    return fetch_news_and_events(horizon_days, fund_codes)

def calc_redemption_fee(fund_code: str, shares_to_sell: float, lot_order: str = "FIFO") -> dict:
    store = _get_store()
    return compute_redemption_fee(fund_code, shares_to_sell, lot_order, store.data, date.today())

def check_rules() -> dict:
    store = _get_store()
    config = load_config()
    market = fetch_market_snapshot()
    events = fetch_news_and_events(7, list(store.data["funds"].keys()))
    return {"results": run_rules(store.data, market, events, config)}

def generate_decision_brief() -> dict:
    """聚合所有数据，一次性返回完整 brief"""
    from datetime import datetime
    store = _get_store()
    fund_codes = list(store.data["funds"].keys())

    holdings = list_holdings()
    realtime = get_fund_realtime(fund_codes)
    market = get_market_snapshot()
    events = get_news_and_events(7)
    rules_result = check_rules()

    now = datetime.now()
    cutoff = now.replace(hour=15, minute=0, second=0)
    minutes_left = max(0, int((cutoff - now).total_seconds() / 60))

    return {
        "as_of": now.isoformat(),
        "minutes_to_cutoff": minutes_left,
        "holdings": holdings,
        "realtime": realtime,
        "market": market,
        "events": events,
        "rule_results": rules_result["results"],
        "data_quality": _assess_data_quality(realtime, fund_codes)
    }

def _assess_data_quality(realtime: dict, codes: list[str]) -> list[str]:
    notes = []
    for code in codes:
        info = realtime.get(code, {})
        if info.get("is_qdii"):
            notes.append(f"{code} 是 QDII，盘中估值不可信")
    return notes
```


### 7.4 Holdings Store（JSON 文件读写）

```python
# src/fund_advisor/holdings.py
import json
import os
from datetime import date
from typing import Optional

DEFAULT_PATH = os.path.expanduser("~/.fund-advisor/holdings.json")

class HoldingsStore:
    def __init__(self, path: str = None):
        self.path = path or os.environ.get("FUND_ADVISOR_HOLDINGS", DEFAULT_PATH)
        self.data = self._load()

    def _load(self) -> dict:
        if not os.path.exists(self.path):
            return {"$schema": "holdings-v1", "available_cash_cny": 0, "funds": {}}
        with open(self.path) as f:
            return json.load(f)

    def _save(self):
        from datetime import datetime
        self.data["updated_at"] = datetime.now().isoformat()
        os.makedirs(os.path.dirname(self.path), exist_ok=True)
        with open(self.path, "w") as f:
            json.dump(self.data, f, ensure_ascii=False, indent=2)

    def add_lot(self, fund_code: str, confirm_date: str, amount_cny: float,
                shares: Optional[float] = None, nav_at_buy: Optional[float] = None):
        if fund_code not in self.data["funds"]:
            raise ValueError(f"基金 {fund_code} 不存在，请先通过 parse_holdings_image 添加")
        lot = {"confirm_date": confirm_date, "amount_cny": amount_cny}
        if shares is not None:
            lot["shares"] = shares
        if nav_at_buy is not None:
            lot["nav_at_buy"] = nav_at_buy
        self.data["funds"][fund_code]["lots"].append(lot)
        self._save()

    def upsert_fund(self, code: str, item: dict, fee_table: list):
        if code not in self.data["funds"]:
            self.data["funds"][code] = {
                "name": item.get("fund_name_zh", ""),
                "share_class": self._guess_share_class(item.get("fund_name_zh", "")),
                "fund_type": "",
                "is_qdii": False,
                "fee_table": fee_table,
                "lots": [],
                "target_hold_days": 180,
                "take_profit_rate": 0.30,
                "stop_loss_rate": -0.15,
            }
        self._save()

    def list_with_computed_fields(self, today: date) -> dict:
        result = []
        for code, fund in self.data["funds"].items():
            lots_computed = []
            for lot in fund["lots"]:
                confirm = date.fromisoformat(lot["confirm_date"])
                holding_days = (today - confirm).days
                lots_computed.append({**lot, "holding_days": holding_days})
            result.append({
                "fund_code": code,
                **{k: v for k, v in fund.items() if k != "lots"},
                "lots": lots_computed,
            })
        return {"funds": result, "available_cash_cny": self.data.get("available_cash_cny", 0)}

    @staticmethod
    def _guess_share_class(name: str) -> str:
        if name.endswith("A"):
            return "A"
        if name.endswith("C"):
            return "C"
        return "A"
```

### 7.5 赎回费计算

```python
# src/fund_advisor/fee/calculator.py
from datetime import date

def compute_redemption_fee(
    fund_code: str,
    shares_to_sell: float,
    lot_order: str,
    holdings_data: dict,
    today: date
) -> dict:
    fund = holdings_data["funds"].get(fund_code)
    if not fund:
        return {"error": f"基金 {fund_code} 不存在"}

    fee_table = fund["fee_table"]
    lots = sorted(fund["lots"], key=lambda l: l["confirm_date"],
                  reverse=(lot_order == "LIFO"))

    if lot_order == "MIN_FEE_FIRST":
        lots = sorted(lots, key=lambda l: _rate_for_days(fee_table, _days(l, today)))

    remaining = shares_to_sell
    breakdown = []
    total_fee = 0

    for lot in lots:
        if remaining <= 0:
            break
        shares_available = lot.get("shares", 0)
        if shares_available <= 0:
            continue
        take = min(shares_available, remaining)
        held_days = _days(lot, today)
        rate = _rate_for_days(fee_table, held_days)
        # 用买入净值估算（精确值需要当前净值，但这里给个保守估计）
        nav = lot.get("nav_at_buy", 1.0)
        gross = take * nav
        fee = gross * rate
        breakdown.append({
            "confirm_date": lot["confirm_date"],
            "shares": take,
            "held_days": held_days,
            "rate": rate,
            "fee_cny": round(fee, 2),
        })
        total_fee += fee
        remaining -= take

    # 优化建议
    hint = _optimization_hint(fund, fee_table, today)

    return {
        "fund_code": fund_code,
        "shares_requested": shares_to_sell,
        "shares_filled": shares_to_sell - remaining,
        "total_fee_cny": round(total_fee, 2),
        "lot_breakdown": breakdown,
        "optimization_hint": hint,
    }

def _days(lot: dict, today: date) -> int:
    return (today - date.fromisoformat(lot["confirm_date"])).days

def _rate_for_days(table: list, days: int) -> float:
    for tier in table:
        max_d = tier.get("max_days") or 10**9
        if tier["min_days"] <= days < max_d:
            return tier["rate"]
    return 0.0

def _optimization_hint(fund: dict, fee_table: list, today: date) -> str:
    hints = []
    for lot in fund["lots"]:
        held = _days(lot, today)
        current_rate = _rate_for_days(fee_table, held)
        # 看下一个 breakpoint
        for tier in sorted(fee_table, key=lambda t: t["min_days"]):
            if tier["min_days"] > held and tier["rate"] < current_rate:
                days_to_wait = tier["min_days"] - held
                hints.append(
                    f"确认日{lot['confirm_date']}的份额再等{days_to_wait}天，"
                    f"费率从{current_rate*100:.1f}%降到{tier['rate']*100:.1f}%"
                )
                break
    return "; ".join(hints) if hints else "当前所有份额已在最低费率区间"
```


---

## 8. 测试策略

### 8.1 必须有的单元测试

| 模块 | 关键用例 |
|---|---|
| `fee/calculator.py` | <7天/=7天/=30天/>1年临界值；FIFO/LIFO排序；份额不够时部分成交 |
| `holdings.py` | 持有天数计算（含周末）；add_lot 写入后可读回 |
| `rules/engine.py` | 每条硬规则触发/不触发；多规则同时触发 |

### 8.2 冒烟测试

- 用 `tests/fixtures/sample_holdings.json` 跑 `generate_decision_brief()`
- 断言：输出含 `rule_results`、`market`、`holdings`、`events` 四个字段
- 断言：H1 blocker 对 <7 天 lot 触发

---

## 9. 风险与限制

| 风险 | 缓解 |
|---|---|
| AKShare 接口偶尔不稳 | 捕获异常，返回 `data_quality` 标注"数据缺失" |
| 盘中估值不准（主动/QDII） | 在 brief 里强制标注 |
| 截图 OCR 误识别 | 标记 `needs_confirm=true`，人工确认 |
| 交易日历（周末/节假日） | 用 `chinese-calendar` 库 |
| LLM 不调工具直接编造 | 系统 prompt 硬约束 + MCP tool 描述强调"必须调用" |
| 合规 | 输出末尾固定免责声明 |

---

## 10. 路线图

### Phase 1（MVP，1-2 天）

- [ ] `holdings.json` 读写 + lot 持有天数计算
- [ ] 赎回费计算（FIFO）
- [ ] 3 条硬规则（H1/H2/H3）
- [ ] AKShare 封装（估值 + 指数 + 北向）
- [ ] MCP server + CLI (`brief` / `holdings` / `fee` / `rules`)
- [ ] 手动维护持仓（通过 `add-lot` CLI 或 MCP 工具）

### Phase 2（1 周）

- [ ] 持仓截图识别（依赖 LLM vision + `parse_holdings_image`）
- [ ] 完整规则集（S1-S6）
- [ ] 新闻 + 财报日历
- [ ] `generate_decision_brief` 聚合
- [ ] Python API（`api.py`）供 OpenClaw import

### Phase 3（按需）

- [ ] 决策日志（`decision_log.jsonl`）
- [ ] 复盘：对比建议 vs 实际操作 vs 实际收益
- [ ] 推送通知（14:30 提醒）
- [ ] 多账户/多人持仓

---

## 11. 给实现 Agent 的交付物清单

实现完成后应当交付：

- [ ] `uv pip install -e .` 后 `fund-advisor-server` 和 `fund-advisor` 两个命令可用
- [ ] `fund-advisor init` 可创建 `~/.fund-advisor/holdings.json`
- [ ] `fund-advisor brief` 输出有效 JSON
- [ ] `claude mcp add fund-advisor -- uv run fund-advisor-server` 后 Claude Code 可调用所有工具
- [ ] `pytest` 全绿
- [ ] `README.md` 含 5 分钟上手说明
- [ ] `config/rules.yaml` + `config/targets.yaml` 示例可用

---

## 附录 A：常用 AKShare 函数

```python
ak.fund_individual_basic_info_xq(symbol="005827")     # 基金基础信息
ak.fund_open_fund_info_em(symbol="005827", indicator="单位净值走势")  # 历史净值
ak.fund_value_estimation_em(symbol="全部")             # 全市场盘中估值
ak.fund_fee_em(symbol="005827", indicator="赎回费率")   # 赎回费率表
ak.fund_portfolio_hold_em(symbol="005827", date="2026") # 重仓股
ak.stock_zh_index_spot_em()                            # 实时指数
ak.stock_hsgt_north_net_flow_in_em(symbol="北上")       # 北向资金
ak.stock_board_industry_name_em()                      # 行业板块
ak.news_cls()                                          # 财联社电报
ak.stock_news_em(symbol="600519")                      # 个股新闻
```

## 附录 B：赎回费速查（典型值，以基金合同为准）

| 持有期 | 股票/混合 A 类 | 债券 A 类 | C 类 |
|---|---|---|---|
| <7 天 | ≥1.5% | ≥1.5% | ≥1.5% |
| 7–30 天 | 0.50% | 0.10% | 0% |
| 30 天–1 年 | 0.25% | 0.05% | 0% |
| ≥1 年 | 0% | 0% | 0% |

## 附录 C：参考资料

- AKShare 文档：https://akshare.akfamily.xyz/data/fund/fund_public.html
- MCP Python SDK：https://github.com/modelcontextprotocol/python-sdk
- chinese-calendar：https://github.com/LKI/chinese-calendar
- 证监会《公开募集开放式证券投资基金流动性风险管理规定》

---

**END OF SPEC**
