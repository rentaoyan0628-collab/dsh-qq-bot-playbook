---
name: start-qq
description: "按用户口令一键启动/自检 QQ 链路：SnowLuma 网关、QQ bridge、GPT-SoVITS 语音、ACE 点歌 worker、DSH 传话箱 watcher 和 DSH 上下文水位守卫，全部后台隐藏运行；并自检直播间只读弹幕工具（bili_live_info / bili_live_feed）与 QQ 侧白名单是否就位。用户说「启动QQ」「开QQ」「拉起QQ」「把QQ服务开起来」或类似话时加载并使用。"
---

# 启动 QQ 链路

## 何时用

用户说「启动QQ」「开QQ」「拉起QQ」「把QQ服务开起来」等，目标是让 QQ 里的
大肥鱼可用，而不是打开某个可见窗口。也适用于排查"QQ 没反应/传话没送到"时先做一遍链路自检。

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
| SnowLuma 网关 | `<SnowLuma 目录>\index.mjs` | 127.0.0.1:3000 / 3001 |
| GPT-SoVITS 语音 | `<GPT-SoVITS 目录>\… -I api_v2.py` | 127.0.0.1:9880 |
| QQ bridge | `<你的用户目录>\qq-bridge\src\bridge.js`（经 `start.bat`，带 `:loop` 自愈） | 127.0.0.1:3100 |
| ACE 点歌 worker | `workers\song-worker.mjs` | `node.exe` 命令行含 `song-worker.mjs` |
| DSH 传话箱 watcher | `autostart\dsh-inbox-watcher.mjs` | 命令行含 `dsh-inbox-watcher.mjs` |
| DSH 上下文水位守卫 | `autostart\dsh-context-guard.mjs` | 命令行含 `dsh-context-guard.mjs` |
| 抖音私信自动回复 | `autostart\douyin-dm-autoreply.mjs` | 命令行含 `douyin-dm-autoreply.mjs` |

watchdog 自带单实例锁（`state\autostart.pid`）；已在跑时再启动会自己退出，不会重复开。
它还会自动把 bridge 的 `baseUrl` 对齐到当前 DSH 端口（DSH 重启后端口会变）。

## 等待并验证

启动后轮询最多 60 秒，直到 `3000/3001/3100/9880` 都监听：

```powershell
$ports = 3000,3001,3100,9880
1..20 | ForEach-Object {
  $down = $ports | Where-Object {
    -not (Test-NetConnection 127.0.0.1 -Port $_ -InformationLevel Quiet -WarningAction SilentlyContinue)
  }
  if (-not $down) { 'ALL_UP'; break }
  Start-Sleep -Seconds 3
}
```

再用控制台令牌确认桥接和 QQ 在线：

```powershell
$token = (Get-Content '<你的用户目录>\qq-bridge\state\console-token' -Raw).Trim()
$h = @{ 'x-console-token' = $token }
Invoke-RestMethod 'http://127.0.0.1:3100/api/status' -Headers $h |
  Select-Object mode,dshReady,ownerQQ,@{n='snowConnected';e={$_.snowluma.connected}},@{n='accountOnline';e={$_.snowluma.accountOnline}}
```

期望 `dshReady=True`、`snowConnected=True`、`accountOnline=True`。

三个后台 worker 是隐藏进程、不占端口，单独查：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" -ErrorAction SilentlyContinue |
  Where-Object { $_.CommandLine -match 'song-worker\.mjs|dsh-inbox-watcher\.mjs|dsh-context-guard\.mjs' } |
  ForEach-Object {
    $m = [regex]::Match($_.CommandLine, '(song-worker|dsh-inbox-watcher|dsh-context-guard)\.mjs')
    [pscustomobject]@{ pid = $_.ProcessId; worker = $m.Value }
  } | Format-Table -AutoSize
```

期望看到三行，**缺任何一行都要补**（尤其 watcher 和 guard 挂了是静默失败）。

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
| 代码 | `<你的用户目录>\.dsh\bili-duty\live_mcp.py` |
| 解释器 | `<你的用户目录>\.dsh\bili-duty\.venv\Scripts\python.exe` |
| 注册 | `<你的用户目录>\.dsh\profiles\desktop\cordis.patch.yml`（`id: mcp-bili-live` → `serverName: bili-live`，stdio） |

三个工具：`bili_live_info`（房间信息）、`bili_live_feed`（弹幕窗口）、`bili_live_send`（发弹幕，**QQ 侧不给**）。

不经过 MCP 直接冒烟（最快确认凭据和直播间接口都还活着）：

```powershell
$py = '<你的用户目录>\.dsh\bili-duty\.venv\Scripts\python.exe'
& $py -c "import asyncio,sys; sys.path.insert(0,r'<你的用户目录>\.dsh\bili-duty'); import live_mcp as m; print(asyncio.run(m.bili_live_info(5370564)))"
```

期望是一段 JSON：`live_status`（0=未开播）、`title`、`online`、`history`。报 `credential` 相关就是 `state\bili-duty-credential.json`（或 `cred.bili`）里的 cookie 过期了。

QQ 侧还要过一道 `<你的用户目录>\qq-bridge\plugins\qq-agent-presets\qq-tool-restrict.mjs` 的白名单：`SAFE_EXACT` 里已有 `bili_live_info` / `bili_live_feed`（精确名，两种注册形态都写了），`bili_live_send` 故意不在。

- 改白名单/改 preset **必须重启 DSH** 才装载；重启后还得有大肥鱼的新会话拿到新 schema，旧会话仍是旧工具清单。
- 没生效时大肥鱼会退化成用传话箱（`类型：看弹幕`），链路照样通，只是慢 60 秒。

## 抖音私信自动回复（独立循环）

`autostart\douyin-dm-autoreply.mjs` 由 watchdog 拉起（不在时）：读私信 → 用专用 DSH 会话生成一句话 → `im-send --mode ui` 无头 Chrome 发出去。只对白名单里的人开（当前 `年银滔`、`月野猫猫卷`），默认间隔 ≥30s，合计日上限 20 条（单人上限 `limits.daily.maxRepliesPerUser`）。

- 状态：`state\douyin-dm-autoreply.json`（每人 `lastMessageText` / `lastCheckAt` / `dayCount`）；回复专用会话记在 `state\douyin-dm-reply-session.json`；日志 `state\douyin-dm-autoreply.log`。
- 调试：`node autostart\douyin-dm-autoreply.mjs --once`（只看不回加 `--dry`，只记基线加 `--baseline`）。
- 停：watchdog 每 5 分钟会把它拉起来，所以「临时停」只能 kill 且 5 分钟内会被重启；**要真停，必须编辑 `autostart\dsh-qq-watchdog.ps1` 第 198–206 行把那段注释掉**。pid 在 `state\douyin-dm-autoreply.pid`。
- 只回「最近新到的一条对方消息」，自己发的不会被当成触发；基线段（启动时那拍）只记录不回。
- 不回时先看日志最后一个 `单拍结果`：`idle` = 没有新消息；`skip_*` = 频率/日上限/白名单。

## 失败时

不要谎报"已启动"。按端口说明是哪一段没起来，并查看：

- `<你的用户目录>\qq-bridge\state\autostart.log`
- `<你的用户目录>\qq-bridge\state\bridge.out.log`
- `<你的用户目录>\qq-bridge\state\dsh-inbox-watcher.log`
- `<你的用户目录>\qq-bridge\state\context-guard.log`
- `<你的用户目录>\qq-bridge\state\song-worker.log`

如果只是 DSH 端口变了，watchdog 会自动把桥接 `baseUrl` 更新到当前 DSH 端口；
若没有，检查 `autostart.log` 里的 `DSH port`。

ACE 点歌另有一层：CLI 报 `could not connect to ACE Studio at \\.\pipe\ace_mcp_bridge_... (os error 2)`
就是 **ACE Studio 没开**（`acestudio-cli app launch` 是假启动，要直接起 `ACE Studio.exe`）。

## 不要做

- 不要创建或依赖 Windows 计划任务；用户已经取消 `DSH QQ Autostart`，也明确拒绝开机自启（2026-10-24 又确认过一次，别再建）。
- 不要弹出 SnowLuma / 桥接 / GPT / worker 的可见窗口。
- 不要在端口已全部就绪时重复启动服务。
- 不要用 `acestudio-cli app launch` 当作真的把 ACE 开起来。
