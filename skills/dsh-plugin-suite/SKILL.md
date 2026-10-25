---
name: dsh-plugin-suite
description: 一套让 大肥鱼/任何 QQ bot「像个人」的 DeepSeek Harness 插件组合——心情、记忆、刷 B 站、鲸鱼挂件、Mnemon 记忆系统、ops 工具、QQ 体检、模式切换、四个 MCP。说明每个插件干什么、配置和数据在哪、哪些能跳过、装法与共同的坑。用户问「这台机器装了哪些插件」「每个插件干嘛的」「怎么配出一只像样的 bot」「心情/记忆/挂件/刷视频 从哪来」时加载。
---

# DSH 插件套装：一只"像人"的 bot 由哪些零件拼成

## 清单（本机实测版本）

| 插件 | 版本 | 一句话 |
|---|---|---|
| `dsh-feiyu-mood` | 0.5.2 | **心情**：每轮判断情绪怎么变，注入上下文 |
| `dsh-fish-memory` | 1.2.6 | **记忆**：会淡忘的长期记忆，半衰期 30 天 |
| `dsh-bili-scroll` | 2.0.4 | **刷 B 站**：挑视频、写日报 |
| `dsh-douyin-scroll` | 0.5.0 | **刷抖音**：候选→打分→日报（可选） |
| `dsh-whale-widget` | 0.4.2 | **鲸鱼挂件**：桌面小挂件 + 每轮结束的"咚" |
| `dsh-mnemon` | 0.5.34 | **记忆系统主体** + 9 provider / 4 source / 5 strategy |
| `dsh-ops` | 0.4.1 | **shell / file 工具**（没有它 agent 碰不到本机） |
| `dsh-qq-healthcheck` | — | **QQ 链路体检**：端口、token、心跳 |
| `qq-mode-console` | — | **模式切换**：`qq-chat` ↔ `qq-chat-v2`（reserved2） |
| MCP `snowluma` | — | QQ 全套工具（收发、记忆、表情、语音…） |
| MCP `snowluma-host` | — | 网关探活（`snowluma_status`） |
| MCP `web-search-safe` | — | 联网搜索/抓取（只读、防内网） |
| MCP `bili-live` | — | 直播间弹幕（读 + 放行后发） |

## 按作用分三组

### A. 让它有"人味"（核心三件）

**心情 `dsh-feiyu-mood`**
- 数据：`.dsh\feiyu-mood\config.json` + `state.json`（v2，按会话分份）
- 每轮一次**很短的模型调用**判断情绪 → 走 `systemPrompt.context` 注入，不破坏前缀缓存
- 半衰期：心情值 3 小时、七根情绪 90 分钟；基线 70
- **装完必须重启**；DSH < `0.1.7-rc.1` 会静默跳过

**记忆 `dsh-fish-memory`**
- 数据：`.dsh\fish-memory-config.json`
- **默认全是关的**，每个会话要手动点成「开」
- 半衰期 30 天、每天最多记 8 条、一次最多注入 6 条
- 「事实键」= 会变的事只留最新

**Mnemon `dsh-mnemon`**
- 管**文档 / 记忆空间 / 运行时记忆**三层，跟 fish-memory 不是一个东西
- 本机走 `mnemon-native` provider（`writeEnabled: true`、`recallMode: guided`）
- 那堆 `provider-*` 是备选后端（hindsight / mem0 / honcho…），**都要外部端点，没配等于没开**

### B. 让它有"活"（内容来源）

**刷 B 站 `dsh-bili-scroll`**：定时挑视频 → 打分 → 写 `digest/<年月>.md`；它自己的 state 在 `.dsh\bili-duty\`

**刷抖音 `dsh-douyin-scroll`**：家目录 `.dsh\douyin-duty\`，三条路（只读接口 / 签名只读 / 无头 Chrome），四层判断；不看画面

**鲸鱼挂件 `dsh-whale-widget`**：配置 `.dsh\.dshw-size.json`（`sound` / `vol` / `soundSet` / `turnCostOn`…）；**那个"咚"就是它**，点挂件 → 音效与提示 → 全局设置可关

### C. 基础设施（不显眼但缺了就跑不动）

**`dsh-ops`**：`ops_bash` / `ops_read_files` 等，agent 执行命令与读文件靠它

**`dsh-qq-healthcheck`**：QQ 链路体检，出问题先跑它

**`qq-mode-console`**：决定 QQ 会话用 `qq-chat` 还是 `qq-chat-v2`；当前 `reserved2`

**四个 MCP**：`snowluma`（QQ 工具全集，工具名 `mcp__snowluma__*`）、`snowluma-host`（探活）、`web-search-safe`（联网）、`bili-live`（弹幕）

## 装法

**标准**（有 `dsh` CLI 时）：

```bash
dsh plugin --profile desktop add github:Yuuyuko-uu/dsh-feiyu-mood
dsh plugin --profile desktop add github:Yuuyuko-uu/dsh-fish-memory
```

装完**必须重启 DSH**。

**桌面版没有 CLI 时**：改 `profiles\<档名>\package.json` + 放 `node_modules` + **改 `pnpm-lock.yaml` 的 commit**。第三步是命门，不改的话 DSH 启动会按旧 lock 重装回去（症状：装了新版、重启又变旧版）。

**MCP 不是插件**：在 `profiles\<档名>\cordis.patch.yml` 里加 `- insert: - id: mcp-xxx / name: '@deepseek-ai/dsh-mcp-client' / config: {serverName, transport: stdio, command, args}`，重启 DSH 才连上。

## 共同的坑

| 现象 | 原因 |
|---|---|
| 装了但设置页没有 | DSH 版本不够（静默跳过）/ 没重启 / 装错 profile |
| 重启后插件变回旧版 | `pnpm-lock.yaml` 还钉旧 commit |
| DSH 起不来报 `Unexpected token ''` | profile 的 `package.json` 带 UTF-8 BOM |
| 心情一直不动 | 判断开关关 / 会话选错 / confidence 低于门槛 |
| 记忆没记东西 | 那个会话还是「关」 |
| 记忆想不起来 | 证据门槛太高 / 注入条数太少 |
| 挂件没声音 | `.dshw-size.json` 里 `sound:false` 或 `vol:0` |
| 刷 B 站/抖音没结果 | 对应 duty 目录的登录态过期（Cookie） |

## 最小可用集（想省事就装这些）

```
必装：dsh-ops（执行能力）+ MCP snowluma（QQ 工具）
想要人味：dsh-feiyu-mood + dsh-fish-memory
想要活：dsh-bili-scroll（+ dsh-douyin-scroll）
想要记忆系统：dsh-mnemon
可省：dsh-whale-widget（纯装饰）、provider-* 那一堆（没端点就是摆设）
```

## 一页流程

```
装 dsh-ops + MCP snowluma → 重启
→ 装 feiyu-mood / fish-memory（github: 地址）→ 重启
→ 设置页：心情填称呼选会话 / 记忆把要记的会话点成「开」
→（可选）bili-scroll 配关键词 → 跑一轮出日报
→ 觉得吵：关 whale-widget 的音效
```
