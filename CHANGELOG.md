# 更新记录

## v1.1.3 — 2026-09-18

**把「大工具结果」真正挡在上下文外**（此前只是契约里请模型自己别一次打那么多）。

- `spill-policy.maxInlineBytes`：12000 → **4000**。超过 4 KB 的非 `read` 结果当场落盘为会话工件，
  模型眼前只剩「头 2KB + 尾 2KB + locator + 省略字节数」，需要时按 locator 取全文。
- `tool-fs` 新增 `readLimit: 300` / `readMaxBytes: 8192` / `readMaxLineLength: 1000`
  （原为 2000 行 / 50 KB / 每行 2000 字符）。**`read` 不参与 spill** —— `dsh-spill-policy` 的
  model-facing arm 开头就是 `exec.name === "read"` 直接返回（避免 `read → spill → 再 read` 死循环），
  所以要给 read 设笼子，只能落在这行配置上。
- 副作用（有意为之）：`skill` 之类的工具结果超过 4 KB 也会被收口成预览 —— 加载一份 14 KB 的 skill
  会变成「预览 + 取工件」两步。嫌碍事就把 4000 调回 8000，只改这一个数。

其他行一字未改；宿主那份 `50000` 不属于本 preset，其他模式的行为一如既往。

## v1.1.2 — 2026-09-18

**再补一条：不许盲开文件。** 依据是一次真实会话（1 轮 43 步，极压模式）的逐步归因：终局 59,862 tok，
只有 1M 窗口的 **6.0%**，离 100K 压缩线还差 40,138 —— 全程**没有触发任何压缩**（无 compaction、无 spill、无 prune）。
增长 +49,185 的来源是：

- **模型自己的输出 33,272 tok（占 68%，其中推理 18,806）**——最大一跳是某步 `todo_write` 花了 8,248 推理 token，
  下一步 prompt 直接 +8,455；
- 工具结果 48 条 × 51.6 KB ≈ 12,890 tok（占 26%）。

即 v1.1.1 那两条最多只管住这 26%。而在那 26% 里，7 次 `read` 有 6 次是窄窗（65/45/30/22/16/14 行），
**唯独首次进场是 `offset=1 limit=110` / 7.8 KB** —— 把 `read` 当侦察工具、从第 1 行读起看「这文件里有什么」，
是唯一一次超标的读。于是新增契约：

- **不盲开文件**：`read` 不是侦察工具。先 `wc -l` 取行数、`grep -n` 找锚点，再让第一个窗口从关键行开始往后读，
  单窗 ≤60 行；多个定位窗口胜过一次大读。

已知边界（本条不解决）：persona 契约只约束**写出来的文本**，管不到推理 token——那次 8,248 token 的
`todo_write` 就是例子。推理量由 `reasoningEffort`（本机为 `high`，来自 host/profile 默认，不在本 preset 内）决定。

`spill-policy.maxInlineBytes` 依旧 12000 未动，preset 其他行一字未改。

## v1.1.1 — 2026-09-18

**persona 契约加严两条**，依据是一次真实会话（8 步）的逐步 token 归因：新增 token 只有三个来源 ——
冷启动前缀（占 47%）、工具结果、模型自己的长输出。该会话里单笔最大的两次工具结果是
`read offset=81 limit=91` 带回 **11 KB**（≈3.4K token）、`sed -n '60,200p'` 带回 **5.5 KB**（≈1.4K token），
它们就是那两步的主要开销。

- **先定位、再窄读**：先 `grep -n`/`glob`/`wc -l` 定位，再读最小窗口（`offset`/`limit`、`sed -n 'A,Bp'`）；
  单次读目标 ≤ 约 60 行 / 4 KB。答案不在窗口里就**再窄一点重读**，而不是把范围放大。
- **输出先进笼子再进上下文**：命令先 `wc -l`/`head -n 20`/`grep -c` 探路，只打印需要的那一段；
  单条命令输出上限约 4 KB。
- 契约里更正一句实话：**pruner 只在压缩那一刻运行**（`compaction-basic` 在 `compactIfNeeded()` 内调用
  `pruneSession()`），在那之前模型打印的每个字节都原样留在上下文里。原文案暗示「结果总会被裁到头尾」，
  是错的。
- `spill-policy.maxInlineBytes` 保持 12000 未动 —— 那是 harness 侧兜底，本次不改配置。

preset 的其他行一字未改。

## v1.1.0 — 2026-09-18

**本仓库现在同时是一个 DSH 插件包**：新增 `package.json`（声明 `dsh.bundle.patch`）与
`cordis.patch.yml`，preset 本体移到 `presets/extreme/`。于是它可以从 GitHub 直接装进 DSHA 的
插件系统——插件页勾选/取消勾选即可装卸，不必手工往 `.agent-presets/` 拷目录。

- `cordis.patch.yml` 只改宿主一行：给 `agent-presets` 补 `roots`（包内 `presets/`，`trust: system`）。
  该行的 config 补丁在 loader 里是**整段替换**（用 `dsh --profile web --patch … --dump-config` 实测确认），
  所以 `default: standard` 一并写全，否则该行的必填项会丢。
- 根的 trust 用 `system`（只读）而不是 `user`：`copy()` 新建 preset 取的是第一个 `user` 根
  （`$DSH_HOME/.agent-presets`），不能把它抢走；顺带让插件里的 preset 不能被 preset 界面删除。
- **文档**：README 开头补「好处与代价」；「约 100K」澄清为**下一次请求的完整注入量**
  （系统提示 + 工具 schema + 消息；本机实测固定开销 ≈13.8K token、留给消息约 86K），
  真实峰值 = 触发线 + 单步几 K（中文增量被 `CHARS_PER_TOKEN = 4` 低估约 2~4 倍）。
- **手装路径变了**：仓库根不再是 preset 目录，老命令 `git clone … .agent-presets/extreme` 不再成立；
  README 给的是「克隆后把 `presets/extreme` 拷进用户根」的新命令。

preset 本体（`presets/extreme/agent.cordis.yml`）与 v1.0.0 逐字相同，一行未改。

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
