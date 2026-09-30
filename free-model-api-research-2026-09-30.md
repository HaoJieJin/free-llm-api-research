# 魔搭 ModelScope 免费推理 API 同类替代品调研报告

- **查证日期：2026-09-30（北京时间）**
- 调研方式：web_search + web_fetch 实抓官方文档/原文；所有结论标注【官方事实】【社区信息】【查不到】
- 重要前提：各平台额度政策变动极快（本次调研中就发现多个"上线即砍"的案例），**接入前务必以官方控制台实时显示为准**

---

## 方向 A：魔搭 ModelScope 自身免费额度

### A1. 关于"魔粒"——该机制名查不到
- 【查不到】用 `"魔粒" modelscope`、`"魔粒" 魔搭` 多组检索，**公开网页（含官网、媒体、社区）均无"魔粒"这一额度机制名**。
- 用户记忆中的"魔粒"，大概率指的是魔搭实际公开的机制——**每日 API-Inference 免费调用次数**（见下）。不排除"魔粒"是平台登录后控制台内的新近叫法/昵称，但**无法从公开来源证实，建议登录 modelscope.cn 控制台截图核实**。

### A2. 每日免费额度：2000 次/天，单模型 500 次/天
- 【社区信息，多源交叉一致】绑定阿里云账号 + 完成阿里云实名认证后，**每天约 2000 次 API-Inference 调用免费、每日刷新**：
  - CSDN 教程（2026-06-27）：https://aicoding.csdn.net/6a696eb7662f9a54cb958e3c.html
  - GitHub ARIS 指南（持续维护的文档）：https://github.com/zhangchenhaobest/Auto-claude-code-research-in-sleep/blob/main/docs/MODELSCOPE_GUIDE.md
- 【社区信息】**单模型上限 500 次/天**（即 2000 次是跨模型总量，单个模型每天最多 500 次）：
  - 80AJ 转 Linux.do（2026-01-24）："每日 2000 次，单模型上限 500 次"：https://www.80aj.com/2026/01/24/modelscope-free-image-api/
  - V2EX 帖（2025-10-14）有疑似官方口吻回复同一口径：https://global.v2ex.co/t/1165033
- 【官方事实（页面元数据）】魔搭官方 API 推理介绍页存在且描述与此一致，但正文为 JS 动态渲染，未能抓到额度原文：
  https://www.modelscope.cn/docs/model-service/API-Inference/intro
- 【官方事实】未绑定/未实名直接调用会报：`401 please bind your alibaba cloud account before use`（上述教程实测截图）。
- ⚠ 即"绑定阿里云账号"本身**不额外赠送一笔独立额度**，它 + 实名就是**开通每日 2000 次的前提条件**；公开来源查不到"绑定另送 X 次/X 魔粒"的说法。

### A3. 支持模型与上下文窗口
- 【官方事实】模型范围 = 模型库中带 **"API 推理"** 标签的模型，ID 形如 `组织/模型名`，如 `Qwen/Qwen3-30B-A3B-Instruct-2507`、`deepseek-ai/DeepSeek-V3.1`、`Qwen/Qwen3-Coder-30B-A3B-Instruct` 等，数量上随平台动态增减。模型列表：https://www.modelscope.cn/models （筛"API 推理"）
- 【查不到】**各模型的上下文窗口没有统一公开数字**，窗口由具体模型决定（社区文档仅笼统写"通常 8K–128K，取决于模型"）。**不要引用固定窗口数，需逐模型在其模型页/实测确认。** 一个 Key 同时支持 OpenAI 与 Anthropic 两种协议（社区文档事实）。

### A4. RPM / 并发限制
- 【官方事实】存在官方限制页：https://www.modelscope.cn/docs/model-service/API-Inference/limits
- 【查不到】该页同样是 JS 渲染，**本次未能抓到具体 RPM/并发数字**；社区文档也只写"有并发限制，具体见 limits 页"。429 即触发限流。**具体 RPM 需登录后实测或查 limits 页。**

---

## 方向 B：其他模型社区 / 模型托管平台的免费推理 API

### B1. Hugging Face（重点结论：是"月度"不是"每日"，且额度极小）
- 【官方事实】HF 官方 pricing 文档（2026 年现行，Inference Providers 体系）：
  https://huggingface.co/docs/inference-providers/pricing
  - **免费用户：每月 $0.10 credits（官方注明 subject to change）**
  - PRO 用户：每月 $2.00；Team/Enterprise：每席位 $2.00
  - credits 每月发放、先扣 credits；**免费层额度每月重置，不是每天**
  - 旧的 "Inference API (serverless)" 已改名 **hf-inference**；截至 2025-07 起 hf-inference 主要做 **CPU 推理**（embedding、分类、BERT/GPT-2 类小模型），大模型走其他第三方 provider
- 对 agent 用途：$0.10/月 对大模型基本是"够打几发"的体验量，**实用价值很低**，且国内需代理。

### B2. OpenXLab 浦源 / 书生 Intern InkStone（国内、值得注册，但非"每日刷新"）
- 【社区信息】NodeLoc 帖（2026-09-19，较新）：https://www.nodeloc.com/t/topic/110021
  - 平台：**Intern InkStone** https://discovery.intern-ai.org.cn/
  - 单账号 **5000 点算力**（顶配约可跑 83 小时，偏开发/训练）
  - 注册送 **10 墨点（约 200M = 2 亿 token）**，平台直接给 API Key：
    - OpenAI 协议：`https://discovery-api.intern-ai.org.cn/v1`
    - Anthropic 协议：`https://discovery-api.intern-ai.org.cn`
  - 可用模型较多（含其自研 Atria-Dawn 等）
- ⚠ 风险：该羊毛已出现**自动化批量注册机**（GitHub Intern-Register-Tool），历史经验是风控会很快收紧/砍额度；"墨点"是注册一次性赠送还是每日刷新，**公开帖未说明 → 按一次性对待，需登录控制台确认**。

### B3. 超算互联网（活动型一次性额度）
- 【社区信息/媒体】"超算互联网 QwQ-32B API 上线，免费 100 万 Tokens"：https://mp.weixin.qq.com/s/La5FT1EXrWjJf_YdT8UazA （微信页本次需验证，未能抓到正文；标题口径来自检索快照）；平台已接入阿里千问：https://www.stdaily.com/web/gdxw/2025-03/10/content_307843.html
- 定性：**活动期一次性 100 万 token，不是每日刷新**，活动是否仍有效需到超算互联网控制台确认。

### B4. 启智社区 OpenI
- 【官方/社区】OpenI 主要提供**算力券/训练集群（智算网络）**与项目托管，定位偏科研协作训练：https://www.openi.org.cn
- 【查不到】**未查到稳定的"每日免费大模型推理 API + 固定 baseURL"公开文档**，不适合直接当 OpenAI 兼容免费源。

### B5. OpenCSG（始智 AI / 开放传神）
- 【官方事实】其产品 **CodeSouler（CSGShip）** 是面向编程的智能体，CSGHub 为模型资产管理：https://opencsg.com/docs/starship/codesouler/codesouler_intro ；社区站 https://sg.opencsg.com
- 【查不到】未查到面向公众、稳定每日刷新的免费推理 token 额度公开规则。

### B6. 阿里云天池
- 【查不到】天池定位是**竞赛/数据集/实训**平台，未查到可长期接入 agent 的免费推理 API 额度。

---

## 方向 C：Serverless GPU / 推理云平台免费额度

| 平台 | 免费额度现状（2026-09-30 查证） | 能否自部署模型暴露 OpenAI API |
|---|---|---|
| **潞晨科技** | 【官方/媒体】2025-02 与华为昇腾联合发布 **DeepSeek-R1 系列 API"无限量限时免费"**：https://hub.baai.ac.cn/view/43118 。**属"限时"活动，2026 年是否延续未知**，需以 https://cloud.luchentech.com/maas/modelMarket 实时为准 | 【官方事实】支持。潞晨云提供昇腾910B / NV H800 **推理镜像**，可一键私有化部署 671B 到蒸馏版 |
| **无问芯穹** | 【社区信息】2025 年初 Infini-AI 平台"百亿 token 补贴、限时全免"：https://mp.weixin.qq.com/s/DJJsUEjGC_haQHEQc-Komg 。**限时补贴，非每日刷新**，现状需查平台 | 平台本身即异构算力 + 模型服务，支持自定义服务，具体以控制台为准 |
| **PPIO 派欧云** | 【官方事实】FAQ：https://resource.ppio.com/docs/support/faq ——①注册自动发**新用户代金券（一次性）**；②提供带**"免费"标签的体验模型（持续可用，按 RPM 限流）**，模型列表 https://ppio.com/docs/model/llm 筛"免费" | 【官方事实】支持。有 **GPU 容器**产品可自部署任意模型 |
| **AutoDL** | 【官方事实】活动页：https://backup.autodl.com/docs/coupon/ ——**新用户注册只送"30 天会员"（会员是折扣/功能权益，≠免费 GPU 时长）**；GPU **按小时计费，无每日免费 GPU 额度** | 【官方事实】支持。容器可跑 vLLM 并"开放端口"暴露 API（公网访问需按其端口/代理机制），是最常用的国内自部署平台，但**要花钱** |
| **揽睿星舟（3s.ai）** | 【查不到】未查到每日免费额度公开规则，按付费 GPU 平台对待 | 支持容器/推理部署 |
| **共绩算力 / "共迹"** | 【社区信息】"共绩算力（算了么 API）"为商业 API，未见免费额度：https://lmspeed.net/zh/provider/api-suanli-cn | 以 API 转售为主 |

**方向 C 总结论：国内 GPU/推理云没有"每天刷新的稳定免费额度"这一类产品**；免费都是"新用户一次性代金券/限时活动/少量免费标签模型"。自部署 OpenAI 兼容 API 技术上都可行，但要么一次性额度、要么按时付费。

---

## 方向 D：关键替代思路——"每天刷新的免费大上下文模型"

### D1. 给"每日/每周免费 GPU 时长、自己部署"的平台
- 【官方事实】**真正周期性免费给 GPU 的主流是海外笔记本平台：Google Colab、Kaggle**；国内平台（方向 C）不做周期性免费 GPU。

### D2. Google Colab 免费 GPU 跑 vLLM 暴露 API —— 明确违反免费层条款，不可作为可靠方案
- 【官方事实】Colab 官方 FAQ：https://research.google.com/colaboratory/faq.html
  - 免费笔记本**最长运行 12 小时**，空闲会断；GPU 型号动态分配、资源不保证、限额动态浮动且不公布
  - **所有托管运行时禁止**："file hosting, media serving, or other **web service offerings** not related to interactive compute"、"**connecting to remote proxies**"、P2P、多账号规避等
  - **免费层（无 compute unit 余额）额外禁止且"可随时无预警终止"**："**remote control such as SSH shells, remote desktops**"、"**bypassing the notebook UI to interact primarily via a web UI**"（即用隧道暴露 API、主要靠网页/API 而非笔记本界面交互）
- 结论：**Colab 免费层上跑 vLLM + cloudflared/ngrok 隧道对外提供 API，正是官方点名会"随时无预警断开"的用法**。社区虽有大量"一键 vLLM"教程（如 https://www.cnblogs.com/bhdk/p/19911614 ），但属于与风控博弈、随时失效，**不是稳定的"每日刷新"生产方案**；且 Colab 国内需代理、需 Google 账号。

### D3. Kaggle 免费 GPU —— 比 Colab 略可玩，但同样被盯上、硬件只够小模型
- 【官方事实】Kaggle 文档：每周约 **30 GPU 小时**配额、单次会话最长 **12 小时**、硬件为 **P100 或双 T4（单卡 16GB）**、需手机验证：https://www.kaggle.com/docs/notebooks ；https://www.kaggle.com/docs/efficient-gpu-usage
- 【官方事实】Kaggle 官方反馈区已出现专门讨论："一波教程正在把 Kaggle Notebook 变成免费公共 LLM 服务器，GPU 可用性正在下降"：https://www.kaggle.com/discussions/product-feedback/743898
- 实操评估：
  - 30 小时/周 ≈ 每天约 4.3 小时，**不是全天在线**，断后 URL/会话变化需重新拉起
  - **T4 16GB 显存**：只够 7B–14B 全精度 / 32B 级 AWQ-GPTQ 量化模型，**跑不动"大上下文的大模型"满血版**；长上下文还会进一步吃 KV cache
  - Kaggle 同样不鼓励把 notebook 当 API 服务器，国内需代理
  - 定性：**适合"每天临时拉起、跑小模型干杂活"，不满足用户"免费大上下文模型稳定日刷新"的核心诉求**

### D4. 其他"周期性免费"AI 算力（多为月度，非每日）
- 【官方/第三方汇编】https://aimultiple.com/free-cloud-gpu （2026-09-16 更新，附官方出处）：
  - **Hugging Face ZeroGPU**：免费登录账号**每天 5 分钟 GPU**，自首次使用起 24 小时滚动重置（非固定 0 点）；硬件 RTX Pro 6000、单次函数默认 60 秒。**只能通过 Gradio 应用调用，不能直接暴露 OpenAI API** → 不适合做 agent 的通用 baseURL。文档：https://huggingface.co/docs/hub/spaces-zerogpu
  - **Lightning AI**：符合条件账号**每月补到 15 credits、月底作废**（月度，非每日）
  - **Modal**：Starter **每月 $30 credits，但必须绑支付方式、超额会扣费**（有风险），非每日
  - **Paperspace Gradient**：免费档仅 8GB M4000、笔记本公开、6 小时断，实用性低
- 【官方事实】**HF Spaces**：免费 CPU Space 可 7×24 托管 Gradio，但 CPU 跑不动大 LLM；上 GPU/ZeroGPU 受上述 5 分钟/天限制 → 无法稳定免费暴露大模型 OpenAI API。

### D5. CDN / 边缘 / 网关上的免费 AI 额度
- **Cloudflare Workers AI**（用户已在用，官方事实）：https://developers.cloudflare.com/workers-ai/platform/pricing/
  - **免费 10,000 Neurons/天，UTC 00:00 每日重置**（这是本次调研中**唯一"额度真正每天重置"的主流海外平台**）；付费 $0.011/1000 neurons
  - ⚠ **kimi-k2.6/k2.7-code、glm-5.2/5.3/5.3-flash、deepseek-v4-flash-0731、deepseek-v4-pro-0813 需付费方式**，免费档用不了；免费档能用的多为中小模型（llama-3.x、gpt-oss、qwen3.8-27b、glm-4.7-flash 等）
- **Vercel AI Gateway**：https://vercel.com/docs/ai-gateway/pricing
  - 【官方事实】存在 free tier：仅含模型子集、按每模型限流（超限 429）；**首次发请求起算；一旦购买 credits 即永久转为付费档、月度免费额度不再适用**
  - 【查不到】**官方定价页未写明免费档具体金额**（社区/第三方博客流传"每月约 $1"，但未在官方文档证实，不采信具体数字）
- **Netlify**：【查不到】AI 能力主要走企业 credit 套餐，未见明确的免费推理额度。
- **Deno Deploy**：【官方事实】它是边缘 **运行时**，可部署自己写的代理/转发脚本（如社区有 Deno 转发 Azure OpenAI 的项目），**但平台本身不送模型推理额度**——背后模型仍需自带 key。
- **Cloudflare AI Gateway / Vercel Gateway 本身**：网关只做路由/缓存/可观测，**不产生免费 token**，后端模型费用另算。

### D6. 方向 D 总结
- **"每天刷新"的免费推理，现实只有两类**：① 魔搭（国内，每日 2000 次）；② Cloudflare Workers AI（海外，每日 1 万 neurons，但大模型要付费、免费档模型偏小）。
- **"每日刷新免费 GPU 自己部署大模型"没有合规稳定解**：Colab 明确禁止且随时断，Kaggle 每周仅 30 小时、16GB 显存只够小模型且已被平台盯防。

---

## 方向 E：接入 agent 框架——OpenAI 兼容性与网络

### E1. OpenAI 兼容（可直接填 baseURL + key）

| 平台 | baseURL | 免费额度性质 | 国内直连 |
|---|---|---|---|
| **魔搭 ModelScope** | `https://api-inference.modelscope.cn/v1` | **每日 2000 次、单模型 500 次** | ✅ 直连 |
| **PPIO 派欧云** | `https://api.ppio.com/openai` | 新用户代金券（一次性）+ "免费"标签模型（持续、低 RPM） | ✅ 直连 |
| **书生 Intern InkStone** | `https://discovery-api.intern-ai.org.cn/v1` | 注册送约 2 亿 token 墨点（疑似一次性，待确认） | ✅ 直连 |
| **硅基流动 SiliconFlow** | `https://api.siliconflow.cn/v1` | 【第三方汇编】多款小模型标价 ¥0（Qwen3-8B、GLM-4-9B-0414 等约 8 款）：https://github.com/mvalentsev/awesome-free-ai-coding ；⚠ 但存在"¥0 模型仍可能要求账户有效/会 402"的情况（本机此前实测 402/403），**需实测** | ✅ 直连 |
| **OpenRouter** | `https://openrouter.ai/api/v1` | 【官方事实】`:free` 模型：**无充值历史 50 次/天、20 RPM；累计充值≥$10 升为 1000 次/天；UTC 按日重置**：https://openrouter.ai/docs/api_reference/limits | ❌ 需代理 |
| **Groq** | `https://api.groq.com/openai/v1` | 【官方事实】免费档有 RPM + **RPD（按模型，如 gpt-oss 系列 1000 次/天级别）**：https://console.groq.com/docs/rate-limits | ❌ 需代理 |
| **Hugging Face** | `https://router.huggingface.co/v1` | 免费 **$0.10/月**（月度） | ❌ 需代理 |
| **Cerebras** | `https://api.cerebras.ai/v1` | 【第三方】有免费层、按分钟级限流（具体数值以控制台为准）：https://www.morphllm.com/cerebras-pricing | ❌ 需代理 |
| **Vercel AI Gateway** | `https://ai-gateway.vercel.sh/v1` | free tier 模型子集 + 每模型 429 限流（金额未公开） | ❌ 需代理 |
| **Cloudflare Workers AI** | `https://api.cloudflare.com/client/v4/accounts/{账户ID}/ai/v1`（**带账户 ID，非纯简单 baseURL 形态**） | **1 万 neurons/天**；大模型需付费 | ❌ 需代理 |

### E2. 网络结论
- **国内直连、可直接填 baseURL 的免费源**：魔搭、PPIO、书生 Intern、（待实测的）硅基流动——**全部是国内平台**。
- **需要代理（127.0.0.1:7890）**：OpenRouter、Groq、HF、Cerebras、Vercel、Cloudflare、Colab、Kaggle。
- 免费层最"抗用"的海外免费路由是 **OpenRouter :free（50 次/天，充值过 $10 则 1000 次/天）和 Groq（按模型有每日上限、速度极快）**，但都要代理、且 :free 模型常为小号/去限流模型，**上下文与稳定性不及魔搭的大模型**。

---

## 总体结论与建议
1. **魔搭在"每日刷新 + 国内直连 + 大模型 + OpenAI 兼容"这一组合上，2026-09 仍无真正同量级替代品。** 其硬约束是：每天 2000 次总量、单模型 500 次、需绑阿里云实名、各模型上下文与 RPM 不透明需实测。
2. 国内可叠加的免费/半免费源：**PPIO 免费标签模型、书生 Intern（约 2 亿 token 一次性）、硅基流动 ¥0 小模型（需实测防 402）**；超算互联网/潞晨/无问芯穹的免费都是限时活动，需逐次确认。
3. 海外（需代理）可叠加：**Cloudflare Workers AI（每日 1 万 neurons，免费档模型偏小、大模型要付费）、OpenRouter :free（50 次/天）、Groq（有日限、快）**；HF 每月仅 $0.10 不实用。
4. **"免费 GPU 自部署每日刷新大上下文模型"不要投入生产**：Colab 官方禁止 web 服务/SSH/远程代理且随时断；Kaggle 每周 30 小时 + 16GB 显存只够小模型，且平台已在打击当 API 服务器的行为。
5. 接入建议：以**魔搭为主**（多模型轮换规避单模型 500 次上限），PPIO/书生做国内备份，OpenRouter/Groq/CF 做代理可用时的补充；任何源上线前先打一发 `/v1/chat/completions` 实测 HTTP 码与真实上下文，不要采信面板宣传数字。

> 免责：额度/窗口/限流均为 2026-09-30 当日公开来源的快照，标注【查不到】的项请以登录后的官方控制台/实测为准；本报告未对任何【查不到】的数字做推测填充。
