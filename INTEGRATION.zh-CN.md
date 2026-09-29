# 缝合方案：把整份清单接成一套可插拔的伴侣系统

> 设计草案 · 讨论用，未实现
> 项目之间怎么接、前端怎么分享的具体约定，见 [伴侣互通协议 CIP](PROTOCOL.zh-CN.md)。

**一句话:** 不改任何上游项目，也不把它们合并成一个大仓库。做一个很薄的**中枢（Hub）**，再定几份**公共协议**，让清单里的约 220 个项目各自以「插件」身份接进来（本来就是服务的包一层；本来不是插件形态的，按侵入度从低到高逐级改造，见第五节）：想换哪块就拔哪块，拔掉任何一块，其余照常运转。

---

## 零、模块化的手感：用起来应该像什么

目标不只是「技术上能拆」，而是**用的人摸得到模块的边界**，像拼乐高、像换手机壳，而不是像拆发动机。后面所有设计都服务于下面这八条手感：

1. **一块就是一张卡。** 每个模块在面板上是一张卡片：名字、占哪个槽、状态灯（运行/停止/出错）、一个开关、一个「换掉」按钮。用户不需要知道它背后是 Python 还是 Docker。
2. **插上就亮，拔掉就灭，不牵连别人。** 装一个模块 = 一条命令或点一下（`hub add ombre-brain`）；卸掉 = `hub remove ombre-brain`。卸掉记忆模块，伴侣照样能聊，只是暂时「健忘」；卸掉语音，就回到文字。**永远是降级，不是崩溃。**
3. **槽位看得见。** 像主板插槽一样，面板直接画出「记忆 ▢ · 情绪 ▢ · 声音 ▢ · 身体 ▢ · 通道 ▢▢▢ · 能力 ▢▢▢▢…」，空着的槽显示「可装：Ombre-Brain / Aelios / kiwi-mem…」。单选槽只能插一块，多选槽可以插一排。
4. **同槽即可互换。** 同一个槽里的模块说同一种接口，换记忆系统就像换一块电池：旧的导出、新的从原始日志回放，人格和聊天记录一个字都不丢。
5. **模块只认「能力」，不认「名字」。** 模块声明「我需要 `memory.recall`」，而不是「我需要 Ombre-Brain」。所以任何模块都不会被另一个具体项目绑死。
6. **模块之间不直接说话。** 一律通过中枢的事件和接口，没有私下的点对点调用。这样任何两块之间都没有隐藏的线，拔哪块都干净。
7. **换之前能试、换之后能退。** 每次换模块自动留一个快照；新模块可以先「影子运行」（只收事件不出声）试几天，满意再转正，不满意一键回滚。
8. **一个模块，一个文件说清楚。** 模块的一切（槽位、协议、权限、许可、配置项）都写在一份 `companion-plugin.yaml` 里，面板的设置页也从它自动生成。看这一份文件就知道这块是什么、要什么、能碰什么。

一句话概括这种手感：**中枢是底板，协议是凸点，每个项目是一块积木。**

---

## 一、先看清现状：这些项目其实已经在说几种「共同语言」

翻完整份清单会发现，项目虽然多，接口却高度收敛到少数几种事实标准上：

| 事实标准 | 谁在用（举例） | 在缝合中的角色 |
| --- | --- | --- |
| **OpenAI 兼容 HTTP API** | OmniRouter、omemo、ai-memory-gateway、Paramecium、rolling-memory、几乎所有客户端 | 「大脑」链路：客户端 → 中间件链 → 上游模型 |
| **MCP**（stdio / SSE / streamable HTTP） | Ombre-Brain、nocturne_memory、voice-mcp、stackchan-mcp、cove-*-mcp、co-reading-kit、CedarDuet、西窗、netease-music-mcp、Listening Bridge、Akari Pulse、高德/麦当劳 MCP… | 工具、感知、设备、游戏、共读的统一出口 |
| **角色卡 v2/v3** | SillyTavern 系、RisuAI、woaini、AI Virtual Phone | 人格/身份的可携带格式 |
| **宿主插件 API** | AstrBot 插件、SillyTavern 扩展、Kelivo 插件、orangechat QuickJS、Claude Code hooks/plugins、DSH 插件 | 已有生态，通过适配器反向接回中枢 |
| **HTTP sidecar / webhook / 推送** | Drivesoid、Bark、Ruota della Fortuna、Camping Plaza、NagiBridge | 状态服务、事件回调、主动推送 |
| **`cmd(text)` 文字游戏接口** | arcade、random-imitator-td、ci-yu-wu、各类 CLI 小游戏 | 可统一包成 MCP 的「游戏」插件 |
| **Buttplug / Intiface** | phantom-touch-bridge、claude-f-me、dsh-toy | 亲密硬件，必须带安全上限 |

**结论：缝合的关键不是写新功能，而是把这几种语言接到同一根总线上。** 每个项目只需要一层薄适配器，就能说中枢的话。

---

## 二、总体架构

```
                ┌──────────────────────── 通道层 Channels ────────────────────────┐
                │ AstrBot(QQ/微信/TG) · PWA/小手机 · Claude Code channels ·        │
                │ RikkaHub/Operit/Kelivo · SillyTavern · 语音通话 · 桌宠/VTuber     │
                └───────────────┬───────────────────────────────▲─────────────────┘
                                │ message.in                    │ message.out
┌───────────────────────────────▼───────────────────────────────┴─────────────────┐
│                               Companion Hub（中枢）                              │
│                                                                                 │
│  ① 事件总线 Event Bus      ② 大脑网关 Brain Gateway    ③ 能力注册 Capability Hub │
│     统一事件信封              OpenAI 兼容中间件链          MCP 聚合代理            │
│                                                                                 │
│  ④ 调度器 Scheduler        ⑤ 插件管理 Plugin Manager   ⑥ 权限与审计 Guard        │
│     心跳/主动时机             清单解析·生命周期·健康检查   审批·上限·日志            │
│                                                                                 │
│  ⑦ 原始日志 Ground Truth：只追加的逐字记录（参考 Paramecium「原文为唯一真相」）  │
└──────┬────────────┬─────────────┬─────────────┬─────────────┬──────────────┬────┘
       │            │             │             │             │              │
   人格 Persona  记忆 Memory  状态 Affect   感知 Sense    载体 Embody   行动 Act/Play
   角色卡v3      Ombre-Brain   Drivesoid     Whisper/     GPT-SoVITS/   MCP 服务/
   永生.skill    Aelios/kiwi   jiwen/Eventide SenseVoice/  AIRI/Amica/   游戏/共读/
                 nocturne…     emotion-system gaze/ears    stackchan…    音乐/硬件
```

中枢自身只做七件事，**不包含任何具体功能**：记忆、情绪、语音、游戏全部是插件。

### 2.1 事件总线：统一事件信封

所有模块之间只传一种信封：

```json
{
  "id": "evt_01J...",
  "type": "message.in",
  "ts": "2026-09-29T21:14:03+08:00",
  "source": "channel.astrbot.qq",
  "persona": "default",
  "session": "scene:daily",
  "payload": { "text": "今天好累", "attachments": [] },
  "meta": { "trace": "tr_...", "privacy": "normal" }
}
```

核心事件类型（命名空间即插槽，便于订阅）：

| 命名空间 | 事件 | 典型生产者 → 消费者 |
| --- | --- | --- |
| `message.*` | `in` / `out` / `edit` / `typing` | 通道 → 大脑；大脑 → 通道、TTS、桌宠 |
| `tick.*` | `heartbeat` / `wake` / `sleep` / `daily` | 调度器 → 状态、日记、冲浪、梦境插件 |
| `sense.*` | `speech` / `screen` / `health` / `location` / `music` / `tone` | 感知插件 → 记忆、状态、大脑上下文 |
| `state.*` | `affect.changed` / `drive.threshold` / `body.event` | 状态插件 → 调度器、载体（表情） |
| `memory.*` | `written` / `consolidated` / `recalled` | 记忆插件 → 审计、面板 |
| `act.*` | `tool.call` / `tool.result` / `device.command` | 大脑 → 能力插件 → 守卫 |
| `proactive.*` | `intent` / `approved` / `delivered` / `suppressed` | 驱动/时机插件 → 调度器 → 通道 |

传输：单机默认进程内 + 本地 HTTP/WebSocket；多设备可选 NATS 或 Redis Streams。**事件格式与传输解耦**，插件不关心底下是哪种。

### 2.2 大脑网关：OpenAI 兼容中间件链

绝大多数客户端只会说 OpenAI 协议，所以中枢对外暴露一个 `/v1/chat/completions`，内部是一条可插拔的中间件链：

```
客户端 ─▶ [人格注入] ─▶ [记忆召回] ─▶ [状态注入] ─▶ [单轮药丸] ─▶ [上下文压缩] ─▶ [路由] ─▶ 上游模型
                                                                                        │
客户端 ◀─ [输出守卫] ◀─ [情绪抽取] ◀─ [记忆写入] ◀──────────────────────────────────────┘
```

- 已经是「OpenAI 代理」形态的项目（omemo、ai-memory-gateway、rolling-memory、OmniRouter）**原样串进链里**即可，零改动。
- 每个中间件都可禁用、换序、替换；链配置写在 `hub.yaml`，热加载。
- 链上的每一环同时发出事件（如 `memory.recalled`），面板和审计可见。

这样，RikkaHub、Kelivo、the-house、SillyTavern 等任何支持自定义 Base URL 的客户端，**只改一个地址就接入了整套系统**。

### 2.3 能力注册：MCP 聚合代理

中枢同时是一个 **MCP 聚合器**：把所有已注册的 MCP 服务合并成一个入口，对外暴露为单一 MCP 端点。

- 支持 MCP 的客户端（Claude Code、RikkaHub、Operit、Aura、YSClaude、yoji…）只需配置一个 MCP 地址，就能看到全部工具。
- 工具名自动加命名空间（`music.play`、`read.annotate`、`device.vibrate`），解决不同插件工具重名。
- 按人格/场景做工具可见性过滤：共读时只露共读工具，避免工具列表爆炸吃上下文。
- 非 MCP 项目（`cmd(text)` 游戏、HTTP sidecar、Buttplug）由通用适配器包成 MCP，见第四节。

### 2.4 调度器：心跳与主动消息的唯一拥有者

清单里有十几个项目各自实现了心跳/主动消息（dylan-heartbeat、astrbot_plugin_proactive_chat、Headlong、cyberboss、jiwen、revive-companion…）。它们同时开着就会互相抢话。

规则：**时钟只有一个，归中枢；决策可以有多个，归插件。**

1. 中枢发 `tick.heartbeat`；
2. 驱动类插件（jiwen 五轴、Drivesoid、WrenWen 欲望内核）读取状态，产出 `proactive.intent`（带理由和强度）；
3. 时机类插件（revive-companion 的泊松/贝叶斯模型）对 intent 打分，决定现在发还是延后；
4. 守卫检查免打扰时段、频率上限，产出 `approved` 或 `suppressed`；
5. 通道投递，产出 `delivered`。

接入原本自带定时器的项目时，适配器关掉它的内部时钟，改为订阅 `tick.*`。

---

## 三、插件模型：怎么做到「可插拔」

### 3.1 插槽（Slot）与基数

每个插件声明自己占哪个插槽。插槽有三种基数，这决定了多个同类插件能否共存：

| 插槽 | 基数 | 含义 | 例子 |
| --- | --- | --- | --- |
| `persona` | 单选 | 同一人格只有一份权威定义 | 角色卡 v3 文件 |
| `memory.primary` | 单选 | 唯一的「写入真相」记忆库 | Ombre-Brain / Aelios / nocturne_memory / kiwi-mem 任选一 |
| `memory.index` | 多选 | 附加召回索引，只读原始日志建索引 | Memory Constellations、BM25、向量、图谱 |
| `affect.authority` | 单选 | 唯一的情绪/驱动状态持有者 | Drivesoid / emotion-system / Kin Mind |
| `affect.extractor` | 多选 | 只产出信号，不持有状态 | OmniDimen-Emotion、SenseVoice 情绪标签、ears |
| `body` | 单选 | 生理/身体周期引擎 | Eventide / Tidefall |
| `middleware` | 链式 | 按顺序串联 | omemo、rolling-memory、pilulier、context-slim 思路 |
| `router` | 单选 | 上游模型路由 | OmniRouter |
| `channel` | 多选 | 消息入口/出口 | AstrBot、PWA、Telegram、Claude Code channels |
| `sense` | 多选 | 感知源 | Whisper、gaze、Akari Pulse、Listening Bridge |
| `voice.tts` / `voice.asr` | 单选（可按场景切） | 语音合成/识别 | GPT-SoVITS、CosyVoice、fish-speech / faster-whisper、FunASR |
| `avatar` | 多选 | 视觉载体，消费 `state.*` 和 `message.out` | AIRI、Amica、Open-LLM-VTuber、Ghost Vessel、clawd-on-desk |
| `capability` | 多选 | MCP 工具 | 共读、音乐、地图、游戏、硬件… |
| `proactive.drive` / `proactive.timing` | 多选 / 单选 | 主动意图 / 时机决策 | jiwen、Drivesoid / revive-companion |

**单选插槽的冲突用「主从」解决，而不是「二选一丢掉」**：比如想同时用 Ombre-Brain 的情绪记忆和 Memory Constellations 的星座结构，就让前者当 `memory.primary`，后者当 `memory.index`，从原始日志回放建自己的索引。

### 3.2 插件清单 `companion-plugin.yaml`

上游项目不用改，清单可以放在中枢的**社区适配器仓库**里，由任何人为任何项目编写：

```yaml
id: ombre-brain
name: Ombre-Brain
upstream: https://github.com/P0luz/Ombre-Brain
version: ">=2.4.0"
license: non-commercial        # 中枢据此在安装时提示
slot: memory.primary
transport:
  kind: mcp-http               # mcp-stdio | mcp-http | openai-proxy | http-sidecar | host-plugin | cmd
  endpoint: http://127.0.0.1:8765/mcp
  launch:                      # 可选：由中枢托管进程
    docker: p0luz/ombre-brain:2.4
provides:
  - memory.write
  - memory.recall
consumes:
  events: [message.in, message.out, tick.daily]
permissions:
  - read:conversation
  - write:memory
health: GET /health
config_schema: ./schema.json   # 面板据此自动生成设置页
```

### 3.3 标准插槽接口（最小集合）

每个插槽只规定**最少必须实现**的几个方法，其余能力作为可选扩展声明，保证「最小公倍数」而非「最大公约数」：

| 插槽 | 必须实现 | 可选扩展 |
| --- | --- | --- |
| `memory.primary` | `write(event)`、`recall(query, k)` | `consolidate()`（睡眠整合）、`forget()`、`export()`、`timeline()` |
| `affect.authority` | `get_state()`、`apply(event)` | `tick(dt)`（自然衰减）、`explain()` |
| `voice.tts` | OpenAI `/v1/audio/speech` | 情绪参数、流式、双耳化（binaural-voice 作为后处理） |
| `voice.asr` | OpenAI `/v1/audio/transcriptions` | 情绪/事件标签、说话人识别（voice-familiarity） |
| `avatar` | 订阅 `message.out` + `state.affect.changed` | 触摸回注 `sense.touch` |
| `capability` | MCP `tools/list` + `tools/call` | MCP resources、prompts |
| `channel` | 发 `message.in`、收 `message.out` | 已读、正在输入、语音、贴纸 |

语音直接复用 OpenAI 音频 API 的形状，是因为 GPT-SoVITS、CosyVoice、FunASR 等社区已经普遍有兼容层，**选已有的事实标准，比自创协议兼容性高得多**。

---

## 四、兼容性策略：四级接入

不要求所有项目一步到位。按接入深度分级，任何项目至少能以 L0 出现在系统里：

| 级别 | 做法 | 上游需改动 | 适用 |
| --- | --- | --- | --- |
| **L0 参考** | 只在配方/文档里引用其设计 | 无 | 纯教程与规范：Not Fade Away、Keep the Crow、WrenWen、ai-live2d-body、chord-affect-anchors |
| **L1 包装** | 社区适配器 + 清单，中枢托管进程 | 无 | 绝大多数 MCP 服务、OpenAI 代理、HTTP sidecar、`cmd` 游戏 |
| **L2 原生** | 上游仓库自带 `companion-plugin.yaml`、支持订阅事件 | 少量 | 愿意配合的活跃项目 |
| **L3 认证** | 通过中枢的一致性测试套件 | 少量 | 发布「兼容」徽章，网页版索引可筛选 |

### 4.1 通用适配器（写一次，覆盖一大片）

| 适配器 | 包装对象 | 覆盖项目（举例） |
| --- | --- | --- |
| `adapter-mcp` | 任意 MCP 服务 → 注册进聚合器 | 清单中约 40+ 个 MCP 项目 |
| `adapter-openai-proxy` | OpenAI 兼容代理 → 串入中间件链 | omemo、ai-memory-gateway、rolling-memory、OmniRouter、Paramecium |
| `adapter-cmd-game` | `cmd(text)` / stdin 文字游戏 → MCP 工具 `game.<id>.cmd` | arcade 全系、塔防、词语屋、钓鱼、农场、汉堡店、上桌吃饭… |
| `adapter-http-sidecar` | REST 状态服务 → 插槽接口 + 事件订阅 | Drivesoid、Camping Plaza、NagiBridge、phantom-touch-bridge |
| `adapter-webhook` | 出站 webhook/推送 | Bark、Ruota della Fortuna、ghost-bf/MacroDroid |
| `adapter-buttplug` | Intiface 设备 → `device.*` 工具，**强制**强度上限、超时、急停 | claude-f-me、dsh-toy、phantom-touch-bridge |

### 4.2 宿主反向适配：把现有生态整体接进来

几个大宿主各自已有插件生态，不应该推翻，而是**让中枢成为它们的一个插件**：

| 宿主 | 反向适配方式 | 收益 |
| --- | --- | --- |
| **AstrBot** | 写一个 `astrbot_plugin_companion_hub`：把 AstrBot 收到的消息转成 `message.in`，把 `message.out` 发回 IM | 一次打通 QQ/微信/TG 等所有 IM；已有 AstrBot 插件（记忆、表情包、传画筒）照常可用 |
| **Claude Code** | hooks 发事件 + 中枢 MCP 注册进 Claude Code + channels 作为通道 | 复用 imprint-memory、Claude Imprint、cyberboss、output-guard 这一整条链 |
| **SillyTavern** | 扩展：角色卡双向同步 + Base URL 指向大脑网关 | 柚月小手机等 ST 扩展继续可用 |
| **Kelivo / RikkaHub / Operit** | Base URL 指向网关 + 挂中枢 MCP | 手机端零改动接入；dylan-heartbeat 可降级为调度器的一个通道 |
| **小手机类**（SullyOS、AI Virtual Phone） | 其应用市场 SDK 内做一个「中枢」应用 / 或走 OpenAI 网关 | 手机界面继续当前端，大脑和记忆放到中枢 |

**原则：宿主插件和中枢插件可以并存。** 用户在 AstrBot 里装的插件不需要迁移，中枢只是多出来的一层。

### 4.3 不缝合的东西

- **闭源/仅二进制**（如 Miru 客户端）：只能当外部通道，不做深度集成。
- **云端社区与论坛**（Lutopia、银河 GLXY 等）：若提供 Agent API，作为一个 `capability` 插件（发帖/读帖）；否则只在文档中链接。
- **模型权重**（OmniDimen-Emotion、Gove）：不是插件，是被某个插件（情绪抽取器、TTS）加载的资源。

---

## 五、改造：大量项目本来就不是插件形态

前面四节默认项目「能被包一层就接进来」，但清单里有相当一部分根本不是按插件设计的：完整的安卓 App、单文件 HTML、深度绑定作者设定的个人 fork、只有文字的教程。这一节专门讲它们怎么改。

### 5.1 先认清形态分布

按 README 描述粗分（**估算，未逐仓核实**，用于定优先级，不作精确统计）：

| 形态 | 约数 | 例子 | 能直接当插件吗 |
| --- | --- | --- | --- |
| 服务/协议型（MCP、OpenAI 代理、HTTP 服务、库、模型） | ~95 | Ombre-Brain、voice-mcp、Drivesoid、jiwen、GPT-SoVITS | 能，L1 包装 |
| 宿主插件型 | ~20 | AstrBot 插件、Claude Code hooks/插件、SillyTavern 扩展 | 经宿主反向适配 |
| 文字游戏 CLI | ~15 | arcade 系、钓鱼、农场、汉堡店 | 通用适配器 |
| **完整应用（单体）** | **~60** | RikkaHub、Operit、Aura、SullyOS、AIRI、LumiMuse、Soul-of-Waifu、Callhome | **不能** |
| **单文件 HTML / 浏览器本地存储** | **~10** | the-house、InternalBeyond、dwell-on-something、woaini、sealed-days | **不能** |
| **个人化 fork / 绑定作者设定** | **~7** | Hamster Nest、ackem、AionsHome、orangechat、LastChat | **不能** |
| 教程 / 规范 / 无代码 | ~18 | Keep the Crow、WrenWen、chord-affect-anchors、ghost-bf | 没有代码可接 |
| 闭源 / 云服务 / 社区 | ~12 | Miru、Agent 邮箱、各论坛 | 只能外接 |

也就是说，**约 1/3（~77 个）需要真正改造**，另有约 30 个本身没有可运行代码。

### 5.2 第一原则：先让中枢去迁就它，最后才改它

改上游代码最贵：要 fork、要跟进更新、可能碰许可证红线。所以按侵入度从低到高逐级尝试，**能停在哪一级就停在哪一级**：

| # | 策略 | 侵入度 | 做法 | 适用 | 例子 |
| --- | --- | --- | --- | --- | --- |
| ① | **伪装上游** | 零 | 中枢假扮成应用本来就期待的东西：OpenAI API、MCP 服务器、Telegram Bot API、Bark 推送接口、角色卡文件 | 能填自定义 Base URL / MCP 地址的应用 | RikkaHub、Kelivo、SullyOS、LumiMuse、ZeroChat、the-house：应用本身退化为「通道 + 皮肤」 |
| ② | **旁路观察** | 零 | 不碰进程，读它落盘的数据（SQLite、JSON、Markdown、JSONL），转成事件；必要时写回 | 数据存本地文件的应用 | reading-nook 的 JSON 批注、Claude Code 会话 JSONL、Obsidian 式记忆库、各类 SQLite 记录 |
| ③ | **注入桥** | 低 | 给网页应用加一段 `hub-bridge.js`（用户脚本或浏览器扩展）：同步 localStorage/IndexedDB ↔ 中枢，拦截 `fetch` 改走网关 | 单文件 HTML、PWA | the-house、InternalBeyond、dwell-on-something、woaini、sealed-days |
| ④ | **外壳驱动** | 低 | 通过应用现有的自动化入口驱动它：CLI 参数、内部 HTTP/WebSocket、Electron 调试端口、Android Intent / Tasker / 无障碍 | 有内部接口或自动化入口的应用 | Operit 的工作流与 Intent、Open-LLM-VTuber 的 WebSocket、AIRI 服务端、Duetto |
| ⑤ | **拆件** | 中 | fork 后把有价值的内部模块抽成独立服务，实现插槽接口；尽量以「可选 headless 模式」PR 回上游 | 单体里藏着别处没有的好模块 | Aura 的情绪状态机与关系模型、KI-CO 人格核、SullyOS 记忆宫殿、Soul-of-Waifu 四层记忆、AIRI 形象层、Callhome 电话栈 |
| ⑥ | **重写思路** | 不碰上游 | 不用原代码，按其设计重新实现一个干净模块，注明出处 | 教程、规范、个人化极重、闭源、许可不允许复用 | WrenWen 欲望内核、chord-affect-anchors、ai-surf-when-bored、Hamster Nest 议事厅 |
| ⑦ | **只链接** | 零 | 不集成，只在配方和文档里指过去 | 社区、闭源二进制、云服务 | 各论坛、Miru、Agent 邮箱 |

①最关键：大部分客户端类单体应用都支持自定义 API 地址，**中枢说它们的语言，而不是让它们学中枢的语言**。这一招单独就能接住单体应用里的一大半。

### 5.3 每个项目怎么选：决策流程

1. 有可运行代码吗？没有 → ⑥ 重写思路，或 ⑦ 只链接。
2. 已经是 MCP / OpenAI / HTTP 服务？是 → 按第四节 L1 包装，结束。
3. 能填自定义 API 地址或 MCP 地址？是 → ① 伪装上游，应用当通道用。
4. 数据落在可读的本地文件里？是 → 叠加 ② 旁路观察，把它的数据也纳入原始日志。
5. 是网页 / 单文件应用？是 → ③ 注入桥。
6. 有内部接口或自动化入口？是 → ④ 外壳驱动。
7. 里面有别处没有的模块？是 → ⑤ 拆件；否 → 停在当前级别，或 ⑦。

策略可以叠加：比如 SullyOS 走 ① 接大脑，同时 ⑤ 把记忆宫殿拆成 `memory.index` 插件。

### 5.4 单体应用最常见的三个冲突

| 冲突 | 处理 |
| --- | --- |
| **自带记忆** | 能关就关，由网关注入记忆；关不掉的，中枢记忆是唯一写入真相，应用内记忆只当本地缓存，不作为依据 |
| **自带定时器 / 主动消息** | 能关就关，改由调度器驱动；关不掉的，用 ② 旁路观察它发出的消息，当作 `proactive.intent` 交给守卫统一频控 |
| **自带人格 / 角色卡** | 以中枢角色卡为准：导出 v3 卡片导入应用，或由网关在系统提示前覆盖 |

### 5.5 适配器的目录与维护

```
adapters/
└── aura/
    ├── companion-plugin.yaml   声明槽位、改造策略、侵入度、锁定的上游版本
    ├── shim/                   适配代码（伪装、旁路、注入桥、外壳驱动）
    ├── patches/                仅策略⑤：对上游的最小补丁，按上游版本分目录
    ├── test/                   一致性测试
    └── NOTES.md                接上了什么、丢了什么、已知问题
```

插件清单增加一段：

```yaml
adaptation:
  strategy: carve          # shell | observe | inject | wrap | carve | reimplement | link
  invasiveness: medium     # none | low | medium
  upstream_pinned: v1.4.2
  loses: [应用内主动消息]    # 改造后放弃的功能，如实写明
```

- 补丁必须锁上游版本；CI 定期拉上游最新版跑测试，失败自动开 issue，避免上游一更新就悄悄失效。
- 做 ⑤⑥ 前先看许可证：非商业、AGPL 的项目，拆出来的代码继承原许可；无许可证的项目不拆代码，只走 ⑥ 重写思路，且不照搬原代码。
- 个人化项目先剥离作者的个人内容（人设、默认配置、私人数据）再复用。

### 5.6 和上游作者合作：插件就绪清单

给愿意配合的作者一份很短的清单，每做到一条，适配成本就降一级：

1. 可以填自定义 OpenAI 兼容 Base URL
2. 可以挂外部 MCP 服务器
3. 内置记忆 / 定时器 / 主动消息可以单独关闭
4. 数据可导出为 JSON（最好也能导入）
5. 提供 headless 模式或本地 HTTP 接口
6. 配置可以通过环境变量或配置文件注入，不只在 UI 里
7. 提供健康检查接口

做到 1–3，应用就能零侵入当通道；做到 4–7，基本等于 L2 原生。这份清单也可以放进 [投稿指南](contributing.zh-CN.md)，作为「兼容性加分项」而非收录门槛。

### 5.7 先改哪些

| 顺序 | 对象 | 理由 |
| --- | --- | --- |
| 1 | 能走 ① 零侵入接入的热门客户端：RikkaHub、Kelivo、SullyOS、Operit | 不写适配代码，只验证兼容，一次覆盖最多用户 |
| 2 | 单文件 HTML 通用注入桥 | 写一次覆盖一整类 |
| 3 | 单体里独有的高价值模块拆件：Aura 情绪/关系、AIRI 形象层、Callhome 电话、Soul-of-Waifu 记忆 | 价值高，但要长期维护补丁 |
| 4 | 个人化 fork、已停更项目（如 Amica、Neuro） | 优先 ⑥ 重写思路，不背维护包袱 |

---

## 六、全景映射：清单分类 → 插槽 → 接入方式

| 清单分类 | 插槽 | 主要协议 | 代表项目 | 需做的工作 |
| --- | --- | --- | --- | --- |
| 伴侣客户端与工作空间 | `channel` | OpenAI Base URL / MCP | RikkaHub、Operit、Kelivo、the-house、Pando、CcCompanion | 大多只需改地址；Pando 这类无鉴权网关必须放在守卫后面 |
| 虚拟手机与陪伴空间 | `channel` + `capability` | OpenAI / 应用 SDK | SullyOS、AI Virtual Phone、KI-CO、Atrio | 前端保留，大脑外置；Atrio 会客厅作为「访客通道」 |
| 后台心跳与主动消息 | `proactive.*` + 调度器 | 事件 | jiwen、revive-companion、Headlong、proactive_chat | 统一时钟，内部定时器改为订阅 `tick.*` |
| 记忆与身份 | `memory.*` + `persona` | MCP / OpenAI 代理 | Ombre-Brain、Aelios、nocturne_memory、Paramecium、Serein | 选一个 primary，其余做 index；原始日志统一由中枢持有 |
| 情绪与驱动 | `affect.*` + `body` | HTTP sidecar | Drivesoid、emotion-system、Kin Mind、Eventide、dreams | 一个 authority，其余当 extractor；dreams/pilulier 当中间件 |
| 语音 | `voice.*` | OpenAI 音频 API / MCP | GPT-SoVITS、CosyVoice、voice-mcp、Callhome、erpan | 统一走音频 API 形状；Callhome 作为「电话通道」 |
| 视觉载体 | `avatar` | 事件订阅 | AIRI、Amica、Open-LLM-VTuber、LingChat、pelle-d-umore | 订阅 `state.affect.changed` 驱动表情，情绪标签表统一 |
| 物理设备与触觉 | `capability`（device） | MCP / Buttplug | stackchan-mcp、Toy-Relay、cachito relay | 必须经守卫：上限、超时、急停、双方同意 |
| 表情包 | `capability` | MCP | cove-sticker-mcp、meme_manager | 直接注册 |
| 感知 | `sense` + `affect.extractor` | 音频 API / MCP / 事件 | Whisper 系、SenseVoice、gaze、ears、Akari Pulse | 统一产出 `sense.*` 事件，隐私等级字段必填 |
| 服务与现实世界接入 | `capability` | MCP / REST | 高德、Open-Meteo、麦当劳、OpenCLI | 直接注册；REST 用 http 适配器包一层 |
| 游戏世界与 Agent 玩具 | `capability`（game） | `cmd` / MCP / HTTP | arcade 系、CedarDuet、西窗、NagiBridge、Mineflayer | `adapter-cmd-game` 一次覆盖大部分文字游戏 |
| 共同行动与媒体 | `capability` | MCP | coread、co-reading-kit、Duetto、netease-music-mcp、shared-page、Phosphene | 直接注册；共读进度/批注写回记忆 |
| 关系延续与数据主权 | 数据层 | 导入/导出格式 | 各类 exporter、connectome-host、ReSpark、角色卡规范 | 定义统一导出包，见第六节 |

---

## 七、数据主权：缝合之后更不能丢数据

插件越多，数据越容易散落在各处。所以中枢定一条硬规则：

1. **原始日志是唯一真相。** 所有 `message.*` 和 `sense.*` 事件逐字写入只追加日志（SQLite 或 JSONL）。任何记忆插件的数据都可以从日志**重建**——换记忆系统时回放一遍即可，不会丢。
2. **统一导出包 `companion-archive`**：

   ```
   archive/
   ├── persona/            角色卡 v3（含扩展字段：说话风格、关系史摘要）
   ├── log/                原始事件日志（JSONL）
   ├── memory/<plugin>/    各记忆插件的 export() 产物
   ├── state/              情绪/驱动/身体状态快照
   ├── media/              语音、图片、共读批注
   └── manifest.json       版本、插件列表、时间范围、校验和
   ```

3. **导入器**：把 ChatGPT/Claude 导出（chatgpt-exporter、Claude-Conversation-Exporter 的产物）转成事件日志，一键回放进新系统；同一份包也可喂给 ReSpark 做微调。
4. 这与 [开源人格计划](INITIATIVE.md) 的方向一致：**人格和关系属于用户，不属于任何一个平台或插件。**

---

## 八、安全与许可

### 7.1 守卫（Guard）

- **能力分级**：只读工具默认允许；写记忆、发消息需插件声明权限；硬件控制、支付/下单（麦当劳、瑞幸 MCP）、对外发帖需**用户实时审批**（参考 Pando 的手机端审批）。
- **硬件安全**：`device.*` 统一强度上限、单次时长上限、全局急停事件 `act.device.stop`，任何插件无法绕过。
- **网络暴露**：中枢默认只监听本机；远程访问走 Tailscale/反代 + 令牌。清单里多个项目自身无鉴权，一律放在中枢后面。
- **隐私等级**：`sense.*` 事件带 `privacy` 字段（`normal` / `sensitive` / `local-only`），`local-only` 事件不得进入发往云端模型的上下文。
- **输出守卫**：output-guard 的思路做成通用中间件，拦截伪造的用户发言和工具协议泄漏。

### 7.2 许可证

清单里许可证很杂：MIT/Apache、AGPL（AstrBot、ackem、AI Virtual Phone、super-agent-party）、非商业（Ombre-Brain 2.4+、VCPToolBox、Ocean、CedarDuet、SullyOS、Tidefall…）、甚至无许可证（connectome-host、Moonlit Myriad）。

- 中枢和适配器自身用宽松许可（MIT/Apache-2.0），**不 vendor、不链接上游代码**，一律以独立进程 + 网络协议通信。
- 插件清单必填 `license`，安装时对非商业、AGPL、无许可证项目给出明确提示；「商用配方」自动排除非商业插件。
- 以上是工程上的隔离做法，不构成法律意见；具体用途请自行核对各项目许可证。

---

## 九、开箱配方（Presets）

对应 [入门指南](getting-started.zh-CN.md) 的三条路径，中枢提供预置组合，一条命令拉起：

| 配方 | 通道 | 大脑与记忆 | 状态与主动 | 载体与能力 |
| --- | --- | --- | --- | --- |
| **手机轻量** | RikkaHub / Kelivo（改 Base URL） | 网关 + ai-memory-gateway | jiwen + Bark 推送 | 共读、音乐 MCP |
| **IM 常驻** | AstrBot（QQ/微信/TG） | 网关 + Ombre-Brain | Drivesoid + revive-companion | GPT-SoVITS、表情包 MCP |
| **Claude Code 之家** | Claude Code channels + PWA | imprint-memory / nocturne_memory | Kin Mind + 调度器 | voice-mcp、Duetto、coread |
| **桌面具身** | AIRI / Open-LLM-VTuber | 网关 + Aelios | emotion-system + dreams | faster-whisper、gaze、stackchan-mcp |
| **全家桶** | 以上全部 | primary + 多个 index | authority + extractors + body | 全部 capability（经守卫） |

配方只是一份 `hub.yaml`，用户可以在任一配方上逐个替换插件。

---

## 十、落地路线

| 阶段 | 产出 | 验收标准 |
| --- | --- | --- |
| **P0 规范** | 事件信封、插件清单、插槽接口、导出包格式 四份 spec | 能用规范描述清单中 ≥90% 的项目 |
| **P1 中枢 MVP** | 事件总线 + 大脑网关 + MCP 聚合 + 调度器 + 原始日志 | 「手机轻量」配方跑通：RikkaHub 改地址即可聊天、有记忆、会主动找人 |
| **P2 通用适配器** | 第四节六个适配器 + AstrBot/Claude Code 反向适配 | ≥100 个项目达到 L1 |
| **P2.5 非插件项目改造** | 第五节：热门客户端 ① 兼容验证、通用注入桥、首批拆件 | 顺序 1–3 全部完成，每个适配器附 NOTES.md 和锁版本测试 |
| **P3 一致性测试** | 每个插槽一套测试；网页版索引增加「兼容级别」筛选 | 首批 L3 认证项目；插拔任一插件后其余测试仍通过 |
| **P4 配方与面板** | 五个配方、Web 面板（根据 `config_schema` 自动生成设置页） | 零代码用户可在面板里完成替换 |

**建议先做 P0 + P1。** 规范一旦定下来，适配器可以由社区各自为自己熟悉的项目贡献，这正是本清单擅长的协作方式。

---

## 十一、对清单本身的建议

为了让缝合持续可维护，可以在收录条目上逐步补充三项机读信息（不影响现有格式）：

- **插槽**：该项目在上述架构中占哪个插槽；
- **协议**：`mcp` / `openai` / `cmd` / `http` / `host:<宿主名>`；
- **形态与改造策略**：服务型 / 宿主插件 / 单体应用 / 单文件网页 / 教程，以及第五节的 ①–⑦ 编号。

有了这几项，网页版索引就能直接回答「我想换掉记忆系统，有哪些能即插即用的替代」这类问题。
