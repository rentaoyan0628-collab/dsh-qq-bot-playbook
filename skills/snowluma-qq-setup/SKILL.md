---
name: snowluma-qq-setup
description: 从零把 SnowLuma 装起来并接上 QQ，得到一个可用的 OneBot v11 接口（HTTP/WS + token），再（可选）转给 DeepSeek Harness / 自己的 bot 用。用户说「装 SnowLuma」「SnowLuma 怎么接 QQ」「QQ 机器人怎么搭」「OneBot 怎么配」「我的 bot 连不上 QQ」时加载。也用于排障：端口不通、token 对不上、事件收不到、登录掉线。
---

# SnowLuma 接入 QQ：从下载到能收发消息

## 0. 先搞清楚它是什么

SnowLuma 是**跑在本机**的 QQ 互操作运行时。它把 QQ 的原生会话，翻译成 **[OneBot v11](https://github.com/botuniverse/onebot-11)** 标准的 HTTP 接口和 WebSocket 事件。

所以它**不是**聊天机器人本身，是**管道**：

```
你的 bot / 脚本  ←→  SnowLuma（OneBot v11）  ←→  QQ 客户端  ←→  QQ 服务器
                     HTTP 发消息 / WS 收事件        注入/登录
```

只要你的程序会说 OneBot v11，接上它就能收发 QQ 消息——不用碰 QQ 的加密和协议。

- 上游仓库：[SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma)
- Release 页示例：[v1.14.21](https://github.com/SnowLuma/SnowLuma/releases/tag/v1.14.21)（版本号一直在涨，取最新即可）
- Windows x64 **完整版**包名：`SnowLuma-v<版本>-win-x64.zip`

---

## 1. 前置条件

| 项 | 要求 |
|---|---|
| 系统 | Windows x64（NTQQ 注入只在 Windows 生效；Linux 版只提供 OneBot 桥接能力） |
| QQ | **官方 NTQQ 客户端**，装好并至少登录过一次（不是旧版 QQ，不是 TIM） |
| Node.js | 完整版 zip **内置 node.exe**，不用自己装。若用 `-lite` 版则需 Node `^24.22.1 || >=26.12.1` |
| 网络 | 能正常上 QQ |

> 版本号有两条线：GitHub Release 上常见 `1.x`（如 1.14.21），运行时分发线可能显示 `2.x`（我们实测过 `@snowluma/runtime 2.1.1`）。装哪种都能用，OneBot v11 是稳定契约。

---

## 2. 下载与解压

1. 打开 [Release 页](https://github.com/SnowLuma/SnowLuma/releases)，下 **`SnowLuma-v<版本>-win-x64.zip`**（带 `-lite` 的是不含 node 的精简版，新手别选）。
2. 解压到一个**路径没有中文和空格**的目录，例如 `<SnowLuma 目录>\…`。
3. 解压后应该看到：

```
<SnowLuma 目录>\…
├─ index.mjs              ← 主程序（单文件 bundle，很大）
├─ launcher.bat           ← 双击这个启动
├─ check-node-version.cjs ← 版本自检
├─ node.exe               ← 内置运行时（完整版才有）
├─ package.json
├─ release.json
├─ client\  config\  data\  logs\  native\
```

---

## 3. 首次启动

1. 先**打开 QQ 客户端并登录**（SnowLuma 要连上它）。
2. 双击 `launcher.bat`。

它会先跑 `check-node-version.cjs`，Node 不达标就会 `pause` 退出——看到窗口停住不动，八成是版本问题（完整版不该出现）。

启动成功后：

- 控制台/日志会显示监听端口
- 浏览器打开 **`http://127.0.0.1:5099`** 进 WebUI（端口在 `config\runtime.json` 的 `webuiPort`，默认 5099）
- 登录密码在 `config\webui.json`（`passwordHash` + `passwordSalt`，首次启动自动生成；忘了就删掉这个文件重启，会重新生成）

WebUI 里完成：**同意 EULA/隐私 → 登录/授权 QQ 账号 → 配置 OneBot**。

> 第一次给的是"要不要授权、要不要注入 QQ"这类确认，必须真人点。程序点不了。

---

## 4. 配置 OneBot（关键一步）

在 WebUI 里新建一个 **账号配置**，会自动生成 `config\onebot_<QQ号>.json`。结构长这样（示例已脱敏）：

```json
{
  "mode": "snapshot",
  "networks": {
    "httpServers": [
      {
        "name": "http-default",
        "accessToken": "<自定的长随机串>",
        "messageFormat": "array",
        "reportSelfMessage": false,
        "host": "127.0.0.1",
        "port": 3000,
        "path": "/",
        "enableWebSocket": false
      }
    ],
    "wsServers": [
      {
        "name": "ws-default",
        "accessToken": "<另一个长随机串>",
        "messageFormat": "array",
        "reportSelfMessage": false,
        "host": "127.0.0.1",
        "port": 3001,
        "path": "/",
        "role": "Universal"
      }
    ],
    "httpClients": [],
    "wsClients": []
  },
  "statusCommand": { "enabled": true, "swallow": false, "cooldownSeconds": 5, "trigger": "#sl" },
  "historySync": { "enabled": false },
  "notifications": { "channelIds": [] }
}
```

几个**必须理解**的字段：

| 字段 | 说明 |
|---|---|
| `accessToken` | **两个方向各一个**。HTTP 那个用于你「发消息」，WS 那个用于你「收事件」。填错是 401 的头号原因 |
| `messageFormat: "array"` | 消息用**段数组**（`[{type:"text",data:{text:"..."}}]`）。设成 `"string"` 则是 CQ 码字符串。**别混** |
| `host: "127.0.0.1"` | 只监听本机。想让别的机器连，改成 `0.0.0.0`——但 token 一定要强，否则等于把 QQ 交给公网 |
| `reportSelfMessage` | 是否上报机器人自己发的消息。设 `true` 会收到自己发的，做回声逻辑时注意别死循环 |
| `port` | HTTP 3000 / WS 3001 是社区常用默认，可改，但要和你 bot 的配置对上 |

---

## 5. 验证：能连上了吗

**第一步，HTTP 通不通**（这是最常用的探活）：

```powershell
$token = "你的 httpServers accessToken"
$r = Invoke-RestMethod -Uri 'http://127.0.0.1:3000/get_login_info' `
  -Headers @{ Authorization = "Bearer $token" }
$r | ConvertTo-Json -Depth 5
```

成功返回类似：

```json
{ "status": "ok", "retcode": 0, "data": { "user_id": 123456789, "nickname": "你的昵称" } }
```

- `status: "failed"` + `retcode: 1400`/`401` → **token 错**
- 连不上 / 拒绝连接 → 端口错，或 SnowLuma 没起来
- 能连但 `status: "failed"` → QQ 掉线了，回 WebUI 看

**第二步，试发一条**：

```powershell
$body = @{
  user_id = 123456789
  message = @(@{ type = 'text'; data = @{ text = '测试一下' } })
} | ConvertTo-Json -Depth 6 -Compress
$bytes = [Text.Encoding]::UTF8.GetBytes($body)

Invoke-RestMethod -Uri 'http://127.0.0.1:3000/send_private_msg' -Method Post `
  -Body $bytes -ContentType 'application/json; charset=utf-8' `
  -Headers @{ Authorization = "Bearer $token" }
# 群聊换成 send_group_msg + group_id
```

> ⚠️ **中文必须走 UTF-8 字节**（`[Text.Encoding]::UTF8.GetBytes($body)`）。PowerShell 直接传字符串会按 GBK 编码，对方收到一串 `???`。这是最容易踩的坑之一。

**第三步，WS 收事件**：

```
ws://127.0.0.1:3001/  ，连接头带 Authorization: Bearer <wsServers 的 accessToken>
```

连上后每 30 秒会有一次 `meta_event/heartbeat`——**这就是最好的"活着"证据**。

---

## 6. 下一步：接进 DeepSeek Harness（上桥）

到这里 SnowLuma 只是把 QQ 变成了一个标准 OneBot v11 接口 —— 它不会思考。要让 DSH 当这个 QQ 背后的人，得把两边接起来。

**本包只覆盖一条路：加一层 qq-bridge。**

```
QQ 客户端 → SnowLuma(3000/3001) → qq-bridge → DSH
```

桥接做的事：

- 收 SnowLuma 的 WS 事件 → 整理成会话上下文 → 交给 DSH agent
- 把 agent 的工具调用（发消息、发语音、查状态）翻译回 OneBot HTTP 动作

为什么还要夹这一层：语音转文字、管理员传话箱、管理员硬闸、跨会话摘要、成长附录、3100 网页控制台，这六件事都长在它上面（设计见同仓库 `playbook/06-custom-features.md`）。

> 另一条路是给 DSH 装一个 OneBot 客户端插件、直接填 3000/3001。**本包不覆盖它** —— 我们没在那条路上实测过。只想要"能聊天"的人可以自己去找，但本包讲的、我实际在跑的都是桥接这条。

下一步去读同级目录的 `qq-bridge-setup`（最小可用：安装六步、`config.json` 每个字段填什么、排障），要语音转文字 / 传话箱 / 跨会话摘要 / 3100 控制台就直接读 `qq-bridge-full`（六项能力各自在哪、怎么验）。

---

## 7. 排障速查

| 症状 | 先查什么 |
|---|---|
| WebUI 打不开 | 端口被占；`config\runtime.json` 的 `webuiPort`；进程是否真起来了 |
| HTTP 401 | **两个 token 搞混了**（发消息用 httpServers 的，收事件用 wsServers 的） |
| 收到了消息，但 bot 发的没人收到 | `messageFormat` 是 `array` 还是 `string`，段结构对不对 |
| 对方收到 `???` | 你这边编码问题（见上面第 5 步的警告），不是 SnowLuma 的锅 |
| 事件收不到、心跳也没有 | WS 端口/路径错；`role` 不对；防火墙 |
| 一会儿能连一会儿断 | QQ 客户端在前台/后台切换、注入掉了；看 `logs\` 下当天的日志 |
| 起来就退出 | Node 版本不达标（看 `check-node-version.cjs` 的输出） |
| 改完配置不生效 | SnowLuma 要**重启**才重读 `config\onebot_*.json` |

日志在 `logs\`，按天分文件，排障时先 `tail` 当天的。

---

## 8. 安全底线（别跳）

1. **token 必须随机够长**，别用 `123456`。它等于你 QQ 的钥匙。
2. `host` 尽量保持 `127.0.0.1`。要外网访问就套反代 + HTTPS，别裸奔。
3. 别把 `config\` 目录（含 token）截图发群里。
4. 这个 QQ 号会被自动化操作，**别用你的主号**练手。
5. 频控：连续快速发消息容易被风控，加间隔、加去重。

---

## 9. 一页流程回顾

```
装 NTQQ 并登录
   ↓
解压 SnowLuma 完整版 → 双击 launcher.bat
   ↓
http://127.0.0.1:5099 → 同意协议 → 登录 QQ
   ↓
WebUI 建 OneBot 配置（记下 http/ws 两个 token）
   ↓
get_login_info 验证 → send_private_msg 试发 → WS 看心跳
   ↓
（可选）接 qq-bridge → 接 DSH / 你自己的 bot
```

---

## 出处与署名

这份文档由 **NOYOT** 整理编写，**不是它描述的工具的作者**。

它讲的工具来自：[SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma) —— QQ ↔ OneBot v11 运行时。

请去原作者仓库下载、按原作者文档操作；觉得好用就给原作者点个星。
如果本文件里有说错的地方，是整理者的问题，不是原作者的问题。
