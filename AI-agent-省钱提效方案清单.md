# 手机版 DSH 省钱 / 提效 高收益方案清单

> 生成时间：2026-09-27 22:5x（Asia/Shanghai）
> 方法：11 个免费子代理（火山方舟 `ark-code-latest`）并行联网检索 → 主模型本地复核 + 抽样验证
> 所有链接均标注是否被真实打开过；本机可落地项标注了**具体配置键**。

---

## 0. TL;DR：按「每完成一个任务的费用」排序，最值钱的 5 件事

| # | 动作 | 预期收益 | 成本 |
|---|---|---|---|
| 1 | **把批量/长任务排到空闲时段**（工作日 9–12、14–18 之外，含周末/节假日全天） | 直接 **5 折** | 零 |
| 2 | **保住缓存前缀**（system prompt + 工具定义 + AGENTS.md 逐字不变、易变内容后置） | 命中价 ¥0.04/M vs 未命中 ¥2/M = **1/50** | 零 |
| 3 | **重活全丢免费子代理**，主模型只决策/把关 | 主模型 token 掉一个量级 | 零 |
| 4 | **关掉/调低思考等级**（思维链按输出价计费，¥8/M 高峰） | 简单任务砍掉大部分输出 token | 零 |
| 5 | **修掉本机失效的 spill 配置 + 调低工具结果裁剪阈值** | 每轮少塞几千 token，且**永久**省 | 改 2 行配置 |

⚠️ **反直觉但重要**（来源：AI Engineer 2026 演讲，已读文字稿）：
**在缓存命中的前提下，盲目压缩历史反而更贵。** 缓存输入便宜 50 倍，摘要必须短到 1/50 才回本；
所以「压缩」要用在**缓存必然失效**的地方（工具结果、易变文件），而不是无脑砍历史。

---

## 1. 立刻可做：本机配置层（零花钱，改完即生效）

### 1.1 ⛔ 本机实测到的真实 bug：`maxInlineBytes` 已经是废弃键
- 本机 `$DSH_HOME/profiles/web/cordis.patch.yml` 第 61–63 行写的是：
  ```yaml
  - id: spill-policy
    config:
      maxInlineBytes: 20000      # ← 本机已装版本里这个键不存在
  ```
- 实测本机安装包：`dsh-spill-policy/README.zh.md` 只接受 **`maxInlineTokens`**（估算 token，不是字节），
  base 默认 **12500**，且全盘 grep 不到 `maxInlineBytes`。
- **结论**：那条「减少进入模型上下文的体积」的注释是**空操作**，实际生效的是默认 12500 token。
- **建议**：改成 `maxInlineTokens`，并按需要下调（如 6000–8000）——
  工具结果会落盘并保留文件路径，模型需要时按需读回，不丢信息。
- 注意：rc.1/rc.2 release notes 明确「旧 `maxInlineBytes` 已删除，自定义配置必须改名重设」。

### 1.2 两个已知的省 token 旋钮（本机已装，默认开启）
| 配置 | 本机现值 | 作用 | 可调方向 |
|---|---|---|---|
| `spill-policy.maxInlineTokens` | 默认 12500（自定义键失效） | 工具文字+图片共享 token 预算，保留首尾 + 完整结果文件路径 | 下调 |
| `compaction/tool-result-pruner.thresholdChars` | 8192（head 4096 / tail 1024） | 超阈值的长工具结果先被裁剪 | 下调到 4000 左右 |

### 1.3 本机**已经**做对的事（不用再动）
- `agent-default-model` = `deepseek-official/deepseek-flash` + `reasoningEffort: low` ✅
- DSH **有**内置压缩与裁剪：`compaction-basic` + `/compact` 命令 + `tool-result-pruner` ✅
  （⚠️ 有个子代理说「DSH 无原生 compact」，**是错的**，本机 patch 第 570–583 行明确挂着这三个插件）
- 子代理并发上限默认 8、委派深度 1 ✅（防止一次炸出上百个任务）

### 1.4 ⚠️ 本机待修：子代理默认路由指向当天已干涸的模型
- `tool-subagent` / `tool-subagent-fork` 的 `agentOptions` 写死
  `modelscope / deepseek-ai/DeepSeek-V4-Pro-0813`，而魔搭额度当天已耗尽
  → **不显式传 provider/model 的子代理会瞬间失败**。
- 两个选择：① 每次都显式指定（用 `node tools/bin/dsh-quota-check --pick default` 拿可用路由）；
  ② 把 preset 默认改为 `openai / ark-code-latest`。
- 改法有风险，务必先 `cp` 备份 + `--dump-config` + 隔离试启动（见技能 `dsh-plugin-install`）。

### 1.5 升级到 v0.1.7-rc.2 的降本项（本机当前 0.1.7-rc.1）
- rc.2：**减少标准模式每轮固定提示的 token 开销**（默认生效）——这是每轮都省的固定成本。
- rc.2：**动态增加工具不破坏 KV Cache**（需模型显式声明能力）——多挂插件不再把缓存打掉。
- rc.2：定时任务与时间上下文 Web/桌面**默认关闭**；开启后默认每 10 分钟刷新一次。
- rc.1：主动压缩预留输出预算 + image-offload（图片超预算时替换最旧图片而非整轮失败）。
- ⚠️ 均为 prerelease；rc.1→rc.2 有破坏性迁移（Messages API 强制、`ptc-runtime`/`workflow-ptc` 改名无别名、
  **不支持 Python PTC**、热更新取消事务回滚）。**Android 端升级说明官方未提供**，本机升级需谨慎。

---

## 2. 路由与调度：钱花在哪条模型上

### 2.1 峰谷分时（最大杠杆，已官方核实）
- 高峰 = 北京时间**周一至周五（不含法定节假日）9:00–12:00、14:00–18:00**；其余全部**半价**（含周末、节假日全天）。
- 做法：批量整理、长文档处理、大批量子代理任务 → 用 `android_schedule` 排到夜间/周末。
- 出处：[api-docs.deepseek.com 定价](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/)（✅官方页已读）

### 2.2 模型分级（官方价，每百万 token，高峰｜空闲）
| 模型 | 输入(未命中) | 输入(缓存命中) | 输出 |
|---|---|---|---|
| `deepseek-flash` | ¥2 ｜ ¥1 | **¥0.04 ｜ ¥0.02** | ¥8 ｜ ¥4 |
| `deepseek-v4-pro` | ¥9 ｜ ¥4.5 | ¥0.30 ｜ ¥0.15 | ¥27 ｜ ¥13.5 |

- Pro 全价约为 Flash 的 **3.4–4.5 倍**；Flash 支持视觉 + 1M 上下文 + 384K 输出，Pro 不支持图像。
- **关键**：别按 token 单价选模型，按「**每完成一个任务的美元**」算（AI Engineer 2026 圆桌共识）。
  贵模型做规划/判断、便宜模型做执行，Devin Fusion 报省约 40%（厂商口径）。
- 出处：[定价页](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/)、[AI Engineer 圆桌](https://ai.engineer/talks/QHBjufYK8TA-state-model-routing-nvidia-cognition-openrouter)（✅已读文字稿）

### 2.3 免费额度阶梯（本机实际可用的）
| 通道 | 额度 | 重置 | 坑 |
|---|---|---|---|
| 火山方舟 `ark-code-latest` | 套餐额度（今日/5h/周窗） | **每日**重置 | 额度到期作废，**优先烧** |
| 魔搭 ModelScope | **魔粒制 250/天**（≈旗舰 125 次） | **每日**重置 | ⚠️ 网上「2000 次/天」与实测不符；余额靠 09:05 抄录，**严重滞后不可信** |
| Cloudflare Workers AI | 10000 neurons/天 | UTC 0 点（北京 08:00） | 全账号共享，用光后**所有** CF 模型（含看图）立刻 429 |
| 商汤 SenseNova | 60000 积分/5h（公测） | 滚动 5h | 属**周**额度池，**最后才用**；官方无余额接口 |
| 硅基流动 | 标注 free 的模型账单为 0 | 分钟级滚动 | 须实名；账户级限流 |
| OpenRouter `:free` | 20 RPM；<10 美元仅 **50 次/日** | UTC 日 | 多账号无效 |
| Google AI Studio | Gemini 3 Flash **20 次/日**、Flash-Lite 500/日、Gemma 14400/日 | 日 | ⚠️ 数字来自社区镜像，官方页未抓到 |
| 通义千问（百炼） | 新用户 7000 万 token | ⚠️ **一次性、90 天、不重置** | 超额默认直接扣费，要手动开「用尽即停」 |
| DeepSeek 官方 | 无额度，余额扣费 | — | 最后手段 |

- 时区坑：CF/OpenRouter 按 **UTC**，魔搭/火山按 UTC+8 —— 路由策略要分开算。
- 出处（已读官方/文档页）：[CF 定价](https://developers.cloudflare.com/workers-ai/platform/pricing/)、
  [OpenRouter limits](https://openrouter.ai/docs/api_reference/limits.md)、
  [硅基流动限流](https://docs.siliconflow.cn/docs/userguide/faqs/rate-limit-and-upgradation)、
  [智谱限流](https://docs.bigmodel.cn/cn/api/rate-limit.md)、
  [商汤 Token Plan](https://www.sensenova.cn/token-plan)、
  [通义免费额度](https://platform.qianwenai.com/docs/resources/free-quota.md?mode=pure)
- ⚠️ 未核实项：Groq / AI Studio / GitHub Models 数字取自社区镜像；火山 Coding Plan 标称次数是预估值，
  Agent 一个回合会放大成多次内部调用，**实际可跑回合远小于标称**。

### 2.4 熔断与退避
- 客户端工程模式：带抖动的指数退避、固定并发队列（用户请求可抢占批量）、错误率 >30% 熔断。
- 出处：[WaveSpeed 博客](https://wavespeed.ai/blog/zh-CN/posts/blog-deepseek-v4-context-caching/)（✅已读）

---

## 3. 上下文与缓存工程（省钱的核心战场）

### 3.1 缓存对齐（DeepSeek 上下文硬盘缓存）
- **默认开启，无需改代码**；按「缓存前缀单元」完整匹配，前缀**中间改任何一个字符即不命中**（A+B→A+C 不命中）。
- 做法：system prompt / 工具定义 / few-shot / 长文档放最前且逐字不变；**时间戳、用户变量、易变上下文全部后置**；
  多轮对话只追加在末尾；同一任务复用同一 `user_id`（user_id 做 KVCache 隔离，混用会互相打不中）。
- 命中率用 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` 字段实测。
- 命中价 vs 未命中价：Flash **1/50**（¥0.04 vs ¥2）；Pro **1/30**（¥0.30 vs ¥9）——⚠️ 网上常说的「1/50」对 Pro 不成立。
- 「最高省 90%」是营销口径：输出为主的任务收益很小。
- 出处：[kv_cache 指南](https://api-docs.deepseek.com/zh-cn/guides/kv_cache)（✅官方页已读）

### 3.2 子代理上下文隔离（本机已是核心打法）
- 子代理用干净窗口烧几万 token 探索，**只回 1–2k token 结论**；细节搜索上下文永不进主会话。
- Anthropic 明确：复杂研究任务上显著优于单代理。
- 出处：[Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（✅已读）

### 3.3 Just-in-time 检索（用标识符代替内容）
- 上下文里只留**文件路径、查询、链接**，运行时用 `head`/`tail`/`grep`/`glob` 按需取片段，绝不整文件/整库载入。
- 对应本机规矩：「定位用 grep/head，别全文 dump」。

### 3.4 分层记忆 + 会话卫生
- `AGENTS.md` 每轮必读、**只放硬规矩**；参考资料按需放 `skills/`；长结论写 `.md`，新会话读文件而不是读历史。
- ⚠️ **`/resume` 会把完整历史全部重新载入**，长会话 resume 本身很贵 → 跨会话衔接靠记忆文件，别靠 resume。
- Cline 建议上下文占用到 **80%** 就压缩或开新任务；各模型有「实用区间」，别按硬上限规划。
- 出处：[Cline Memory Bank](https://docs.cline.bot/best-practices/memory-bank)、[Cline Context Windows](https://mintlify.wiki/cline/cline/models/context-windows)

### 3.5 工具集瘦身
- 功能重叠的臃肿工具集是最常见失败模式：人分不清，模型更会误用，且**每轮都占上下文**。
- 本机对应：极简模式（仅 2 个工具）适合基准/省钱；插件挂载越多，每轮固定开销越大。

### 3.6 输出侧封顶
- 输出单价是输入未命中价的 4 倍，跑题/啰嗦最烧钱；显式设 `max_tokens` 兜底，prompt 里要求简短输出。

---

## 4. 手机端（Android）特有优化

| 项 | 结论 | 出处 |
|---|---|---|
| 特权通道 | 无 root 首选 **Shizuku/adb**（uid 2000 直调 Binder），比每次 fork shell 快得多；重启不持久、ColorOS 需关「权限监控」类包装器 | [Shizuku 官方](https://shizuku.rikka.app/introduction/) ✅ |
| 读屏成本 | **控件树（`android_screen`）几乎零 token，截图贵**；虚拟屏读不到控件树只能截图 → 降频 + 裁剪 + 交便宜视觉模型 | 本机实测 |
| uiautomator | `dump` 可能**永久卡住**（持续动画/WebView 最坏）；必须带超时，失败回退截图 | [appium#14726](https://github.com/appium/appium/issues/14726) ✅ |
| 虚拟屏 | VirtualDisplay 跑 App + 模型 tool call 控制主屏不受扰，有先例（ShadowAuto）；受 secure surface 限制 | [ShadowAuto](https://github.com/android-notes/ShadowAuto) ✅ |
| 保活 | 长后台服务约 6 小时可能被系统停（OEM 而异）；电池设「不限制」+ 自启动白名单 + 前台服务 + 任务分段 | [Doze 官方](https://developer.android.google.cn/training/monitoring-device-state/doze-standby?hl=en) ✅ |
| 发热限频 | 长任务是头号物理约束：插电 + 散热；Vulkan GPU offload 报 3–5 倍提速 | [Unstore 实测](https://unstore.io/discover/best-apps-for-old-android-local-llm-server-android/) ✅ |
| 本地小模型 | 定位=摘要/改写/分类，不做大脑。Gemini Nano 上下文仅 4096、无原生 function calling、需手写 ReAct；**ColorOS 上大概率不可用** | [Luciq](https://www.luciq.ai/blog/android-on-device-ai-gemini-nano-guide) ✅ |
| 本地零成本路径 | Ollama × DSH：`ollama launch dsh` 可接本地模型或 cloud 模型 | [Ollama 集成](https://docs.ollama.com/integrations/deepseek-harness) ✅ |
| Google 端侧实测 | 270M 小模型经合成数据微调，函数调用 46%→90%+；**技能目录先给描述、按需加载细节**（与 DSH skills 机制一致） | [AI Engineer 演讲](https://ai.engineer/talks/-TiET_K-E_g-from-46-90-fine-tuning-tiny-llms) ✅ |
| ⚠️ 反向坑 | 为适配 DeepSeek V4.1，**默认请求图片尺寸与编码质量被提高** → 图片 token 反而增加 | rc.2 release notes ✅ |

---

## 5. 来源清单（文章 / 视频 / 演讲）

### 5.1 官方（最高可信）
- [DeepSeek 定价页](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/) ✅ · [KV 缓存指南](https://api-docs.deepseek.com/zh-cn/guides/kv_cache) ✅ · [思考模式](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) ✅ · [限流](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit) ✅
- [DSH 官网「一切皆插件」](https://www.deepseek.com/harness/) ✅ · [providers 配置文档](https://deepseek-harness.github.io/deepseek-harness/guide/providers) ✅ · [subagent 参考](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/subagent.md) ✅
- [DSH Release API](https://api.github.com/repos/deepseek-ai/deepseek-harness/releases?per_page=8) ✅（rc.2 / rc.1 / alpha.1-2 / 0.1.6 / 0.1.5 全文）

### 5.2 视频 / 播客（✅=已读文字稿）
1. [Context Engineering in 2026: Compaction, Memory & Cost](https://ai.engineer/talks/WP3hjUXd918-context-engineering-in-2026-compaction-memory-cost) ✅ ←「有缓存时压缩反而更贵」
2. [FinOps for AI Agents: Who Spent All the Tokens?](https://ai.engineer/talks/GJX19pNhmSw-finops-ai-agents-who-spent-all-tokens) ✅（预算将尽时「引导输出变短」优于熔断）
3. [Preferences Over Benchmarks: Model Routing](https://ai.engineer/talks/FvxY8oPoI8o-preferences-over-benchmarks-model-routing) ✅（同一 coding 会话 $0.44→$0.14）
4. [The State of Model Routing（NVIDIA/Cognition/OpenRouter 圆桌）](https://ai.engineer/talks/QHBjufYK8TA-state-model-routing-nvidia-cognition-openrouter) ✅（上下文别超 100–200K）
5. [Agents for Everything Else — swyx](https://ai.engineer/talks/zepu8Kk6FBQ-agents-everything-else-swyx) ✅
6. [Fine-Tuning Tiny LLMs for On-Device Agents](https://ai.engineer/talks/-TiET_K-E_g-from-46-90-fine-tuning-tiny-llms) ✅
7. B站：[缓存命中，使用成本再砍 85%](https://www.bilibili.com/video/BV1DGVM6VEBV/) ⚠️仅标题 · [V4-pro 接入 Claude Code](https://www.bilibili.com/video/BV1pQRNBsEGs/) ⚠️仅标题 · [有效提问的法则](https://www.bilibili.com/video/BV1NJtgztEyx/) ⚠️仅标题
8. MLOps Podcast #340（端侧 agent 生产化）⚠️未打开正文
9. Microsoft Build BRKSP92（端侧到云编排）⚠️未打开正文

### 5.3 中文文章（✅=已读原文）
- [DeepSeek Flash 降价落地：缓存输入 0.02 元、成本测算全教程（阿里云）](https://developer.aliyun.com/article/1762718) ✅
- [Prompt Cache、Token 成本与 Plan Compiler（CSDN）](https://blog.csdn.net/yc2284238579/article/details/165891391) ✅ ←省钱优先级：砍调用 > 砍 context > 稳定前缀 > 砍工具调用，最后才换模型
- [DeepSeek V4 上下文缓存（WaveSpeed）](https://wavespeed.ai/blog/zh-CN/posts/blog-deepseek-v4-context-caching/) ✅
- [手机版 Andclaw：无 Root 纯无障碍安卓 AI 自动化（53AI）](https://www.53ai.com/news/Openclaw/2026031768934.html) ✅（竞品参照）
- 掘金两篇（Token 管理 / KV Cache）⚠️JS 墙未读

### 5.4 英文工程文档（✅）
- [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ✅
- [Claude Code 命令参考](https://code.claude.com/docs/en/commands) ✅ · [Subagents](https://code.claude.com/docs/en/sub-agents) ✅ · [Hooks](https://code.claude.com/docs/en/hooks-guide) ✅
- [Cursor Rules](https://cursor.com/docs/rules) ✅ · [Aider repo map](https://aider.chat/docs/repomap.html) ✅ · [Cline auto-compact](https://docs.cline.bot/features/auto-compact) ✅
- [AI coding agent 成本优化（dev.to）](https://dev.to/agdex_ai/ai-coding-agent-cost-optimization-in-2026-cut-claude-code-cursor-aider-token-spend-5a5l) ✅（⚠️其中 30–70% 省幅是单篇厂商博客口径）
- [LLMLingua 论文（EMNLP 2023）](https://aclanthology.org/2023.emnlp-main.825/) ✅（有损，agent/代码任务需 A/B）

---

## 6. ⚠️ 待核实 / 有争议（别当结论用）

1. **Batch API**：官方 sitemap 已无 batch 页、`/guides/batch` 404、定价页无折扣字样 → **倾向已下线**；网上流传的「半价」不成立。
2. **缓存最小前缀 64 token**：官方现行页无此数字（只说「按固定 token 间隔落盘」），**未能核实**。
3. **缓存 TTL**：官方只说「几小时到几天」，无精确值。
4. **魔搭「2000 次/日」**：与实测（魔粒制 250/天）冲突，**以控制台为准**。
5. **火山方舟 Coding Plan**：条款限编程工具，**裸 API 调用可能被判违规停用**。
6. **「省 85–90%」**：营销口径，取决于缓存 token 占比；输出型任务收益很小。
7. **各家省幅数字**（省 40%/78.9%/46%→90%）均为厂商单方 demo，小样本、LLM 当裁判、无方差说明。
8. **Gemini Nano 在 ColorOS 16 上大概率不可用**，投入前先查是否有 AICore。
9. 两个 `mintlify.wiki` 引用是**文档镜像站**，非官方域名（官方为 docs.cline.bot / code.claude.com），结论一致但引用时留意。
10. **第三方 API 中转站**（0.06x 之类）：Key 泄露、稳定性、合规风险高，**不建议采纳**。

---

## 7. 本次任务的复核修正（子代理说错/说得不准的地方）

| 子代理结论 | 复核结果 |
|---|---|
| 「DSH 无内置压缩，需靠纪律替代」 | ❌ **错**。本机已装 `compaction-basic` + `/compact` + `tool-result-pruner`（patch 570–583 行） |
| 「缓存命中价 = 未命中价 1/50」 | ⚠️ 只对 Flash 成立；**Pro 是 1/30** |
| 「`maxInlineBytes: 20000` 在减小上下文」 | ❌ 该键在本机版本**已不存在**，实际生效默认 `maxInlineTokens: 12500` |
| 「Batch API 半价」 | ⚠️ 官方文档已无此页，**倾向下线**，不可作为方案 |
| 「魔搭 2000 次/日」 | ❌ 与实测（魔粒制）冲突 |

→ **结论：子代理的检索面很宽、出处标注也诚实，但事实性断言必须主模型本地/官方页复核。**

---

## 8. 建议执行顺序（一次性做完的清单）

> ✅ 2026-09-27 23:05 已执行第 1、3 项（详见文末「已执行记录」）。
> 其余项涉及能力变化或破坏性迁移，**需用户确认后再做**。

1. [x] 改 `cordis.patch.yml`：`maxInlineBytes: 20000` → `maxInlineTokens: 12500`（行为不变，去掉空操作键）
2. [ ] 改 `tool-result-pruner.thresholdChars`：8192 → 4000
3. [x] 子代理 preset 默认模型改为 `openai / ark-code-latest`（**新会话生效**）
4. [ ] 长任务/批处理排到空闲时段（工作日 9–12、14–18 之外）——用 `android_schedule`（需半夜唤醒手机，见风险清单）
5. [ ] 检查 AGENTS.md / system prompt 是否有易变内容（时间戳、动态值）混在前缀里 → 全部后置
6. [ ] 评估升级到 v0.1.7-rc.2（先备份，注意破坏性迁移）
7. [ ] 给免费额度池做「日重置优先、周额度最后」的每日巡检

---

## 9. 已执行记录（2026-09-27 23:03–23:05）

**改动文件**：`$DSH_HOME/profiles/web/cordis.patch.yml`（备份：`cordis.patch.yml.bak-20260927-230353`）

| # | 改动 | 为什么是无影响/极小影响 |
|---|---|---|
| 1 | `spill-policy.config`: `maxInlineBytes: 20000` → `maxInlineTokens: 12500` | 旧键在本机版本**根本不存在**（全盘 grep 无），实际一直走 base 默认 12500；改成显式同值 = **行为完全不变**，只是把误导性的空操作去掉 |
| 2 | 6 处 subagent `agentOptions`: `modelscope/DeepSeek-V4-Pro-0813` → `openai/ark-code-latest` | 仅改「不显式指定模型时」的兜底。原默认模型当天额度已耗尽，不传参数必然失败；ark 本会话 11 个子代理实测全成功 |

**验证证据**：
- `node <dsh-bin> --profile web --dump-config` → **exit 0**；解析后配置里 `maxInlineTokens: 12500`（第 738 行）、6 处 `agentOptions: openai/ark-code-latest`（1180/1189/1368… 行）均已生效。
- **隔离试启动**（独立 `DSH_HOME=tools/trialhome`，端口 3099）→ 打印
  `dsh web: http://127.0.0.1:3099/?token=…`，**TRIAL_OK**，无报错。
- 线上引擎（127.0.0.1:3080）热重载后仍 **HTTP 200**，未受影响。
- 试启动产物与临时文件已清理。

**⚠️ 重要限制**：agent preset 与模型白名单一样，**在会话组合时快照**。
本会话发起的「不指定模型」探针仍走旧默认（modelscope）→ `subagent run failed`；
显式 `openai/ark-code-latest` 探针 → `ARK-PROBE-OK`。
→ **改动 1 立即生效，改动 2 需要新开会话才生效。**

