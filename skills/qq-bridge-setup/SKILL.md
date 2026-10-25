---
name: qq-bridge-setup
description: 在 SnowLuma 和 DeepSeek Harness 之间装一层 qq-bridge，把「QQ 能说话」升级成「像个人」。这是本包唯一覆盖的接入路线（不做直连）。覆盖：安装六步、config.json 每个字段、六项能力各自在哪与怎么验、日常运维、排障。用户说「装 qq-bridge」「接进 DSH」「要传话箱」「语音转文字怎么接」「管理员闸怎么做的」「3100 控制台打不开」「跨会话摘要」时加载。也用于排障：连上了不回、重启后进程消失、白名单不生效。
---

# 上桥：把 DSH 接到 QQ 上

## 先搞清这一层是干什么的

| | DSH 原生 QQ 插件 | qq-bridge 完整版 |
|---|---|---|
| 链路 | `QQ → SnowLuma(3000/3001) → DSH` | `QQ → SnowLuma → **qq-bridge** → DSH` |
| 能聊天 | ✅ | ✅ |
| 语音转文字 | ❌ | ✅ |
| 管理员传话箱 | ❌ | ✅ |
| 管理员硬闸（防冒用） | ❌ | ✅ |
| 跨会话摘要 | ❌ | ✅ |
| 成长附录 | ❌ | ✅ |
| 网页控制台（3100） | ❌ | ✅ |

**本包只走这条路（上桥），不覆盖直连。** 装它的理由就是上面那六项——它们全部长在这一层上。如果你的目标只是「最快能聊」，另一条路是给 DSH 装一个 OneBot 客户端插件直连 3000/3001；那条路本包没有覆盖，我们也没实测过。

## 前置

1. **DSH 在跑**（任意端口，脚本会自己找）
2. **SnowLuma 已配好 OneBot**：HTTP `127.0.0.1:3000`、WS `127.0.0.1:3001`，有 accessToken
3. **Node ≥ 22.13**
4. 可选（没有也能跑，只是对应功能不亮）：GPT-SoVITS（语音）、ACE Studio（点歌）、Python venv（语音转写）

## 安装六步

```bash
git clone https://github.com/Derpyu520/qq-bridge.git
cd qq-bridge
npm install                 # postinstall 会自动修补 @snowluma/sdk 的 ESM 打包 bug，别跳过
copy config.example.json config.json
# 改 config.json（见下表）
node scripts/setup-dsh.mjs  # 装 preset + 挂 MCP
# 重启 DSH
start.bat                   # 或 node src/bridge.js
```

**第 5、6 步最容易漏**：只跑 bridge 不装 DSH 端，QQ 上会毫无反应（不报错，只是不回）。

## config.json 关键字段

| 字段 | 填什么 |
|---|---|
| `dsh.baseUrl` | 他的 DSH 地址（模板写 3080，实际按端口改） |
| `dsh.authToken` | **留空**。留空时用 `~/.dsh/.credentials.yaml` 离线铸 Cookie；填了更容易 401 |
| `snowluma.wsUrl` / `httpUrl` | `ws://127.0.0.1:3001` / `http://127.0.0.1:3000`（**别把 WS 端口填进 httpUrl**，会 426） |
| `snowluma.accessToken` | SnowLuma 里那个 token |
| `snowluma.followDiscoveredEndpoint` | 默认 `true`，换账号会自动跟随端点；跑第二个实例时才设 `false` |
| `allow.private` / `allow.groups` | QQ 号 / 群号。**模板是空数组，空着 + `allowAllWhenEmpty:false` = 谁都不回**（刻意的 fail-closed） |
| `admins` | 管理员 QQ 号数组，管理员闸与点歌/传话箱都看它 |
| `agentPreset` / `socialV2.agentPreset` | 默认 `qq-chat` / `qq-chat-v2` |
| `consolePort` | 默认 `3100` |
| `consoleToken` | 留空则启动时自动生成到 `state/console-token` |

## 六大功能：在哪、怎么验

### 1. 语音转文字

**作用**：收到的语音条渲染成 `[语音:转写文字]`，AI 才能"听懂"。

**依赖**：一个带 faster-whisper 的 Python venv + 一个转写脚本。

```powershell
python -m venv .venv-asr
.\.venv-asr\Scripts\pip install faster-whisper
```

`src/bridge.js` 里三个常量指向它们：

```js
const VOICE_ASR_PYTHON = process.env.QQ_VOICE_ASR_PYTHON || '<venv>\Scripts\python.exe';
const VOICE_ASR_SCRIPT = '<你的 asr.py>';
const VOICE_ASR_TIMEOUT_MS = 90000;
```

脚本约定：`python asr.py <wav路径> --lang zh`，**把文字打到 stdout**（UTF-8）。

开关：`socialV2.voice.transcribe`（不是 `false` 就开）。

验证：QQ 发一条语音，日志里会出现 `[voice] 转写失败（降级为 [语音]）` 或正常出字。

### 2. 管理员传话箱

**作用**：管理员在 QQ 里说「转告 dsh / 帮我放进传话箱」，消息进 `state/dsh-inbox.json`，由守护进程投给 DSH 会话。

**链路**：
```
qq_dsh_inbox(工具) → POST /api/admin/dsh-inbox → enqueueDshInbox → state/dsh-inbox.json
                                                              ↓
                              autostart/dsh-inbox-watcher.mjs → 投给某个 live DSH 会话
```

**watcher 要一直跑**，掉了就没人投递：

```powershell
Start-Process node -ArgumentList "autostart\dsh-inbox-watcher.mjs" `
  -WorkingDirectory <bridge目录> -WindowStyle Hidden `
  -RedirectStandardOutput state\dsh-inbox-watcher.out.log -RedirectStandardError state\dsh-inbox-watcher.err.log
```

日志：`state/dsh-inbox-watcher.log`，正常会有 `delivered <id> -> <sessionId>`。

**投错会话的经典坑**：`state/dsh-inbox-target.json` 会钉死一个 sessionId。如果那个会话停了/是内部 agent 会话，消息全进黑洞。watcher 里必须把内部目录排除干净（`/[\\/]qq-bridge[\\/]state[\\/]/i`），并且**排行不能用 `updatedAt`**（注入 prompt 会顶新 updatedAt，导致自我延续）。改用 `sessionListMetadata.lastPromptAt`，拿不到就返回 0。

### 3. 管理员硬闸

**作用**：防"群友冒充管理员"。核心是**不信任模型自填的 adminUserId**，回头查真实入站消息。

```js
function recentSpeakerMatches(key, uid, windowMs = 15*60*1000, maxScan = 60) {
  // 在该会话的 recentMessages 里找 userId === uid 且时间在窗口内的
}
function adminGateError(key, uid) {
  // 私聊 key: private:<uid> 直接放行（uid 来自 key 本身）
  // 群聊 key: 必须该 uid 本人最近 15 分钟在这个群说过话
}
```

套在三处：`/api/admin/song-request`、`/api/admin/dsh-inbox`、`/api/admin/profile`。

三条判定：
- 私聊管理员 → 过
- 群聊冒用别人的号 → 403「这个要管理员本人来说才行」
- 群聊非管理员 → 403「只有管理员可以使用这个功能」

**残留风险**：模型若在上下文里见过别的会话的 agentToken，可以伪造 key。彻底解法是一 QQ 会话一 DSH 会话，或一次性 action token。

### 4. 跨会话摘要

**作用**：让它在私聊里记得跟别人聊过什么，但**不越界播报**。

三件套：
- `state/shared-summaries.json` —— 摘要快照（人写/模型偶尔写）
- `state/social-v2.json` —— 每个会话的 `recentMessages`，**现场重建的原料**
- `sharedVisibleKeysV2(key, st)` —— 决定"这个会话能看到谁"

关键设计（别改坏）：
- 只在**注入 prompt 时**现算，不落盘、不每轮写 → 问起时最新
- 超过 15 分钟就用 `recentMessages` 现场重建，标签写「现场刚取」；旧的标「X 分钟前」
- 群聊：能看到群里那些人的私聊摘要
- 私聊：默认只能看到**群**的内容；只有对方在最近两条消息里**点名**某人（名字或 QQ 号）才放开那个人的私聊

```js
function privateMentionedKeysV2(key, st) {
  // 取最近两条非自己的消息，命中 people 表里的名字/QQ号 → 放开 private:<uid>
}
```

### 5. 成长附录（只准追加）

**作用**：给它一本只能往后写的本子。

- 文件 `state/growth-appendix.jsonl`，一行一条 `{ts, at, key, kind, reason, text, prev, hash}`
- **格式 `at` 必须带日期**（`2026-10-25 17:29`）。只存时分会让「每天最多 30 条」永远失效
- 每行 `hash = sha256(prev|ts|key|text).slice(0,12)` 串链，手工改历史 → 注入时标 `⚠️链断`
- 只有 `GET/POST /api/socialV2/growth`，**没有 update/delete**
- 注入：最近 6 条，放在【跨会话共享摘要】之后

### 6. 3100 控制台

```powershell
$tok = (Get-Content "<bridge>\state\console-token" -Raw).Trim()
Start-Process "http://127.0.0.1:3100/?token=$tok"
```

- 页面：`public/console.html`
- 无 token → 401 并弹 prompt 要密码
- 有 Host 头校验，只认 `127.0.0.1` / `localhost` / `[::1]`，反代会 403
- 所有 `/api/*` 也要 `x-console-token` 头（**这是用 curl 测接口总是 401 的真因**）

## 日常运维

| 要看什么 | 命令/位置 |
|---|---|
| 活着吗 | `GET http://127.0.0.1:3100/api/status`（带 `x-console-token`） |
| 正常状态 | `mode=reserved2 dshReady=True`，`snowluma.connected=true`、`accountOnline=true` |
| 日志 | `state\bridge.log`、`state\bridge.err.log`、`state\qq-activity.log` |
| 启动 | `start.bat`（自带 `:loop` 自动重启；退出码 2 = 已在别的窗口跑） |
| 重启 | `Stop-Process` 旧 pid → **等 2–3 秒** → 隐藏启动（见下） |

**重启的坑**：杀完立刻起会因端口没释放而启动失败（日志看着正常，但进程随即消失、3100 不通）。必须：

```powershell
Stop-Process -Id <pid> -Force
Start-Sleep -Seconds 3
Start-Process node -ArgumentList "src\bridge.js" -WorkingDirectory <bridge> `
  -WindowStyle Hidden -RedirectStandardOutput state\bridge.out.log -RedirectStandardError state\bridge.err.log
Start-Sleep -Seconds 8   # 再确认进程还在
```

## 排障

| 现象 | 原因 |
|---|---|
| QQ 毫无反应（不报错） | 没跑 `setup-dsh.mjs` 或装完没重启 DSH |
| 连上了但不回 | 白名单空 + `allowAllWhenEmpty:false` / preset 没装 / persona 里写了 `{{cwd}}`（fail loud） |
| 起的来又消失 | 端口没释放（等 2 秒）/ 两个实例抢 3100（start.bat 退出码 2） |
| 用 curl 测接口全 401 | 没带 `x-console-token` |
| 传话箱消息没人收 | watcher 没跑，或 target session 是死会话 |
| 重启后插件变回旧版 | `pnpm-lock.yaml` 还钉旧 commit |
| 语音不转写 | venv 缺 faster-whisper / 路径常量不对 / `voice.transcribe=false` |
| 群里明明有这个消息它说不知道 | `recentMessages` 被清（控制台「清除会话上下文」会删） |

## 一页流程

```
SnowLuma 就位（3000/3001）
→ clone qq-bridge → npm install
→ copy config.example.json config.json → 填 baseUrl / snowluma / allow / admins
→ node scripts/setup-dsh.mjs
→ 重启 DSH
→ start.bat
→ 开 http://127.0.0.1:3100/?token=... 看 status
→ QQ 里说一句，确认有回复
→（可选）配 venv 开语音转写 → 起 inbox-watcher
```
