# Cloudflare Workers AI 接不通 —— 排查与修复（2026-09-27）

## 症状
会话里选 `DeepSeek V4 Pro / V4 Flash` 后每次都被顶回：

```
本轮运行失败  Provider is not configured: cloudflare-workers-ai   PI_AI_ERROR
```

会话日志（`session-e38483bb-…/session.v4.jsonl.zstd`）里 4 轮全是同一个错误，
`usage` 全 0 —— 请求根本没发出去。

## 根因（已用实验证实，不是猜）
pi-ai 的 Cloudflare 鉴权**同时**需要两样东西（见
`pi-ai/dist/providers/cloudflare-auth.js` → `resolveCloudflareEnv`）：

1. **API Key** —— 来自 profile 的 `apiKeyEnv: CLOUDFLARE_WORKERS_AI_API_KEY`（已有 ✅）
2. **Account ID** —— 读 `ctx.env("CLOUDFLARE_ACCOUNT_ID")`（**缺失 ❌**）

两个都在才返回鉴权结果；只要缺一个就返回 `undefined`，pi-ai 的 `applyAuth`
随即抛 `Provider is not configured: <provider>`（`pi-ai/dist/models.js:366`）。

harness 侧的 `ctx.env()` 兜底链（`dsh-llm-pi-ai` 的 `authContextFrom`）是：
**credential refs（`$DSH_HOME/.credentials.yaml` 的 `refs:`）→ 进程环境变量**。
所以只要 refs 里有同名 `CLOUDFLARE_ACCOUNT_ID` 就能被读到。

复现实验（同一份 pi-ai 代码，直接跑真实请求）：

| authContext 里的 refs | 结果 |
|---|---|
| 无 `CLOUDFLARE_ACCOUNT_ID` | `stopReason: error`，`errorMessage: "Provider is not configured: cloudflare-workers-ai"`，usage 全 0（= 用户截图里的一模一样） |
| 有 `CLOUDFLARE_ACCOUNT_ID` | `stopReason: stop`，正常返回 `"pong"` |

## 已做的修复
1. `$DSH_HOME/.credentials.yaml` 的 `refs:` 增加
   `CLOUDFLARE_ACCOUNT_ID: <REDACTED>`
   （备份：`.credentials.yaml.bak-20260927-131228`）
   → 该文件由 `dsh-credentials-local` **chokidar 监听并热生效**，不用重启引擎。
2. 实测验证：在跑着的引擎（pid 8938）上用 `workflow` 发了一发真实请求
   （provider `cloudflare-workers-ai` / model `@cf/openai/gpt-oss-120b`）→ **成功返回**。
   （同一通道换成 `@cf/meta/llama-3.3-70b-instruct-fp8-fast` 会 400
   `CONTEXT_WINDOW_EXCEEDED`：只有 24k 上下文，装不下 DSH 的系统提示词 + 全部工具 schema。）

## 第二个坑：这个账号是 Workers **Free** 计划
用该 token 直连 `https://api.cloudflare.com/client/v4/accounts/<id>/ai/v1/chat/completions`
逐个实测 18 个模型：

**❌ 403 / code 5035「not available on the Workers Free plan」**（正是用户最初想用的那几个）：
`@cf/deepseek-ai/deepseek-v4-pro-0813`、`@cf/deepseek-ai/deepseek-v4-flash-0731`、
`@cf/moonshotai/kimi-k2.6`、`@cf/moonshotai/kimi-k2.7-code`、
`@cf/zai-org/glm-5.2`、`@cf/zai-org/glm-5.3`、`@cf/zai-org/glm-5.3-flash`

**✅ 可用**：gpt-oss-120b、gpt-oss-20b、qwen3.8-27b(图片)、nemotron-3-120b、
gemma-4-26b(图片)、llama-4-scout(图片)、mistral-small-3.1-24b、glm-4.7-flash、
granite-4.0-h-micro、（偏小）qwen3-30b-a3b-fp8 32k、llama-3.3-70b 24k

token 本身没问题：`/user/tokens/verify` 返回 `status: active`。

## 配置改动（`profiles/web/cordis.patch.yml`）
- 模型列表按「实测可用 + 上下文够用」重排：9 个推荐模型在前，2 个 32k/24k 的标注偏小，
  7 个需要 Workers Paid 的**整段注释掉**并写明原因（升级后取消注释即可）。
- 各模型 `maxTokens` 从「等于 contextWindow」（如 262144）改成 8192–32768，
  避免向 CF 发送离谱的 `max_tokens`。
- 备份：`cordis.patch.yml.bak-20260927-131228`
- 校验：`dsh --profile web --dump-config` 退出码 0，composed 配置里 CF 有 11 个模型；
  另用插件的 `Config` schema 跑了一遍校验，通过。
- ⚠ **profile patch 不热加载**：模型列表要**重启引擎**才生效
  （凭据那条已经热生效，不重启也能用已有的模型）。

## 另一条「官方」路子（可选）
设置 → 模型 → Cloudflare Workers AI → 登录/添加密钥：pi-ai 的 CF 登录流程本来就会
**同时问 API Key 和 Account ID**，并把它俩一起写进凭据记录（`{key, env:{CLOUDFLARE_ACCOUNT_ID}}`）。
比手工配 `apiKeyEnv` 少踩这个坑。

## 回滚
```sh
cd $DSH_HOME
cp .credentials.yaml.bak-20260927-131228 .credentials.yaml
cp profiles/web/cordis.patch.yml.bak-20260927-131228 profiles/web/cordis.patch.yml
```
