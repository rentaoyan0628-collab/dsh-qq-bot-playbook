---
name: bili-live-hangout
description: "去 B 站直播间「挂着陪聊」：找到某人的直播间（昵称/UID→房间号）、经管理员放行后发弹幕、开一个后台 worker 收弹幕 + 定时轮判断该不该接话、直播结束或到点后道谢退场并清理提醒。用户说「去某某的直播间发弹幕/陪聊」「挂直播间」「看看这房间里在聊什么」「直播间测试」「到点收尾」「感谢主播」时加载。核心纪律：没管理员放行不进房；宁可少说别硬找话；中文绝不走 pwsh 命令行（cp936 会乱码）。"
---

# B 站直播间陪聊：耳朵挂在后台，脑子定时轮

一次完整的陪聊有两半，别混在一起：

- **耳朵 + 手** = 一个后台 python worker（`live_chat_worker.py`）：收实时弹幕写日志、按冷却把队列里的话发出去、每 45 秒检查直播是否结束、收到 `stop.flag` 立刻退出。
- **脑子** = 本会话里的一条定时提醒：每 60–120 秒醒一次，读增量弹幕、判断够不够聊、最多写 1–2 句进队列。

worker 会自己退场（直播结束/到点），提醒也要自己删。**没人清理的提醒会一直烧调用。**

---

## 0. 先拿到房间号（最容易错的一步）

**直播间号 ≠ 主播 UID。** 用户常给 UID 或昵称。

```bash
# 昵称 → UID（带 Cookie 才可用）
curl -s "https://api.bilibili.com/x/web-interface/search/type?search_type=bili_user&keyword=小小只suki" \
  -H "Cookie: $(cat <你的用户目录>\.dsh/bili-duty/cookie.txt 2>/dev/null)" | head -c 2000

# UID → 房间号 + 开播状态（不需要 Cookie）
curl -s "https://api.live.bilibili.com/room/v1/Room/getRoomInfoOld?mid=627306137"
#   → data.roomid / data.title / data.liveStatus(1=在播) / data.online
```

判人：看粉丝数、签名里的「直播时间」、以及是不是同名小号。重名时把候选念给用户确认，别猜。

拿到 `roomid` 后确认 `liveStatus == 1`，再往下走。

## 1. 硬闸：没放行不进房

`<你的用户目录>\.dsh\bili-duty\live-allow.json`（`enforce: true`）是唯一入口名单，`bili_live_info` / `bili_live_feed` / `bili_live_send` 都受它约束。

- 只有管理员能改名单（`bili_live_allow` 不在 QQ 侧白名单里，QQ 会话开不了新房）。
- 加房：`bili_live_allow add 32438129 anchor 小小只suki`（可 `until_offline=true` 只放这一场）。
- ⚠️ **这个文件会被别的写者覆盖**：加完必须 `cat live-allow.json` 复查一遍；发现被冲掉就再 add 一次。同理 `live_mcp.py` 也可能被别的 agent 改过，动手前先 `git diff` / 看 `tail`。

## 2. 发弹幕

```bash
cd <你的用户目录>\.dsh/bili-duty
PYTHONIOENCODING=utf-8 ./.venv/Scripts/python.exe -c "
import live_mcp
print(live_mcp.bili_live_send(32438129, '路过测试一下弹幕，主播好呀', confirm=True))
print(live_mcp._history(32438129)[-3:])
"
```

- **绝对不要**把中文当 pwsh/native 参数传（控制台 cp936 → 乱码）。要么 `PYTHONIOENCODING=utf-8` + 脚本内联，要么写成 UTF-8 文件再读。
- 成功看 `sent_from_me: true` + `id_str`；再 `_history` 回读确认没乱码、时间对。
- 弹幕 ≤40 字、不能带链接、**发出去删不掉**、同房间受冷却。

## 3. 冷却（`live_mcp.py`）

```python
MIN_INTERVAL_SEC = 60       # 2026-10-25 主人定：同一房间 60 秒一条（以本 skill 为准）
```

- 护栏：`if hits >= 3 and remaining > MIN_INTERVAL_SEC:` —— 按 60 秒写，冷却 ≤60 时照常等，不会误杀。
- 文档串（`live_mcp.py:8`、`:485` 附近）和 worker 文件头里的「每房间 N 秒 1 条」要一起改，改完 `grep -n "120 秒"` 确认没残留。
- **物理下限 ≈ 33–38 秒**：worker 轮询 1 秒 + 节流后 sleep `cooldown_remaining + 0.2`。想让它真按 60 秒走，就别用 5 秒轮询。
- 2026-10-25 起 QQ 侧也有这条口子：`bili_live_send` 直接可发（不再要草稿过目），约束靠「60 秒冷却 + 每次调用都进 `logs/danmaku.jsonl`」；挂机听直播间走 `bili_live_watch_start/status/stop`（worker 由 MCP 拉起，不再需要 DSH 会话里的定时提醒）。

## 4. worker

`<你的用户目录>\.dsh\bili-duty\live_chat_worker.py`，参数 `--room --mid --dir [--wake-key group:xxx] [--wake-cooldown 90] [--judge-sec 60]`：

| 行为 | 说明 |
|---|---|
| 收弹幕 | `bilibili_api.live.LiveDanmaku` 收 `DANMU_MSG` → `danmaku.jsonl`，跳过自己 uid；`text` 命中 `SELF_NAME` 时标 `"at_me": true` |
| 补漏 | 每 20 秒 `live_mcp._history(room)` 回填（`"src": "history"`）——弹幕流会丢，历史接口不会 |
| 去重 | `_seen` 集合**两条路径共享**（只在 `on_danmaku` 里 add）。只给回填用会双写同一条 |
| 发话 | 轮询 `queue.jsonl`（1 秒一次），`throttled` / `stop_retrying` 时 sleep `cooldown_remaining + 0.2` |
| 检查存活 | 每 45 秒 `getRoomInfoOld?mid=` 看 `liveStatus`；掉 0 → 写 `closed.jsonl` 并退出 |
| 退场 | 有 `stop.flag` 立即退出 |
| 进场闸 | 启动前必须 `armed.json` 且 `allow: true`，否则 `REFUSED` 不进场 |
| **叫醒（2026-10-25）** | 每 `--judge-sec` 秒读 `danmaku.jsonl` 增量（byte offset 存 `cursor.txt`）→ 写 `hint.json`；够得着（`at_me` 或有实质弹幕且最近一条 ≤3 分钟）且有 `--wake-key` 时 POST 桥接 `/api/socialV2/wake` 把 QQ 会话叫醒，两次之间至少隔 `--wake-cooldown` 秒，每次记 `wake.jsonl`。**脑子还是 QQ 会话里的模型**，worker 只判断值不值得叫 |

状态目录 `<你的用户目录>\.dsh\bili-duty\live-chat\`：
`armed.json` `cursor.txt` `queue.jsonl` `danmaku.jsonl` `sent.jsonl` `closed.jsonl` `stop.flag` `worker.log` `schedule-id.txt`

启动＝后台 job（`run_in_background`），不要占屏；先写好 `armed.json`：

```json
{"room": 32438129, "mid": 627306137, "allow": true, "armed_at": "2026-10-25T02:30:00"}
```

## 5. 定时提醒（脑子）

`schedule_create` 建一条 recurring（`every_seconds: 60~120`），prompt 里写死这六步，别指望自己临场记：

1. 先看 `closed.jsonl` 非空？→ 直播结束：删提醒、一句话收工，**不再进房**。
2. `tail -n 40 danmaku.jsonl` 看走向 + `cursor.txt` 得已读行数 → 之后算增量。
3. **判断够不够聊**（最重要，见下）；不够就只推 cursor。
4. 优先级：`at_me` / 问我 / 回我 → 必回（≤2 条）；其余值得接 ≤1 条；明显在跟别人聊的不插手。
5. 句子 ≤40 字、口语化、当同场观众；不复读原话。
6. `{"text":"..."}` 追加 `queue.jsonl`，尾巴把总行数写回 `cursor.txt`。

**够不够聊的判据**（这条决定它像人还是像复读机）：

- 只有当【增量里 ≥1 条实质内容（提问/感慨/吐槽/自述/接话，不含表情、刷屏、点歌系统话术）】**且**【最近一条弹幕 ≤3 分钟】才考虑发。
- 增量 0 条、或最新弹幕已经超过 3 分钟没人说话 → **本轮一律不发**，只推 cursor。
- 拿不准就不发。宁可少说，也别为了不冷场硬找话。

**叫醒判据**（2026-10-25 起，worker 内实现，`--wake-cooldown` 默认 90 秒、`--max-wakes-per-hour` 默认 30）：

- **被 @ / 被点名（含「大肥鱼」「本鲸」「猫猫鱼」）→ 立刻叫醒，不受冷却限制**（仍受每小时 cap 约束）。
- 其余情况必须**增量里 ≥2 条实质内容**才叫醒 —— 1 条不够，半句话不值得起一个完整模型回合。
- 叫醒只是「让她看一眼」，发不发弹幕仍按上面那条判据来；叫醒后被冷却挡住时只写 `hint.json`。

## 6. 退场纪律

- 到点/结束：先发一句道谢弹幕（举例「谢谢主播今晚陪着聊天，听得很舒服，我先撤啦，晚安」）→ 回读 `sent.jsonl` 确认 → `echo stop > live-chat/stop.flag` → `job_list` 确认 worker 已结束（没结束 `job_kill`）→ `schedule_delete` 删掉 recurring 提醒和收尾提醒自己 → 汇报。
- **没有管理员放行，不得再进这个直播间。**
- 收尾用一个一次性提醒（`after_seconds`）来钉住时间点，别靠"我等会儿记得"。

## 7. 踩过的坑

| 症状 | 真因 | 处理 |
|---|---|---|
| 弹幕发出去变问号/乱码 | 中文走了 pwsh 命令行（cp936） | `PYTHONIOENCODING=utf-8` + 内联脚本或 UTF-8 文件 |
| 加好的房间突然不见了 | `live-allow.json` 被别的写者覆盖 | 加完 `cat` 复查，必要时再 add |
| 30 秒冷却下消息被丢 | 护栏按 120 秒写死 | 护栏改成 `remaining > MIN_INTERVAL_SEC` |
| 同一条弹幕回了两遍 | 去重集合只喂给回填 | `_seen` 两条路径共享 |
| 间隔总是 +60s 而不是 30/60 | worker 5 秒轮询 | 改 1 秒轮询，sleep 用 `remaining + 0.2` |
| 觉得"读不到弹幕" | 房间本身话少 | 用 `_history()` 对账；一致就是没人说话，不是漏读 |
| 接话显得尬 | 没做"够不够聊"判断 | 见第 5 节，宁可少说 |

## 8. 最小剧本（抄了就能跑）

```bash
# 1) 找房
curl -s "https://api.live.bilibili.com/room/v1/Room/getRoomInfoOld?mid=<UID>"
# 2) 放行 + 复查
#    bili_live_allow add <room> anchor <主播名>   →   cat live-allow.json
# 3) 进场（后台 job）
#    armed.json {room, mid, allow:true}  →  python live_chat_worker.py --room .. --mid .. --dir live-chat
# 4) 定时提醒 every_seconds=60~120，prompt 用第 5 节六步
# 5) 想先说一句就 queue.jsonl 追加 {"text":"..."}
# 6) 到点：道谢 → stop.flag → 删提醒 → 汇报
```
