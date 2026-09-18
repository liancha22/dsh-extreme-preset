# 更新记录

## v1.0.0 — 2026-09-18

首个公开版本。

**好处**

- 每步重发的上下文压到约 100K：长会话的 token 曲线基本走平，不再越跑越贵。
- 模型写下的字被约束：状态、摘要、注记一律英文、≤40 词、只留事实，不复述工具结果和 diff。
- 状态与历史分离：todo 是唯一状态存储（整表替换、只允许一个 `in_progress`），旧上下文折叠成有界摘要。
- 原始内容仍可回取：被压掉的工具结果全文落盘为会话级工件，可按 locator 读回。
- 其他模式零影响：宿主组成一行没改，只有加入本 preset 的会话受这套预算约束。
- 改动面小、可回滚：与标准模式的差异只有七处 `config` 值加一段 persona，卸载就是删一个目录。

**代价**

- 保留窗口（≈50K）之前的逐字内容会丢，只剩摘要；被压掉的工具结果只剩头 1024 / 尾 512 字符。
- 12KB~50KB 的工具结果在 preset 这层落盘一次（一跳）；>50KB 的结果会在宿主那层之后再多落一次盘
  （两跳，全文仍可达）。原因与顺序见 README。
- `compaction-basic` 的两个比例是按 1M 窗口反算出约 100K 预算的。换小窗口模型必须重算，
  否则会频繁压缩、反而更贵（README 给了 `retainTokens` 的绝对值写法）。
- 不适合需要精确回溯全部原始输出的任务（例如逐行比对两份大文件）。

**它是什么**：DSH 的 agent preset。把随发行版附带的「标准模式」（preset id `standard`）按
《极致上下文管理插件 · 精简方案》的 S0–S6 布局重新调参：约 100K 的上下文预算、工具结果只留头尾、
旧上下文快速折叠成 ≤4K 英文摘要，并用 persona 契约约束模型每一步写下的字。

**与标准模式的七处差异**

| 行 | 标准模式 | 极压模式 |
| --- | --- | --- |
| `persona.prefix` | 一句话身份 | 追加极压契约（英文、逐字稳定、整段命中缓存） |
| `agent-instructions.maxBytes` | 65536 | 16384 |
| `tool-todo.allowParallelInProgress` | 允许多个 | 只允许一个（todo 成为唯一的覆盖式状态存储） |
| `compaction-basic` | 0.8 / 0.16 / 8192 | 0.1 / 0.05 / 4096 |
| `compaction-tool-result-pruner` | 8192 / 4096 / 1024 | 4096 / 1024 / 512 |
| `tool-fs-search` | glob 100 / grep 250 / 行 2000B / meta 64KB | glob 50 / grep 100 / 行 1000B / meta 16KB |
| `spill-policy` | 无此行 | 新增，`maxInlineBytes: 12000` |

行的集合、顺序、`isolate` realm 划分与 `standard` 一致，只改 `config` 值加一行。

**不影响其他模式**：宿主组成与 `profiles/*/cordis.patch.yml` 一律未改，宿主那份 `spill-policy`
保持默认值；全部改动都挂在 preset 自己的 standing scope 上，只覆盖加入该 preset 的 agent。

**验证**

- YAML 用 loader 自己的方言（含 `!!js`）解析通过：顶层 19 行、展开 29 行。
- 每个 `name:` 都能在部署里解析到；每个 `config` 都通过对应插件的加载期校验。
- `agentPresets.standingKeyFor('extreme')` → `MOUNTED OK`，29 行全部真正激活
  （同时排除：包解析不了、config 不合法、行 waiting for service、服务被发布到 root realm）。

**兼容**：实测 `@deepseek-ai/dsh` 0.1.5-rc.2，web profile。
