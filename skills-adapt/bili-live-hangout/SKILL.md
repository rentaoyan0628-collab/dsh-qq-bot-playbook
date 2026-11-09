---
name: bili-live-hangout
description: "去 B 站直播间「挂着陪聊」：把昵称/UID/房间号解析成直播间、经管理员放行后进房、收实时弹幕、判断该不该接话并自动发弹幕，直播结束或到点后道谢退场。用户说「去某某的直播间发弹幕/陪聊」「挂直播间」「看看这房间里在聊什么」「直播间测试」「到点收尾」「感谢主播」时加载。核心纪律：没管理员放行不进房；宁可少说别硬找话；中文绝不走 pwsh 命令行（cp936 会乱码）。"
---

# B 站直播间陪聊：插件把耳朵和手都包了

2026-10-31 起，这件事由插件 **dsh-bili-danmaku-lite** 独立完成，不再是「一个 python worker + 一条 DSH 定时提醒」的两半结构：

| 以前 | 现在 |
|---|---|
| worker 收弹幕 + 写队列；**DSH 会话里的定时提醒**负责判断和写话 | 插件自己每 `tickSec`（默认 20 秒）看一次增量弹幕，自己判断，自己发 |
| 一次判断 = 一个完整模型回合 + 一条提醒 | 判断在插件内联完成，不占 DSH 会话 |
| 定时提醒没人删就一直烧钱 | 只有插件的 `setInterval`，没有提醒要清 |

所以「脑子」不再是 DSH 会话里的提醒了。**只有在插件不可用、或你想手动接一句的时候**，才回到老的 worker + queue 手动路径。

关键位置：

```text
插件代码   <你的用户目录>\.dsh\plugins\dsh-bili-danmaku-lite\
                index.js                     # autoChat 判断/发送/setInterval
                python\live_mcp.py           # MCP 工具 + 房间解析 + 放行名单
                python\live_chat_worker.py   # 收弹幕/发队列/叫醒（手动拉起时才用）
插件家目录 <你的用户目录>\.dsh\bili-live\
                config.json                  # autoChat 参数
                live-allow.json              # 放行名单（唯一入口）
                bili_credential.json         # B 站 cookie
                logs\plugin.log               # 插件日志（判断/排句/SKIP 都在这）
                live-chat\<room>\             # 每个房间：danmaku.jsonl / queue.jsonl / worker.log
```

> 旧路径 `\.dsh\bili-duty\` 是插件化之前的位置。看到两套都在，以 `.dsh\bili-live` 为准。

---

## 0. 先拿到房间号（最容易错的一步）

**直播间号 ≠ 主播 UID。** 用户常给 UID 或昵称。

```bash
# 昵称 → UID（带 Cookie 才可用）
curl -s "https://api.bilibili.com/x/web-interface/search/type?search_type=bili_user&keyword=<昵称>" \
  -H "Cookie: $(cat <你的用户目录>/.dsh/bili-live/bili_credential.json 2>/dev/null)" | head -c 2000

# UID → 房间号 + 开播状态（不需要 Cookie）
curl -s "https://api.live.bilibili.com/room/v1/Room/getRoomInfoOld?mid=<UID>"
#   → data.roomid / data.title / data.liveStatus(1=在播) / data.online
```

判人：看粉丝数、签名里的「直播时间」、以及是不是同名小号。重名时把候选念给用户确认，别猜。

拿到 `roomid` 后确认 `liveStatus == 1`，再往下走。

### ⚠️ 10 位房间号会被当成 UID

`_resolve_room()` 以前的规则是「纯数字 ≥9 位 → UID」。B 站现在有 10 位房间号（例如 `1796291118`），于是它跑去按 UID 查，报：

```text
RuntimeError: UID 1796291118 没有开通直播间（code=0）
```

**这个错只在「发」的时候出现**：收弹幕走的是房间号那条路，正常；发弹幕走 `_resolve_room`，全失败。表现就是「直播间看着在正常挂机，一条弹幕都没发出去」。

修法（`live_mcp.py`）：9 位以上先按房间号验一次，真的存在就用它，验不到才当 UID。

```python
if s.isdigit():
    rid = int(s)
    if len(s) <= 8:
        return rid
    if _room_exists(rid):      # 先验房间
        return rid
    return _roomid_of_uid(rid) # 再退回 UID
```

```python
def _room_exists(rid: int) -> bool:
    url = f"https://api.live.bilibili.com/room/v1/Room/get_info?room_id={int(rid)}"
    ...
    return bool(data.get("code") == 0 and (data.get("data") or {}).get("room_id"))
```

坑中坑：**函数的 docstring 还写着旧规则**（`纯数字 ≥9 位 → 当 UID`）。改代码时别只看 docstring；两处要一起改，否则下一个人照着注释又改回去。这个文件是 **CRLF 行尾**，用脚本替换时记得带 `\r\n`。

## 1. 硬闸：没放行不进房

`<你的用户目录>\.dsh\bili-live\live-allow.json`（`enforce: true`）是唯一入口名单，`bili_live_info` / `bili_live_feed` / `bili_live_send` / `bili_live_watch_start` 都受它约束。

- 只有管理员能改名单；QQ 会话里的模型开不了新房。
- 加房：`bili_live_allow add <roomid> anchor <主播名>`。
- 键是**房间号**。写 UID 进去、拿房间号来比对，永远 `not_allowed`。

### ⚠️ 放行条目会自己消失

`_ensure_allowed()` 对带 `until_offline: true` 的条目，会在 `_live_status(room_id) != 1` 时把条目 pop 掉并回写文件：

```python
if isinstance(entry, dict) and entry.get("until_offline"):
    if await _live_status(room_id) != 1:
        rooms.pop(key, None)
        dropped = True
```

于是「主播刚好没播」→ 你查一次房间 → 放行被删了。表现是「明明放行过，怎么又要放行」。

应对：

- 要长期挂的房间，加放行**别带 `until_offline`**；
- 只用这一场的，接受它会自动收回，下次开播前重新 add；
- 加完 `cat live-allow.json` 复查一遍，别假设写成功了。

## 2. 发弹幕

```bash
cd <你的用户目录>/.dsh/plugins/dsh-bili-danmaku-lite/python
export DSH_BILI_LIVE_HOME=<你的用户目录>/.dsh/bili-live
PYTHONIOENCODING=utf-8 <venv python> -c "
import asyncio, live_mcp
print(asyncio.run(live_mcp.bili_live_send(32438129, '路过测试一下弹幕，主播好呀', confirm=True)))
"
```

- **绝对不要**把中文当 pwsh/native 参数传（控制台 cp936 → 乱码）。要么 `PYTHONIOENCODING=utf-8` + 脚本内联，要么写成 UTF-8 文件再读。
- 成功看 `sent_from_me: true` + `id_str`；再回读 `_history` 确认没乱码、时间对。
- 弹幕 ≤40 字、不能带链接、**发出去删不掉**、同房间受冷却。
- 走 MCP 的 `bili_live_send` 默认**只出草稿**，要 `confirm=true` 才真发。

## 3. 冷却

```python
MIN_INTERVAL_SEC = 60       # 同一房间 60 秒一条
```

- 护栏 `if hits >= 3 and remaining > MIN_INTERVAL_SEC:` —— 按 60 秒写，冷却 ≤60 时照常等，不会误杀。
- **物理下限 ≈ 33–38 秒**：worker 1 秒轮询 + `cooldown_remaining + 0.2`。想真按 60 秒走，别用 5 秒轮询。
- 改冷却要同时改：`live_mcp.py` 常量、文件头文档串、worker 文件头注释。改完 `grep -n "120 秒"` 确认没残留。

## 4. 自动陪聊（插件，默认这条）

配置在 `<你的用户目录>\.dsh\bili-live\config.json` 的 `autoChat`：

```json
"autoChat": {
  "enabled": true,
  "provider": "deepseek-official",
  "model": "deepseek-flash",
  "persona": "你是直播间里的一个老观众，嘴快、爱接梗、说话短，看到有意思的就顺口接一句，不当客服不复读。",
  "tickSec": 20,
  "windowSec": 150,
  "minLines": 2,
  "minGapSec": 60,
  "perHour": 10,
  "maxChars": 25
}
```

| 字段 | 作用 |
|---|---|
| `tickSec` | 每多少秒看一次增量（默认 20） |
| `windowSec` | 只看最近多少秒的弹幕（默认 150） |
| `minLines` | 至少几条实质弹幕才值得判断 |
| `minGapSec` | 两次发言最小间隔（下限被 clamp 到 60） |
| `perHour` | 每小时上限（1–60） |
| `maxChars` | 单条字数上限（10–40） |
| `persona` | 接话口吻，直接进 system prompt |

**不热加载。** `loadCfg()` 只在插件启动时读一次，改完 `config.json` 必须**重启 DSH**。改之前先备份。

日志怎么看：`logs\plugin.log` 每 20 秒一条，正常长这样 ——

```text
自动陪聊：模型决定 SKIP（房间 23941953）
自动陪聊：排队 1 条 → 那本时尚女人杂志八成是伏笔
```

长时间只有 SKIP 不一定坏了：可能真没人说话，也可能 `persona` 太拘谨。先看 `danmaku.jsonl` 有没有量，再调 persona。

## 5. 叫醒 QQ 会话（可选，`--webhook`）

插件自己会发弹幕，但「直播间有人说话 → 把 QQ 会话里的模型叫醒」是另一条路，走 worker 的 POST。它曾经长期 **401**：

- worker 是独立 python 进程，不持控制台令牌；`/api/socialV2/wake` 走控制台鉴权 → 401；
- 带 agent token 又被 `adminOnlyV2Paths` 拒 → 403。

修法是给这条通道一个专用头，而不是放宽整个鉴权：

```js
// bridge.js：带这个头就不要求控制台令牌
const liveWakeReq = req.method === 'POST'
  && url.pathname === '/api/socialV2/wake'
  && String(req.headers['x-live-wake'] ?? '') === '1';
if (consoleToken && !tokenMatches && !liveWakeReq) { /* 401 */ }
```

wake handler 里的 live 分支**目标固定** `private:${cfg.ownerQQ}`，不接受调用方指定 `key`（否则等于给了任意会话的唤醒权）。

worker 侧：

```python
# live_chat_worker.py 的 _post()
headers = {..., "x-live-wake": "1"}
```

worker 参数（`live_chat_worker.py`）：

| 参数 | 默认 | 说明 |
|---|---|---|
| `--room` | — | 房间号 |
| `--mid` | — | 主播 UID |
| `--dir` | — | 状态目录（`live-chat\<room>`） |
| `--name` | 空 | 自己叫什么，判「点名」用；空则读 `DSH_BILI_LIVE_NAME` |
| `--webhook` | 空 | 够得着时 POST 到这里；**空 = 只写 `hint.json`，不打扰任何人** |
| `--wake-cooldown` | 90 | 两次叫醒最小间隔（被 @ 时不受限） |
| `--judge-sec` | 60 | 多久看一次增量 |
| `--max-wakes-per-hour` | 30 | 每小时上限，防刷爆 token |

> 老文档里写的 `--wake-key` 已经不存在了，别照抄。

### ⚠️ 手动拉起 worker 必须设家目录

```powershell
$env:DSH_BILI_LIVE_HOME = '<你的用户目录>\.dsh\bili-live'
```

不设的话，`live_mcp.py` 默认把脚本目录当家目录，第一行 `CRED_FILE.stat()` 就抛：

```text
FileNotFoundError: ...\plugins\dsh-bili-danmaku-lite\python\bili_credential.json
```

还有两件容易误判的事：

- **一套 worker = 两个 pid**：venv 的 python 会再 spawn 一个 uv-python 子进程（同参数）。杀掉父进程后子进程可能还在，`job_kill`/`Stop-Process` 之后要回查。
- 同一个房间**只能有一套**。两套同时跑会双发弹幕；判断依据是命令行里 `--room <id>` 出现了几次，不是看日志。

## 6. 自动判断「挂住」了怎么发现（真实事故）

`index.js` 的 `autoTick()` 里：

```js
await askLlm(llm, sys, user)
```

**没有超时。** 模型请求一挂住，`autoState.busy` 就永远是 `true`，而 `setInterval` 开头是 `if (autoState.busy) return` —— 之后每 20 秒的 tick 全部直接返回，**连一行日志都不写**。

表现：插件「还活着」（进程在、端口在），但自动陪聊静默 60 分钟，看起来像模型不接话。换房间也没用，因为和房间无关。

修法（三件套）：

```js
function withTimeout(p, ms, label) {
  let t
  const timeout = new Promise((_, rej) => {
    t = setTimeout(() => rej(new Error(`${label} 超时 ${ms}ms`)), ms)
  })
  return Promise.race([p, timeout]).finally(() => clearTimeout(t))
}
// 调用处
const r = await withTimeout(askLlm(llm, sys, user), 60000, '自动陪聊模型')
```

```js
// autoState 加 busySince；interval 开头兜底解锁
if (autoState.busy && Date.now() - autoState.busySince >= 120000) {
  log(`自动陪聊：上一轮已卡住 ${...}s，强制解锁重试`)
  autoState.busy = false
}
autoState.busy = true
autoState.busySince = Date.now()
// finally
autoState.busy = false; autoState.busySince = 0
```

排查口诀：**日志停更 ≠ 进程死了**。日志停更 + 进程在 = 某个 await 卡住了，去找没设超时的那一个。

## 7. 退场纪律

- 到点/结束：先发一句道谢弹幕（「谢谢主播今晚陪着聊天，听得很舒服，我先撤啦，晚安」）→ 回读 `sent.jsonl` / `danmaku.jsonl` 确认 → `bili_live_watch_stop` 或 `echo stop > live-chat/<room>/stop.flag` → 确认进程退出 → 汇报。
- **没有管理员放行，不得再进这个直播间。**
- 用一次性提醒（`after_seconds`）钉住收尾时间，别靠"我等会儿记得"。

## 8. 踩过的坑

| 症状 | 真因 | 处理 |
|---|---|---|
| 弹幕发出去变问号/乱码 | 中文走了 pwsh 命令行（cp936） | `PYTHONIOENCODING=utf-8` + 内联脚本或 UTF-8 文件 |
| 加好的房间突然不见了 | `until_offline` 条目在未开播时被 `_ensure_allowed` 自动 pop | 长期房别带 `until_offline`；加完 `cat` 复查 |
| 收得到弹幕，一条也发不出去 | 10 位房间号被判成 UID | 改 `_resolve_room`：先验房间再退回 UID |
| 报 `UID … 没有开通直播间（code=0）` | 同上 | 同上 |
| 手动起 worker 立刻 `FileNotFoundError: …bili_credential.json` | 没设 `DSH_BILI_LIVE_HOME` | 先 export 家目录 |
| 同一句话发了两遍 | 两套 worker 同时挂在同一房间 | 按命令行 `--room` 数个数，只留一套 |
| 自动陪聊静默几十分钟 | `askLlm` 无超时 → `busy` 永久 true | `withTimeout` + `busySince` 强制解锁 |
| 叫醒一直 401 | `/api/socialV2/wake` 走控制台鉴权 | 加 `x-live-wake: 1` 专用头 + bridge 放行分支 |
| 觉得"读不到弹幕" | 房间本身话少 | 用 `_history()` 对账；一致就是没人说话 |
| 接话显得尬 | 没做"够不够聊"判断 | 宁可少说，见下方判据 |

**够不够聊的判据**（插件里就是这个思路）：

- 只有【增量里 ≥1 条实质内容（提问/感慨/吐槽/自述/接话，不含表情、刷屏、点歌系统话术）】**且**【最近一条弹幕 ≤3 分钟】才考虑发。
- 增量 0 条、或最新弹幕已经超过 3 分钟没人说话 → 本轮一律不发。
- 拿不准就不发。宁可少说，也别为了不冷场硬找话。

## 9. 最小剧本（插件版）

```text
1) 找房：昵称/UID → 房间号，确认 liveStatus == 1
2) 放行：bili_live_allow add <room> anchor <主播名>  →  cat live-allow.json
3) config.json 的 autoChat：enabled / perHour / minGapSec / persona
4) 重启 DSH（插件不热加载）→ 看 logs\plugin.log 有没有「插件启动」
5) 盯一轮：日志出 SKIP 或排句 = 活着；只有 SKIP 很久 = 看 persona 或 danmaku.jsonl 有没有量
6) 到点：道谢 → stop.flag / bili_live_watch_stop → 汇报
```

---

## 出处与署名

这份文档由 **NOYOT** 整理编写，**不是它描述的工具的作者**。

它讲的工具来自：[c1overzz/dsh-bili-scroll](https://github.com/c1overzz/dsh-bili-scroll)（B 站接口）；直播间弹幕与自动陪聊是本仓库作者自己写的插件 [dsh-bili-danmaku-lite](https://github.com/rentaoyan0628-collab/dsh-bili-danmaku-lite)。

请去原作者仓库下载、按原作者文档操作；觉得好用就给原作者点个星。
如果本文件里有说错的地方，是整理者的问题，不是原作者的问题。
