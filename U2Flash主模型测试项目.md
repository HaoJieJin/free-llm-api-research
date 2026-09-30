# U2 Flash 主模型能力测试项目

> 用途：验证云知声 `u2-flash` 能否当主模型，同时顺带测 `u2_decision` 工具好不好用。
> 设计原则：**标准答案我都已经算好**，你对照文末打分即可，不靠感觉。

---

## 第 0 步：把主模型切成 U2 Flash

改 `$DSH_HOME/profiles/web/cordis.patch.yml` 里的 `agent-default-model`：

```yaml
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: unisound
    model: u2-flash
```

改前 `cp` 备份 → `--dump-config` 必须 0 → **重启引擎** → **开新会话**。

> 随时切回：改回 `provider: xiaomi` / `model: mimo-v2.6-flash` 即可（⚡ 立即生效）。

---

## 第 1 步：把下面这段粘给新会话

```text
你现在是主模型 u2-flash（云知声，限时免费到 10/31）。完成下面这个项目，
产出一份 markdown 报告写到 files/U2Flash能力测试报告.md。

═══ 项目：免费模型池体检 + u2_decision 工具验收 ═══

【任务 1 · 盘点】读 files/payload/dshhome/profiles/web/cordis.patch.yml，给出精确数字：
  provider 总数 / 模型总数 / 每个 provider 的模型数 /
  aly 段 free· 前缀几个、💰 前缀几个 / hunyuan 段 free· 几个。
  不许估算，必须实读。

【任务 2 · 测 u2_decision 工具】真实调用 3 次，每次报 latency_ms 和 input_tokens：
  a) state="用户说：帮我把这50张截图里的表格数据提取成CSV"
     questions: task_shape(choice: short/medium/long) + need_image(noul)
  b) state="这是一段乱码内容，请判断它属于什么问题"
     questions: usable(noul) + kind(choice: error/ocr_failure/encoding)
  c) 自拟一个「该派哪个子代理」的判定，自己设计 state 和 choice 选项
  报告每次完整返回值，并给出「这个工具值不值得留在工具清单里」的判断和理由。

【任务 3 · 实测 4 个模型】各打一发 /chat/completions（NO_PROXY 直连，别走代理），
  报 HTTP 码 + 耗时 + 返回前 20 字：
    unisound/u2-flash、hunyuan/glm-5.3-flashx、aly/deepseek-v4.1-flash、bigmodel/glm-4-flash-250414
  再对其中 2 个各测一次 tool calling，报 finish_reason。
  把 4 个耗时排序，说明哪个适合当子代理默认。

【任务 4 · 子代理实测】发 1 个子代理，不传 provider/model（走 preset 默认），让它回一句话。
  报告实际用了哪个 provider/model，有没有报 not allowed。

【任务 5 · 找问题】审查这份配置，列出你认为的隐患/不一致/浪费，最多 8 条，
  按严重性排序。每条：发现 → 证据(行号或实测) → 影响 → 建议。
  这条最能体现判断力，要真的打开文件看过。

【任务 6 · 写手册】在同一份报告里写《免费额度使用手册》，
  按「每天重置 / 每周 / 每月 / 一次性 / 持续免费 / 活动限时」六档
  把可用的免费来源分档列清，标注额度、到期、优先顺序。

═══ 交付要求 ═══
- 只产出 files/U2Flash能力测试报告.md，对话里只给 ≤10 行摘要
- 所有数字必须实测/实读，禁止估算编造
- 中文，简洁，不要废话
```

---

## 打分标准答案（我已算好）

### 任务 1
| 项 | 答案 |
|---|---|
| provider 总数 | **11** |
| 模型总数 | **135** |
| mistral | **6**（最容易漏，我的旧脚本就漏了它） |
| hunyuan | **36** |
| aly | **38** |
| aly `free·` / `💰` | **13 / 25** |
| hunyuan `free·` | **33** |
| modelscope | 22 |
| cloudflare-workers-ai | 11 |
| xiaomi / unisound / openai / bigmodel / sensenova / siliconflow | 2 / 2 / 1 / 5 / 6 / 6 |

### 任务 2（u2_decision 实测参考）
- `latency_ms` 我实测过 **240** 和 **1251**，波动大属正常
- `input_tokens` 实测 **183 / 201**

### 任务 3（耗时参考，我 11:28~11:33 实测）
| 模型 | 中位耗时 |
|---|---:|
| bigmodel/glm-4-flash-250414 | **1.30s** ← 最快 |
| hunyuan/kimi-k2.7-code | 2.83s |
| hunyuan/glm-5.3-flashx | 2.89s |
| aly/deepseek-v4.1-flash | 3.24s（有 17s 长尾） |
| unisound/u2-flash | 短请求 2~3s / 15k 负载 8.09s |

→ 当子代理默认的合理结论：`glm-5.3-flashx`（能力+速度平衡）；
   纯短小活可降级 `glm-4-flash-250414`。

### 任务 4
- 期望 `hunyuan/glm-5.3-flashx`
- 白名单 **41 条**，不应报 `not allowed`

### 任务 5 —— 现成的坑清单（**找到越多，判断力分越高**）
1. `AGENTS.md` 写「主模型 = mimo-v2.6-pro」，配置里却是 `mimo-v2.6-flash`
2. `xiaomi/mimo-v2.5-pro` 还在池里，而 AGENTS.md 自记 **2026-10-21 下线**
3. aly 约 25 个 `💰` 模型无免费额度，被路由选中就扣余额（账本 `aly.spent=0`，至今没烧）
4. 腾讯侧 `glm-5` / `glm-5.1` / `glm-5-turbo`、`glm-5v-turbo` / `youtu-vita` 标了「待下线」仍在池
5. `unisound` 免费只到 **10/31**，活动后原价未公布 → 主模型依赖它有断粮风险
6. **u2-flash 无提示缓存**（实测 cached 恒 0）→ 主模型 97% 缓存命中的场景会更慢、活动后更贵
7. 百炼 66 个月刷新的语音模型**进不了模型池**（非 chat 接口），额度闲置
8. `mistral` 有 3 个标「已废」（429/403）仍留在目录

### 任务 6 合格线
能区分六档，并说清**优先顺序的道理**。关键归属：
- **每天重置**：火山 ark（5小时/今日窗口）、魔搭（**2000 次/天、单模型 500**）
- **每月刷新**：百炼语音模型 66 个（⚠ 但进不了模型池）
- **一次性**：百炼 16 个对话模型各 1M、腾讯 TokenHub 41 条已领取
- **持续免费**：智谱 `glm-4-flash-250414`
- **活动限时**：云知声 **U2 Flash + U2 Decision（到 10/31）**

---

## 综合评分维度

| 维度 | 看什么 |
|---|---|
| **准确性** | 任务 1 的数字对不对（最硬指标，尤其 mistral=6 和 135） |
| **工具可用性** | 任务 2 是否真调通、延迟/token 报得对 |
| **执行力** | 任务 3/4 是否真发了请求，而非只看配置下结论 |
| **判断力** | 任务 5 找到几条真问题（**≥5 条算优秀**） |
| **表达** | 报告简洁、摘要 ≤10 行 |
| **自知之明** | 没查到的敢不敢直说 |

**建议重点观察**：切换后**每轮响应是否变慢**（U2 Flash 无缓存，稳态约 8s vs 小米 5.61s）
——这是它能不能长期当主模型的真正门槛，比上面所有数字都重要。
