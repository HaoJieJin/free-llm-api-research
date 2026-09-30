# 魔搭社区 ModelScope 免费 API —— 配置记录（2026-09-27）

## 接入信息
- Base URL：`https://api-inference.modelscope.cn/v1`（OpenAI 兼容）
- API Key：存为 credential ref `MODELSCOPE_API_KEY`（值 `ms-fcc7f677-…`）
- 模型清单：`GET /v1/models`（该 key 实测返回 35 个条目）
- 注意：旧文档里的 `Qwen/Qwen3-235B-A22B` 已下线（400 has no provider supported）；
  `MiniMax/MiniMax-M3` 也挂在列表但无服务（400）。

## 22 个实测可用模型（2026-09-27，逐个真实请求验证）
- **DeepSeek V4（推理）**：deepseek-ai/DeepSeek-V4-Pro-0813、DeepSeek-V4-Pro、
  DeepSeek-V4-Flash-0731、DeepSeek-V4.1-Flash
  → reasoning 字段与 compat（thinkingFormat: deepseek 等）照抄 pi-ai 内置 deepseek catalog；
  返回里思考内容在 `reasoning_content`。
- **Qwen**：Qwen3.5-397B-A17B、Qwen3.5-122B-A10B、Qwen3.5-35B-A3B、Qwen3.5-27B、
  Qwen3.8-27B、Qwen3.8-Flash-Next
- **GLM**：ZhipuAI/GLM-5.2、GLM-4.7-Flash
- **阶跃**：stepfun-ai/Step-3.7-Flash、Step-3.5-Flash
- **MiniMax**：MiniMax/MiniMax-M1-80k
- **书生**：Shanghai_AI_Laboratory/Intern-S2-Preview、Intern-S1-mini
- **文心**：PaddlePaddle/ERNIE-4.5-300B-A47B-PT
- **视觉（可发图）**：OpenGVLab/InternVL3_5-241B-A28B、PaddlePaddle/ERNIE-4.5-VL-28B-A3B-PT
- **轻量**：meituan-longcat/LongCat-Flash-Lite、nex-agi/Nex-N2.5-Pro

亮点：**Cloudflare Workers Free 用不了的 DeepSeek V4 Pro/Flash，魔搭免费就能用。**

## 配置落点
- 路由 `modelscope` 写在 `$DSH_HOME/profiles/web/cordis.patch.yml` 的 `llm-pi-ai.config.providers`
  （pi-ai 不认识的路由 → 显式 `api: openai-completions` + baseURL + 每模型全字段）
- Key 写在 `$DSH_HOME/.credentials.yaml` refs
- 备份：`cordis.patch.yml.bak-20260927-132810`、`.credentials.yaml.bak-20260927-132810`
- 校验：`dsh --profile web --dump-config` 退出码 0（22 个模型）；
  独立 trialhome 隔离试启动（端口 3099）干净通过。

## ⚠ 生效方式
profile patch 不热加载 → **重启引擎**后模型列表才出现（把 App 划掉重开即可）。
凭据文件热生效，无需单独处理。
