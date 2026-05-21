# Fund AI Advisor

场外基金 AI 辅助决策系统。把大模型从"决策者"降级为**纪律检查员 + 事实核对员 + 方案解释员**。

## 定位

- ✅ 帮你算清赎回费、持有天数、规则触发
- ✅ 把实时数据 / 新闻 / 政策日历送到 LLM 嘴边
- ✅ 通过 MCP / CLI / Python import 三种方式对外暴露
- ❌ 不预测涨跌、不自动下单

## 支持的调用方式

| 工具 | 接入方式 |
|---|---|
| Claude Code CLI | `claude mcp add fund-advisor -- uv run fund-advisor-server` |
| Hermes | MCP server 配置 |
| OpenClaw | `from fund_advisor.api import get_decision_brief` 或 shell 调 `fund-advisor brief` |
| 手动 | `fund-advisor brief` 输出 JSON → 复制到任意 LLM |

## 快速开始

```bash
git clone <repo>
cd fund-ai-advisor
uv venv && source .venv/bin/activate
uv pip install -e .

# 初始化
fund-advisor init

# 添加持仓
fund-advisor add-lot 005827 --confirm-date 2026-05-10 --amount 5000 --shares 1234.56

# 获取决策 brief
fund-advisor brief

# 接入 Claude Code
claude mcp add fund-advisor -- uv --directory $(pwd) run fund-advisor-server
```

## 每日使用流程

1. 14:30 在 Claude Code 中输入"今天基金怎么操作"
2. AI 自动调 `generate_decision_brief` 获取实时数据+规则检查
3. AI 输出决策矩阵（保持/加仓/减仓/观察）
4. 你在支付宝中执行

## 文档

- **[SPEC.md](./SPEC.md)** — 完整工程方案（实现直接参照此文档）
- `config/rules.yaml` — 规则配置
- `config/targets.yaml` — 目标仓位配置

## 数据存储

一个 JSON 文件：`~/.fund-advisor/holdings.json`，不用数据库。

## License

MIT
