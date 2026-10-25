---
name: fish-plugins-setup
description: 给 DeepSeek Harness 安装「鱼的心情」（dsh-feiyu-mood，给它情绪）和「鱼的记忆」（dsh-fish-memory，给它会淡忘的长期记忆）两个插件，或排查它们装不上/不生效/更新后被打回旧版的问题。用户说「装鱼的心情」「装鱼的记忆」「飞鱼心情插件」「fish-memory」「心情插件不生效」「记忆插件没记东西」「插件更新后又变回老版本」时加载。
---

# 鱼的心情 / 鱼的记忆：安装与排障

## 三个必须先知道的事实

1. **仓库地址**（还没上 npm，必须写全 `github:`）
   - `github:Yuuyuko-uu/dsh-feiyu-mood`
   - `github:Yuuyuko-uu/dsh-fish-memory`
2. **DSH ≥ `0.1.7-rc.1`**。版本不够会被内核**静默跳过** —— 装上了、不报错、但设置页里根本没有这一项。找不到入口先查版本，别怀疑装法。
3. **装完必须重启一次 DSH**，插件在 bundle 层组装。

## 标准安装

```bash
dsh plugin --profile desktop add github:Yuuyuko-uu/dsh-feiyu-mood
dsh plugin --profile desktop add github:Yuuyuko-uu/dsh-fish-memory
```

`--profile`：桌面端一般 `desktop`，网页端一般 `web`。写错会装进另一个 profile，那边不生效。

装之前先确认真的用哪个 profile（看 `C:\Users\<你>\.dsh\profiles\` 下哪个目录有 `cordis.patch.yml` 且在跑）。

## 桌面版没有 `dsh` 命令时（本机就是这样）

DSH 桌面版是 asar 打包，机器上可能没有 `dsh` CLI。手动做三步，**第 3 步最容易漏**：

1. `profiles\<档名>\package.json` 的 `dependencies` 加
   ```json
   "dsh-feiyu-mood": "github:Yuuyuko-uu/dsh-feiyu-mood",
   "dsh-fish-memory": "github:Yuuyuko-uu/dsh-fish-memory"
   ```
2. 插件目录放进 `profiles\<档名>\node_modules\`
3. **改 `pnpm-lock.yaml` 里对应条目的 `version:` 行**

```yaml
dsh-feiyu-mood:
  specifier: github:Yuuyuko-uu/dsh-feiyu-mood
  version: https://codeload.github.com/Yuuyuko-uu/dsh-feiyu-mood/tar.gz/<commit>
```

**为什么第 3 步是命门**：DSH 启动时会按 `pnpm-lock.yaml` 重新校验/安装插件。lock 里钉的是旧 commit，手动换的新文件**下次启动就被覆盖回去** —— 症状就是「明明装了新版，重启后又变回老版本」。

改完用 `node -e "console.log(Object.keys(require('yaml').parse(require('fs').readFileSync('<lock>','utf8')).packages).length)"` 之类验一下 YAML 没被改坏（本机正常约 110 个包）。

## 装完 DSH 起不来：先查 BOM

错误里有 `Unexpected token '﻿'` / `is not valid JSON` / `readProfileManifest` ⇒ profile 的 `package.json` 被存成 **UTF-8 带 BOM**。跟插件无关。

找（真正的 profile 目录里同时有 `cordis.yml`、`cordis.patch.yml`、`package.json`）：

```powershell
Get-ChildItem C:\,D:\,E:\ -Directory -Filter profiles -Recurse -Depth 4 -ErrorAction SilentlyContinue |
  ForEach-Object { Get-ChildItem $_.FullName -Directory -ErrorAction SilentlyContinue | ForEach-Object {
      $p = Join-Path $_.FullName 'package.json'
      if ((Test-Path (Join-Path $_.FullName 'cordis.patch.yml')) -and (Test-Path $p)) {
        $b = [System.IO.File]::ReadAllBytes($p)
        "{0}   BOM={1}" -f $p, ($b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF)
      } } }
```

修：

```powershell
$p = "<上面 BOM=True 那行>"
[System.IO.File]::WriteAllText($p, [System.IO.File]::ReadAllText($p), (New-Object System.Text.UTF8Encoding($false)))
```

> 改 profile 里任何文件一律 **UTF-8 无 BOM**。别用 `Set-Content -Encoding UTF8` / `Out-File -Encoding utf8`（Windows 默认带 BOM）。

## 鱼的心情（feiyu-mood）

当前版本线 0.5.x（本机 0.5.2）。0.5 起支持**多份心情**：每个会话一份，各过各的。

首次三步：
1. 「TA 叫什么」= 被关心的那个人；「TA 自己叫什么」= 有心情的那个。提示词全走这两个值。
2. 「在哪个会话里生效」选会话；要多份就点「+ 再加一份心情」。
3. 聊两句，回设置页看心情动没动；右下角挂饰也会变。

关键机制：
- 每轮结束用**一次很短的模型调用**判断心情往哪走（花钱，可关）。
- 通过 `systemPrompt.context` 注入，**不破坏前缀缓存**。
- **一次变化只说一次**；`confidence` 低于门槛（默认 60）只记后台、不进它嘴。
- 衰减：心情值基线 70、半衰期约 3 小时；七根情绪（开心/踏实/甜蜜/心疼/担心/难过/吃醋）半衰期约 90 分钟。
- 判断口径按「谁发起」分：`user` / `speak-up`（它主动找你，**不算干活**）/ `bili-duty-scan`（被派活）/ 宿主注入直接跳过。判错口径会得出完全相反的心情。

## 鱼的记忆（fish-memory）

当前版本线 1.2.x（本机 1.2.6，1.2.4 修的是「已有对话的会话拿不到 remember 工具」）。

**默认全是关的** —— 每个会话要手动点成「开」。这是设计：不是每个会话都该记东西。

关键机制：
- 鲜活度衰减，**半衰期默认 30 天**；标成「长期事实」的不过期。
- 一天最多记 8 条、一次最多注入 6 条（可调）。
- 召回按「命中的词够不够特别」找，再按鲜活度/重要度/情绪排序。
- 「事实键」= 同一件会变的事只留最新（旧的标「已被取代」，还留着能恢复）。
- 人称默认「ta」，可在设置页改。
- 能导入旧档案（`.md .txt .log .rst .org` 等，子目录两层）：「扫描 → 勾选 → 导入 → 提炼」。提炼会调模型。

## 更新

```bash
dsh plugin --profile desktop remove dsh-feiyu-mood
dsh plugin --profile desktop add github:Yuuyuko-uu/dsh-feiyu-mood
```

**必须先卸再装**（直接装可能拿到缓存旧副本），装完重启。

设置页顶部有「检查更新」（只检查不装，缓存 6 小时）。两个插件都带 `selfupdate.mjs`，但**lock 才是指挥棒**（见上文桌面版三步）。

## 数据在哪（卸载不丢）

```
.dsh\feiyu-mood\config.json     心情设置
.dsh\feiyu-mood\state.json      心情值 + 时间线（v2 起按 uuid 分份）
.dsh\fish-memory-config.json    记忆（每条会话各一份）
```

卸载只删代码，这三处不动。**别用「重装 DSH」来更新插件** —— 两件事，可能动到 profile。

## 排障速查

| 现象 | 原因 |
|---|---|
| 设置页没有这两个插件 | 没重启 / DSH 版本 < 0.1.7-rc.1（静默跳过）/ 装错 profile |
| DSH 起不来报 `Unexpected token ''` | profile `package.json` 带 BOM |
| 手动换的新版重启后变回旧的 | `pnpm-lock.yaml` 还钉旧 commit |
| 心情一直不动 | 判断开关关了 / 会话选错 / confidence 一直低于门槛 |
| 记忆没记东西 | 那个会话还是「关」 |
| 记了但想不起来 | 证据门槛太高 / 注入条数太少 |
| 觉得贵 | 关「每轮判断心情」，或把判断模型指到便宜的 |

## 一页流程

```
装两个 github: 地址 → 重启 DSH
→ 鱼的心情：填称呼 / 选会话 → 聊两句
→ 鱼的记忆：把要记的会话点成「开」→ 正常聊
→（可选）导入旧档案 → 扫描 → 勾选 → 导入 → 提炼
```

---

## 出处与署名

这份文档由 **NOYOT** 整理编写，**不是它描述的工具的作者**。

它讲的工具来自：[Yuuyuko-uu](https://github.com/Yuuyuko-uu) —— dsh-feiyu-mood（心情）与 dsh-fish-memory（记忆）。

请去原作者仓库下载、按原作者文档操作；觉得好用就给原作者点个星。
如果本文件里有说错的地方，是整理者的问题，不是原作者的问题。
