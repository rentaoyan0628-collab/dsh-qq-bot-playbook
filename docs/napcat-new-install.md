# 新安装：用 NapCat 替代教程中的 SnowLuma

[返回 README](../README.md) · [已有 SnowLuma？看迁移篇](napcat-migration.md)

> 更新：2026-11-10。教程中的 SnowLuma 仓库目前已转为私密，普通读者无法按旧链接获取；新安装可以改用 NapCatQQ 提供 OneBot v11 接口。GitHub 未授权访问的 404 无法单独区分私密/删除，此处按教程维护者提供的情况说明，不推断原因，也不提供非官方搬运包。
>
> 本文已核对 NapCat 官方文档、配置源码及 qq-bridge 的连接配置；**未在本教程维护者电脑上安装 NapCat，也未完成 NapCat → qq-bridge → DSH 的端到端实测**。这是接入配置方案，不是全功能兼容认证；请按文末验收后使用。

## 1. 什么变、什么基本不变

旧视频：QQ → SnowLuma → qq-bridge → DSH。
现在：QQ → **NapCat（OneBot v11）** → **同一个 qq-bridge** → **同一个 DSH**。

**只替换最前面的网关。后面的桥接安装、DSH preset/MCP、白名单、管理员、人格卡、记忆和模型配置基本沿用原教程。** 不需要为此换成 dsh-channel-qq，也不是再装一套 AI。

| 项目 | 如何处理 |
|---|---|
| 网关安装、QQ 启动、管理页 | 改按 NapCat 官方步骤；不照抄 SnowLuma 的 5099 管理页 |
| OneBot HTTP / 正向 WS | 本文约定本机 3000 / 3001，两边明确配置，不是自动开好 |
| qq-bridge 配置字段名 | 仍是历史名称 `snowluma`（兼容字段，不改名） |
| 3100 控制台、会话权限 | 属于 qq-bridge，不属于 SnowLuma；基本沿用 |
| DSH 模型、人格、记忆 | 沿用；换网关不会自动增加视频中的定制功能 |
| 网关自动发现、自愈、守护 | 要改，不能继续读取/拉起 SnowLuma |
| 图片、语音、表情收藏、拍一拍等扩展 | 逐项验收；同为 OneBot v11 不保证扩展实现一致 |

## 2. 从哪里下载 NapCat

只从原作者渠道获取：
- [NapCatQQ 项目](https://github.com/NapNeko/NapCatQQ)
- [NapCatQQ Releases](https://github.com/NapNeko/NapCatQQ/releases)
- [官方 Shell 安装文档](https://doc.napneko.icu/guide/boot/Shell)

本文以本地 Windows x64 + Shell 为例，不混用 Docker、Framework、桌面管理器的路径。

1. 按当前官方发布说明选择适配的 QQ / NapCat 版本，不照抄旧截图里的最低 QQ 版本。
2. 已有官方 QQ 时，可按手动安装说明下载并解压 `NapCat.Shell.zip`，按该版本说明运行 `launcher.bat`（Windows 10 分支以官方要求为准）。
3. 想用一键包，按官方说明获取 `NapCat.Shell.Windows.OneKey.zip`，运行 `NapCatInstaller.exe`；安装后从生成目录按说明运行 `napcat.bat`。包名随版本变化，以当前 Release 为准，不混两套启动器。
4. 由本人完成扫码/登录，确认登录的是专用机器人 QQ，不是主人日常主号。
5. 按启动日志打开 WebUI（常见为 `http://127.0.0.1:6099/webui`，**以日志为准**）。管理页 token 取本机日志/配置，不假定固定密码，更不能当 OneBot token。

先完成授权和来源核验再运行安装器。不默认开放公网、不加开机自启。NapCat 同样是非官方 QQ 接入，不保证免风控或永久兼容。

## 3. NapCat 网络配置：两个服务，别选反了

在当前账号的 WebUI 网络配置中新建并启用：

| 设置 | HTTP 服务器 | 正向 WebSocket 服务器 |
|---|---|---|
| 地址 | 127.0.0.1 | 127.0.0.1 |
| 端口 | 3000 | 3001 |
| Token | 同一个你本机生成的长随机串 | 与 HTTP 完全相同 |
| 消息格式 | array | array |
| 上报自身消息 | 不额外开启 | false |

**这里是 WebSocket 服务器（正向 WS），不是 WebSocket 客户端/反向 WS。** bridge 主动连接 NapCat；不填 DSH 地址或 3100 作为反向 WS 地址。

HTTP/WS 使用同一个 token 是为了适配当前 bridge 只有一个 `accessToken` 的配置习惯；不是协议规定必须相同。保持两个端点都鉴权，禁止空 token。

推荐用 WebUI 保存。下面是按官方源码字段核对的**network 局部配置示例**；不要用它覆盖整个配置文件，不要将 SnowLuma JSON 直接复制进 NapCat：

```json
{
  "network": {
    "httpServers": [{
      "name": "bridge-http", "enable": true,
      "host": "127.0.0.1", "port": 3000,
      "enableCors": false, "enableWebsocket": false,
      "messagePostFormat": "array", "token": "REPLACE_WITH_LOCAL_RANDOM_TOKEN", "debug": false
    }],
    "websocketServers": [{
      "name": "bridge-ws", "enable": true,
      "host": "127.0.0.1", "port": 3001,
      "messagePostFormat": "array", "reportSelfMessage": false,
      "token": "REPLACE_WITH_LOCAL_RANDOM_TOKEN", "heartInterval": 30000,
      "enableForcePushEvent": true, "debug": false
    }],
    "httpClients": [], "websocketClients": []
  }
}
```

注意：NapCat 是 `network` / websocketServers / token / messagePostFormat；旧 SnowLuma 示例是 `networks` / wsServers / accessToken / messageFormat。字段不同，不能互换。常见 NapCat 文件名为 `onebot11_<QQ号>.json`，位置随安装方式变化，请从 WebUI/日志确认；修改后保存，按当前版本要求重载网络或重启 NapCat。

## 4. 安装并配置同一套 qq-bridge

按 [桥接安装 skill](../skills/qq-bridge-setup/SKILL.md) 或 [完整版 skill](../skills/qq-bridge-full/SKILL.md) 安装 **同一个** [Derpyu520/qq-bridge](https://github.com/Derpyu520/qq-bridge)。两份 skill 是说明的详略区别，不是两个程序，不要重复安装。Node 要求以当前桥接项目为准。

修改自己的 bridge 配置中已有的 `snowluma` 段，不要将字段名改成 napcat：

```json
{
  "snowluma": {
    "httpUrl": "http://127.0.0.1:3000",
    "wsUrl": "ws://127.0.0.1:3001",
    "accessToken": "REPLACE_WITH_THE_SAME_ONEBOT_TOKEN",
    "followDiscoveredEndpoint": false,
    "launcherPath": "",
    "homeDir": ""
  }
}
```

这也是局部示例，保留其余配置。桥接使用了 `@snowluma/sdk`，不要因包名含 SnowLuma 就删除依赖；它能否覆盖某项 NapCat 扩展，应以实测和当前源码为准。

- 清除**启动 bridge / MCP / worker 的环境**中旧的 `SNOWLUMA_HOME`；若有 `SNOWLUMA_HTTP_URL`、`SNOWLUMA_TOKEN`，同步为新端点/token 或移除过期值。只改一个终端不会改变已运行进程环境，要正常重启对应进程。
- `followDiscoveredEndpoint:false` 仅限制端点跟随，不等于关闭全部 token 自愈。旧 homeDir、launcherPath、环境变量仍可能让程序读取 SnowLuma 旧凭证，必须一并检查。
- 不要把 NapCat 目录填到 SnowLuma 的 homeDir/launcherPath：旧解析器不认识 NapCat 配置，也不能假定旧启动器支持 NapCat。
- 不改 `ownerQQ`、`admins`、`allow.private`、`allow.groups` 的语义；按自己目标填，保留 `allowAllWhenEmpty:false`。
- `dsh.baseUrl` 填你当前 DSH 地址，模型填已注册项。不要抄作者机器端口/QQ 号。
- 按原教程运行安装 DSH preset/MCP 的步骤，需要时重启 DSH。只有网关端点变化时，不必为了迁移重复安装 DSH 或清空会话。

若有 SnowLuma 专用 watcher/自动拉起逻辑，先按 [迁移篇第 2 节](napcat-migration.md#2-先停自动回复和旧守护再换网关) 处理。最小验收期间先手动启动 NapCat 和一个 bridge，暂不接入旧 watchdog。

## 5. 分层验收：HTTP 通不等于能聊天

### 5.1 只读核验账号

PowerShell 中执行（token 只在自己电脑输入，不贴截图/日志）：

```powershell
$token = Read-Host '输入本机 OneBot token（不要公开）'
$r = Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:3000/get_login_info' -Headers @{ Authorization = "Bearer $token" } -ContentType 'application/json' -Body '{}'
$r | ConvertTo-Json -Depth 5
```

应为 `status=ok`、`retcode=0`，且 `data.user_id` 是预期机器人号。不是预期号立即停，不通过改白名单来迁就错账号。

### 5.2 验证入站与实际回复

1. 启动一个 bridge，查看其实际日志及 3100 控制台；界面中的 SnowLuma 字样可能仍是旧标签，不代表当前真在用旧网关。
2. 检查 WS 已连接，再由一个白名单测试好友发一条唯一文字；日志必须能看到对应入站事件。心跳只能证明连接活着，不能替代消息事件验收。
3. 经本人同意，仅向这个测试会话回复一条；确认工具返回、聊天窗口及会话目标一致。
4. 再做一轮问答，确认没有重复回复、没回自己发的消息、没有第二个自动响应入口。
5. 白名单外不回复；遇到验证码、安全限制、重复发送时先停自动化，不切后端硬发。

验收记录写清 NapCat/QQ/bridge/DSH 版本、测试时间、项目与结果，不含 token。没有测试图片/语音就写“未验”，不能写全功能通过。

## 6. 什么继续读旧教程，什么不要照搬

继续读：DSH 设置、bridge 的白名单和管理员、3100 控制台、人格卡、记忆插件、传话箱、模型与媒体设计。它们的配置逻辑基本不变。

不要照搬：SnowLuma 安装包、5099 WebUI、配置文件格式、自动发现、token 自愈来源、自动拉起脚本、SnowLuma 专用 API。图片下载、合并转发、语音转码、收藏表情、改资料等逐项验收，不支持时关闭该工具，别让 AI 假装成功。

媒体特别注意：同机文件路径才可能共享；若改成 Docker/远端 NapCat，bridge 的本机音频/图片路径不会自动在另一环境存在，需要单独设计挂载或上传，本篇不覆盖。

## 7. 常见错误

| 现象 | 先处理 |
|---|---|
| 3000/3001 拒绝连接 | HTTP / WS 是否分别启用，是否已保存、端口被占、登录是否完成 |
| HTTP 401 或 WS 连不上 | OneBot token 是否一致，是否误用 WebUI/3100 token，是否被旧环境变量覆盖 |
| HTTP 正常但没事件 | 是否选成反向 WS、array 是否正确、自身消息是否造成回环 |
| 地址/token 又被改回去 | 旧 homeDir、launcherPath、SNOWLUMA_*、端点跟随、旧守护是否仍在读 SnowLuma |
| 端口冲突或回复两遍 | 旧 SnowLuma/旧 bridge/其他 bot 是否仍在运行 |
| 能收不能回 | HTTP token、白名单、工具权限与 DSH 模型是否正常；不是一律重装 |
| 能聊但某工具报错 | 查具体 OneBot 动作与参数差异，禁用未兼容扩展 |

## 8. 来源与维护范围

- [NapCat 官方安装说明](https://doc.napneko.icu/guide/boot/Shell)
- [NapCat OneBot 网络配置源码](https://github.com/NapNeko/NapCatQQ/blob/main/packages/napcat-onebot/config/config.ts)
- [qq-bridge 配置模板](https://github.com/Derpyu520/qq-bridge/blob/main/config.example.json)
- [qq-bridge 端点发现实现](https://github.com/Derpyu520/qq-bridge/blob/main/src/snowluma-conn.js)

上游 main 和安装包会更新，字段以你安装版本为准。本仓库只维护教程，不是 NapCat、SnowLuma 或 qq-bridge 的作者，也不分发这些项目的程序。
