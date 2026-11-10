# 已有 SnowLuma：迁移到 NapCat，保留 bridge 与记忆

[返回 README](../README.md) · [新安装篇（含 NapCat 网络配置）](napcat-new-install.md)

> 更新：2026-11-10。教程维护者说明：旧教程的 SnowLuma 仓库已私密，普通读者无法再从原入口下载。**已有本地安装不因此自动失效，也不需要强制迁移。** 本文面向主动选择换 NapCat 的用户。
>
> 已核对官方 NapCat 配置与现有 bridge 的静态代码，尚未进行 NapCat 端到端迁移实测。不要把本方案当成“只改端口，所有功能保证通用”的承诺。

## 1. 迁移范围：不重装整套大肥鱼

替换的是 QQ 收发网关，不是 AI、bridge 或记忆系统：

```text
dsh / 模型 / MCP / 人格卡 / 记忆 / qq-bridge 保留
QQ ← SnowLuma → qq-bridge  改为  QQ ← NapCat → qq-bridge
```

| 保留 | 需要修改或重新验证 |
|---|---|
| 原 bridge 安装目录与源码改动 | 网关 HTTP/WS、OneBot token |
| 人格卡、记忆库、会话状态、共享摘要 | 旧 SnowLuma 自动发现、自愈来源 |
| 白名单、管理员、owner、模型配置 | 自动启动 SnowLuma 的守护与控制台按钮 |
| DSH preset/MCP（一般不必重装） | MCP/点歌 worker 等独立进程的旧地址/token |
| 3100 控制台与权限设计 | 图片/语音/表情/拍一拍/资料扩展接口 |

使用原来那个机器人 QQ 才能自然沿用会话目标。更换机器人账号是另一项迁移，要重新确认权限和身份，不能偷偷混在本次操作中。

## 2. 先停自动回复和旧守护，再换网关

### 2.1 先做本机备份

记录 bridge/QQ/SnowLuma/DSH 版本、启动入口、进程命令行、实际监听端口；备份以下内容到不公开的本地目录：

- bridge 的配置、源代码改动、roles、state 和媒体目录，及其外部记忆库路径。
- DSH profile / MCP 连接设置、环境变量来源。
- watchdog、启动脚本、现有计划任务/服务配置；只检查已有项，不新增自启。
- SnowLuma 安装与配置（含凭证，限制访问），保留回滚所需的合法安装包。

备份 state/数据库时先停止相关写入者，或使用其一致性备份方式；不要把正在写的文件随便复制后当作可靠恢复点。不能把包含 token、Cookie、私聊原文的备份推到 GitHub。

### 2.2 确认停止范围

1. 在 bridge 控制台暂停 AI 自动回复和主动发送；记录原开关状态。
2. 停止会重拉 SnowLuma/bridge 的 watchdog、启动批处理循环，以及会回传消息的 worker；按查明的 PID/命令行停止，不按“全部 node/python”批量杀进程。
3. 等在途发送结束。未知投递记为 unknown，迁移后不要补发；不要把老积压闲聊重新灌给 AI。
4. 正常停止 bridge 和 SnowLuma，关闭 SnowLuma 自动加载路径。AI 暂停不等于所有出站入口都停：控制台、MCP、点歌失败通知可能走另一条路。
5. 若当前 QQ 会话曾被旧网关加载，保存工作后按正常流程退出该机器人客户端，再按 NapCat 官方流程重新启动；不要将两套网关加载到同一 QQ 进程。

**先停守护再停组件。** 否则看到进程消失，几秒后又自动回来，会误判迁移成功或产生双回复。

### 2.3 旧 watchdog 是最容易漏掉的地方

仅清空 bridge 配置中的 launcherPath/homeDir 不足以阻止旧脚本启动 SnowLuma。检查实际脚本：

- 监控 5099 后硬编码启动 SnowLuma index.mjs 的分支；删除/禁用这一分支，或改为明确选择的 NapCat 后端。
- start.bat 等退出后循环重拉 bridge 的逻辑；迁移维护窗口先停循环。
- 控制台“启动 SnowLuma”、管理员系统调用、自愈脚本中的硬编码目录。
- 已有开机项、服务、计划任务，只禁用本次相关项并记录回滚方法。

最稳妥的第一阶段是**暂不启用网关自动拉起**：人工启动 NapCat，手动起一个 bridge。文字验收后再单独改守护为 NapCat 当前版本的启动命令与运行目录。

若保留后台守护，检测 OneBot 端口及预期账号，不再用 5099 判断新网关是否运行。NapCat WebUI 与 bridge 3100 是两个服务。不要把旧 SnowLuma 的 node.exe + index.mjs 命令原样换个目录就运行。

## 3. 安装 NapCat 与设置端口

按 [新安装篇第 2、3 节](napcat-new-install.md#2-从哪里下载-napcat) 下载、扫码并创建两个服务：

- HTTP 服务器：127.0.0.1:3000。
- 正向 WebSocket 服务器：127.0.0.1:3001。
- 两者启用，消息格式 array，同一个新生成的强 OneBot token；WS reportSelfMessage=false。
- WebUI 登录 token、OneBot token、bridge 控制台 token、模型 API key 分开管理。

沿用 3000/3001 便于少改上层，前提是旧网关已停止并释放端口。若另选端口，bridge、MCP、worker 中的地址要全部同步；不要边跑两套边抢同一端口。

NapCat 的网络字段为 network/websocketServers/token/messagePostFormat；SnowLuma 的旧配置格式不同。不要导入整个旧 onebot 配置，也不要复制其账号登录数据覆盖 NapCat。用官方登录流程，由本人确认。

## 4. bridge 只改网关段，不清空原配置

将已有配置中的 snowluma 段调整为以下内容（局部示例）：


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

字段仍叫 snowluma 是 bridge 的历史兼容命名，不是告诉它继续启动旧网关。不要自行改为 napcat，否则现有代码可能读不到。

另外必须检查：

1. 清除旧的 SNOWLUMA_HOME；SNOWLUMA_HTTP_URL / SNOWLUMA_TOKEN 设为新值或移除过期覆盖。检查 shell、启动脚本、MCP env、系统/用户环境及 worker 的配置来源；不要公开值。
2. homeDir/launcherPath 清空，不能指向 NapCat 目录让旧 SnowLuma 解析器读配置。
3. followDiscoveredEndpoint=false 防止发现旧地址后切回；**不代表彻底禁用发现/自愈**，旧配置文件仍可能参与 token 更新，必须消除旧来源。
4. 保留 dsh、ownerQQ、admins、allow、socialV2、人格卡、记忆及会话设置，不用模板覆盖整份文件。
5. 正常重启所有实际读取过旧配置的进程；已运行进程不会自动继承后来修改的环境。

当前 bridge 依赖 @snowluma/sdk：不要删包、改 package.json 或另装第二套 bridge。先测标准 OneBot 接口；若某扩展确有不兼容，记录动作和返回值，最小修改或暂时关闭该能力。本文不附未经验证的“通用兼容补丁”。

## 5. MCP 和 worker：地址一样也不能跳过

逐个核对这些入口的实际来源，而不只检查 bridge 主配置：

- QQ MCP / SnowLuma 状态探活工具：是否读取旧配置、环境变量或旧网关目录。
- 点歌 worker、失败通知、语音回传：是否直接 HTTP 访问旧 token；生成音频成功不等于回传成功。
- 管理员代发、文件/图片发送、资料与空间动态：是否走旧扩展 API。
- 自动端点发现和 token 自愈脚本：是否会覆盖已经手工写好的 NapCat token。

代码中检索 snowluma、SNOWLUMA_、3000、3001、5099、onebot_、get_record 等作为排查线索，再读上下文，不做全局字符串替换。3100、模型代理、B站/抖音插件与 QQ 网关不是同一层，不要连带重装。

媒体兼容的具体边界：
- 某些二开 bridge 的语音转写依赖 get_record 返回 data.base64；NapCat 若只返回路径/URL，需要安全下载或读取的适配，不能宣称已兼容。
- 收藏表情可能调用 fetch_custom_face_detail 等扩展；动作名、参数、数组返回格式都要核对。
- 本机图片/音频路径必须对 NapCat 进程可见；Docker/远端路径不是 Windows 本机路径。
- 历史消息 ID、引用和合并转发资源不保证跨网关可读；本地记忆保留不等于新网关能查询旧网关的每条消息。

## 6. 验收与恢复自动回复

按 [新安装篇验收](napcat-new-install.md#5-分层验收http-通不等于能聊天) 从只读检查开始。不要以“端口通了”宣布迁移完成。

| 验收项 | 通过标准 |
|---|---|
| 账号 | get_login_info 返回预期机器人 QQ |
| HTTP / WS | 正向连接稳定，无旧 token 自愈循环 |
| 入站 | 一个白名单测试好友的新消息能进入 bridge |
| 出站 | 经授权一条文字准确发到同一测试会话 |
| 连续问答 | 至少两轮，不重复、不自己回自己 |
| 权限 | 白名单外不回复，管理员限制未放宽 |
| 数据 | 原人格、记忆和会话配置仍在；旧引用另验 |
| 媒体 | 图片、语音、引用、转发、表情、拍一拍按需逐项记录 |
| 守护 | 只启目标后端，不重拉 SnowLuma、不起第二个 bridge |

先在单个测试私聊中验收文字，再恢复原来批准过的群聊范围。未测/不兼容的媒体工具暂时关闭。恢复自动回复后观察实际新消息，不批量重放积压。

没有账号/验证码问题时，可做一次受控 bridge 重启，核对端点/token 不回滚、数据仍在；之后再验守护。出现验证码、账号限制或不明重复发送就停，不通过反复重启/换后端绕过。

## 7. 失败时怎样回滚

1. 暂停并停止新链路自动发送，停新守护、bridge 和 NapCat；不补发 unknown 项。
2. 正常退出相关机器人 QQ 会话，避免两套加载同时存在。
3. 恢复迁移前 bridge 网关配置、环境和旧守护设置；保护迁移后新产生的数据，不用旧 state 全量覆盖新 state。
4. 仅在旧 QQ/SnowLuma 版本仍兼容且账号状态正常时启动旧链路；先只读确认账号，再经授权测试。
5. 账号急停/安全限制不能因恢复备份而被清掉。旧后端也不可用时，停止自动化，改人工收发，不无限切换。

不删除旧安装，直到新方案的所需能力验收完成并获得本人确认。仓库私密不等于已安装程序立即不能运行；迁移是可选项，不是强制重装。

## 8. 给 AI 的复制模板


```text
请读取本仓库 README.md、docs/napcat-new-install.md 和 docs/napcat-migration.md。
我选择把现有 SnowLuma 网关迁移为 NapCat，保留 qq-bridge、DSH、人格卡、记忆和白名单。
先只读检查并备份，列明要停的进程/守护及预计影响，确认后再操作。
不要改成 dsh-channel-qq，不清空 state，不把 snowluma 配置字段擅自改名。
清查旧网关发现、自愈、环境变量、MCP、worker 和自动启动入口。
先单个私聊验收文字，媒体逐项测试；需要我扫码和批准测试目标时问我。
未经验证的写未验证，失败保留回滚，不默认新增自启或放开公网。
```

## 9. 来源

NapCat 的官方仓库、安装与网络 schema，以及 bridge 配置和发现逻辑来源见 [新安装篇来源](napcat-new-install.md#8-来源与维护范围)。本篇只给迁移方案；没有安装 NapCat、修改维护者现有 QQ 运行链路或发送真实消息作为本次文档发布的一部分。
