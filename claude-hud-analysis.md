# Claude HUD（jarrodwatts/claude-hud）安装、使用流程与实现原理分析

> 分析日期：2026-03-22（UTC）

## 0) 下载结果（环境限制说明）

我先在终端中尝试直接拉取仓库：

```bash
git clone https://github.com/jarrodwatts/claude-hud.git
```

当前环境对 GitHub 出站连接受限，返回：

- `fatal: unable to access ... CONNECT tunnel failed, response 403`

因此无法在本机完成完整 `git clone`。后续分析基于可访问的 `raw.githubusercontent.com` 源文件与 GitHub 页面内容完成（README、plugin/marketplace 配置、核心 TS 源码、命令脚本）。

---

## 1) 项目是什么

Claude HUD 是一个 Claude Code 插件，核心目标是把「会话状态」持续显示在 statusline 中，重点包括：

- 上下文窗口占用（context）
- 使用额度（usage，Pro/Max/Team）
- 工具调用活动（Read/Edit/Grep 等）
- 子代理（agent）状态
- Todo 进度

它的定位不是新建 TUI/窗口，而是挂在 Claude Code 原生 statusline API 上，输出到标准输出由 Claude Code 渲染。

---

## 2) 安装流程（用户视角）

根据 README，标准安装是 3 步：

1. 添加 marketplace

```text
/plugin marketplace add jarrodwatts/claude-hud
```

2. 安装插件

```text
/plugin install claude-hud
```

3. 执行 setup，把 HUD 配置进 statusline

```text
/claude-hud:setup
```

完成后重启 Claude Code，使新的 statusLine 配置生效。

### Linux 特殊事项（EXDEV）

README 和 setup 命令都强调：Linux 下若 `/tmp` 与 home 不同文件系统，可能出现 `EXDEV: cross-device link not permitted`。建议在同一文件系统下指定 TMPDIR 后再安装：

```bash
mkdir -p ~/.cache/tmp && TMPDIR=~/.cache/tmp claude
```

然后在该会话里继续 `/plugin install claude-hud`。

### Setup 命令内部流程（从 `commands/setup.md` 看）

`/claude-hud:setup` 的设计非常“运维化”，大致是：

- **Step 0**：先检查“幽灵安装”状态（cache/registry/temp 三处一致性）
- 如发现残留，给出清理流程（含是否重置 `installed_plugins.json`）
- Linux 下额外检查 cross-device 风险
- **Step 1**：探测平台、shell、运行时（优先 bun，其次 node）
- 根据运行时拼接最终 statusline 执行命令（bun 走 `src/index.ts`，node 走 `dist/index.js`）

这说明作者重点处理了「插件装过但不可用」和「跨平台 shell 差异」两个高频故障场景。

---

## 3) 使用流程（用户日常）

- 初次用 `/claude-hud:setup` 进行接入
- 日常通过 `/claude-hud:configure` 做引导式开关（布局、预设、元素启用/禁用、Git 展示粒度、自定义文案等）
- 高级用户可直接编辑 `~/.claude/plugins/claude-hud/config.json`

`/configure` 的脚本里显式区分两条流程：

- 新用户（无配置）
- 老用户（有配置）

并且强调：一些核心项（如 model/context）始终显示，不给关闭；同时保留高级手工字段（如颜色、阈值）以避免引导界面覆盖手工配置。

---

## 4) 实现原理（技术视角）

## 4.1 插件声明与命令入口

`.claude-plugin/plugin.json` 声明了：

- 插件名/描述/版本
- 两条命令：`./commands/setup.md` 与 `./commands/configure.md`

`.claude-plugin/marketplace.json` 定义 marketplace 元数据和可安装 plugin 清单。

这意味着安装层面依赖 Claude Code 插件机制，本项目本身不负责安装器逻辑，而是提供规范化元数据与命令文档。

## 4.2 主处理链路（数据流）

`src/index.ts` 的主函数可概括为：

1. `readStdin()`：读取 Claude Code 传入的 statusline JSON
2. `parseTranscript(transcript_path)`：解析 transcript JSONL，提取工具/agent/todo 状态
3. 读取配置计数（CLAUDE.md/rules/MCP/hooks）
4. `loadConfig()`：加载并合并用户配置
5. 视配置决定是否取 Git 状态、usage 数据
6. 计算会话时长
7. 组装 `RenderContext` 后 `render(ctx)` 输出 statusline

即：**stdin + transcript + 本地配置 + Git + usage API => 渲染字符串**。

## 4.3 Context 占用显示（为什么“更准”）

`src/stdin.ts` 中：

- 优先读取 Claude Code v2.1.6+ 的 `context_window.used_percentage`（原生百分比）
- 不可用时才 fallback 到 token 手算

这能保证 HUD 与 Claude Code 自身 `/context` 更一致，避免纯估算偏差。

## 4.4 Tool / Agent / Todo 的实时感来源

`src/transcript.ts` 会流式读取 transcript JSONL，跟踪：

- `tool_use` / `tool_result` 对应关系（running/completed/error）
- `Task` 作为 agent 活动
- `TodoWrite`、`TaskCreate`、`TaskUpdate` 维护 todo 列表
- 保存最近 N 条（tools/agents）和最新 todos

因此 HUD 的“活动行”并非来自猜测，而是来自 Claude 会话日志增量解析。

## 4.5 Usage（额度）显示机制

`src/usage-api.ts` 体现出较完整的健壮性设计：

- 从本地 OAuth 凭证读取 access token（区分 API 用户与订阅用户）
- 调 Anthropic usage API 拉取 5h / 7d 利用率
- 识别 custom endpoint（如非官方 base URL）时直接跳过
- 使用文件缓存（成功/失败 TTL 可配置）
- 针对 429 有指数退避 + Retry-After + lastGoodData 回退展示
- 用文件锁避免多进程并发下缓存竞争

因为 statusline 进程刷新频率高（README 提到约 300ms），所以它必须把网络请求“慢路径”缓存化，否则会拖慢或打爆 API。

## 4.6 配置系统设计

`src/config.ts` 显示该项目配置层有这些特点：

- 有完整 `DEFAULT_CONFIG`
- 对字段进行类型与范围校验（路径层级、阈值、颜色、layout 等）
- 支持旧版本配置迁移（legacy layout -> 新字段）
- `mergeConfig` 采用“默认 + 用户增量覆盖”

这使得配置损坏/升级后字段变化时，HUD 仍可回落到可用状态。

## 4.7 渲染层

`src/render/index.ts` 负责最终文本输出，关键点包括：

- ANSI 颜色处理
- 终端宽度探测（stdout/stderr/COLUMNS）
- 可见宽度计算（含 grapheme、emoji、宽字符）
- 超宽截断/换行拆分

这解释了为什么它在不同终端和多字节字符场景下能保持较稳显示。

---

## 5) 一句话总结其“工作模型”

Claude HUD 本质是一个 **状态聚合器 + 文本渲染器**：

- 从 Claude Code stdin 读「当下上下文」
- 从 transcript 读「过程活动」
- 从本地配置决定「展示策略」
- 从 usage API/Git 补充「外部状态」
- 最终渲染为 statusline

所以它不是“替代 Claude Code UI”，而是把 Claude Code 原有能力做成更连续、可观测、可定制的会话仪表带。

---

## 6) 对你实际落地的建议

如果你要自己复用这套思路（例如做企业版 HUD）：

1. **保留 stdin 优先**：原生数据准确度最高
2. **所有外部 IO 都做缓存和退避**：尤其 usage/API 类
3. **命令脚本优先做故障自愈**：setup 中的 ghost install 检测很值得借鉴
4. **配置永远做 merge + migration**：减少升级破坏
5. **渲染层重视宽字符与 ANSI**：否则一旦跨终端就容易错位


---

## 7) 从“用户监控”视角看：插件到底能看到什么？

先给结论：**通常不能无条件拿到“所有”输入输出**，但在 Claude HUD 这类 statusline 插件模式下，
可以拿到的会话信号已经足以构建“行为级监控”。

### 7.1 能看到哪些信息（以本项目实现为参照）

结合本项目的数据来源，插件可见面主要来自：

1. **stdin statusline payload**
   - 会话级元数据（如模型、上下文占用、token 相关统计、路径信息等，具体取决于 Claude Code 注入字段）

2. **transcript JSONL**
   - 工具调用轨迹（何时调用了 Read/Edit/Grep/Bash 等）
   - 工具结果状态（成功、失败、耗时趋势）
   - agent / task / todo 的活动变化
   - 这类数据可推断“用户在做什么类型工作”

3. **本地环境信息（按插件自身读取能力）**
   - Git 分支、脏文件计数
   - 本地配置文件（规则、hooks、MCP 开关状态）

4. **外部 API 信息**
   - 用量额度（5h/7d），可用于负载与成本监控

### 7.2 不能默认保证拿到的内容

- **终端里用户输入的每一个按键**（除非宿主平台明确暴露）
- **宿主未提供的完整对话正文**（是否有、给到什么粒度由平台决定）
- **插件权限范围外的系统信息**（文件/网络/进程访问受宿主和本机权限控制）

所以“插件=全量录屏/键盘记录器”这个等式并不成立；它更像是**受宿主接口约束的数据订阅器**。

### 7.3 从监控能力看，还能扩展什么

如果你把它当成“工程监控探针”，除 HUD 展示外还能做：

1. **行为审计（Activity Audit）**
   - 记录工具调用序列，形成可回放事件流（不必保存全文）

2. **风险检测（Policy Guard）**
   - 识别高风险动作：大规模删除、越权命令、可疑外连
   - 触发告警或强制二次确认

3. **效率分析（Productivity Analytics）**
   - 统计任务周期、工具成功率、失败重试率、上下文爆满频次

4. **成本与容量治理（Cost Governance）**
   - 结合 usage + 会话时长做预算阈值告警

5. **质量门禁（Quality Gate）**
   - 将“是否跑测试/是否通过 lint”纳入会话状态，作为交付前检查项

### 7.4 风险与合规建议（很重要）

从监控角度上线插件时，建议最少做到：

- **最小化采集**：只采事件，不采不必要正文
- **明示告知**：用户可见“采集哪些字段、用途、保留期”
- **脱敏与加密**：token、路径、命令参数做脱敏；日志静态加密
- **可关闭/可导出/可删除**：满足审计与隐私请求
- **分级权限**：监控插件与执行插件分离，避免一体化高权限

### 7.5 一句话回答你的问题

- **是否可获取所有输入输出？** 一般不行，取决于宿主暴露接口与权限模型。
- **还有什么监控能力？** 在现有可见信号下，已经能做活动审计、风险检测、效率分析、成本治理和质量门禁。

