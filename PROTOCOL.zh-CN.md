# 伴侣互通协议 CIP v0.1（草案）

> Companion Interop Protocol。本清单收录项目之间「怎么互相接上」的约定。
> 配套文件：[`spec/companion.schema.json`](spec/companion.schema.json)（描述文件的 JSON Schema）、[`spec/examples/`](spec/examples/)（按真实项目写的示例）。
> 整体架构见 [INTEGRATION.zh-CN.md](INTEGRATION.zh-CN.md)。本文只管边界：项目怎么描述自己、怎么被配置、说什么话、怎么交换数据、前端要满足什么。

---

## 0. 定位

### 0.1 这是协议，不是框架

- **不需要装任何库。** 遵守协议 = 在仓库里放一个 JSON 文件，外加按约定暴露几个配置项。
- **不规定你怎么写代码。** 语言、架构、存储、界面风格全不管。
- **描述现状优先于提出要求。** 协议先描述大家已经在做的事。已有的常见写法（`system_prompt.txt`、`.env.example`、AstrBot `_conf_schema.json`、MCP 工具）默认就算合规。
- **不是收录门槛。** 清单收录与是否遵守 CIP 无关，CIP 只影响「能不能被一键接入」。

### 0.2 为什么现在能定

对清单里 195 个 GitHub 仓库做了目录结构统计（按文件名，偏粗）：

| 现象 | 数量 | 协议里对应 |
| --- | --- | --- |
| 目录里有 MCP 相关文件 | 69（35%） | §6.1 MCP 约定 |
| 带配置样例（`.env.example`、`config.example.*`、`_conf_schema.json`） | 62 | §5.1、§4 `config` 字段 |
| 提示词 / 人设放在独立文件或目录 | 44 | §5.2 提示词外置 |
| 已自带工具描述文件 `tool-schema.json` | 2（词语屋、钓鱼游戏） | §6.3 工具描述 |

同时也看到了分歧，协议要消解的正是这些：

- 词语屋的 `tool-schema.json` 用 MCP 的 `inputSchema`，钓鱼游戏用 OpenAI function 的 `parameters`。两种都要认（§6.3）。
- 前端各自发明本地存储键名，例如 chatnest 用 `chat_token`、`chat_conversation`、`chat_model`，而且默认模型名写死在代码里。外部工具想同步提示词，只能逐个猜（§8.4）。
- 很多客户端能填 Base URL，但也有的只认自家后端 API。对接前得先读代码才知道（§8.2）。

### 0.3 用词

- **必须**：不做到就不算达到该级别。
- **应当**：强烈建议，有理由可以不做，但要在描述文件里说明。
- **可以**：纯可选。

---

## 1. 角色

| 角色 | 是什么 | 例子 |
| --- | --- | --- |
| 项目 | 清单里的任意一项 | 词语屋、kiwi-mem、RikkaHub |
| 宿主 | 能加载插件的应用 | AstrBot、SillyTavern、Claude Code |
| 中枢 | 把多个项目编排在一起的程序，参考实现见 INTEGRATION | Companion Hub |
| 适配器 | 替某个项目补齐协议的薄层，可由第三方维护 | `adapters/rikkahub/` |
| 前端 | 负责和人交互的界面，本身可以不含任何「大脑」 | chatnest、the-house、Kelivo |
| 描述文件 | `companion.json`，项目的「名片」 | 见 §4 |

一个项目可以同时是多种角色，例如 SullyOS 既是前端，又自带记忆。

---

## 2. 形态 `kind`

描述文件里必须写 `kind`，只选最主要的一个；次要形态写在 `also` 里。

| `kind` | 含义 | 典型 |
| --- | --- | --- |
| `service` | 独立运行的服务：MCP 服务器、HTTP 服务、代理网关 | kiwi-mem、ai-memory-gateway、各类 `*-mcp` |
| `host-plugin` | 必须装进某个宿主才能用 | `astrbot_plugin_*`、Claude Code 技能 |
| `game` | 给 AI 玩的游戏，文字接口 | 词语屋、钓鱼游戏、aifarm |
| `app` | 完整应用，自带界面和大脑 | Operit、Aura、Open-LLM-VTuber |
| `frontend` | 以界面为主，大脑在外面 | chatnest、LumiMuse、ZeroChat |
| `web-page` | 单文件或纯静态网页，无后端 | the-house、dwell-on-something |
| `hardware` | 硬件桥：玩具、机器人、传感器 | stackchan-mcp、cachito-ble-mcp-relay |
| `prompt-pack` | 提示词、角色卡、技能包，没有可执行代码 | 人设模板、Skill 合集 |
| `doc` | 教程、规范、清单 | XSJDeveloperGuide、character-card-spec |

---

## 3. 分级

分两条线：**能力等级 C0–C3** 适用于所有项目；**前端等级 F0–F3** 只对有界面的项目（`frontend`、`app`、`web-page`）额外适用，见 §8。

每一级单独都有用，不要求按顺序做完。检查器按实际满足的条件计算等级，描述文件里的 `level` 只是自报。

| 等级 | 名称 | 判定条件（必须全满足） | 作者成本 | 换来什么 |
| --- | --- | --- | --- | --- |
| **C0** | 可发现 | 有一份合法的 `companion.json`：可以放在自己仓库，也可以由社区放进 `registry/` | 5 分钟，不改代码 | 中枢知道你是什么、怎么启动、提示词和数据在哪，可以对你做「强行兼容」 |
| **C1** | 可配置 | C0，且满足 §5 五个接缝中与你相关的**全部**必须项 | 每个接缝 5–30 行 | 不用改你的代码就能换模型、同步提示词、导出数据、关掉冲突功能 |
| **C2** | 说通用语言 | C1，且至少实现一种 §6 的通用接口 | 大部分项目已经做到 | 即插即用，不需要专门写适配器 |
| **C3** | 可协作 | C2，且实现 §7 的事件、健康检查、导入 | 较多 | 可以接入中枢的调度、守卫、快照回滚 |

「与你相关」的判定：没有调用模型就跳过 S1，没有提示词就跳过 S2，依此类推。描述文件里对应字段写 `null` 表示「不适用」，与「没写」区分开。

---

## 4. 描述文件 `companion.json`

### 4.1 放在哪

按优先级查找，找到第一个即停：

1. 仓库根目录 `companion.json`
2. `.companion/companion.json`
3. 本清单仓库的 `registry/<owner>/<repo>.json`：**社区代写**，作者无需任何操作

作者自己的文件优先于社区代写。社区代写的文件必须带 `"maintainer": "community"`。

### 4.2 通用规则

- 编码 UTF-8，标准 JSON，不允许注释。需要注释时用 `"$comment"` 字段。
- 遇到不认识的字段**必须忽略**，不得报错。这是协议能只做加法的前提。
- 自定义字段以 `x-` 开头，例如 `"x-sullyos": {...}`。
- **已有的宿主清单不必重复。** AstrBot 插件的 `metadata.yaml` 里已经有名称、版本、作者、支持平台，`companion.json` 只写那里没有的东西；读取方会合并两者。`package.json`、`pyproject.toml` 同理。
- 路径一律相对仓库根目录，用 `/` 分隔，可以用 glob（`characters/*.yaml`）。
- 「配置位置」统一用**定位串**表示，格式见 4.4。

### 4.3 字段

只有前五个必填，其余按需。

| 字段 | 必填 | 类型 | 说明 |
| --- | --- | --- | --- |
| `cip` | 是 | string | 协议版本，目前为 `"0.1"` |
| `id` | 是 | string | 小写字母、数字、短横线，全清单唯一，默认用仓库名 |
| `name` | 是 | string | 显示名，可以是中文 |
| `kind` | 是 | enum | 见 §2 |
| `repo` | 是 | string | 仓库 URL；闭源项目写主页 |
| `also` | | enum[] | 次要形态 |
| `summary` | | string | 一句话介绍 |
| `license` | | string | SPDX 标识；没有许可证写 `"NONE"`，适配器据此决定能不能拆代码 |
| `maintainer` | | enum | `author`（默认）或 `community` |
| `level` | | string | 自报等级，如 `"C1"`、`"C2+F1"` |
| `run` | | object | 怎么启动，见 4.5 |
| `host` | | object | `host-plugin` 必填：`{"name": "astrbot", "min_version": "..."}` |
| `speaks` | | object[] | 已经会说的通用接口，见 §6 |
| `tools` | | string / object | 工具描述文件路径或内联，见 §6.3 |
| `llm` | | object / null | S1 模型端点，见 §5.1 |
| `prompts` | | object[] / null | S2 提示词位置，见 §5.2 |
| `data` | | object[] / null | S3 数据位置与导出方式，见 §5.3 |
| `autonomy` | | object / null | S4 自主行为开关，见 §5.4 |
| `events` | | object / null | S5 webhook 与 C3 事件入口，见 §5.5、§7 |
| `health` | | string | 健康检查定位串，见 §7.2 |
| `config` | | object | 其余配置：`{"schema": "_conf_schema.json", "example": ".env.example"}` |
| `ui` | | object | 前端档案，见 §8 |
| `permissions` | | string[] | 需要的敏感权限：`network`、`filesystem`、`microphone`、`camera`、`location`、`device.ble`、`device.intimate`、`payment`、`shell` |
| `adaptation` | | object | 仅适配器使用，见 INTEGRATION §5 |

### 4.4 定位串

用一个字符串说清「这个值在哪、怎么改」。所有需要指向某个配置的字段都用它。

| 前缀 | 格式 | 例子 |
| --- | --- | --- |
| `env:` | 环境变量名 | `env:OPENAI_BASE_URL` |
| `file:` | `file:<路径>#<JSON Pointer>`；YAML、TOML、JSON 都用 JSON Pointer；没有 `#` 表示整个文件 | `file:conf.yaml#/character_config/persona_prompt` |
| `block:` | `block:<路径>#<块名>`，指文件里的托管块，见 5.2.3 | `block:CLAUDE.md#persona` |
| `http:` | `http:<方法> <路径>`，相对 `run.url` | `http:PUT /api/persona/{id}` |
| `cli:` | `cli:<命令>` | `cli:python export.py --out {file}` |
| `storage:` | `storage:<localStorage|indexedDB>:<键或库/表>` | `storage:localStorage:chat_model` |
| `ui:` | `ui:<人能看懂的操作路径>`，只能手动 | `ui:设置 → 模型 → 自定义地址` |
| `host:` | `host:<宿主>:<宿主内的位置>` | `host:astrbot:persona` |

`{id}`、`{file}` 这类花括号是占位符，由调用方替换。

### 4.5 `run`

```json
"run": {
  "type": "stdio",
  "command": ["python", "server.py"],
  "cwd": ".",
  "env": {
    "OPENAI_API_KEY": { "required": true, "secret": true, "description": "模型密钥" },
    "DATA_DIR": { "required": false, "default": "./data" }
  },
  "requires": ["python>=3.10"]
}
```

| `type` | 附加字段 | 用于 |
| --- | --- | --- |
| `stdio` | `command` | 本地 MCP、命令行游戏 |
| `http` | `command`（可选）、`url`、`port` | 本地或远程服务 |
| `static` | `entry`，如 `dist/index.html` | 纯前端、单文件网页 |
| `host` | 无，看 `host` 字段 | 宿主插件 |
| `app` | `platforms`：`android`、`ios`、`windows`、`macos`、`linux`、`web` | 需要安装的应用 |
| `cloud` | `url` | 只提供在线服务 |
| `none` | 无 | 提示词包、文档 |

`env` 里 `secret: true` 的变量，工具在显示、日志、导出时必须遮蔽。

### 4.6 最小例子

社区代写一份 C0，只需要这些：

```json
{
  "cip": "0.1",
  "id": "the-house",
  "name": "the-house",
  "kind": "web-page",
  "repo": "https://github.com/wuliu0012/the-house",
  "maintainer": "community",
  "run": { "type": "static", "entry": "index.html" }
}
```

完整示例见 [`spec/examples/`](spec/examples/)。

---

## 5. 五个接缝（C1）

「接缝」指预留给外部的最小入口。每个接缝都给出：要求、推荐写法、描述文件怎么写。

### 5.1 S1 模型端点可改

**要求**
- 必须：凡是调用大模型的地方，Base URL、API Key、模型名三者都能在**不改代码**的前提下修改（环境变量、配置文件或界面都行）。
- 应当：Base URL 填写到 `/v1` 为止，程序自己拼 `/chat/completions`。已经要求填完整 endpoint 的项目（例如 kiwi-mem 的 `API_BASE_URL` 填到 `/chat/completions`）不用改，在 `llm.url_style` 写 `"full-endpoint"`，外部工具据此拼接。
- 应当：支持 OpenAI Chat Completions 兼容格式。只支持某家原生格式（如 Anthropic Messages）的，在 `llm.formats` 写明。
- 应当：不同用途用不同模型的（对话、摘要、嵌入），每一路单独可配。

**变量名不强制。** kiwi-mem 用 `API_KEY` / `API_BASE_URL`，chatnest 用 `ANTHROPIC_BASE_URL`，都合规：外部工具看描述文件里的定位串，不看名字。新项目可以直接用社区最通行的名字：

```
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
```

**描述文件**

```json
"llm": {
  "formats": ["openai-chat"],
  "url_style": "base",
  "routes": {
    "chat":  { "base_url": "env:OPENAI_BASE_URL", "api_key": "env:OPENAI_API_KEY", "model": "env:OPENAI_MODEL" },
    "embed": { "base_url": "env:EMBED_BASE_URL",  "api_key": "env:EMBED_API_KEY",  "model": "env:EMBED_MODEL" }
  }
}
```

`formats` 可选值：`openai-chat`、`openai-responses`、`anthropic-messages`、`gemini`、`ollama`。`url_style`：`base`（默认，填到 `/v1`）或 `full-endpoint`。

这一条做到后，中枢就能用 INTEGRATION 里的策略①「伪装上游」接管模型调用，项目本身不用知道中枢存在。

### 5.2 S2 提示词外置

**要求**
- 必须：人设、系统提示词不能硬编码在源代码里。放进文件、配置项或界面都可以。
- 必须：在 `prompts` 里声明每一处提示词的位置和格式。
- 应当：说明修改后何时生效（`reload`）。
- 应当：支持多角色的，声明角色文件的位置和切换方式。

**描述文件**

```json
"prompts": [
  {
    "id": "persona",
    "role": "persona",
    "at": "file:conf.yaml#/character_config/persona_prompt",
    "format": "text",
    "reload": "restart",
    "variants": "file:characters/*.yaml#/character_config/persona_prompt"
  }
]
```

| 字段 | 说明 |
| --- | --- |
| `id` | 项目内唯一 |
| `role` | `persona`（角色设定）、`system`（系统规则）、`style`（说话风格）、`memory-policy`（记忆规则）、`tool-guide`（工具说明）、`greeting`（开场白）、`other` |
| `at` | 定位串 |
| `format` | 值本身的格式：`text`、`markdown`、`jinja`、`card-v2`、`card-v3`、`messages`（OpenAI 消息数组） |
| `reload` | `hot`（立刻生效）、`next-turn`（下一轮对话生效）、`restart`（重启生效）、`switch`（切换角色时生效） |
| `variants` | 多角色时每个角色的提示词位置 |
| `vars` | 模板里用到的变量，如 `["{{user}}", "{{char}}"]` |
| `managed` | 托管方式，见 5.2.3 |

#### 5.2.1 变量

- 沿用角色卡生态的写法：`{{user}}`、`{{char}}`。
- 其他变量用 `{{snake_case}}`，并在 `vars` 里列出。
- 工具同步提示词时必须原样保留变量，不能替换。

#### 5.2.2 编码与换行

UTF-8，LF 换行。提示词文件末尾的空行可有可无，工具不得因此判定为「内容有改动」。

#### 5.2.3 托管块

外部工具（例如统一提示词管理器）写入提示词时，常常需要只改其中一段，保留作者或用户手写的其余部分。约定用托管块标记：

```markdown
<!-- companion:begin persona -->
这里的内容由提示词管理器维护，手动修改会在下次同步时被检测为冲突。
<!-- companion:end persona -->
```

YAML、TOML、纯文本等以 `#` 注释的格式写成：

```yaml
# companion:begin persona
# companion:end persona
```

规则：
- 工具只能改动标记之间的内容。
- 文件里没有标记时，工具只有在 `managed` 为 `"whole"` 时才能整段覆盖；为 `"block"` 时先插入标记再写；为 `"none"` 或未写时只读。
- 工具写入前必须比对上次同步时的内容哈希。不一致说明有人手动改过，必须提示冲突，不得静默覆盖。

### 5.3 S3 数据可导出

**要求**
- 必须：对话记录、记忆、角色状态中至少有一类能导出为本节定义的格式，或导出为结构清晰、在 `data` 里声明过字段的 JSON / JSONL / SQLite。
- 必须：导出文件里不得包含 API Key 等密钥。
- 应当：同一格式能导回来。
- 应当：给出导出方式的定位串：命令、HTTP 接口或界面路径。

**描述文件**

```json
"data": [
  { "id": "chat",   "type": "message", "at": "file:data/chat.db", "format": "sqlite", "export": "cli:python tools/export.py --out {file}" },
  { "id": "memory", "type": "memory",  "at": "file:memory/*.md",  "format": "markdown" }
]
```

`type` 可选值：`message`、`memory`、`state`、`persona`、`media`、`other`。

#### 5.3.1 交换格式 `companion-export`

JSONL，一行一条记录。第一行是文件头：

```json
{"format":"companion-export","version":"0.1","source":"kiwi-mem","exported_at":"2026-09-29T12:00:00+08:00","persona":"xiaoju"}
```

之后每行一条记录，`type` 决定字段：

```json
{"type":"message","id":"m_01","session":"s_1","ts":"2026-09-28T23:10:00+08:00","role":"user","content":"晚安"}
{"type":"message","id":"m_02","session":"s_1","ts":"2026-09-28T23:10:04+08:00","role":"assistant","content":[{"type":"text","text":"晚安，明天见。"},{"type":"sticker","ref":"media/goodnight.png","alt":"[挥手]"}]}
{"type":"memory","id":"k_17","ts":"2026-09-28T23:15:00+08:00","text":"用户最近在准备考试，晚上会早睡","kind":"fact","importance":0.6,"source_ids":["m_01"],"tags":["作息"]}
{"type":"state","key":"affect.mood","value":{"valence":0.4,"arousal":0.2},"ts":"2026-09-28T23:15:00+08:00"}
{"type":"persona","card":"cards/xiaoju.json","spec":"card-v3"}
```

| 记录 | 必填字段 | 说明 |
| --- | --- | --- |
| `message` | `id`、`ts`、`role`、`content` | `role` 同 OpenAI：`system`、`user`、`assistant`、`tool`。`content` 是字符串或内容块数组 |
| `memory` | `id`、`ts`、`text` | `kind`：`fact`、`event`、`preference`、`relation`、`summary`、`diary`；`source_ids` 指回原始消息 |
| `state` | `key`、`value`、`ts` | 键名用点分命名空间，如 `affect.mood`、`relation.intimacy` |
| `persona` | `card` 或 `inline` | 人设统一用角色卡 v3，不另起格式 |

内容块沿用 OpenAI 的 `text`、`image_url`、`input_audio`，另加以下几种：

| `type` | 字段 | 用途 |
| --- | --- | --- |
| `sticker` | `ref`、`alt` | 表情包 |
| `reasoning` | `text` | 思考过程 |
| `tool_call` | `name`、`arguments`、`call_id` | 工具调用 |
| `tool_result` | `call_id`、`content` | 工具结果 |
| `action` | `text` | 动作描写，如「*摸摸头*」被结构化后 |
| `file` | `ref`、`mime`、`name` | 附件 |

**每个非文本内容块都应当带 `alt`**：读不懂这个类型的程序就显示 `alt`，保证至少能降级成文字。

媒体文件和 JSONL 放在同一目录（或同一个 zip）里，`ref` 用相对路径。

### 5.4 S4 自主行为可关闭

很多项目自带记忆、定时器、主动消息、情绪系统。单独用没问题，但和中枢或其他插件同时开，就会出现两套记忆各记各的、两个定时器轮流发消息。

**要求**
- 必须：在 `autonomy` 里如实声明自带了哪些自主行为。
- 应当：每一项都能单独关闭，并给出关闭方式的定位串。
- 可以：提供「只关闭主动发送、保留内部计算」这类细分开关。

```json
"autonomy": {
  "memory":    { "has": true,  "off": "file:config.yaml#/memory/enabled", "off_value": false },
  "scheduler": { "has": true,  "off": "env:DISABLE_SCHEDULER",           "off_value": "1" },
  "proactive": { "has": true,  "off": null, "$comment": "暂时关不掉，已在 issue 里记录" },
  "affect":    { "has": false }
}
```

可选的键：`memory`（记忆）、`scheduler`（定时任务）、`proactive`（主动发消息）、`affect`（情绪/状态）、`persona-evolution`（人设自我修改）、`tools`（自主调用工具）。

关不掉的项写 `"off": null`。中枢会按 INTEGRATION §5 的冲突表处理，例如改为旁路观察。

### 5.5 S5 出站 webhook

**要求**（只对有「事件」可说的项目适用：收到消息、发出消息、状态变化等）
- 应当：可以配置一个 URL，事件发生时向它 POST 一条事件。
- 必须（如果实现了）：使用 §7.1 的事件信封格式。
- 应当：支持签名，见下。

**请求**

```
POST {webhook_url}
Content-Type: application/json
X-Companion-Event: message.out
X-Companion-Delivery: 01J8Z6...        # 本次投递的唯一 ID，重试时不变
X-Companion-Signature: sha256=<hex>    # HMAC-SHA256(secret, 原始请求体)

{ ...事件信封... }
```

- 接收方返回 2xx 即成功。
- 发送方应当在失败时重试，最多 3 次，间隔 1s、5s、30s，超过就丢弃。**不得因为 webhook 失败而影响主功能。**
- 超时 5 秒。

```json
"events": {
  "webhook": {
    "url": "env:COMPANION_WEBHOOK_URL",
    "secret": "env:COMPANION_WEBHOOK_SECRET",
    "types": ["message.in", "message.out", "proactive.sent"]
  }
}
```

### 5.6 各形态要做哪些接缝

✓ 必须（有相应功能时）　○ 应当　— 不适用

| 形态 | S1 模型 | S2 提示词 | S3 导出 | S4 自主开关 | S5 webhook |
| --- | --- | --- | --- | --- | --- |
| `service` | ✓ | ✓ | ✓ | ✓ | ○ |
| `host-plugin` | 走宿主，— | ✓ | ○ | ✓ | — |
| `game` | — | — | ○ 存档 | — | — |
| `app` | ✓ | ✓ | ✓ | ✓ | ○ |
| `frontend` | ✓ | ✓ | ✓ | ○ | — |
| `web-page` | ✓ | ✓ | ○ | ○ | — |
| `hardware` | — | — | — | ○ | ○ |
| `prompt-pack` | — | ✓ 本身就是 | — | — | — |

---

## 6. 通用接口（C2）

C2 只要求实现下面**任意一种**。`speaks` 里声明：

```json
"speaks": [
  { "protocol": "mcp", "transport": "stdio" },
  { "protocol": "openai-chat", "role": "server", "path": "/v1" },
  { "protocol": "cmd-text" }
]
```

| `protocol` | `role` | 含义 |
| --- | --- | --- |
| `mcp` | server（默认） | MCP 服务器；`transport`：`stdio`、`streamable-http`、`sse` |
| `openai-chat` | server | 自己就是一个 OpenAI 兼容端点（代理网关、记忆网关） |
| `openai-chat` | client | 调用 OpenAI 兼容端点（等同 S1） |
| `cmd-text` | server | 文字游戏协议，见 6.2 |
| `webhook-out` | | 实现了 S5 |
| `card-v3` | | 能导入导出角色卡 v3 |
| `buttplug` | | 设备控制走 Buttplug 协议 |
| `telegram-bot-api` 等 | client | 本身不是标准协议，但能接到某个现成平台 |

### 6.1 MCP

- 工具名应当加项目前缀，用下划线：`ciyuwu_new_game`、`kiwi_search`。多个 MCP 聚合到一起时才不会撞名。
- 工具描述写给模型看，应当说清：什么时候该调用、返回什么、有什么副作用。
- 有副作用的工具（发消息、控制设备、花钱、删数据）应当在描述开头写明，并设置 MCP 的 `annotations`（`destructiveHint` 等）。
- 需要用户身份或会话时，从参数读取，不依赖进程全局状态。这样同一个服务能同时服务多个角色。
- 远程 MCP 应当支持 `Authorization: Bearer`。

### 6.2 文字游戏协议 `cmd-text`

给 AI 玩的游戏，最省事的接口是「一句指令进，一段文字出」。协议只固定三件事：

1. **一个工具**：`cmd(text: string) -> string`。可以另外提供 `new_game(seed?)`、`status()`，但只靠 `cmd` 就必须能玩。
2. **可以串联**：多条指令用 `;` 或换行分隔，一次调用按顺序执行，返回合并结果。词语屋、钓鱼游戏都已经这样做。
3. **状态行**：返回文字的最后一行应当是 `📊 ` 开头、后接一个 JSON 对象的状态行，方便中枢和日志解析。人类看的内容写在前面。

```
你抛下鱼竿……一条银鳞鲫上钩了（普通）。
📊 {"points":120,"location":"reed_river","season":"autumn","turn":37}
```

存档：应当支持通过参数或环境变量指定存档路径，并在 `data` 里声明为 `type: "state"`。

### 6.3 工具描述文件

不想跑 MCP、只想让宿主知道「有哪些工具」时，可以提供一个静态文件（默认 `tool-schema.json`），在 `tools` 字段指向它。

两种现有写法都合规，读取方必须都能识别：

```json
{ "tools": [ { "name": "...", "description": "...", "inputSchema": { ... } } ] }
```

```json
{ "name": "...", "description": "...", "parameters": { ... } }
```

- 前者是 MCP 风格（词语屋）。后者是 OpenAI function 风格（钓鱼游戏），可以是单个对象或数组。
- 读取方统一转换：`parameters` 等同于 `inputSchema`。

---

## 7. 协作（C3）

### 7.1 事件信封

webhook、事件入口、中枢日志共用一种格式：

```json
{
  "id": "01J8Z6Q2V3X4...",
  "type": "message.out",
  "ts": "2026-09-29T08:00:00.123+08:00",
  "source": "kiwi-mem",
  "persona": "xiaoju",
  "session": "s_1",
  "payload": { "role": "assistant", "content": "早安" },
  "meta": { "cip": "0.1" }
}
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id` | 是 | 全局唯一，推荐 ULID（可按时间排序） |
| `type` | 是 | 点分命名 |
| `ts` | 是 | RFC 3339，带时区 |
| `source` | 是 | 发出方的 `id` |
| `persona` | | 角色 ID；单角色项目可以省略 |
| `session` | | 会话 ID |
| `payload` | 是 | 内容，结构由 `type` 决定 |
| `meta` | | 附加信息，如 `trace_id`、`reply_to` |

**保留的事件类型**，含义固定，payload 结构见 `spec/`（后续补充）：

| 命名空间 | 类型 | 含义 |
| --- | --- | --- |
| `message` | `message.in`、`message.out` | 收到用户消息 / 发出回复；payload 同 §5.3.1 的 message |
| `proactive` | `proactive.intent`、`proactive.sent`、`proactive.suppressed` | 想主动说话 / 已发出 / 被守卫拦下 |
| `memory` | `memory.write`、`memory.recall` | 写入记忆 / 召回记忆 |
| `state` | `state.change` | 状态变化，payload：`{key, from, to}` |
| `sense` | `sense.*` | 感知输入：天气、位置、音乐、屏幕、设备 |
| `act` | `act.*` | 对外动作：设备控制、发帖、打电话 |
| `tick` | `tick.*` | 时钟事件 |
| `game` | `game.*` | 游戏内事件 |

自定义类型用 `x.<项目id>.<名字>`，例如 `x.aura.relationship.milestone`。不要占用上面的命名空间。

### 7.2 健康检查

```
GET /healthz
200 {"status":"ok","version":"1.4.2","cip":"0.1"}
```

- `status`：`ok`、`degraded`、`down`。`degraded` 表示能用但有问题，例如模型不可达、正在降级运行。
- 可以附加 `checks`：`{"llm":"ok","db":"ok"}`。
- 不得在返回里暴露密钥和内部路径。
- stdio 类服务不需要这个接口：进程在、能响应 MCP `ping` 就算健康。

### 7.3 事件入口与导入

- 事件入口：`POST` 一个接收事件信封的地址，在 `events.inbound` 声明。项目收到自己认识的类型就处理，不认识的返回 `202` 并忽略。
- 导入：能读 §5.3.1 格式，在 `data[].import` 声明方式。导入必须幂等：按 `id` 去重，同一个文件导两次结果不变。

```json
"events": {
  "webhook": { "url": "env:COMPANION_WEBHOOK_URL" },
  "inbound": "http:POST /companion/events",
  "accepts": ["message.in", "sense.weather", "proactive.intent"]
}
```

---

## 8. 前端档案（F0–F3）

有人把自己在用的前端分享出来，别人拿到之后能不能直接用，关键在这几件事：有没有夹带作者的私人设置、能不能接别人的后端、数据能不能带走、能不能和其他东西联动。

对 `kind` 为 `frontend`、`app`、`web-page` 的项目，额外适用本章。

### 8.1 F0 可分享（分享前必须做到）

这一级是底线。不满足的话，别人拿到手的其实是「作者的私人配置」，而不是一个前端。

| 编号 | 要求 | 常见反例 |
| --- | --- | --- |
| F0.1 | 代码和默认配置里**没有密钥**：API Key、Token、Webhook 地址、私人服务器地址都没有 | 把自己的 Key 写在 `config.js` 里当默认值 |
| F0.2 | **没有作者的私人内容**：人设、昵称、聊天记录、照片、记忆。示例内容必须是虚构的，并标明是示例 | 默认人设就是作者自己的伴侣 |
| F0.3 | **首次打开是可用的空状态**：引导用户填后端、导入或新建角色，而不是报错或白屏 | 没配后端时整个页面报错 |
| F0.4 | **没有未声明的外部请求**：统计、埋点、第三方字体和 CDN 都要在 `ui.privacy` 里列出；统计必须默认关闭 | 偷偷接入统计脚本 |
| F0.5 | **有许可证** | 没有 LICENSE，别人不能合法改 |
| F0.6 | **说明怎么跑起来**：README 里有运行步骤，或者直接双击 `index.html` 就能用 | 需要作者本地的某个服务才能启动 |

### 8.2 F1 能换后端

| 编号 | 要求 |
| --- | --- |
| F1.1 | 必须：能接任意 OpenAI 兼容端点，即满足 S1。Base URL、Key、模型名在界面里可填 |
| F1.2 | 必须：支持流式输出（SSE `text/event-stream`），并在流中断时保留已收到的内容 |
| F1.3 | 必须：如果前端配套了自家后端，要么同时支持直连 OpenAI 兼容端点，要么公开自家后端的 API 文档，写在 `ui.backend.own_api` |
| F1.4 | 应当：模型列表从 `GET {base_url}/models` 读取，不写死模型名 |
| F1.5 | 应当：请求带 `X-Companion-Persona`、`X-Companion-Session` 请求头（值为角色 ID、会话 ID）。普通端点会忽略它们；中枢可以据此路由到正确的记忆和人设 |
| F1.6 | 应当：纯网页前端说明跨域需求。浏览器直连需要后端允许 CORS，否则要说明需要代理 |

### 8.3 F2 数据可带走

| 编号 | 要求 |
| --- | --- |
| F2.1 | 必须：人设能导入导出**角色卡 v3**（JSON；PNG 可选）。能读 v2 更好 |
| F2.2 | 必须：聊天记录能导出为 §5.3.1 `companion-export`，导出文件中不含密钥 |
| F2.3 | 必须：系统提示词和人设在界面里能看到全文、能修改，没有隐藏的内置提示词。确实需要内置规则（如格式说明）的，也要能看到 |
| F2.4 | 应当：导出能再导回来，按 `id` 去重 |
| F2.5 | 应当：在 `ui.storage` 里声明本地存储的位置和键名，见 8.4 |

### 8.4 存储声明

浏览器前端的数据存在 localStorage / IndexedDB 里，外部工具（提示词同步脚本、备份扩展、中枢的注入桥）只能靠猜。声明出来，就能不改前端代码也能同步：

```json
"ui": {
  "storage": {
    "kind": "localStorage",
    "prefix": "chat_",
    "keys": {
      "settings.model": "chat_model",
      "auth.token": { "key": "chat_token", "secret": true },
      "session.current": "chat_conversation"
    },
    "schema_version_key": "chat_schema_version"
  }
}
```

- `keys` 的键是协议里的逻辑名，值是实际键名。这样外部工具不用管每个前端各叫什么。上面的键名取自 chatnest 的 `frontend-demo`。
- 逻辑名优先使用：`settings.base_url`、`settings.api_key`、`settings.model`、`auth.token`（自家后端的登录凭证）、`persona.active`、`persona.list`、`prompt.system`、`history`、`session.current`。
- 值为对象时可以写 `{"key": "...", "path": "/a/b"}`，表示该键里存的是 JSON，要取的是其中某个字段。
- 所有键应当带统一前缀，避免和同域名下的其他页面冲突（单文件网页经常放在同一个 GitHub Pages 域名下）。
- 应当有一个存数据结构版本号的键。前端升级改了结构时，外部工具据此判断能不能写。

### 8.5 F3 可联动

| 编号 | 要求 |
| --- | --- |
| F3.1 | 应当：能**接收主动消息**。方式在 `ui.push` 里声明：`sse`（订阅 `GET /events` 之类的流）、`websocket`、`webpush`、`poll`（定时拉取）、`none` |
| F3.2 | 应当：能渲染 §5.3.1 的内容块。在 `ui.renders` 里声明支持哪些；**遇到不支持的类型必须显示 `alt` 或原始文字，不能吞掉，也不能报错** |
| F3.3 | 可以：支持被嵌入或嵌入别人，走 8.6 的 postMessage 约定 |
| F3.4 | 可以：支持主题变量，见 8.7 |

`ui.renders` 可选值：`text`、`markdown`、`image`、`audio`、`sticker`、`reasoning`、`tool_call`、`action`、`file`、`live2d`、`vrm`、`tts`。

### 8.6 嵌入约定（可选）

用于「前端里开一个面板显示别的插件」或「把前端嵌进中枢控制台」。只用浏览器原生的 `postMessage`，不需要任何库。

所有消息都是 `{"cip": "0.1", "type": "...", ...}`：

| 方向 | `type` | 字段 | 说明 |
| --- | --- | --- | --- |
| 子 → 父 | `companion.ready` | `id`、`capabilities` | 子页面加载完成 |
| 父 → 子 | `companion.context` | `persona`、`session`、`theme`、`locale` | 下发上下文 |
| 父 → 子 | `companion.event` | `event`（事件信封） | 转发事件 |
| 子 → 父 | `companion.send` | `content` | 请父页面代为发送一条消息 |
| 子 → 父 | `companion.resize` | `height` | 请求调整高度 |

安全要求：
- 接收方必须校验 `event.origin`，只处理白名单内的来源。
- 不得通过 postMessage 传递 API Key。

### 8.7 主题变量（可选）

支持被换肤的前端，读取以下 CSS 变量，没有就用自己的默认值：

```css
:root {
  --companion-bg: #fffaf5;
  --companion-fg: #2b2622;
  --companion-muted: #8a817a;
  --companion-accent: #e0826a;
  --companion-bubble-user: #f3e9df;
  --companion-bubble-char: #ffffff;
  --companion-radius: 14px;
  --companion-font: system-ui, sans-serif;
}
```

### 8.8 体验基线（应当）

不计入等级，但作为分享时的推荐项：

- 手机宽度（360px）下可用，没有横向滚动。
- 支持深色模式，或至少不被系统深色模式弄得看不清。
- 输入框在手机软键盘弹出时不被遮挡。
- 基础无障碍：按钮有文字标签，对比度足够。
- 断网时已有记录仍能查看。
- 界面文字集中管理，方便翻译。

### 8.9 `ui` 字段汇总

```json
"ui": {
  "distribution": ["static", "pwa"],
  "entry": "dist/index.html",
  "backend": { "mode": "openai-compatible" },
  "streaming": true,
  "cards": { "import": ["v2", "v3"], "export": ["v3"] },
  "renders": ["text", "markdown", "image", "sticker", "reasoning"],
  "push": "sse",
  "storage": { "kind": "indexedDB", "prefix": "nest_", "keys": { "history": "nest_db/messages" } },
  "embed": true,
  "theme_vars": true,
  "privacy": { "telemetry": false, "third_party": ["fonts.googleapis.com"] }
}
```

| 字段 | 取值 |
| --- | --- |
| `distribution` | `static`（构建产物，任意静态托管）、`single-file`（一个 HTML）、`pwa`、`userscript`（油猴脚本换皮）、`extension`（浏览器扩展）、`android`、`ios`、`desktop` |
| `backend.mode` | `openai-compatible`、`own`（只认自家后端）、`both` |
| `backend.own_api` | 自家后端 API 文档的路径或 URL |
| `privacy.third_party` | 会访问的第三方域名 |

### 8.10 分享前自查表

投稿一个前端时，可以直接把这张表贴进 PR：

```markdown
- [ ] F0.1 代码和默认配置里没有任何密钥、私人地址
- [ ] F0.2 没有作者的私人人设 / 记录 / 照片，示例内容已标明
- [ ] F0.3 首次打开有引导，不报错
- [ ] F0.4 外部请求已在 ui.privacy 列出，统计默认关闭
- [ ] F0.5 有 LICENSE
- [ ] F0.6 README 有运行步骤
- [ ] F1 能填任意 OpenAI 兼容地址并流式输出
- [ ] F2 能导入导出角色卡 v3 和聊天记录
- [ ] 已提供 companion.json（或请社区代写）
```

---

## 9. 预留接口与演进

协议要能长期用下去，靠的是下面几条规则，而不是一次想全：

1. **只做加法。** 新版本只增加字段和取值，不删除、不改含义。0.x 期间如需破坏性修改，至少提前一个版本标记为「废弃」。
2. **按能力判断，不按版本号判断。** 读取方检查「有没有 `ui.storage`」，而不是「`cip` 是不是 ≥ 0.2」。
3. **不认识的字段必须忽略**（§4.2）。
4. **扩展走 `x-` 前缀和 `x.<id>.*` 事件命名空间**。某个扩展被三个以上项目采用后，可以提议转正。
5. **保留的名字**：`X-Companion-*` 请求头、`companion.*` postMessage 类型、`companion:begin/end` 托管块、§7.1 列出的事件命名空间、`/healthz`。项目自己的东西不要用这些名字。
6. **提案流程**：在本仓库开 issue，标题以 `[CIP]` 开头，附至少一个已经在用这种做法的真实项目。没有实例的提议不进入协议。

---

## 10. 减负措施

个人开发者的时间最宝贵。协议能不能被接受，取决于这几件事：

| 措施 | 做什么 | 状态 |
| --- | --- | --- |
| 社区代写 | `registry/` 里由社区给项目写 `companion.json`，作者什么都不用做 | 计划中；本次统计的 195 个仓库数据可直接生成草稿 |
| 生成器 `cip init` | 读仓库目录，自动推断 `kind`、`run`、提示词文件、MCP 工具、配置样例，生成草稿让作者确认 | 计划中 |
| 检查器 `cip check` | 校验描述文件，计算实际等级，列出差哪几项 | 计划中；目前可用 JSON Schema 校验 |
| 项目模板 | 按形态提供模板仓库：MCP 服务、AstrBot 插件、文字游戏、单文件前端，模板本身就满足 C1 / F1 | 计划中 |
| 单文件辅助代码 | 每个接缝一份可以直接复制的片段（Python / TypeScript），无依赖，几十行 | 计划中 |
| 徽章 | 清单里显示等级徽章，作为加分展示，不影响收录 | 计划中 |

---

## 11. 协议不管什么

- 界面长什么样、交互怎么设计
- 用什么语言、框架、数据库
- 记忆算法、情绪模型、人格设计
- 是否开源、是否收费（只要求如实声明许可证和权限）
- 项目内部的模块划分

协议只管五个边界：**怎么描述自己、怎么被配置、说什么话、怎么交换数据、前端怎么分享**。

---

## 附录 A：真实项目示例

以下按目前仓库结构推断，**未经作者确认**，完整文件在 [`spec/examples/`](spec/examples/)。

| 文件 | 项目 | 展示的要点 |
| --- | --- | --- |
| `ci-yu-wu.json` | 词语屋 | `game` + `cmd-text` + 现成的 `tool-schema.json` |
| `open-llm-vtuber.json` | Open-LLM-VTuber | `app`，提示词在 YAML 深层字段，多角色 `variants` |
| `kiwi-mem.json` | kiwi-mem | `service`，提示词是根目录 `system_prompt.txt` |
| `astrbot-plugin-proactive-chat.json` | AstrBot 主动消息插件 | `host-plugin`，复用 `_conf_schema.json` 和 `metadata.yaml`，提示词在宿主的插件配置里 |
| `chatnest.json` | chatnest | `frontend`，自家后端 + Anthropic 原生格式（`backend.mode: own`），存储键声明 |

## 附录 B：等级速查

| 我是…… | 最少做什么能到 C1 / F1 |
| --- | --- |
| MCP 服务作者 | 写 `companion.json`；用到模型的话，Base URL / Key / 模型名走环境变量 |
| AstrBot 插件作者 | 写 `companion.json`，`config.schema` 指向已有的 `_conf_schema.json`；在里面加上关闭主动消息 / 记忆的开关 |
| 文字游戏作者 | 写 `companion.json`；确认 `cmd` 能串联，结尾加一行 `📊` 状态行 |
| 前端作者 | 过一遍 §8.10 自查表；能填 Base URL；能导入导出角色卡 |
| 单文件网页作者 | 去掉写死的 Key；本地存储键加前缀并在 `ui.storage` 声明 |
