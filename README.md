# Stock Agent Pipeline

Stock Agent Pipeline 是一个以 Skill 为核心的 AI 主观选股与模拟调仓项目。

项目主线：

```text
查询模拟仓事实源现状
  -> 数据 Skill 收集 A 股候选和热点
  -> 主智能体按基础硬限制形成候选池
  -> 按 style-contract.md 启用风格 Skill 做主观筛选
  -> 主智能体判断买入、卖出、持有、空仓、观察或不动
  -> 非仓位类硬纪律检查
  -> 若需要交易，调用 account-source.md 定义的模拟仓事实源执行模拟买卖
  -> 若发生交易，再次查询模拟仓事实源
  -> 生成本次运行的独立调仓报告
```

## 当前目标

- 先完成一个小而可用的 AI 主观选股 MVP。
- 首版只做 A 股。
- 首版默认每日运行一次，但架构允许同日多次运行。
- 支持盘前、盘中、收盘后和临时复查四类运行。
- 启用风格以 `skills/stock-agent-pipeline/references/style-contract.md` 中的风格契约为准。
- 模拟仓事实源以 `skills/stock-agent-pipeline/references/account-source.md` 中的事实源契约为准。
- 模拟调仓模式表示允许交易，不表示必须交易；风格判断可以选择空仓、持有、观察或不动。
- 本地不维护模拟仓资金、持仓、委托或成交账本。
- 本地只保存 AI 决策记忆和调仓报告，不作为模拟仓事实源；记忆保持精简，只保留重要判断和关键复盘。

## 报告保存规则

每次运行都保存一份独立报告，避免同日多次看盘时覆盖过程信息：

```text
reports/YYYY-MM-DD/HHmm-adjustment-report.md
reports/YYYY-MM-DD/daily-summary.md
reports/YYYY-MM-DD/daily-dashboard.html
```

具体运行时间由外部调用或 Codex 自动化控制；项目只按实际运行时间保存报告。同一分钟重复运行时，使用 `HHmmss-adjustment-report.md`。如果交易型任务实际执行时间与计划时段明显偏离，项目降级为选股研究或临时复查，不执行模拟买卖。`daily-summary.md` 只汇总当天 AI 判断、模拟交易结果和复查重点，不记录或替代模拟仓事实源账户状态。`daily-dashboard.html` 是可选可视化聚合视图，用于承接详细表格、时间线、持仓明细和可视化复盘，不替代 Markdown 报告或模拟仓事实源。

## Codex 自动化时间说明

在 Codex 自动化的自定义 RRULE 中，时间按 UTC 等价配置。这个规则只适用于 Codex 自动化，不是项目通用时间规则。例如北京时间 `08:00 / 09:35 / 14:30 / 20:00`，应配置为 UTC `00:00 / 01:35 / 06:30 / 12:00`。

## 非目标

- 不接真实券商。
- 不接 QMT。
- 不做港股通模拟交易。
- 不恢复 Python 脚本化流水线。
- 不做本地回测、绩效归因或因子消融。
- 不把本地文件当作模拟仓真实状态。

## 触发方式

在 Codex 中使用项目级 Skill：

```text
使用 $stock-agent-pipeline 运行今天的 AI 主观选股模拟调仓。
```

模拟调仓模式会先判断是否值得交易；如果结论是空仓或继续观察，也必须生成报告，但不会调用买卖接口。

也可以明确要求只查询不交易：

```text
使用 $stock-agent-pipeline 做今天的查询模式，只查资金、持仓、委托，不买卖。
```

如果需要可视化日报，可以明确要求：

```text
使用 $stock-agent-pipeline 生成今天的可视化日报 HTML 页面。
```

Markdown 调仓报告默认保持简短，重点说明本次动作、核心理由、关键风险、触发条件和失效条件；完整表格和可视化复盘优先放入 `daily-dashboard.html`。

## 项目结构

```text
AGENTS.md
README.md
reports/
  memory/
    trading-memory.md
  YYYY-MM-DD/
    HHmm-adjustment-report.md
    daily-summary.md
    daily-dashboard.html
skills/
  stock-agent-pipeline/
    SKILL.md
    agents/openai.yaml
    references/
      workflow.md
      hard-rules.md
      style-contract.md
      account-source.md
      memory-rules.md
      report-template.md
      dashboard-report.md
```

## 关键依赖

- 模拟仓事实源：以 `account-source.md` 为准；具体事实源、调用位置和认证方式只在该契约文件中维护。
- 数据 Skill：使用当前环境可用的数据能力收集行情、主题、行业、新闻、公告、事件和候选线索；具体工具不写入主流程。
- 风格 Skill：以 `style-contract.md` 为准；具体风格只在该契约文件中维护。

## 安全边界

- 模拟仓事实源以 `account-source.md` 为准。
- `reports/memory/trading-memory.md` 是 AI 决策记忆，不是账户账本。
- `reports/YYYY-MM-DD/daily-dashboard.html` 是可选可视化聚合视图，不是账户账本。
- 每次买卖前必须查询模拟仓事实源当前资金、持仓、委托。
- 每次买卖后必须再次查询模拟仓事实源当前资金、持仓、委托；交易后状态确认优先使用稳定的后端查询方式。
- 模拟仓事实源状态不明时不交易；`委托已提交` 不等于 `已成交`，只有持仓或委托/成交状态确认后才能写成已成交。
- 无论是否发生买卖，每次运行都必须生成独立报告。
- 候选池基础硬限制：A 股 6 位代码、非 ST、非退市整理、非停牌、基础字段完整、成交额不低于 1 亿元、按代码去重；高风险标的可以进入研究，但必须明确说明风险。
- 买卖失败、非交易时间、资金不足或接口错误，只写入报告，不做本地补单。
- 不打印或保存模拟仓事实源密钥、token 或其他认证信息。
