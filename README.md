# dsh-extreme-preset · 极压模式

[DSH（DeepSeek Harness）](https://github.com/deepseek-ai/deepseek-harness) 的一个 **agent preset**。

把随发行版附带的「标准模式」（preset id `standard`）按《极致上下文管理插件 · 精简方案》的
S0–S6 布局重新调参：**把每一步都要重发的上下文压到约 100K，并约束模型自己写下的字**。

> 这是 **preset（会话组成）**，不是插件（bundle）。它不装新包、不改宿主组成，
> 全部内容就是一份 `agent.cordis.yml` 里的 `config` 值加一段 persona 文本。
> 所以它既不需要重启，也不可能因为它的存在让别的模式变得不一样。

## 安装

preset 就是一个目录。DSH 从 `<dshHome>/.agent-presets/`（默认 `~/.dsh/.agent-presets/`）扫描本机 preset，
目录名就是 preset id：

```bash
git clone https://github.com/liancha22/dsh-extreme-preset.git \
  "${DSH_HOME:-$HOME/.dsh}/.agent-presets/extreme"
```

不想用 git（没有 SSH key 也行）：

```bash
dest="${DSH_HOME:-$HOME/.dsh}/.agent-presets"
mkdir -p "$dest"
tmp=$(mktemp -d)
curl -sL https://github.com/liancha22/dsh-extreme-preset/archive/refs/tags/v1.0.0.tar.gz | tar -xz -C "$tmp"
mv "$tmp"/dsh-extreme-preset-1.0.0 "$dest/extreme"
```

然后**新建**会话时在 preset 选择器里选「极压模式」即可。不需要重启：roster 每次读取都重新扫描该目录，
standing mount 也按文件戳失效。

> 只有空会话能切换 preset（换了工具组成会让历史里的工具调用对不上），所以是「新建时选」，不是「中途切」。

## 它改了什么（七处，其余与标准模式逐字相同）

| 文档里的一段 | DSH 里对应的行 | 标准模式 | 极压模式 |
| --- | --- | --- | --- |
| S0 System Prompt（永久缓存段） | `persona.prefix` | 一句话身份 | 身份 + 极压契约（英文、逐字稳定、整段命中缓存） |
| S2 指令注入预算 | `agent-instructions.maxBytes` | 65536 | 16384 |
| S3 State（覆盖式） | `tool-todo` | 允许多个 `in_progress` | 只允许一个，todo 即唯一状态存储 |
| S4 Step Log（有界摘要） | `compaction-basic` | 阈值 0.8 / 保留 0.16 / 摘要 8192 token | 阈值 0.1 / 保留 0.05 / 摘要 4096 token |
| S6 Recent Raw（按需、放末尾） | `compaction-tool-result-pruner` | 8192 / 4096 / 1024 字符 | 4096 / 1024 / 512 字符 |
| ArtifactStore（原始内容可回取） | `spill-policy` | 50000（宿主平面默认，未改） | **preset 内 12000** |
| S6 检索结果上限 | `tool-fs-search` | glob 100 / grep 250 / 行 2000B / meta 64KB | glob 50 / grep 100 / 行 1000B / meta 16KB |

行的集合、顺序、`isolate` realm 划分与 `standard` 一致，只在 `tool-fs-search` 之后多了一行
`spill-policy`；其余都是改 `config` 值。这是刻意的：一个已被证明可加载的组成，改动面越小，
重挂失败的面越小。

### persona 契约写了什么

放在 `persona.prefix`（section order = `DEPLOYMENT_PERSONA_PREFIX`，整棵树最靠前）的稳定段里，
每轮逐字相同，因此是 100% 缓存命中段。要点：

- **只留事实**：不复述工具结果、不复述文件与 diff，只留标识符、数字、路径、结论。
- **英文、≤40 词**：凡是写给自己未来看的那一行（状态、摘要、给后续步骤的注记）都必须英文、≤40 词、只讲事实。
- **状态优先于历史**：todo 列表是唯一状态存储，整表替换、≤7 项，不从早前消息里重新推导进度。
- **摘要优先于重读**：读范围（`offset`/`limit`、`sed -n`）而不是整文件；宽输出先 `head`/`tail`；先搜再列。
- **批量**：互不依赖的调用放在同一步。
- **稳定前缀**：指令与工具目录本会话固定，不要求重复、不重写、不在输出里复述。

它**不改变工具调用协议**——DSH 用原生 function calling，不套 JSON 信封，契约只约束模型写下的文本。

## 与文档的对应关系

文档提的三个组件在 DSH 里都有现成实现，不需要新写插件：

- **ArtifactStore**（原始内容保留 300 步）→ `spill-policy` + `spill-local`：结果超限时全文落盘为
  会话级工件，上下文里只留 head/tail 预览和 locator；清理由 `spill-local` 的
  `cleanupPeriodDays`（默认 30 天，启动时一次性清扫）兜底。
- **StepLog**（英文摘要保留 3000 步、满则合并最旧）→ `compaction-basic`：把旧上下文折叠成摘要并
  shadow 掉原始事件，语义就是「有界、追加式、满了合并最旧」；`maxTokens: 4096` 是摘要的硬上限。
- **StateStore**（结构化状态、覆盖式）→ `tool-todo`：整表替换、有界、每步重发。

两点**没有照搬**，是有意的：

1. **不套 JSON 信封**（`thought` / `action` / `action_input` / `result_summary` 那套）。DSH 是原生
   function calling，不是 ReAct 文本协议，强行改成 JSON 输出会破坏工具调用。这条约束被翻译成
   「模型写下的文本必须英文、≤40 词、只留事实」。
2. **没有 `artifact_ref` / `state_patch` 字段**。DSH 的工具体系不接受模型回写元字段；等价能力由
   harness 侧（spill / pruner / compaction）和 todo 承担。

## 为什么 `spill-policy` 放在 preset 里，而不是改宿主

`spill-policy` 在 `dsh-base` 里是**宿主平面**行，默认对所有 preset 生效。要让只有极压模式变紧、
其他模式纹丝不动，做法是在 preset 里再挂一份，宿主那份保持默认不动。之所以可行，是两条代码事实：

1. **它是消费者，不是提供者。** 它只 `inject: ["tools"]`、注册一个 `tools/post-execute` 监听器，
   再 `ctx.get("spillStore")` 取宿主后端；它不提供任何服务。所以它**不能**放进 `isolate` realm——
   包进 realm 连宿主的 `tools` / `spillStore` 都解析不到，它是一行散装 row。
2. **事件在 agent 作用域上触发：**

   ```js
   await this.ctx.waterfall(scopeTarget(this, exec.agent), "tools/post-execute", exec, result, …)
   ```

   投递目标是该次工具调用所属 agent 的作用域，所以挂在 preset standing scope 上的监听器会收到它，
   其他 preset 的 agent 不会。

**宿主那份关不掉。** Cordis 的 `dispatch` 用一张全局监听表加作用域过滤
（`hook.global || !filter || filter.call(thisArg, hook.ctx)`），而根作用域永远是任何 agent 作用域的祖先，
所以宿主那份必然也对这些 agent 触发。配置层面没有任何按 preset 覆盖宿主行的办法
（`!!js` 在 loader 上下文求值，不是按 agent 求值）。

### 两层叠加后的实际行为

`register()` 里 `prepend: true` 就是 `unshift`，两者都用了 prepend：宿主先注册、preset 后注册，
所以 **preset 那份在外层**（先 `next()` 拿到内层结果，再收口）：

| 原始结果大小 | 发生什么 |
| --- | --- |
| ≤12KB | 两层都不动 |
| 12KB~50KB | 宿主内层不触发，preset 外层落盘一次 → **一跳** |
| >50KB | 宿主内层先落盘（工件 A = 全文），preset 外层再把那份预览收口到 ≤12KB（工件 B，含 A 的 locator）→ **两跳，全文仍可达** |

模型眼前的结果始终 ≤12KB，再被 `tool-result-pruner` 压到头 1024 / 尾 512 字符。提示文字是固定模板
（约 100~300 字节），落在尾 512 内，所以 locator 不会被截掉。两跳只是多一次读取。

如果你更想要「一跳但全局收紧」：把本 preset 里的 `spill-policy` 行删掉，改在
`<dshHome>/profiles/<profile>/cordis.patch.yml` 里加

```yaml
- id: spill-policy
  config:
    maxInlineBytes: 12000
```

代价是所有模式一起变紧，且该 profile 默认 `patchReload: startup`，需要重启宿主才生效。

## 「窗口约 100K」是怎么算出来的

文档的目标是固定约 100K 的峰值窗口。DSH 的压缩阈值是**比例**而非绝对值，所以按路由的上下文窗口反算：

```
thresholdRatio 0.1  × 1,000,000 ≈ 100K   触发压缩的表面 token（文档的峰值窗口）
retainRatio    0.05 × 1,000,000 ≈  50K   压缩后逐字保留的近期上下文
maxTokens 4096                           摘要本身的上限（文档的有界 step log）
```

当前实测部署的路由是 `deepseek-official / deepseek-flash`，窗口按 1M 计，于是落在文档的 100K 上。

**换模型要重算。** 比例会同比缩放：128K 窗口的模型在同样配置下变成 12.8K 触发、6.4K 保留——
那已经过头了，会频繁压缩、反而更贵。要固定绝对预算，把 `compaction-basic` 里那两行换成：

```yaml
        thresholdRatio: 0.3
        retainTokens: 24000
```

唯一约束是 `retainTokens` 必须小于该模型上的 `thresholdTokens`（`0.3 × 窗口 ≥ 24000`，即窗口 ≥ 80K 时成立）。

## 代价与适用面

- **原始内容会丢**：保留窗口（≈50K）之前的逐字内容不再回到上下文，只剩摘要。
- **细节会丢**：被压掉的工具结果只剩头 1024 / 尾 512 字符（全文在 spill 工件里，可按 locator 回取）。
- **适合**：长周期批量任务、状态机式 Agent、按步推进的实现工作。
- **不适合**：需要精确回溯全部原始输出的任务（例如逐行比对两份大文件）。

这一条也写进了 persona 契约：模型知道本模式会丢历史，遇到需要全量回溯的任务应当主动说明。

## 不影响其他模式

- 宿主组成、`profiles/*/cordis.patch.yml`、`agent-presets` 的默认 preset——**一律没改**。
- 极压模式的每一行都挂在它自己的 standing scope 上，只覆盖加入该 preset 的 agent。
- 默认 preset 仍是 `standard`；不选「极压模式」就没有任何影响。

## 卸载

```bash
rm -rf "${DSH_HOME:-$HOME/.dsh}/.agent-presets/extreme"
```

目录就是全部状态。已运行的会话挂的是各自的 composition generation，不受影响。

## 验证记录

1. **YAML 解析**：用 loader 自己的方言（含 `!!js` 平台判定）解析通过；顶层 19 行、展开后 29 行。
2. **包可解析**：每个 `name:` 都能在部署里解析到（与 `standard` 同一份包集合，多出的
   `dsh-spill-policy` 是宿主已在用的那个包）。
3. **加载期约束**（逐条对着对应插件的 Config / `resolveConfig` 核对）：
   - `compaction-basic`：`retainRatio(0.05) < thresholdRatio(0.1)` ✓
   - `tool-result-pruner`：`headChars + marker + tailChars = 1024+38+512 = 1574 ≤ thresholdChars(4096)` ✓
   - `tool-fs-search`：四个上限均为正整数；`sampleOverCapGlobResults` 是必填布尔且已给 ✓
   - `spill-policy`：`maxInlineBytes` 必须是非负整数，12000 ✓（缺省时该插件是纯 no-op，必须给值）
   - `agent-instructions.maxBytes` / `tool-todo.allowParallelInProgress`：类型与必填均满足 ✓
4. **结构检查**：`spill-policy` 是散装 row、不在任何 group 的 `isolate` realm 里。
5. **挂载校验**：`agentPresets.standingKeyFor('extreme')` → `MOUNTED OK`。这一条会真正 compose
   整棵插件子树，同时排除挂载期四种失败——包解析不了、config 不合法、行没激活（waiting for service）、
   服务被发布到 root realm。29 行全部真正激活。

## 兼容性

- 实测部署：`@deepseek-ai/dsh` **0.1.5-rc.2**。
- 依赖的行全部来自随发行版附带的包（`dsh-persona`、`dsh-compaction-basic`、
  `dsh-compaction-tool-result-pruner`、`dsh-spill-policy`、`dsh-tool-fs-search`、
  `dsh-agent-instructions`、`dsh-tool-todo` 等）。行名或配置键在别的版本里若已改名，
  挂载校验会直接报出来（`Cannot find package` / `invalid config: $.<field>`）。
- 本 preset 只在 **web profile** 下实测过（该 profile 把 agent 平面的行 disabled 掉，
  改由 preset 提供）。

## License

MIT，见 [LICENSE](LICENSE)。
