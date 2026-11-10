---
name: start-qq
description: "按用户口令一键启动/自检 QQ 链路：SnowLuma 网关、QQ bridge、GPT-SoVITS 语音、ACE 点歌 worker、DSH 传话箱 watcher 和 DSH 上下文水位守卫，全部后台隐藏运行；并自检直播间只读弹幕工具（bili_live_info / bili_live_feed）与 QQ 侧白名单是否就位。用户说「启动QQ」「开QQ」「拉起QQ」「把QQ服务开起来」或类似话时加载并使用。"
---

# 启动 QQ 链路

## 何时用

用户说「启动QQ」「开QQ」「拉起QQ」「把QQ服务开起来」等，目标是让 QQ 里的
大肥鱼可用，而不是打开某个可见窗口。也适用于排查"QQ 没反应/传话没送到"时先做一遍链路自检。

> **NapCat 用户先停一下：**下列脚本是 SnowLuma 时代的本机适配示例，可能监控 5099 并硬编码启动旧网关。已换 NapCat 时，先按 [迁移篇](../../docs/napcat-migration.md) 修改守护、清查旧发现/自愈及环境变量，不能直接照跑。

## 一键启动

在 PowerShell 里执行（必须隐藏窗口，不要 `start` 出黑框）：

```powershell
Start-Process -FilePath 'powershell.exe' `
  -ArgumentList '-NoLogo','-NoProfile','-NonInteractive','-WindowStyle','Hidden','-ExecutionPolicy','Bypass','-File','<你的用户目录>\qq-bridge\autostart\dsh-qq-watchdog.ps1' `
  -WindowStyle Hidden
```

这个 watchdog 每 15 秒巡一圈，缺什么补什么：

| 组件 | 位置 | 端口/进程特征 |
|---|---|---|
| SnowLuma 网关 | `<SnowLuma 目录>\index.mjs` | 127.0.0.1:3000 / 3001（OneBot） |
| SnowLuma WebUI | 同上（网关自带） | 127.0.0.1:5099 |
| GPT-SoVITS 语音 | `<GPT-SoVITS 目录>\… -I api_v2.py -a 127.0.0.1 -p 9881` | 127.0.0.1:**9881**（不是 9880） |
| QQ bridge | `<你的用户目录>\qq-bridge\src\bridge.js`（经 `start.bat`，带 `:loop` 自愈） | 127.0.0.1:3100 |
| ACE 点歌 worker | `workers\song-worker.mjs` | `node.exe` 命令行含 `song-worker.mjs` |
| DSH 传话箱 watcher | `autostart\dsh-inbox-watcher.mjs` | 命令行含 `dsh-inbox-watcher.mjs` |
| DSH 上下文水位守卫 | `autostart\dsh-context-guard.mjs` | 命令行含 `dsh-context-guard.mjs` |
| 抖音私信自动回复 | `autostart\douyin-dm-autoreply.mjs`（受 `state\douyin-dm-autoreply.disabled` 门控） | 命令行含 `douyin-dm-autoreply.mjs` |
| **本地调参代理**（可选） | `tools\llm-tuning-proxy.cjs` | 127.0.0.1:8787；8787 不监听时 watchdog 会隐藏拉起，日志 `state\llm-proxy.out.log` |

watchdog 自带单实例锁（`state\autostart.pid`）；已在跑时再启动会自己退出，不会重复开。
它还会自动把 bridge 的 `baseUrl` 对齐到当前 DSH 端口（DSH 重启后端口会变）。

## 等待并验证

**验收只等 3000 / 3001 / 3100 三个端口**，语音端口 9881 不算验收条件。

原因：GPT-SoVITS 首次加载模型要预热 100 秒左右，把它算进「链路没起来」会让人以为启动失败、去反复重启，反而更慢。语音是后台慢慢起的，watchdog 下一轮看到没起会自己补。

```powershell
$ports = 3000,3001,3100
1..30 | ForEach-Object {
  $down = $ports | Where-Object {
    -not (Test-NetConnection 127.0.0.1 -Port $_ -InformationLevel Quiet -WarningAction SilentlyContinue)
  }
  if (-not $down) { 'ALL_UP'; break }
  Start-Sleep -Seconds 2
}
```

**SnowLuma 的 OneBot 端口不是立刻开的。** 实测从 QQ 客户端被 hook 注入到 3000/3001 真正 listen，大约要 **90 秒**（不是文档里常见的 6~8 秒）。这期间 5099 WebUI 可能已经起来了，但 3000/3001 还没开——这是正常的，不要因此去重启 QQ 客户端。

```text
05:09:00  [OneBot] session started: UIN=...
05:09:00  listening 3000/3001
```

看到这两行再断言「QQ 在线」。

再用控制台令牌确认桥接和 QQ 在线：

```powershell
$token = (Get-Content '<你的用户目录>\qq-bridge\state\console-token' -Raw).Trim()
$h = @{ 'x-console-token' = $token }
Invoke-RestMethod 'http://127.0.0.1:3100/api/status' -Headers $h |
  Select-Object mode,dshReady,ownerQQ,@{n='snowConnected';e={$_.snowluma.connected}},@{n='accountOnline';e={$_.snowluma.accountOnline}}
```

期望 `dshReady=True`、`snowConnected=True`、`accountOnline=True`。

后台 worker 是隐藏进程、不占端口，单独查：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" -ErrorAction SilentlyContinue |
  Where-Object { $_.CommandLine -match 'song-worker\.mjs|dsh-inbox-watcher\.mjs|dsh-context-guard\.mjs|douyin-dm-autoreply\.mjs|llm-tuning-proxy\.cjs' } |
  ForEach-Object {
    $m = [regex]::Match($_.CommandLine, '(song-worker|dsh-inbox-watcher|dsh-context-guard|douyin-dm-autoreply)\.mjs|(llm-tuning-proxy)\.cjs')
    [pscustomobject]@{ pid = $_.ProcessId; worker = $m.Value }
  } | Format-Table -AutoSize
```

期望看到三到五行，**缺任何一行都要补**（尤其 watcher 和 guard 挂了是静默失败）。

两个容易漏的：

- `douyin-dm-autoreply.mjs` 存在 `state\douyin-dm-autoreply.disabled` 标记时**故意不跑**（插件自己接私信了），查不到是正常的；删掉标记文件，下一轮 watchdog 就会拉起。
- `llm-tuning-proxy.cjs` 只有配了「本地调参代理」才需要（见 [playbook/10](../../playbook/10-models-and-media.md)）。它挂了不会报错，只是 QQ 的每条消息都在等一个连不上的 8787。

## 启动期三个真实的坑

### 1. 别用 WMI 拉常驻进程

`Invoke-CimMethod -ClassName Win32_Process -MethodName Create` 看起来最"正规"：不弹窗、返回 `ReturnValue: 0`、像成功。但在这台机器上，它拉起来的常驻进程会在**约 10 秒内被静默杀掉**，watchdog 日志里连一行 "started" 都没有。

诊断起来很费时间：ReturnValue=0 让你以为成功了，端口一直不通又让你以为是别的组件坏了。

结论：

- 常驻进程（watchdog、worker、代理）一律用**隐藏窗口的 `Start-Process`** 从 PowerShell 起。
- `-Once` 可以拿来**验证能不能起来**，但它不等于"起来后能活"——真正的判据是隔 1~2 分钟回来看端口/进程还在不在。

```powershell
Start-Process -FilePath 'powershell.exe' `
  -ArgumentList '-NoLogo','-NoProfile','-NonInteractive','-WindowStyle','Hidden','-ExecutionPolicy','Bypass','-File','<你的用户目录>\qq-bridge\autostart\dsh-qq-watchdog.ps1' `
  -WindowStyle Hidden
```

### 2. `restart.bat` 不是可靠的重启入口

桥接目录里那个 `restart.bat` 里有 `pause`。隐藏启动时它停在那里等人按键，于是你以为"重启了"，其实旧进程还在、新进程没起。

可靠的重启顺序：

1. 杀掉命令行含 `bridge.js` 的 node（排除 MCP 子进程）、以及命令行含 `start.bat` 的 cmd（排除 `restart.bat` 自己）；
2. 删掉 `state\bridge.lock`；
3. 隐藏起 `start.bat`（不是 `restart.bat`）。

### 3. DSH 重启会带走整棵进程树

DSH 自己重启时，从 DSH 里拉起来的 watchdog 和 worker 会被一起回收。典型现象：重启脚本日志只写到 "stopping DeepSeek Harness processes" 就断了，后面的步骤全没跑。

两个应对：

- 要跨 DSH 重启存活，得从 `explorer.exe` 起一个 .vbs 之类**脱离作业对象**的方式；即便如此，DSH 进程被杀的那一刻，脚本自己也常被带走，所以别把"重启 + 验收"写在同一个脚本里指望它跑完。
- 更稳的做法：重启后**单独跑一次** self-check（本 skill 的「等待并验证」），别赌脚本跑完了。

## 传话箱（qq_dsh_inbox）

QQ 管理员把任务转给本机 DSH 的通道，链路：

```
QQ 管理员 → qq_dsh_inbox 工具 → POST /api/admin/dsh-inbox
          → 写入 state\dsh-inbox.json（status=pending）
          → autostart\dsh-inbox-watcher.mjs 每 60 秒挑 pending
          → 投给最近活跃的桌面 DSH 会话（sessions.prompt, mode=queue）
          → 标记 delivered
```

查有没有积压：

```powershell
Get-Content '<你的用户目录>\qq-bridge\state\dsh-inbox.json' -Raw -Encoding UTF8 |
  ConvertFrom-Json | Where-Object { $_.status -ne 'delivered' } |
  Select-Object id,status,text | Format-Table -AutoSize -Wrap
```

排障顺序：

1. `state\dsh-inbox-watcher.log` 最后一行时间 —— 停更说明 watcher 挂了（watchdog 会拉起）。
2. 管理员权限：`qq_dsh_inbox` 只允许 `admins` 里的人，群聊必须带 `adminUserId`，否则 403 `只有管理员可以传话给 DSH`。
3. 目标会话：`state\dsh-inbox-target.json` 记录上次投到哪个 sessionId；watcher 会优先用最近有人类 prompt 的会话。
4. 大肥鱼"嘴上答应但不做事"通常是 preset 里没这条协议 —— 让它调工具时要点名"传话箱/DSH"。

## 上下文水位守卫（context guard）

只监控**非 QQ** 的桌面 DSH 会话：读到 `contextPressure.pressureTokens / contextWindow` 超过阈值，
就往那个会话投一条"把结论写成 Mnemon 文档"的指令（QQ 侧有自动轮换，不归它管）。

- 阈值默认 **75%**（留出写文档本身的余量），环境变量 `CONTEXT_GUARD_THRESHOLD`
- 每 3 分钟查一次，`CONTEXT_GUARD_INTERVAL_MS`
- 同一会话 30 分钟冷却，且须比上次提醒时再涨 5% 才会再次提醒，`CONTEXT_GUARD_COOLDOWN_MS`
- 日志 `state\context-guard.log`，已提醒记录 `state\context-guard.json`

只看当前水位、不投递：

```powershell
node "<你的用户目录>\qq-bridge\autostart\dsh-context-guard.mjs" --dry
Get-Content '<你的用户目录>\qq-bridge\state\context-guard.log' -Tail 10 -Encoding UTF8
```

## 直播间只读弹幕（bili-live MCP）

不是后台常驻服务，由 **DSH 自己按需拉起**（stdio MCP），DSH 一关就没了 —— 所以自检放在启动之后、和 `dshReady` 一起看。

| 项 | 位置 |
|---|---|
| 插件 | `<你的用户目录>\.dsh\plugins\dsh-bili-danmaku-lite\` |
| 代码 | `…\python\live_mcp.py`、`…\python\live_chat_worker.py` |
| 家目录（配置/凭据/日志） | `<你的用户目录>\.dsh\bili-live\`（`config.json`、`live-allow.json`、`bili_credential.json`、`logs\plugin.log`） |
| 解释器 | 插件自带的 venv，见 `config.json` 的 `pythonPath` |
| 注册 | 插件自己注册 MCP（`cordis.patch.yml` 里 `serverName: bili-live` 那段旧手工注册已注释掉） |

> 旧路径 `<你的用户目录>\.dsh\bili-duty\` 是插件化之前的位置，还留着历史文件。**新装/排障一律看 `.dsh\bili-live`**；两套同时存在时，MCP 加载的是插件那份。

三个工具：`bili_live_info`（房间信息）、`bili_live_feed`（弹幕窗口）、`bili_live_send`（发弹幕，**QQ 侧不给**）。

不经过 MCP 直接冒烟（最快确认凭据和直播间接口都还活着）：

```powershell
# 先设家目录，否则 live_mcp.py 会去脚本目录找 bili_credential.json
$env:DSH_BILI_LIVE_HOME = '<你的用户目录>\.dsh\bili-live'
$py = (Get-Content "$env:DSH_BILI_LIVE_HOME\config.json" -Raw | ConvertFrom-Json).pythonPath
& $py -c "import asyncio,sys; sys.path.insert(0,r'<你的用户目录>\.dsh\plugins\dsh-bili-danmaku-lite\python'); import live_mcp as m; print(asyncio.run(m.bili_live_info(5370564)))"
```

期望是一段 JSON：`live_status`（0=未开播）、`title`、`online`、`history`。报 `credential` 相关就是 `<你的用户目录>\.dsh\bili-live\bili_credential.json` 里的 cookie 过期了（旧位置 `.dsh\bili-duty\` 下那份已经不用了）。

QQ 侧还要过一道 `<你的用户目录>\qq-bridge\plugins\qq-agent-presets\qq-tool-restrict.mjs` 的白名单：`SAFE_EXACT` 里已有 `bili_live_info` / `bili_live_feed`（精确名，两种注册形态都写了），`bili_live_send` 故意不在。

- 改白名单/改 preset **必须重启 DSH** 才装载；重启后还得有大肥鱼的新会话拿到新 schema，旧会话仍是旧工具清单。
- 没生效时大肥鱼会退化成用传话箱（`类型：看弹幕`），链路照样通，只是慢 60 秒。

## 抖音私信自动回复（独立循环）

`autostart\douyin-dm-autoreply.mjs` 由 watchdog 拉起（不在时）：读私信 → 用专用 DSH 会话生成一句话 → `im-send --mode ui` 无头 Chrome 发出去。只对白名单里的人开（当前 `年银滔`、`月野猫猫卷`），默认间隔 ≥30s，合计日上限 20 条（单人上限 `limits.daily.maxRepliesPerUser`）。

- 状态：`state\douyin-dm-autoreply.json`（每人 `lastMessageText` / `lastCheckAt` / `dayCount`）；回复专用会话记在 `state\douyin-dm-reply-session.json`；日志 `state\douyin-dm-autoreply.log`。
- 调试：`node autostart\douyin-dm-autoreply.mjs --once`（只看不回加 `--dry`，只记基线加 `--baseline`）。
- 停：watchdog 每 5 分钟会把它拉起来，所以「临时停」只能 kill 且 5 分钟内会被重启；**要真停，得改 `autostart\dsh-qq-watchdog.ps1` 里带 `douyin-dm-autoreply.disabled` 判断的那一段**（或直接 touch 那个标记文件，比改进程更省事）。pid 在 `state\douyin-dm-autoreply.pid`。
- 只回「最近新到的一条对方消息」，自己发的不会被当成触发；基线段（启动时那拍）只记录不回。
- 不回时先看日志最后一个 `单拍结果`：`idle` = 没有新消息；`skip_*` = 频率/日上限/白名单。

## 失败时

不要谎报"已启动"。按端口说明是哪一段没起来，并查看：

- `<你的用户目录>\qq-bridge\state\autostart.log`
- `<你的用户目录>\qq-bridge\state\bridge.out.log`
- `<你的用户目录>\qq-bridge\state\dsh-inbox-watcher.log`
- `<你的用户目录>\qq-bridge\state\context-guard.log`
- `<你的用户目录>\qq-bridge\state\song-worker.log`
- `<你的用户目录>\qq-bridge\state\llm-proxy.out.log`（配了调参代理才有；每请求一行 `tuned` / `plain`）

如果只是 DSH 端口变了，watchdog 会自动把桥接 `baseUrl` 更新到当前 DSH 端口；
若没有，检查 `autostart.log` 里的 `DSH port`。

ACE 点歌另有一层：CLI 报 `could not connect to ACE Studio at \\.\pipe\ace_mcp_bridge_... (os error 2)`
就是 **ACE Studio 没开**（`acestudio-cli app launch` 是假启动，要直接起 `ACE Studio.exe`）。

## 不要做

- 不要创建或依赖 Windows 计划任务；用户已经取消 `DSH QQ Autostart`，也明确拒绝开机自启（2026-10-24 又确认过一次，别再建）。
- 不要弹出 SnowLuma / 桥接 / GPT / worker 的可见窗口。
- 不要在端口已全部就绪时重复启动服务。
- 不要用 WMI（`Invoke-CimMethod Win32_Process Create`）拉常驻进程：返回 0 也会被静默回收。
- 不要用 `acestudio-cli app launch` 当作真的把 ACE 开起来。

---

## 出处与署名

这份文档由 **NOYOT** 整理编写，**不是它描述的工具的作者**。

它讲的工具来自：[SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma)、[Derpyu520/qq-bridge](https://github.com/Derpyu520/qq-bridge)。

请去原作者仓库下载、按原作者文档操作；觉得好用就给原作者点个星。
如果本文件里有说错的地方，是整理者的问题，不是原作者的问题。
