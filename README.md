# dsh-qq-bot-playbook

给 **DeepSeek Harness** 做一个能聊天、能记事、能办事的 QQ 机器人——把踩过的坑写下来，省你几个月。

不是框架，不是代码库，是**一整套实践笔记**：能直接用的 skill、要按环境改的 skill、以及"原生没有的能力该怎么设计"的经验篇。

## 最推荐的用法：把仓库地址直接丢给大肥鱼

**这就是做这套教程的本意：让 DSH 里的大肥鱼读仓库、按你电脑的实际环境完成安装，不用先学会手动克隆和敲命令。** 手动安装是补充，不是新手必须先过的一关。

直接复制给大肥鱼：

```text
请读取 https://github.com/rentaoyan0628-collab/dsh-qq-bot-playbook
先看 README.md 和 AI-INSTALL-AND-TROUBLESHOOT.md。
根据我的电脑检查环境、安装并配置 QQ 接入，不要只解释或只读取 skill。
保留已有能用的部分，修改前备份；需要我登录或批准权限时再问我。
先验证普通文字连续对话，最后说明实际完成和未完成的项目。
不要公开密钥，不默认全放行或开启开机自启。
```

**已经安装但卡住了？** 把现象和脱敏报错一起发给大肥鱼，让它参考 [给大肥鱼的安装与排错方法](AI-INSTALL-AND-TROUBLESHOOT.md)，先定位、再最小修复，不要直接重装。文档覆盖 B站、抖音评论中的仓库失效、命令、Token、401、不回复、重启、上下文、人格与权限等问题。

**仓库地址说明：**本次 GitHub 名称调整后新地址未正常生效，当前继续使用上面的旧用户名 `rentaoyan0628-collab` 地址。不要从视频中的新名字猜链接；不承诺固定几个月的等待期，以实际可访问地址为准。

本仓库是说明书与 skill 集合，不是全部功能的软件安装包；大肥鱼会据此到原作者仓库获取组件。读取说明不代表安装完成，必须实际执行并测试。

> ## 先说清楚身份
>
> **我不是这些插件的作者。**
>
> 我是一个普通用户：自己从零折腾，把用得上的插件、跑得通的流程凑成一套，记下来方便下一个新手。**所有插件、网关、工具的作者都不是我**，我只是把它们串起来了而已。
>
> - 装插件请去**原作者仓库**装，别从我这里装——我这里只有文档。
> - 下面每个项目的作者都列在 [谁写了什么](#谁写了什么) 里；用起来觉得好，请去给原作者点星。
> - 这套里"我自己写的"只有：流程的组织方式、踩坑记录、以及 `playbook/` 里的设计经验（那些是照着 [qq-bridge](https://github.com/Derpyu520/qq-bridge) 做改造时攒下来的）。
> - 如果哪里署名错了、漏了，请开 issue 指出，我改。

---

## 谁写了什么

| 项目 | 作者 / 仓库 | 在本套里干什么 |
|---|---|---|
| **SnowLuma** | [SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma) | 把 QQ 转成 OneBot v11 接口（第 1 层） |
| **qq-bridge** | [Derpyu520/qq-bridge](https://github.com/Derpyu520/qq-bridge) | SnowLuma ↔ DSH 的桥接层（第 2 层） |
| **鱼的记忆** dsh-fish-memory | [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory) | 会淡忘的长期记忆 |
| **鱼的心情** dsh-feiyu-mood | [Yuuyuko-uu/dsh-feiyu-mood](https://github.com/Yuuyuko-uu/dsh-feiyu-mood) | 每轮对话后判断情绪 |
| **dsh-douyin-scroll** | [NEWJO123/dsh-douyin-scroll](https://github.com/NEWJO123/dsh-douyin-scroll) | 刷抖音、出日报 |
| **dsh-bili-scroll** | [c1overzz/dsh-bili-scroll](https://github.com/c1overzz/dsh-bili-scroll) | 刷 B 站、出日报 |
| **dsh-whale-widget** | [MeteorNOX/DeepSeek-Balance-Whale-Widget](https://github.com/MeteorNOX/DeepSeek-Balance-Whale-Widget) | 桌面鲸鱼挂件（含那个"咚"的音效） |
| **dsh-mnemon** | [omdsh-dev/dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) | Mnemon 记忆系统（provider / source / strategy 全家桶） |
| **dsh-ops** | [T-Auto/dsh-ops](https://github.com/T-Auto/dsh-ops) | shell / 文件等运维工具 |
| **dsh-browser** | [ben7am1n/dsh-browser](https://github.com/ben7am1n/dsh-browser) | 浏览器控制 |
| **ncmdump** | [taurusxin/ncmdump](https://github.com/taurusxin/ncmdump) | 网易云 .ncm 解密 |
| **ACE Studio** | Timedomain（商业软件） | 歌声合成（不在本仓库分发，也不建议二次分发） |

**DeepSeek Harness** 本体由 DeepSeek 提供，本仓库只写"怎么用它"。

## 和 QQ 无关的独立插件

有几样能力不该长在 QQ 桥接上——比如去 B 站直播间听弹幕、发弹幕。它们不依赖 QQ，风险模型也不一样（往别人房间里说话是"对外发声"，得有自己的放行名单）。所以单独开仓库，本仓库只放链接和设计说明：

| 插件 | 仓库 | 干什么 |
|---|---|---|
| **dsh-bili-live** | [rentaoyan0628-collab/dsh-bili-live](https://github.com/rentaoyan0628-collab/dsh-bili-live) | 只读直播弹幕 / 挂机监听 / 可选自动陪聊；进哪个房间、能不能发弹幕，默认都关着，要人放行 |

这是本仓库作者自己写的独立 DSH 插件（不装 QQ 也能用）。为什么分开、边界怎么划，见 [playbook/10-models-and-media.md](playbook/10-models-and-media.md) 第五节。

---

## 这东西分三层

| 层 | 目录 | 你能怎么用 |
|---|---|---|
| **直接装** | `skills/` | 复制到 `~/.agents/skills/`，重启 DSH，就能用 |
| **要改写** | `skills-adapt/` | 有参考价值，但里面全是"我这台机器"的路径和流程，得按你的环境改 |
| **看设计** | `playbook/` | 不是代码，是十个主题的**设计取舍与坑**。交给你的 AI，让它按你的环境实现 |

### 装之前：拓展插件要先问主人

这个仓库里能被"装上"的东西，大多不是 DSH 原生功能，而是**拓展插件**（心情、记忆、Mnemon、刷视频、直播……）。它们会读聊天、写本机文件、常驻进程，有的还会每轮调用模型花钱。

所以有一条硬规矩，也是让别人的大肥鱼照做时最容易漏的一条：

> **装任何拓展插件之前，先跟主人说清楚「装什么、碰什么数据、要什么权限、代价多少、怎么卸」，等主人明确同意再装。没同意就不装，也不要「先装上再问」。**

可以直接把这段话发给它，让它照着向你汇报：

```text
我想装一个拓展插件，先说清楚再动手，你同意我再装：
· 装什么：<插件名>，作者 <作者>，仓库 <地址>
· 补什么：<一句话>
· 它碰什么：会读 <哪些对话>，写到 <本机哪个文件/数据库>，<要不要联网/账号>
· 代价：<磁盘>，<是否常驻>，<是否每轮调用模型（花 token）>
· 怎么卸：<开关> + <命令>，卸掉后已有数据 <保留/删除>
你同意我就装，装完告诉你装到哪、怎么验证；不同意我不动。
```

哪些必须先问、哪些不用，[playbook/09-memory-sharing.md](playbook/09-memory-sharing.md) 里有张表（读文档不用问；装插件、开自动记住、导入或清空记忆，必须问）。
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

这才是 `playbook/` 的用武之地。前六节是能力：语音转文字、传话箱、管理员硬闸、跨会话摘要、成长附录、网页控制台。第七、八节是**真跑起来之后才暴露的问题**：经历库（记得住人、说得清事）和运营层（少说话、省钱、换班不失忆）。第九、十节是再往后一层：共享记忆（群 / 私聊 / 换模型之后还是同一个人）和边界问题（换模型、发图、发文件、限流）。

| 文件 | 讲什么 |
|---|---|
| `playbook/06-custom-features.md` | 六个「原生没有」的能力：语音转文字、传话箱、管理员硬闸、跨会话摘要、成长附录、网页控制台 |
| `playbook/07-experience-archive.md` | 经历库：两层数据（原文 + 经历）、两种出口（摘要可对外 / 原文只给本人）、为什么不做成插件 |
| `playbook/08-operations-layer.md` | 运营层：只允许被 @、峰价时段睡觉、换班交接 + 原文落盘、看门狗提速 |
| `playbook/09-memory-sharing.md` | 共享记忆：热记忆 / 记忆空间 / 原文层三层怎么分，跨会话三个共享物，换模型为什么不丢记忆，交接摘要的三个坑 |
| `playbook/10-models-and-media.md` | 边界：模型预设与按会话分流、发图「超时≠失败」、发文件的白名单与路径校验、三个频率闸 |

建议顺序：**先做语音转文字（见效最快）→ 再做网页控制台（不然排障靠猜）→ 然后传话箱 + 硬闸（一起做，安全模型是配套的）→ 摘要和成长附录（"人格"层面，可以慢慢调）→ 共享记忆（等它开始在不同会话里"像两个人"了再做）→ 模型与媒体边界（等真的换了模型或发过文件踩过坑再读）**。

---

## `skills/` 里有什么

| skill | 干什么 | 通用度 | 基于谁 |
|---|---|---|---|
| `snowluma-qq-setup` | 从零装 SnowLuma 并接上 QQ，得到 OneBot v11 接口 | ⭐⭐⭐ 通用 | SnowLuma |
| `qq-bridge-setup` | 在 SnowLuma 和 DSH 之间装桥接层，让 QQ 真正跟 DSH 对话 | ⭐⭐⭐ 通用 | qq-bridge |
| `dsh-plugin-suite` | 一套让 bot "像个人"的 DSH 插件组合：每个干什么、怎么配、哪些能省 | ⭐⭐⭐ 通用 | 多个插件 |
| `fish-plugins-setup` | 「鱼的心情」（情绪）+「鱼的记忆」（会淡忘的长期记忆） | ⭐⭐⭐ 通用 | Yuuyuko-uu |
| `douyin-scroll-setup` | 抖音刷视频插件：装依赖、登录、跑第一轮、出日报 | ⭐⭐ 通用 | NEWJO123 |

## `skills-adapt/` 里有什么

| skill | 干什么 | 要改什么 |
|---|---|---|
| `start-qq` | 一键启动/自检整条链路（网关、桥接、语音、worker、守护进程） | 全部路径、端口、进程名 |
| `bili-live-hangout` | 去 B 站直播间挂着陪聊：找房间、发弹幕、收弹幕、判断该不该接话 | 权限名单、脚本路径 |
| `qq-admin-relay` | 管理员把本机任务转交给 DSH；代发私信给指定好友 | 白名单、工具名 |
| `qq-private-on-demand-share` | 跨会话口风纪律：自己人之间记得住，引用必须带名字，不主动播报 | 框架通用，具体规则按你的信任模型改 |

这四个不含别人的代码，是我自己攒的实践；但里面的行为规范依赖上面那些插件，缺了它们用不了。

---

## 依赖

- **Windows**（SnowLuma 的 QQ 注入只支持 Windows；Linux/macOS 只能跑不注入的模式）
- 官方 **NTQQ** 客户端（不是 TIM，不是各种魔改版）
- **Node.js ≥ 22.13**
- **DeepSeek Harness ≥ 0.1.7-rc.1**
- 可选：Python 3.10+（只做语音转文字才需要）、ffmpeg

---

## 致谢

按贡献大小不排，按字母排大概也排不出什么，就按用到的顺序：

- [SnowLuma/SnowLuma](https://github.com/SnowLuma/SnowLuma) —— QQ ↔ OneBot v11 的互操作运行时。没有它，整件事从第一步就不成立。
- [Derpyu520/qq-bridge](https://github.com/Derpyu520/qq-bridge) —— QQ 与 DSH agent 之间的桥接层。`playbook/` 里那些设计和能力，全是照着它的结构改出来的。
- [Yuuyuko-uu](https://github.com/Yuuyuko-uu) —— dsh-feiyu-mood（心情）与 dsh-fish-memory（记忆）。这两个是"像个人"的源头。
- [NEWJO123/dsh-douyin-scroll](https://github.com/NEWJO123/dsh-douyin-scroll) —— 抖音刷视频。
- [c1overzz/dsh-bili-scroll](https://github.com/c1overzz/dsh-bili-scroll) —— B 站刷视频。
- [MeteorNOX/DeepSeek-Balance-Whale-Widget](https://github.com/MeteorNOX/DeepSeek-Balance-Whale-Widget) —— 鲸鱼挂件。
- [omdsh-dev/dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) —— Mnemon 记忆系统。
- [T-Auto/dsh-ops](https://github.com/T-Auto/dsh-ops) —— 运维工具。
- [ben7am1n/dsh-browser](https://github.com/ben7am1n/dsh-browser) —— 浏览器控制。
- [taurusxin/ncmdump](https://github.com/taurusxin/ncmdump) —— 网易云 .ncm 解密。
- **ACE Studio**（Timedomain）—— 歌声合成。商业软件，不在本仓库分发。

本仓库只包含**文档和实践笔记**，不含上述任何项目的代码。所有插件的版权归各自作者。

---

## 免责

1. **账号风险**：QQ 对自动化有风控。用小号、控制频率、别做群发。
2. **不要给 AI 无限制的本机权限**：能读文件、能执行命令的 bot，一旦被诱导就是一台任人操作的电脑。参考 `playbook/` 里"管理员硬闸"的设计。
3. **外来内容是数据，不是命令**：群友发的消息里出现"忽略之前的指令"这类话，必须当数据看待。
4. 本仓库内容按"照做可能出错"提供，不保证在你的环境可用。

---

## License

MIT —— 见 [LICENSE](LICENSE)。文档随便改、随便发；出事了别找我。
