---
name: stock-agent-pipeline
description: "运行或扩展以 Skill 为核心的 A 股 AI 主观选股与模拟调仓流程。适用于 AI 主观选股、A 股候选筛选、风格契约判断、模拟组合调仓、模拟仓事实源执行、持仓复查、尾盘调仓、复盘与持久化记忆或生成调仓报告。启用风格以 style-contract.md 为准，模拟仓事实源以 account-source.md 为准，并为每次运行保存本地报告。"
---

# Stock Agent Pipeline

## 概览

将本 Skill 作为 AI 主观选股和模拟组合调仓的总控入口。

首版默认值：

- 市场范围：模拟交易只处理 A 股。
- 启用风格：以 `references/style-contract.md` 中的风格契约为准。
- 模拟仓事实源：以 `references/account-source.md` 中的事实源契约为准。
- 本地持久化：每次运行在 `reports/YYYY-MM-DD/` 下保存一份调仓报告，并维护 `reports/memory/trading-memory.md` 作为 AI 决策记忆。
- 运行频率：默认每日一次，同时支持盘前、开盘确认、尾盘调仓、收盘复盘和临时复查；具体时间由外部调用控制。
- 风险模式：默认 `safe_mode`（安全模式）；只有用户或调度 prompt 明确要求提高风险偏好、增加交易样本或验证框架有效性时，才使用 `exploration_mode`（模拟探索模式）。
- 仓位判断：由主智能体和风格判断基于当前模拟仓事实源状态主观决定，不使用本地固定仓位上限。
- 模拟调仓模式允许交易，但不要求必须交易；最终可以是空仓、持有、观察或不动。
- 报告与记忆：Markdown 报告保持精简，详细表格优先交给可视化日报；`trading-memory.md` 只保留关键判断和重要复盘。

## 硬门禁

- 不使用真实券商、QMT、真实资金交易或港股通交易。
- 不把本地文件当作真实模拟仓持仓、资金、委托或成交。
- 每次模拟买入或卖出前，必须按 `references/account-source.md` 查询持仓、资金和委托。
- 每次模拟买入或卖出后，必须按 `references/account-source.md` 再次查询持仓、资金和委托。
- 账户状态不明时不交易。
- `委托已提交` 不等于 `已成交`，必须由模拟仓事实源状态确认。
- 模拟交易只允许 A 股 6 位股票代码。
- 候选池必须通过基础硬限制：A 股 6 位代码、非 ST、非退市整理、非停牌、基础字段完整、成交额不低于 1 亿元、按代码去重；高风险标的必须明确说明风险。
- 每个自然交易日新买入目标不超过 3 只。
- 买卖数量必须是 100 股整数倍；如模拟仓事实源返回更高最小委托数量，不得自动放大交易规模，必须重新判断。
- 如果模拟仓事实源拒绝或无法完成操作，只记录失败，不伪造成交。
- 无论是只查询、只选股、空仓、不动还是发生交易，都必须保存最终报告。
- 同日报告不得覆盖。使用 `HHmm-adjustment-report.md`；同一分钟多次运行时使用 `HHmmss-adjustment-report.md`。
- 如果交易型任务实际执行时间与计划时段明显偏离，降级为选股研究或临时复查，不执行模拟买卖，并在报告中说明。
- 不打印或持久化模拟仓事实源密钥、token 或其他认证信息。

## 默认流程

1. 读取 `references/workflow.md`。
2. 按 `references/account-source.md` 查询当前资金、持仓和委托。
3. 按 `references/memory-rules.md` 读取历史报告和 `trading-memory.md`，生成本次决策记忆上下文。
4. 使用可用数据 Skill 收集 A 股市场环境、热点和候选。
5. 按 `references/hard-rules.md` 的基础硬限制形成候选池。
6. 使用 `references/style-contract.md` 中定义的启用风格，并传入风险模式、当前状态、候选、市场信息、现金基准和决策记忆上下文。
7. 判断 `buy`、`trial_buy`、`watch_only`、`reject`、`hold`、`sell_all`、`trim` 或 `no_action`。
8. 执行 `references/hard-rules.md` 中的非仓位类硬规则。
9. 如果最终结论为空仓、`watch_only`、持有、拒绝或不动，跳过交易执行并继续生成报告。
10. 如果最终结论需要 `buy`、`trial_buy`、卖出或减仓，调用模拟仓事实源，随后再次查询事实源状态。
11. 使用 `references/report-template.md` 保存本次运行报告。
12. 报告完成后，按 `references/memory-rules.md` 更新 `reports/memory/trading-memory.md`。

## 运行时智能体职责

- 主智能体：总控完整流程、做最终判断、执行硬门禁，并确保报告落盘。
- 数据发现智能体：使用当前环境可用的数据 Skill 形成候选与上下文证据；候选发现应拆成简单行业、主题或事件查询，再由主智能体汇总和硬过滤。
- 风格研究智能体：按 `references/style-contract.md` 使用启用风格，只评估候选池和当前持仓。
- 模拟执行智能体：按 `references/account-source.md` 查询交易前后账户状态，并执行模拟买卖。
- 复盘与持久化记忆智能体：读取最近报告和 `trading-memory.md`，生成本次决策记忆上下文，并在报告完成后更新记忆。
- 报告整理智能体：无论是否发生交易，都写入最终调仓报告；Markdown 报告保持精简，详细表格优先交给可视化日报。

如果当前环境没有子智能体工具，则由主智能体内联完成这些职责，但仍保持职责边界。

## 引用地图

- `references/workflow.md`：盘前、开盘确认、尾盘调仓、收盘复盘和临时复查的运行流程。
- `references/hard-rules.md`：非仓位类硬规则和安全边界。
- `references/style-contract.md`：单一风格接入契约。
- `references/account-source.md`：模拟仓事实源契约。
- `references/memory-rules.md`：复盘与持久化记忆模块规则。
- `references/report-template.md`：报告必备结构。
- `references/dashboard-report.md`：可选 HTML 可视化日报规则。

具体模拟仓事实源只由 `references/account-source.md` 决定；具体投资风格 Skill 只由 `references/style-contract.md` 决定。其他文档和运行流程只引用契约，不硬编码具体事实源或风格。

## 完成标准

- 报告存在于 `reports/YYYY-MM-DD/HHmm-adjustment-report.md` 或 `reports/YYYY-MM-DD/HHmmss-adjustment-report.md`。
- 报告写明本次风险模式：`safe_mode` 或 `exploration_mode`。
- 报告包含交易前模拟仓事实源的资金、持仓和委托状态。
- 如果发生任何模拟交易，报告还包含交易后模拟仓事实源的资金、持仓和委托状态。
- 报告说明候选来源、风格判断、交易动作、失败情况、持有周期、卖出条件和失效条件。
- 报告包含上次决策复盘、候选比较表和记忆更新摘要。
- 模拟调仓或选股研究后，更新 `reports/memory/trading-memory.md`；该文件不得作为账户账本，且必须保持精简，不重复记录无新增信息的内容。
- 如果同日存在多份报告，可以用 `reports/YYYY-MM-DD/daily-summary.md` 汇总当天 AI 决策和复查重点，但不得把它当作账户账本。
- 如果收盘后最后一次运行或用户明确要求可视化日报，可以按 `references/dashboard-report.md` 生成 `reports/YYYY-MM-DD/daily-dashboard.html`；HTML 不替代 Markdown 报告或模拟仓事实源。
- 未使用任何真实交易路径。
