# U2Flash 能力测试报告

> 测试时间：2026-09-30 15:42~15:58（Asia/Shanghai）｜主模型：unisound/u2-flash（本会话）
> 数据来源：全部为实读 `payload/dshhome/profiles/web/cordis.patch.yml` + 真实 HTTP 请求 + 工具真实返回，无估算。
> 说明：项目里写的 `files/...` 相对路径在本机即工作目录 `/data/user/0/com.deepseek.harness/files` 下的文件。

---

## 任务 1 · 免费模型池盘点（实读 cordis.patch.yml，共 1631 行）

- **provider 总数：11**
- **模型总数：135**
- 各 provider 模型数（脚本按 `models:` 段的 `- id:` 逐条统计）：

| provider | 模型数 |
|---|---:|
| xiaomi | 2 |
| unisound | 2 |
| openai | 1 |
| bigmodel | 5 |
| hunyuan | 36 |
| aly | 38 |
| modelscope | 22 |
| cloudflare-workers-ai | 11 |
| sensenova | 6 |
| siliconflow | 6 |
| mistral | 6 |

- **aly 段**：`name: "free·` 前缀 **13** 个，`name: "💰` 前缀 **25** 个（13+25=38 ✓）
- **hunyuan 段**：`name: "free·` 前缀 **33** 个（36 个模型 − 3 个无前缀的 hy4-preview/hy3/hy-vision-2.0-instruct = 33 ✓）

---

## 任务 2 · u2_decision 工具验收（3 次真实调用）

### 调用 a
- state：`用户说：帮我把这50张截图里的表格数据提取成CSV`
- questions：`task_shape(choice: short/medium/long)` + `need_image(noul)`
- 完整返回值：
```json
{"answers":{"task_shape":{"type":"choice","choice":"medium","probabilities":{"short":0.171594,"medium":0.562636,"long":0.26577},"confidence":0.5626355989395554},"need_image":{"type":"noul","noul":0.9820137910906878}},"latency_ms":180,"input_tokens":201}
```
- 解读：判 medium（0.56）；need_image noul=0.982（需要看图）——合理。

### 调用 b
- state：`这是一段乱码：�...`
- questions：`usable(noul)` + `kind(choice: error/ocr_failure/encoding)`
- 完整返回值：
```json
{"answers":{"usable":{"type":"noul","noul":0.016914909302112344},"kind":{"type":"choice","choice":"encoding","probabilities":{"error":0.045484,"ocr_failure":0.09629,"encoding":0.858225},"confidence":0.8582254703583022}},"latency_ms":169,"input_tokens":186}
```
- 解读：usable noul=0.017（不可用）；kind=encoding（编码问题）——合理。

### 调用 c（自拟：该派哪个子代理）
- state：`主代理收到一个任务：读取本地一份 2000 行的日志文件，找出所有报错并分类汇总。当前子代理额度：hunyuan/glm-5.3-flashx 可用、bigmodel/glm-4-flash-250414 可用、openai/ark-code-latest 今日已耗尽。`
- questions：`delegate(choice: glm_5_3_flashx / glm_4_flash / ark / no_delegate)` + `need_file_read(noul)`
- 完整返回值：
```json
{"answers":{"delegate":{"type":"choice","choice":"glm_4_flash","probabilities":{"glm_5_3_flashx":0.378882,"glm_4_flash":0.586824,"ark":0.005077,"no_delegate":0.029216},"confidence":0.5868244015180337},"need_file_read":{"type":"noul","noul":0.9982992772849691}},"latency_ms":227,"input_tokens":400}
```
- 解读：正确排除已耗尽的 ark（概率 0.005），选了 glm_4_flash（持续免费，0.587）；need_file_read noul=0.998（需读文件）——合理。

### 结论：u2_decision 值不值得留在工具清单里

**值得留。** 理由：
1. 3 次全部成功，延迟 169~227ms、输入 186~400 tokens，开销极低（约 200~400 tok/次）。
2. 输出是结构化概率（choice + probabilities + confidence），不是自然语言，适合程序化分支（任务分型 / 要不要看图 / 路由决策 / 门槛过滤）。
3. 判定质量合理：能识别「50 张截图 → 需要看图」、乱码类别、「已耗尽的路由不该派」。
4. 局限：概率有时不够尖锐（如 medium 仅 0.56），适合做「快速预筛」，不能完全替代主模型判断。

---

## 任务 3 · 4 个新模型直连实测（NO_PROXY，未走代理）

### /chat/completions 直连结果（多次实测）
| 模型 | HTTP | 耗时（多次） | 返回前 20 字 |
|---|---|---|---|
| unisound/u2-flash | 200 | 3.37s / 3.53s | 我是由云知声研发的人工智能助手U2.1， |
| hunyuan/glm-5.3-flashx | 200 | 2.63s / 3.30s / 4.67s / 11~13s* | 我是Z.ai开发的GLM大语言模型，通过海量文本数 |
| aly/deepseek-v4.1-flash | 200 | 1.66s / 2.91s / 3.06s | 我是由深度求索公司开发的AI助手，随时愿 |
| bigmodel/glm-4-flash-250414 | 200 | 0.50s / 0.77s / 1.22s | 我是人工智能助手智谱清言，基于智谱 AI |

\* 注：glm-5.3-flashx 是**思考型模型**（响应带 `reasoning_content`）。max_tokens≤1000 时多次出现 `finish_reason=length` 且 content 为空（推理吃光预算），7 次实测里 4 次空正文；max_tokens 加大后能正常返回。

### tool calling 实测（get_weather 工具，测了 3 个）
| 模型 | finish_reason | 工具参数 |
|---|---|---|
| hunyuan/glm-5.3-flashx | **tool_calls**（4.68s） | `{"city":"北京"}` ✓ |
| aly/deepseek-v4.1-flash | **tool_calls**（1.65s） | `{"city": "北京"}` ✓ |
| bigmodel/glm-4-flash-250414 | **tool_calls**（0.46s） | `{"city": "北京"}` ✓ |

### 耗时排序（快 → 慢）
1. **bigmodel/glm-4-flash-250414**：0.5~1.2s
2. **aly/deepseek-v4.1-flash**：1.7~3.1s
3. **hunyuan/glm-5.3-flashx**：2.6~4.7s（思考型，有 11~13s 长尾）
4. **unisound/u2-flash**：3.4~3.5s

### 谁适合当子代理默认
- **短小简单任务默认**：`bigmodel/glm-4-flash-250414`——最快、稳定、持续免费、tool call 通。
- **长/难任务**：`aly/deepseek-v4.1-flash`——1M 窗口、免费 1M tokens、tool call 通。
- **不建议**把 `hunyuan/glm-5.3-flashx` 当默认：思考型 + 空 content 风险 + 延迟不稳定（引擎侧子代理实测能出正文，但直连表现差）。

---

## 任务 4 · 子代理实测（不传 provider/model）

- 发送 1 个子代理，prompt 为「报告你当前使用的模型名称」。
- 子代理回复：**「我当前运行在 glm-5.3-flashx 模型上（provider 未在系统提示中说明）」**。
- **实际路由：hunyuan/glm-5.3-flashx**（与 patch 中 preset 的 `agentOptions` 一致）。
- **未报 `not allowed`**（白名单 41 条覆盖该路由）。

---

## 任务 5 · 配置审查（8 条，按严重性排序）

### 1【高】主模型三方不一致：mimo-v2.6-pro / mimo-v2.6-flash / u2-flash
- 发现：AGENTS.md 写「主模型 = xiaomi/mimo-v2.6-pro」；patch L115 注释「mimo-v2.6-pro … ← 主模型用它」；patch L176 注释「主模型继续用 xiaomi/mimo-v2.6-flash」；实际 `agent-default-model`（L91-92）= unisound/u2-flash。
- 证据：patch L88-92、L115、L176 + AGENTS.md。
- 影响：测试结束后按注释切回会切错型号；文档互相矛盾。
- 建议：统一为唯一主模型（建议 mimo-v2.6-pro），同步 AGENTS.md、patch 注释、实际配置三处。

### 2【高】默认子代理 glm-5.3-flashx 是思考型，直连多次空 content
- 发现：2026-09-30 三改后 default 路由 = `hunyuan/glm-5.3-flashx`（L1166-1167 等 6 处 agentOptions），但直连实测它带 reasoning_content，max_tokens≤1000 时 4/7 次空正文、finish=length、耗时 11~13s。
- 证据：本报告任务 3 实测。
- 影响：子代理可能拿到空结果（若引擎侧 max_tokens 不够大）；思考型烧 token、延迟高。
- 建议：default 改回 `glm-4-flash-250414`（0.5s 直出），或在 agentOptions 显式加大 max_tokens 并验证。

### 3【中】aly 池 25 个 💰 付费模型无免费额度，误选即扣余额
- 发现：aly 38 个模型里 25 个标 💰；《阿里云百炼89免费额度统计》明确「池里有约 17 个百炼模型没有免费额度」。
- 证据：patch L425-525；`阿里云百炼89免费额度统计-2026-09-30.md`；state.json 中 `aly.spent=0`（至今未烧）。
- 影响：模型选择器可见，用户误选一次就扣 ALY 余额（当前 ¥10.01）。
- 建议：从目录移除无免费额度的付费模型，或加「⚠付费」强标识。

### 4【中】mimo-v2.5-pro 已公告 2026-10-21 下线，仍留在两个目录
- 发现：L119 写「目录里不列，别用 v2.5」，但 hunyuan L389 有 `free·mimo-v2.5-pro ⚠10-21下线`、aly L522 有 `💰xiaomi/mimo-v2.5-pro`。
- 证据：patch L119 vs L389/L522。
- 影响：用户可能选到即将下线的模型；10-21 后必然报错。
- 建议：两条都从目录移除。

### 5【中】hunyuan 池 5 个「待下线」模型仍展示
- 发现：glm-5v-turbo（L353）、youtu-vita（L359）、glm-5（L371）、glm-5.1（L379）、glm-5-turbo（L387）名字标了「⚠待下线」仍留在目录。
- 证据：patch 对应行。
- 影响：冗余展示、可能误选；下线后报错。
- 建议：移除或折叠到「已下线备查」区。

### 6【中】u2-flash 限时免费到 10/31、无提示缓存、活动后原价未公布，却当主模型
- 发现：patch L160-162 注明活动 09/30-10/31 全 0 元、活动后原价未列出；L168-169 实测无提示缓存（cached 恒 0）。
- 证据：patch L160-169；state.json 中 unisound 已 41 次请求。
- 影响：10/31 后主模型可能断粮或价格未知；无缓存导致每轮稳态 ~8s。
- 建议：测试完立即切回小米主模型；u2-flash 只作免费子代理，10/31 前用完。

### 7【低】dsh-quota-check ROUTES.default 有重复条目
- 发现：`aly/deepseek-v4.1-flash` 出现两次（L98/L102）、`hunyuan/glm-5.3-flashx` 出现两次（L93/L103）。
- 证据：`tools/bin/dsh-quota-check` L92-108；`--pick default` 输出也重复打印。
- 影响：无功能影响，但输出冗余、易误导。
- 建议：清理重复项。

### 8【低】文档滞后：AGENTS.md / 技能仍写子代理默认 = ark-code-latest
- 发现：AGENTS.md 与 cost-saving-routing 技能仍写「子代理默认路由 = openai/ark-code-latest」，patch 已在 2026-09-30 11:18 三改为 hunyuan/glm-5.3-flashx（L1157-1167）。
- 证据：AGENTS.md；patch L1157-1167。
- 影响：按旧文档理解会误判实际路由。
- 建议：同步更新 AGENTS.md 与技能文档。

---

## 任务 6 · 免费额度使用手册（明天就能用）

### 每天重置（先用光，不用就浪费）
| 来源 | 额度 | 说明 |
|---|---|---|
| 火山方舟 `openai/ark-code-latest` | 5小时/今日窗口，每天重置（当前剩 85.5%） | 实测 2.4s 直出、256K；短小任务首选 |
| 魔搭 ModelScope `modelscope/*` | 魔粒制：每天登录 +200、绑阿里云 +50 = 250 魔粒/天，有效期 1 天（旧说法「2000 次/天」已失效） | DeepSeek-V4-Flash-0731（1.31M）、Qwen3.8-27B 等；今天已用完，明天重置 |
| Cloudflare Workers AI | 10k neurons/天，北京 08:00 重置 | 已移出子代理，专供 ModLens 识图 |

### 每周重置
| 来源 | 额度 | 说明 |
|---|---|---|
| 商汤 SenseNova `sensenova/*` | 积分制：通用池 60 万/周 + 6 万/5h | deepseek-v4-flash（1.31M）；日额度用光才轮到它 |

### 每月重置
| 来源 | 额度 | 说明 |
|---|---|---|
| 阿里云百炼语音模型（~66 个 sambert/paraformer） | 每月 1 日重置、长期有效 | ⚠ 非 chat 接口，进不了模型池，对 agent 不可用（额度闲置） |

### 一次性（过期作废，优先用）
| 来源 | 额度 | 到期 |
|---|---|---|
| 阿里云百炼对话模型 16 个 | 各 1M tokens，一次性 | 2026/10/23~12/21；重点：deepseek-v4.1-flash 到 12/13、deepseek-v4-flash-0731 到 10/31、qwen3.7-flash 到 10/23 |
| 腾讯 TokenHub 41 条（`free·`） | 各 100%（新账号一次性免费约 100 万 tokens，官方 techpedia）；刷新规则页面未显示 | 未显示到期日；已领取，尽快用 |

### 持续免费（保底，无额度概念）
| 来源 | 说明 |
|---|---|
| 智谱 `bigmodel/glm-4-flash-250414` | 官方免费页未标到期；实测 0.5~1.3s 直出、tool call 通 → 最佳保底 |
| 智谱 `glm-4.7-flash` | 免费但会 429 + 思考型（实测一次 146s），降级用 |
| 智谱 `glm-4.6v-flash` | 免费视觉，可看图/视频（高峰会 429） |

### 活动限时
| 来源 | 说明 |
|---|---|
| 云知声 U2 Flash + U2 Decision | 2026.09.30-10.31 输入/缓存/输出全 0 元；512K 窗口；⚠ 无提示缓存、稳态 ~8s；活动后原价未公布 → 10/31 前用完，别当长期主模型 |

### 明天（10/1）优先顺序
1. **先烧日额度**：火山 ark（今日窗口）+ 魔搭魔粒（250/天，明天重置）
2. **再烧一次性**：百炼临近到期的（deepseek-v4-flash-0731 到 10/31、qwen3.7-flash 到 10/23）、腾讯 TokenHub 41 条
3. **保底**：智谱 `glm-4-flash-250414`（不耗任何额度池）
4. **周额度**：商汤（最后）
5. **活动**：U2 Flash 可作免费子代理（10/31 截止）
6. **付费兜底**：小米 mimo / DeepSeek 官方（尽量不碰）

---

### 附：本报告的关键实测证据
- patch 文件：`payload/dshhome/profiles/web/cordis.patch.yml`（1631 行，实读全文）
- 模型直连：Python urllib + `ProxyHandler({})` 直连（NO_PROXY），真实 HTTP 200
- u2_decision：工具真实返回 3 次（latency_ms 169/180/227，input_tokens 186/201/400）
- 子代理：后台子代理真实回复「glm-5.3-flashx」，未报 not allowed