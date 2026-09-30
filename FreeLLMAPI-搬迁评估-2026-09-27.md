# FreeLLMAPI 搬到本机（OPPO PKS110 / Android 16）评估

日期：2026-09-27　结论：**能搬，已实测跑通；配置简单；不挂代理也有一批上游可用**

## 一、项目是什么

- 仓库：https://github.com/tashfeenahmed/freellmapi
- ★29,143　MIT　TypeScript　最近提交 2026-09-26（极活跃）
- 把 34 家免费 LLM 提供商的免费额度，聚合成**一个 OpenAI 兼容端点** `/v1`
- 智能路由 + 自动 failover + 按 key 记速率上限 + key 加密存 SQLite
- 同时暴露 `/v1/chat/completions`、`/v1/responses`、`/v1/embeddings`、`/v1/images`、
  Anthropic `/v1/messages`、Gemini `/v1beta`、Ollama 模拟、MCP server
- 商业模式：**路由器本身 MIT 永久免费**；免费版用「月度快照」目录（304 个模型），
  $19/年或 $49 买断解锁「实时目录」。功能不阉割，只是新模型晚 30 天到。

## 二、实测结果（本机真实跑通，非纸面推演）

服务现在就跑着：`http://127.0.0.1:3001`

| 验证项 | 结果 |
|---|---|
| 依赖安装 | ✅ 919 个包，pnpm 装成功 |
| 服务启动 | ✅ 监听 3001，`/api/ping` → 200 |
| 数据库 | ✅ 自动建库，走 **node:sqlite**（Android 分支） |
| 目录同步 | ✅ 从 api.freellmapi.co 拉到 **314 模型** / 35 embedding / 23 media / 73 quirk |
| 模型列表 | ✅ `/v1/models` 带 key → 200，返回 **250 个模型** |
| 鉴权 | ✅ 无 key → 401 |
| 路由引擎 | ✅ 返回 `no_provider_key`，正确报告 312 个模型缺 key |
| 前端 dashboard | ✅ vite/rolldown 构建成功，`/` → 200 |

跑通的关键：本机 Node 26.4.0 是 **Android 原生构建**（`process.platform === 'android'`），
FreeLLMAPI 正好为这种情况写了 `node:sqlite` 降级分支 → 一路畅通。

## 三、网络实测（不挂代理，直连）

| 上游 | 可达性 | 目录里模型数 |
|---|---|---|
| ModelScope 魔搭 | ✅ 200 | 11 |
| Cloudflare Workers AI | ✅ 200 | 25 |
| OpenRouter | ✅ 200 | 12 |
| 智谱 Z.ai | ✅ 200 | — |
| Groq | ✅ 403(=通) | 8 |
| Cerebras | ✅ 403 | — |
| NVIDIA NIM | ✅ 404(=通) | 18 |
| Cohere | ✅ 403 | 15 |
| Google Gemini | ❌ 超时 | 11 |
| Mistral | ❌ 超时 | 12 |
| HuggingFace | ❌ 超时 | 117 |

**结论：不挂代理能直接吃到 OpenRouter + Cloudflare + 魔搭 + 智谱 + Groq + Cerebras + NVIDIA + Cohere，
池子已经不小**（OpenRouter 一家就能转出很多免费模型）。Gemini/Mistral/HF 需要代理。

## 四、配置复杂度

极简。真正必须的只有一个：

```env
ENCRYPTION_KEY=<64位hex>
PORT=3001
```

其余全在 Web dashboard 点：粘贴各家 key → 排 Fallback Chain → 从 Keys 页抄走统一 key
（本项目现在这个：`freellmapi-<REDACTED>`）。

首次打开 dashboard 要用一次性 setup code 建账号（本次：`DLXMD5LKTK`，重启会变）。

也支持声明式 JSON（`keys` / `customProviders` / `models` / `routing`），适合写入版本管理。

**亮点**：支持「custom OpenAI 兼容端点」→ 可以把它指向别的网关，或反过来被别人指向。

## 五、搬运时踩到的坑（都已解决）

1. **pnpm 不认 `package.json` 的 `workspaces` 字段** → 必须建 `pnpm-workspace.yaml`
2. **pnpm 10+ 默认 `linkWorkspacePackages: false`** → 内部包 `@freellmapi/shared` 被当外部包去 registry 找，报 404
   → 开 `linkWorkspacePackages: true` + `preferWorkspacePackages: true`
3. **pnpm 12 换了构建脚本批准键**：老的 `onlyBuiltDependencies` 无效，会报 `ERR_PNPM_IGNORED_BUILDS`
   且**回滚整个 node_modules** → 正确写法是在 `pnpm-workspace.yaml` 里用：
   ```yaml
   allowBuilds:
     esbuild: true
     better-sqlite3: false
   ```
4. **`node_modules` 曾被回滚后 pnpm 误判「已是最新」**→ 需 `pnpm install --force`
5. **npm / git 本机没有** → 用 pnpm 代替 npm；源码用 codeload tarball 下（`raw.githubusercontent.com` 被墙，
   但 `api.github.com` 和 `codeload.github.com` 通）
6. **`tsc` 构建报一堆 TS2742**（pnpm 的 @types/express 隔离结构导致，不影响产物）
   → 官方 Android 路线本就走 `tsx` 免编译，直接 `tsx src/index.ts` 即可
7. **`sharp` 没有 Android 预编译包** → 无需处理：它是动态 import，缺失即静默关闭图片自动缩放，视觉功能仍可用
8. **`better-sqlite3` 在 Android 上装不了** → 无需处理：它是 optionalDependencies，官方就是这么设计的

## 六、接入 DSH 的注意事项

CLI 有 `npx freellmapi setup-dsh`，生成器写的是 **`$DSH_HOME/settings.yaml`**：
```yaml
llm-pi-ai:
  providers:
    freellmapi:
      displayName: FreeLLMAPI
      apiKeyEnv: FREELLMAPI_API_KEY
      api: openai-completions
      baseURL: http://localhost:3001/v1
      models: [...]
agent-default-model:
  provider: freellmapi
  model: <auto>
```

⚠ **本机不适用**：`settings.yaml` 是 DSH 旧格式，已被导入并改名为 `settings.yaml.imported`，
当前配置在 Cordis（`profiles/web/cordis.patch.yml`）。新写一个 `settings.yaml` 不会被再次导入。

✅ **正确做法**：手抄。生成器的 `llm-pi-ai.providers.<路由>` 结构和我们
`@deepseek-ai/dsh-llm-pi-ai` 条目的 `config.providers` **完全同构**，
把 freellmapi 那一段贴进 `config.providers` 下即可，约 10 行 YAML。

## 七、当前部署状态与待办

- 位置：`files/tmp/freellmapi/freellmapi-main`（**临时目录**）
- 进程：DSH bash 后台任务，**DSH 重启即死**
- 已有：`.env`（ENCRYPTION_KEY 已生成）、建好的 `server/data/freeapi.db`
- 待办：① 挪到 `files/freellmapi` 固定位置 ② 做常驻自启 ③ 灌 provider key ④ 接进 DSH

## 八、结论

- **能不能搬**：能，已经在本机跑通（不是"理论上可行"）
- **好不好用**：路由/鉴权/failover/目录同步全部正常；免费目录 314 个模型够玩；
  不挂代理也有 8 家上游可达；但定位是**个人实验**，无 SLA、免费额度会撞日限，别当生产用
- **配置方不方便**：非常方便。服务端只要 1 个 `ENCRYPTION_KEY`；其余全在 Web UI 点。
  唯一的"不方便"是接入 DSH 那一步因为本机配置已迁移到 Cordis，得手抄一段 YAML
