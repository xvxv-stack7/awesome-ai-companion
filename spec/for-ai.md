# CIP 接入指南（写给 AI 编程助手）

> 这份文件是给 AI 看的操作手册。开发者会对你说类似「读这个文件，按步骤检查并改造我的项目」的话。
> 完整规范在 [PROTOCOL.zh-CN.md](../PROTOCOL.zh-CN.md)，校验用的 Schema 在 [companion.schema.json](companion.schema.json)。本文件已包含完成接入所需的全部信息，一般不需要再去读它们。

## 你的任务

让开发者的项目能被 awesome-ai-companion 生态里的其他工具（统一的提示词管理、记忆服务、主动消息调度等）直接接上。做法是：

1. 在仓库根目录写一个描述文件 `companion.json`，如实说明项目是什么、怎么启动、各项设置在哪里。**这一步不改任何代码。**
2. 检查五项常见的接入条件（A–E），告诉开发者哪些已经满足、哪些没满足，**经开发者同意后**再用最小改动补上。
3. 如果项目是给别人用的前端，额外做一次分享前检查（F）。
4. 校验，然后按固定格式汇报。

## 必须遵守的规则

1. **先描述，后修改。** 项目已有的写法（环境变量名、配置格式、目录结构）只要能用，就在 `companion.json` 里如实描述，不要为了“规范”去改名或改结构。
2. **最小改动。** 不重构，不改变默认行为，不改默认值，不升级依赖，不动与本任务无关的代码。
3. **不编造。** 只写你在代码里确认过的路径、变量名、命令。确认不了的字段就不写，并在汇报里列出来问开发者。
4. **不写密钥。** 任何 API Key、Token、私人地址都不能出现在 `companion.json` 或示例文件里。环境变量只写名字。
5. **分步确认。** 第 1 步（写描述文件）可以直接做。第 2、3 步每一处代码改动，先列出计划让开发者确认；开发者说不用改，就只在描述文件里如实写“没有”。
6. **不改已有的工具名、接口路径、配置键名。** 别人可能已经在用，改了会坏。新加的才按本文建议命名。

---

## 第 0 步：弄清你在替谁写

- 你在改的是**开发者自己的项目** → 描述文件写到项目根目录 `companion.json`，改动提交到项目里。
- 你在替**别人的项目**写（比如给 awesome-ai-companion 清单做贡献，不能改原项目）→ 只写描述文件，放到清单仓库的 `registry/<owner>/<repo>.json`，并加 `"maintainer": "community"`。跳过所有代码改动。

## 第 1 步：读项目，判断形态

读 README、依赖文件（`package.json`、`pyproject.toml`、`requirements.txt`、`build.gradle` 等）、入口文件和配置样例（`.env.example`、`config*.yaml`、`_conf_schema.json`、`metadata.yaml`）。

按下表选 `kind`，从上往下匹配第一个符合的。项目同时有多种形态时，主形态写 `kind`，其他写进 `also` 数组。

| 项目是 | `kind` |
| --- | --- |
| 某个宿主程序的插件（AstrBot、NoneBot、SillyTavern、Koishi、Operit 等）。特征：有 `metadata.yaml`、`_conf_schema.json`、宿主 SDK 的 import | `host-plugin` |
| 靠工具调用来玩的文字游戏 | `game` |
| 控制实体设备（蓝牙、串口、智能硬件） | `hardware` |
| 只有提示词、人设、角色卡，没有程序 | `prompt-pack` |
| 教程、文档、资料汇总 | `doc` |
| 单个 HTML 文件，打开就能用 | `web-page` |
| 只有聊天界面，连接别人的后端 | `frontend` |
| 有界面也有自己的后端或完整功能（桌宠、VTuber、手机 App、桌面应用） | `app` |
| 没有界面的后台程序：MCP 服务器、API 网关、记忆服务、机器人 | `service` |

## 第 2 步：写 `companion.json`（必做，不改代码）

### 2.1 必填字段

```json
{
  "cip": "0.1",
  "id": "仓库名，小写字母、数字、短横线",
  "name": "显示名，可以是中文",
  "kind": "第 1 步选出的形态",
  "repo": "https://github.com/owner/repo"
}
```

### 2.2 能确认就写的字段

| 字段 | 写什么 | 从哪里找 |
| --- | --- | --- |
| `summary` | 一句话介绍 | README 第一段 |
| `license` | SPDX 标识，如 `MIT`、`AGPL-3.0-only`；没有许可证写 `"NONE"` | `LICENSE` 文件 |
| `also` | 次要形态 | 第 1 步 |
| `run` | 怎么启动，见 2.3 | README、Dockerfile、启动脚本 |
| `host` | 宿主插件必填：`{"name": "astrbot", "min_version": "4.24.0", "manifest": "metadata.yaml"}` | `metadata.yaml`、依赖声明 |
| `speaks` | 已支持的通用接口，见 2.4 | 代码 |
| `tools` | 工具描述文件路径，如 `"tool-schema.json"` | 仓库里现成的工具描述 |
| `config` | `{"schema": "_conf_schema.json", "example": ".env.example"}`，指向已有的配置说明文件 | 仓库 |
| `permissions` | 用到的敏感权限，从 `network` `filesystem` `microphone` `camera` `location` `device.ble` `device.intimate` `payment` `shell` 中选 | 代码 |
| `llm` `prompts` `data` `autonomy` `events` | 第 3 步的检查结果 | 第 3 步 |
| `ui` | 有界面的项目填，见第 4 步 | 第 4 步 |

- 宿主插件：`metadata.yaml` 里已有的名称、版本、作者不要重复写。
- 自定义字段以 `x-` 开头。
- 标准 JSON，不能有注释。需要备注时写 `"$comment": "..."`。

### 2.3 `run`

```jsonc
// 命令行启动、通过标准输入输出通信（多数本地 MCP 服务器）
"run": { "type": "stdio", "command": ["python", "server.py"], "env": { "OPENAI_API_KEY": { "required": true, "secret": true } } }

// 启动后监听端口
"run": { "type": "http", "command": ["uv", "run", "run_server.py"], "port": 12393 }

// 静态网页
"run": { "type": "static", "entry": "index.html" }

// 装进宿主运行（插件）
"run": { "type": "host" }

// 需要安装的应用
"run": { "type": "app", "platforms": ["android", "windows"] }

// 线上服务，不需要本地运行
"run": { "type": "cloud", "url": "https://example.com" }
```

`type` 只能是 `stdio` `http` `static` `host` `app` `cloud` `none`。`env` 里敏感的变量标 `"secret": true`。

### 2.4 `speaks`

在代码里找到证据才写：

| 证据 | 写法 |
| --- | --- |
| import 了 MCP SDK（`mcp`、`FastMCP`、`@modelcontextprotocol/sdk`） | `{"protocol": "mcp", "transport": "stdio"}`，HTTP 方式写 `"streamable-http"` 或 `"sse"`，并加 `"path"` |
| 自己提供 `/v1/chat/completions` 接口 | `{"protocol": "openai-chat", "role": "server", "path": "/v1"}` |
| 游戏只靠一个“输入一句指令、返回一段文字”的工具就能玩 | `{"protocol": "cmd-text"}` |
| 能导入导出角色卡 v3 | `{"protocol": "card-v3"}` |
| 设备控制走 Buttplug | `{"protocol": "buttplug"}` |

### 2.5 定位串：告诉别人“这个设置在哪”

描述文件里凡是指向某个设置的地方，都用一个带前缀的字符串：

| 前缀 | 格式 | 例子 |
| --- | --- | --- |
| `env:` | 环境变量名 | `env:OPENAI_BASE_URL` |
| `file:` | 文件路径，`#` 后接 JSON Pointer（YAML、TOML、JSON 通用）；整个文件就不加 `#` | `file:conf.yaml#/character_config/persona_prompt` |
| `block:` | 文件里的托管块，见 3B | `block:CLAUDE.md#persona` |
| `http:` | 方法和路径 | `http:PUT /api/persona/{id}` |
| `cli:` | 命令，`{file}` 表示输出文件 | `cli:python export.py --out {file}` |
| `storage:` | 浏览器存储 | `storage:localStorage:chat_model` |
| `host:` | 宿主里的配置位置 | `host:astrbot:plugin_config/<插件名>#/friend_settings/proactive_prompt` |
| `ui:` | 只能手动在界面里改 | `ui:设置 → 模型 → 自定义地址` |

JSON Pointer 用 `/` 分隔层级，从文件的最外层开始，按实际键名写。写之前打开配置文件核对一遍。

---

## 第 3 步：检查 A–E 五项

先按形态查下表，知道哪些是**必须**的（缺了就不算完成接入），哪些是**建议**的。标“—”的项在描述文件里写 `null`，不用检查。

| 形态 | A 模型 | B 提示词 | C 导出 | D 自主行为 | E 通知 |
| --- | --- | --- | --- | --- | --- |
| `service` | 必须 | 必须 | 必须 | 必须 | 建议 |
| `host-plugin` | —（模型由宿主提供，写 `"llm": {"via_host": true}`） | 必须 | 建议 | 必须 | — |
| `game` | — | — | 建议（存档） | — | — |
| `app` | 必须 | 必须 | 必须 | 必须 | 建议 |
| `frontend` | 必须 | 必须 | 必须 | 建议 | — |
| `web-page` | 必须 | 必须 | 建议 | 建议 | — |
| `hardware` | — | — | — | 建议 | 建议 |
| `prompt-pack` | — | 必须（本身就是） | — | — | — |
| `doc` | — | — | — | — | — |

“必须”也只在项目有这个功能时才适用：项目根本不调用模型，A 就写 `"llm": null`；没有提示词，B 就写 `"prompts": null`。**`null` 表示“不适用”，不写表示“没检查”，两者要区分。**

每项的检查都按：**怎么查 → 算不算满足 → 没满足怎么补（需开发者同意） → 描述文件怎么写**。

### A. 模型地址、密钥、模型名能不改代码就换

**怎么查**：搜索 `api.openai.com`、`api.anthropic.com`、`base_url`、`baseURL`、`OPENAI_`、`ANTHROPIC_`、`api_key`、`model=`、`"gpt-`、`"claude-`、`"deepseek`。

**算满足**：地址、密钥、模型名三样都能通过环境变量、配置文件或界面修改。变量叫什么名字都可以。

**没满足怎么补**：把写死的值改成从环境变量读取，**原来的值作为默认值**，行为不变。

```python
import os
BASE_URL = os.getenv("OPENAI_BASE_URL", "原来写死的地址")
API_KEY  = os.getenv("OPENAI_API_KEY", "")
MODEL    = os.getenv("OPENAI_MODEL", "原来写死的模型名")
```

```js
const BASE_URL = process.env.OPENAI_BASE_URL ?? "原来写死的地址";
```

浏览器前端没有环境变量：在设置界面加输入框，存进 localStorage，并在第 4 步的 `ui.storage` 里声明。

同时在 `.env.example` 里补上新变量（值留空或写示例），README 里加一行说明。

**描述文件**：

```json
"llm": {
  "formats": ["openai-chat"],
  "url_style": "base",
  "routes": {
    "chat": { "base_url": "env:OPENAI_BASE_URL", "api_key": "env:OPENAI_API_KEY", "model": "env:OPENAI_MODEL" }
  }
}
```

- `formats`：项目支持的请求格式，从 `openai-chat` `openai-responses` `anthropic-messages` `gemini` `ollama` 中选。
- `url_style`：地址填到 `/v1` 为止写 `"base"`；要求填完整地址（到 `/chat/completions`）写 `"full-endpoint"`。**按项目现状写，不要改项目。**
- `routes`：不同用途用不同模型时分开写，键名用 `chat`、`summary`、`embedding`、`vision`、`tts`、`stt` 等。
- 某一样改不了，对应值写 `null`。

### B. 人设和系统提示词放在代码外面

**怎么查**：搜索源代码里的长字符串，关键词 `你是`、`You are`、`system`、`persona`、`prompt`、`角色`、`人设`。

**算满足**：提示词在独立文件、配置文件、数据库或界面里，改它不用改代码。

**没满足怎么补**：把字符串原样移到 `prompts/system.md`（或项目习惯的位置），启动时读取；文件不存在时退回原来的字符串，行为不变。

```python
from pathlib import Path
_p = Path(__file__).parent / "prompts" / "system.md"
SYSTEM_PROMPT = _p.read_text(encoding="utf-8") if _p.exists() else """原来的提示词"""
```

提示词里如果有用户名、角色名之类的占位，沿用角色卡的写法 `{{user}}`、`{{char}}`，已有的其他写法不要改，写进 `vars` 即可。

**描述文件**：

```json
"prompts": [
  { "id": "persona", "role": "persona", "at": "file:prompts/system.md", "format": "markdown", "reload": "restart", "managed": "whole" }
]
```

| 字段 | 取值 |
| --- | --- |
| `role` | `persona` 人设 · `system` 系统规则 · `style` 说话风格 · `memory-policy` 记忆规则 · `tool-guide` 工具说明 · `greeting` 开场白 · `other` |
| `format` | `text` `markdown` `jinja` `card-v2` `card-v3` `messages` |
| `reload` | 改了之后什么时候生效：`hot` 立刻 · `next-turn` 下一轮对话 · `restart` 重启后 · `switch` 切换角色时。看代码在哪里读取来判断 |
| `managed` | 外部工具能怎么改：`whole` 整个文件都可以覆盖（文件里只有这段提示词时）· `block` 只改托管块 · `none` 只读 |
| `variants` | 有多个角色时，其他角色所在位置，可用通配符：`file:characters/*.yaml#/persona_prompt` |
| `vars` | 用到的占位变量，如 `["{{user}}", "{{char}}"]` |

**托管块**：提示词和别的内容混在一个文件里时，用标记圈出可以被外部工具改写的部分，写 `"managed": "block"`，`at` 用 `block:文件#块名`。

```markdown
<!-- companion:begin persona -->
这里的内容由外部的提示词管理工具维护
<!-- companion:end persona -->
```

YAML、TOML、纯文本文件用 `# companion:begin persona` / `# companion:end persona`。

### C. 数据能导出

**怎么查**：找数据存在哪里：SQLite 文件、JSON/JSONL、Markdown、数据库、localStorage、IndexedDB。再找有没有导出功能（按钮、命令、接口）。

**算满足**：对话、记忆、状态至少一类满足以下任一条：能导出为下面的 `companion-export` 格式；或者存储格式清楚，在描述文件里声明了位置和格式。导出结果不能含密钥。

**已有的存储如实声明就算满足**，不需要额外开发：

```json
"data": [
  { "id": "chat",   "type": "message", "at": "file:data/chat.db", "format": "sqlite" },
  { "id": "memory", "type": "memory",  "at": "file:memory/*.md",  "format": "markdown" }
]
```

`type`：`message` `memory` `state` `persona` `media` `other`。`format`：`companion-export` `sqlite` `postgres` `json` `jsonl` `markdown` `yaml` `other`。有导出或导入命令时加 `"export": "cli:..."`、`"import": "cli:..."`；没有就不写。

**开发者想加导出功能时**，输出 `companion-export`：JSONL，一行一条，第一行是文件头。

```json
{"format":"companion-export","version":"0.1","source":"项目id","exported_at":"2026-09-29T12:00:00+08:00"}
{"type":"message","id":"m_01","session":"s_1","ts":"2026-09-28T23:10:00+08:00","role":"user","content":"晚安"}
{"type":"message","id":"m_02","session":"s_1","ts":"2026-09-28T23:10:04+08:00","role":"assistant","content":[{"type":"text","text":"晚安，明天见。"},{"type":"sticker","ref":"media/goodnight.png","alt":"[挥手]"}]}
{"type":"memory","id":"k_17","ts":"2026-09-28T23:15:00+08:00","text":"用户最近在准备考试","kind":"fact","source_ids":["m_01"]}
{"type":"state","key":"affect.mood","value":{"valence":0.4},"ts":"2026-09-28T23:15:00+08:00"}
{"type":"persona","card":"cards/xiaoju.json","spec":"card-v3"}
```

- `message` 必填 `id` `ts` `role` `content`。`role` 为 `system` `user` `assistant` `tool`。`content` 是字符串，或 OpenAI 格式的内容块数组。
- 额外的内容块类型：`sticker`（`ref` `alt`）、`reasoning`（`text`）、`tool_call`（`name` `arguments` `call_id`）、`tool_result`（`call_id` `content`）、`action`（`text`）、`file`（`ref` `mime` `name`）。**每个非文字块都带 `alt`**，给不认识这个类型的程序显示。
- `memory` 必填 `id` `ts` `text`；`kind` 为 `fact` `event` `preference` `relation` `summary` `diary`。
- `state` 必填 `key` `value` `ts`，键名用点分隔，如 `affect.mood`。
- 人设统一用角色卡 v3。
- 媒体文件和 JSONL 放在同一目录，`ref` 用相对路径。
- 时间用带时区的 ISO 8601。

### D. 自带的自主行为能单独关掉

项目自带的记忆、定时任务、主动发消息，和别的插件同时开会冲突：两套记忆各记各的，两个定时器轮流发消息。所以要能关。

**怎么查**：
- 记忆：向量库、记忆文件、摘要逻辑。
- 定时任务：`apscheduler`、`schedule`、`cron`、`setInterval`、`asyncio.sleep` 循环。
- 主动发消息：没有用户输入时也会发消息的代码。
- 情绪/状态：好感度、心情数值。
- 人设自我修改：程序会改写自己的提示词。
- 自主调用工具：不经用户同意自动调用工具。

**算满足**：如实声明了有哪些（这一条是必须的）；每一项能单独关掉（建议）。

**没满足怎么补**：给每项加一个开关，默认开启，行为不变。

```python
MEMORY_ENABLED = os.getenv("MEMORY_ENABLED", "true").lower() != "false"
```

**描述文件**：只写项目有的和明确没有的。

```json
"autonomy": {
  "memory":    { "has": true, "off": "env:MEMORY_ENABLED", "off_value": "false" },
  "proactive": { "has": true, "off": null },
  "affect":    { "has": false }
}
```

键名：`memory` `scheduler` `proactive` `affect` `persona-evolution` `tools`。`off` 指向开关，`off_value` 是关闭时要设的值；关不掉写 `"off": null`。

### E. 发生事件时通知外部（可选）

只在项目有事件可说（收到消息、发出消息、主动消息已发送、状态变化）**并且开发者想要**时才做。

做法：读取一个可配置的 URL，事件发生时 POST 下面的 JSON。

```http
POST {COMPANION_WEBHOOK_URL}
Content-Type: application/json
X-Companion-Event: message.out
X-Companion-Delivery: <本次投递的唯一 ID，重试时不变>
X-Companion-Signature: sha256=<HMAC-SHA256(COMPANION_WEBHOOK_SECRET, 原始请求体) 的十六进制>
```

```json
{
  "id": "01J8Z6Q2V3X4...",
  "type": "message.out",
  "ts": "2026-09-29T08:00:00+08:00",
  "source": "项目id",
  "persona": "角色id",
  "session": "会话id",
  "payload": { "role": "assistant", "content": "早安" },
  "meta": { "cip": "0.1" }
}
```

- `id` 用 ULID 或 UUID。
- 常用 `type`：`message.in` `message.out` `proactive.sent` `proactive.suppressed` `memory.write` `state.change`。自定义的写成 `x.<项目id>.<名字>`。
- 超时 5 秒；失败重试最多 3 次，间隔 1、5、30 秒，然后放弃。**发送放到后台，失败不能影响主功能。**
- URL 没配置时什么都不做。

**描述文件**：

```json
"events": {
  "webhook": { "url": "env:COMPANION_WEBHOOK_URL", "secret": "env:COMPANION_WEBHOOK_SECRET", "types": ["message.in", "message.out"] }
}
```

---

## 第 4 步：有界面的项目（`frontend` `app` `web-page`）

### 4.1 分享前检查（全部满足才算可分享）

逐条检查，发现问题**先告诉开发者**，由开发者决定怎么处理。涉及删除内容时不要自己删。

| # | 检查 | 怎么查 |
| --- | --- | --- |
| 1 | 没有密钥 | 搜索 `sk-`、`sk-ant-`、`Bearer `、`api_key`、`token`、`webhook`、私人 IP 和域名；也查 git 历史里是否提交过（`git log -p -S "sk-"`），提交过就提醒开发者作废那个密钥 |
| 2 | 没有作者的私人内容 | 默认人设、昵称、示例对话、图片是否像真人真事。示例内容要虚构并标明是示例 |
| 3 | 第一次打开能用 | 没配后端、没有角色时是引导界面，不报错、不白屏 |
| 4 | 外部请求都列出来了 | 统计脚本、第三方字体、CDN。统计必须默认关闭 |
| 5 | 有许可证 | `LICENSE` 文件 |
| 6 | 写了怎么运行 | README 有步骤，或双击 `index.html` 即可 |

### 4.2 能力检查

| 等级 | 条件 |
| --- | --- |
| 能换后端 | 能在界面填任意 OpenAI 兼容地址、密钥、模型名；支持流式输出，中断时保留已收到的内容；只能连自家后端的，公开了后端接口文档 |
| 数据能带走 | 角色卡 v3 能导入导出；聊天记录能导出为 `companion-export`（不含密钥）；系统提示词在界面里能看到全文、能修改 |
| 能联动 | 能接收主动消息（或明确声明不能）；遇到不认识的内容块显示它的 `alt`，不报错也不吞掉 |

建议但不强制：模型列表从 `GET {base_url}/models` 读取；请求带 `X-Companion-Persona`（角色 id）和 `X-Companion-Session`（会话 id）两个请求头；手机 360px 宽度可用；支持深色模式。

### 4.3 描述文件的 `ui`

```json
"ui": {
  "distribution": ["single-file"],
  "entry": "index.html",
  "backend": { "mode": "openai-compatible" },
  "streaming": true,
  "cards": { "import": ["v2", "v3"], "export": ["v3"] },
  "renders": ["text", "markdown", "image"],
  "push": "none",
  "storage": {
    "kind": "localStorage",
    "prefix": "myapp_",
    "keys": {
      "settings.base_url": "myapp_base_url",
      "settings.api_key": { "key": "myapp_key", "secret": true },
      "settings.model": "myapp_model",
      "prompt.system": { "key": "myapp_settings", "path": "/systemPrompt" }
    },
    "schema_version_key": "myapp_version"
  },
  "privacy": { "telemetry": false, "third_party": ["fonts.googleapis.com"] }
}
```

- `distribution`：`static` `single-file` `pwa` `userscript` `extension` `android` `ios` `desktop`。
- `backend.mode`：`openai-compatible` 能直连任意兼容接口 · `own` 只连自家后端（加 `"own_api": "接口文档路径"`）· `both`。
- `push`：`sse` `websocket` `webpush` `poll` `none`。
- `storage.keys` 的**键是固定的逻辑名，值是项目里实际的键名**。逻辑名从这些里选：`settings.base_url` `settings.api_key` `settings.model` `auth.token`（自家后端的登录凭证） `persona.active` `persona.list` `prompt.system` `history` `session.current`。存的是 JSON 对象时，用 `{"key": ..., "path": "/字段"}` 指到里面。**照抄代码里的实际键名，不要改项目的键名。**
- 敏感的键标 `"secret": true`。

---

## 第 5 步：校验

```bash
pip install jsonschema
curl -sSLo /tmp/companion.schema.json https://raw.githubusercontent.com/DasterProkio/awesome-ai-companion/main/spec/companion.schema.json
python -c "import json,jsonschema; jsonschema.validate(json.load(open('companion.json')), json.load(open('/tmp/companion.schema.json'))); print('OK')"
```

不能联网时，至少确认：JSON 能解析；五个必填字段都在；`kind`、`run.type` 等取值在本文列出的范围内；所有定位串都以 `env:` `file:` `block:` `http:` `cli:` `storage:` `ui:` `host:` 之一开头。

再逐个打开描述文件里 `file:` 指向的文件，确认路径和 JSON Pointer 真实存在。

## 第 6 步：计算等级并汇报

**等级**（取满足的最高一级）：

- **C0**：有合法的 `companion.json`。
- **C1**：第 3 步表里该形态“必须”的项全部满足。
- **C2**：C1，且 `speaks` 里有 `mcp`、`openai-chat`（server）或 `cmd-text` 之一。
- **C3**：C2，且实现了 E（通知）、提供 `GET /healthz`（返回 `{"status": "ok"}`）、能导入 `companion-export` 且重复导入不产生重复数据。

有界面的项目另算前端等级：**F0** 分享前检查全过；**F1** 加上“能换后端”；**F2** 再加上“数据能带走”；**F3** 再加上“能联动”。

在描述文件里写 `"level": "C1"` 或 `"level": "C1+F2"`。

**汇报格式**：用开发者的语言，不用术语，按下面的结构：

```
已完成：
- 写了 companion.json（形态：xxx，等级：C1）
- ……（每处代码改动一行，说明改了什么、为什么行为不变）

已经满足、无需改动：
- 模型地址可以通过 OPENAI_BASE_URL 修改
- ……

没做，需要你决定：
- ……（每项说明：不做会怎样，做的话要改哪里、大概多少行）

我不确定、没写进去的：
- ……（请你补充）

分享前请注意：（仅前端）
- ……
```

---

## 附：完整例子

一个本地 MCP 记忆服务，改造后的 `companion.json`：

```json
{
  "cip": "0.1",
  "id": "my-memory-mcp",
  "name": "我的记忆服务",
  "kind": "service",
  "repo": "https://github.com/you/my-memory-mcp",
  "summary": "给 AI 伴侣用的长期记忆 MCP 服务",
  "license": "MIT",
  "level": "C2",
  "run": {
    "type": "stdio",
    "command": ["python", "server.py"],
    "env": {
      "OPENAI_BASE_URL": { "required": false },
      "OPENAI_API_KEY": { "required": true, "secret": true }
    }
  },
  "speaks": [{ "protocol": "mcp", "transport": "stdio" }],
  "llm": {
    "formats": ["openai-chat"],
    "url_style": "base",
    "routes": {
      "summary": { "base_url": "env:OPENAI_BASE_URL", "api_key": "env:OPENAI_API_KEY", "model": "env:SUMMARY_MODEL" },
      "embedding": { "base_url": "env:OPENAI_BASE_URL", "api_key": "env:OPENAI_API_KEY", "model": "env:EMBEDDING_MODEL" }
    }
  },
  "prompts": [
    { "id": "summary", "role": "memory-policy", "at": "file:prompts/summary.md", "format": "markdown", "reload": "hot", "managed": "whole" }
  ],
  "data": [
    { "id": "memory", "type": "memory", "at": "file:data/memory.db", "format": "sqlite", "export": "cli:python export.py --out {file}" }
  ],
  "autonomy": {
    "memory": { "has": true, "off": "env:MEMORY_ENABLED", "off_value": "false" },
    "scheduler": { "has": false },
    "proactive": { "has": false }
  },
  "events": null,
  "config": { "example": ".env.example" },
  "permissions": ["network", "filesystem"]
}
```

更多按真实项目写的例子：[examples/](examples/)。
