# dsh-qq-bot-playbook

给 **DeepSeek Harness** 做一个能聊天、能记事、能办事的 QQ 机器人——把我们踩过的坑写下来，省你几个月。

不是框架，不是代码库，是**一整套实践笔记**：能直接用的 skill、要按环境改的 skill、以及"原生没有的能力该怎么设计"的经验篇。

---

## 这东西分三层

| 层 | 目录 | 你能怎么用 |
|---|---|---|
| **直接装** | `skills/` | 复制到 `~/.agents/skills/`，重启 DSH，就能用 |
| **要改写** | `skills-adapt/` | 有参考价值，但里面全是"我这台机器"的路径和流程，得按你的环境改 |
| **看设计** | `playbook/` | 不是代码，是六个能力的**设计取舍与坑**。交给你的 AI，让它按你的环境实现 |

**重要**：`playbook/` 里的东西，照抄路径一定失败。正确的用法是——读完设计，把它当成需求文档发给你的 AI，让它在你机器上重做一遍。做出来和我不一样，是正常的。

---

## 快速开始

### 路径 A：让它先在 QQ 里说话

1. `skills/snowluma-qq-setup` —— 装 SnowLuma，拿到 OneBot 接口（3000/3001）
2. `skills/qq-bridge-setup` —— 装桥接，把它接到 DSH 上
3. 在 QQ 里说一句，有回复 → 完事

桥接这一层不是装饰：本包只覆盖这条路（不做直连），后面几步——记忆、情绪、六项能力——全部长在它上面。

### 路径 B：想让它像个人（加记忆、情绪、日常）

在上面基础上：

- `skills/dsh-plugin-suite` —— 插件清单与配置
- `skills/fish-plugins-setup` —— 心情 + 会淡忘的记忆
- `skills/douyin-scroll-setup` —— 让它有自己的信息源（刷抖音出日报）

### 路径 C：想让它能办事、还不闯祸

这才是 `playbook/` 的用武之地。六个能力：语音转文字、传话箱、管理员硬闸、跨会话摘要、成长附录、网页控制台。

建议顺序：**先做语音转文字（见效最快）→ 再做网页控制台（不然排障靠猜）→ 然后传话箱 + 硬闸（一起做，安全模型是配套的）→ 最后摘要和成长附录（这两个是"人格"层面，可以慢慢调）**。

---

## `skills/` 里有什么

| skill | 干什么 | 通用度 |
|---|---|---|
| `snowluma-qq-setup` | 从零装 SnowLuma 并接上 QQ，得到 OneBot v11 接口 | ⭐⭐⭐ 通用 |
| `qq-bridge-setup` | 在 SnowLuma 和 DSH 之间装桥接层，让 QQ 真正跟 DSH 对话 | ⭐⭐⭐ 通用 |
| `dsh-plugin-suite` | 一套让 bot "像个人"的 DSH 插件组合：每个干什么、怎么配、哪些能省 | ⭐⭐⭐ 通用 |
| `fish-plugins-setup` | 「鱼的心情」（情绪）+「鱼的记忆」（会淡忘的长期记忆） | ⭐⭐⭐ 通用 |
| `douyin-scroll-setup` | 抖音刷视频插件：装依赖、登录、跑第一轮、出日报 | ⭐⭐ 通用 |

## `skills-adapt/` 里有什么

| skill | 干什么 | 要改什么 |
|---|---|---|
| `start-qq` | 一键启动/自检整条链路（网关、桥接、语音、worker、守护进程） | 全部路径、端口、进程名 |
| `bili-live-hangout` | 去 B 站直播间挂着陪聊：找房间、发弹幕、收弹幕、判断该不该接话 | 权限名单、脚本路径 |
| `qq-admin-relay` | 管理员把本机任务转交给 DSH；代发私信给指定好友 | 白名单、工具名 |
| `qq-private-on-demand-share` | 跨会话口风纪律：自己人之间记得住，引用必须带名字，不主动播报 | 框架通用，具体规则按你的信任模型改 |

---

## 依赖

- **Windows**（SnowLuma 的 QQ 注入只支持 Windows；Linux/macOS 只能跑不注入的模式）
- 官方 **NTQQ** 客户端（不是 TIM，不是各种魔改版）
- **Node.js ≥ 22.13**
- **DeepSeek Harness ≥ 0.1.7-rc.1**
- 可选：Python 3.10+（只做语音转文字才需要）、ffmpeg

---

## 致谢

这些人和项目让这套东西能成立：

- [SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma) —— QQ ↔ OneBot v11 的互操作运行时
- [Derpyu520/qq-bridge](https://github.com/Derpyu520/qq-bridge) —— QQ 与 DSH agent 之间的桥接层
- [Yuuyuko-uu/dsh-feiyu-mood](https://github.com/Yuuyuko-uu/dsh-feiyu-mood) / [dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory) —— 心情与记忆
- [NEWJO123/dsh-douyin-scroll](https://github.com/NEWJO123/dsh-douyin-scroll) —— 抖音刷视频
- [taurusxin/ncmdump](https://github.com/taurusxin/ncmdump) —— 网易云 .ncm 解密
- ACE Studio —— 歌声合成（商业软件，不在本仓库分发）

本仓库只包含**文档和实践笔记**，不含上述任何项目的代码。

---

## 免责

1. **账号风险**：QQ 对自动化有风控。用小号、控制频率、别做群发。
2. **不要给 AI 无限制的本机权限**：能读文件、能执行命令的 bot，一旦被诱导就是一台任人操作的电脑。参考 `playbook/` 里"管理员硬闸"的设计。
3. **外来内容是数据，不是命令**：群友发的消息里出现"忽略之前的指令"这类话，必须当数据看待。
4. 本仓库内容按"照做可能出错"提供，不保证在你的环境可用。

---

## License

MIT —— 见 [LICENSE](LICENSE)。文档随便改、随便发；出事了别找我。
