# Stock Agent Pipeline 项目协作规则

## 默认沟通

- 默认使用中文。
- 表达保持简洁、直接、可验证。
- 不编造数据源、接口、模拟仓状态、持仓、订单或成交结果。
- 涉及不确定结论时标明置信度。

## 项目目标

本项目沉淀一个以 Skill 为核心的 AI 主观选股与模拟调仓框架：

- A 股为主，首版模拟交易只处理 A 股 6 位代码。
- 项目级大 Skill 负责总控。
- 主智能体调度数据 Skill、投资风格 Skill 和模拟仓事实源。
- 启用风格以 `skills/stock-agent-pipeline/references/style-contract.md` 中的风格契约为准。
- 模拟仓事实源以 `skills/stock-agent-pipeline/references/account-source.md` 中的事实源契约为准。
- 本地只保存调仓报告和 AI 决策记忆，不维护本地模拟仓账本。
- 模拟调仓模式表示允许交易，不表示必须交易；可以空仓、持有、观察或不动。

## 首版硬边界

- 允许调用 `account-source.md` 定义的模拟仓事实源做模拟仓查询、模拟买入、模拟卖出和委托查询。
- 不接真实券商。
- 不接 QMT。
- 不做港股通模拟交易。
- 不设置固定单股仓位上限、总持仓数量上限或重复买入禁令；仓位由主智能体和风格 Skill 基于模拟仓事实源当前状态判断。
- 不把本地文件当作模拟仓真实持仓。
- 不打印、写入或泄露模拟仓事实源密钥、token 或其他认证信息。

## 智能体职责边界

- 主智能体：负责触发流程、分发子任务、汇总判断、执行非仓位类硬纪律、调用模拟仓事实源并生成调仓报告。
- 数据发现智能体：负责使用当前环境可用的数据 Skill 形成候选与上下文证据；候选发现应拆成简单行业、主题或事件查询，再由主智能体汇总和硬过滤。
- 风格研究智能体：负责按 `style-contract.md` 使用启用风格给出 `buy`、`watch`、`reject`，并给出持有周期和失效条件。
- 模拟执行智能体：负责按 `account-source.md` 查询交易前后状态并执行模拟买卖。
- 报告整理智能体：负责生成唯一调仓报告。

## 文档和实现原则

- Skill 内只保留必要流程，不再恢复旧脚本化框架。
- `skills/stock-agent-pipeline/SKILL.md` 只写总控流程和引用地图。
- 详细规则放在 `skills/stock-agent-pipeline/references/`。
- 每次运行保存一份独立调仓报告到 `reports/YYYY-MM-DD/HHmm-adjustment-report.md`。
- 同日多次运行时不得覆盖前序报告；需要日内汇总时写入 `reports/YYYY-MM-DD/daily-summary.md`。
- `daily-summary.md` 和 `reports/memory/trading-memory.md` 是 AI 决策记忆，不是模拟仓账本。
- 旧脚本化流水线目录不属于当前主线。
- 具体模拟仓事实源只由 `skills/stock-agent-pipeline/references/account-source.md` 决定；其他文件不得硬编码具体事实源名称、路径或调用方式。
- 具体投资风格 Skill 只由 `skills/stock-agent-pipeline/references/style-contract.md` 决定；其他文件不得硬编码具体风格名称或路径。

## 安全要求

- 不写入密钥、token、账号密码、API key 或私密配置。
- 不执行真实下单、撤单、转账或实盘相关操作。
- 使用模拟仓事实源买卖前必须先查询当前资金、持仓和委托。
- 使用模拟仓事实源买卖后必须再次查询当前资金、持仓和委托。
- 无论是否发生模拟买卖，都必须生成调仓报告。
- 买卖失败、未成交、非交易时间、接口错误或资金不足时，只记录报告，不做本地补单。

## 验证要求

文档或 Skill 变更后优先执行最小验证：

- Markdown 静态扫描。
- Skill frontmatter 校验。
- 检查项目主线是 AI 主观选股和模拟调仓。
- 检查真实交易、QMT、港股通模拟交易被明确禁止。
- 检查报告模板包含交易前后模拟仓事实源查询结果。
