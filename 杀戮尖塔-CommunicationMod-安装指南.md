# 杀戮尖塔 CommunicationMod 安装指南 (macOS)

## 概述

目标：安装 CommunicationMod，让外部程序（AI agent）能通过 stdin/stdout JSON 协议控制杀戮尖塔。

## 安装清单

| 组件 | 来源 | 最终版本 | 安装位置 |
|------|------|---------|---------|
| ModTheSpire | **Steam 创意工坊** | v3.30+ | `SlayTheSpire/ModTheSpire.jar` |
| BaseMod | **Steam 创意工坊** | v5.56.0 | `SlayTheSpire/mods/BaseMod.jar` |
| StSLib | **Steam 创意工坊** | v2.12.0 | `SlayTheSpire/mods/StSLib.jar` |
| CommunicationMod | GitHub Releases | v1.2.1 | `SlayTheSpire/mods/CommunicationMod.jar` |

## 关键错误和教训

### 错误 1：从 GitHub 下载 ModTheSpire 和 BaseMod（版本过时）

**问题：** GitHub 上的 ModTheSpire (kiooeht/ModTheSpire) 最新版 v3.6.3 是 2018 年的，作者已声明所有后续更新只在 Steam 创意工坊发布。BaseMod GitHub 版 v5.5.0 同样过时。

**后果：** CommunicationMod 需要 ModTheSpire >= v3.18.1，旧版 MTS 会导致 mod 列表中 CommunicationMod **无法勾选**（灰掉）。

**正确做法：** 在 Steam 创意工坊订阅 ModTheSpire 和 BaseMod，然后从 Workshop 文件夹复制 JAR。

- ModTheSpire Workshop: https://steamcommunity.com/sharedfiles/filedetails/?id=1605060445
- BaseMod Workshop: 在 Slay the Spire 创意工坊搜索 "BaseMod"
- StSLib Workshop: 在 Slay the Spire 创意工坊搜索 "StSLib"
- Workshop 下载路径: `~/Library/Application Support/Steam/steamapps/workshop/content/646570/`

### 错误 2：终端命令中路径空格处理

**问题：** `Application Support` 路径中有空格，终端运行脚本时如果不用引号包裹会报错 `No such file or directory`。

**错误写法：**
```bash
bash ~/Library/Application Support/Steam/...    # 空格导致路径截断
bash ~/Library/Application\                     # 换行时反斜杠换行也容易截断
  Support/Steam/...
```

**正确写法（必须用引号包裹整个路径）：**
```bash
bash "$HOME/Library/Application Support/Steam/steamapps/common/SlayTheSpire/MTS_mac.sh"
```

### 错误 3：ModTheSpire 找不到 desktop-1.0.jar 和 JRE

**问题：** 运行 MTS 脚本时报：
```
desktop-1.0.jar: Does not exist
SlayTheSpire.app: Does not exist
Exception: NullPointerException at ProcessBuilder.start
```

**原因：**
1. 脚本没有先 `cd` 到游戏目录，MTS 在当前工作目录找文件
2. 新版 MTS 的 `SteamSearch.findJRE()` 方法查找 `jre/bin/java`（相对工作目录），但 JRE 实际在 `SlayTheSpire.app/Contents/Resources/jre/bin/java`

**修复方法：**

1. 脚本必须先 `cd` 到游戏根目录：
```bash
#!/bin/bash
DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$DIR"
./SlayTheSpire.app/Contents/Resources/jre/bin/java -jar ModTheSpire.jar
```

2. 创建 JRE 符号链接让 MTS 能找到：
```bash
cd "SlayTheSpire目录"
ln -sf SlayTheSpire.app/Contents/Resources/jre jre
```

### 错误 4：安装 Mod 后游戏存档"丢失"

**问题：** 通过 ModTheSpire 启动后，所有进度归零。

**原因：** 原版游戏和 ModTheSpire 使用不同的工作目录，存档位置不同：
- 原版存档：`SlayTheSpire.app/Contents/Resources/preferences/` 和 `.../saves/`
- MTS 读取的存档：`SlayTheSpire/preferences/` 和 `SlayTheSpire/saves/`（游戏根目录）

**修复：** 将 .app 包内的存档复制到游戏根目录：
```bash
GAME_DIR="$HOME/Library/Application Support/Steam/steamapps/common/SlayTheSpire"
cp "$GAME_DIR/SlayTheSpire.app/Contents/Resources/preferences/"* "$GAME_DIR/preferences/"
mkdir -p "$GAME_DIR/saves"
cp "$GAME_DIR/SlayTheSpire.app/Contents/Resources/saves/"* "$GAME_DIR/saves/"
```

### 错误 5：试图修改 Steam 启动方式

**问题：** 为了从 Steam 直接启动 mod 版，修改了 `.app/Contents/MacOS/SlayTheSpire` 二进制文件（替换为脚本）。

**后果：** Steam 无法启动游戏。

**结论：** 不要修改 .app 包内的原生二进制文件。使用独立的启动脚本（.command 文件）。

## 最终可用方案

### 启动脚本位置

桌面上的 `杀戮尖塔_Mod.command` 文件，内容：
```bash
#!/bin/bash
GAME_DIR="$HOME/Library/Application Support/Steam/steamapps/common/SlayTheSpire"
cd "$GAME_DIR"
./SlayTheSpire.app/Contents/Resources/jre/bin/java -jar ModTheSpire.jar
```

双击即可启动 ModTheSpire，勾选 BaseMod、StSLib、CommunicationMod 后点 Play。

### Steam 原版启动

Steam 中正常点 Play = 原版游戏（无 mod）。

### 游戏目录关键文件结构

```
SlayTheSpire/
├── ModTheSpire.jar          # 从 Steam Workshop 复制的 mod 加载器
├── jre -> SlayTheSpire.app/Contents/Resources/jre   # 符号链接
├── mods/
│   ├── BaseMod.jar          # 从 Steam Workshop 复制
│   ├── StSLib.jar           # 从 Steam Workshop 复制
│   └── CommunicationMod.jar # 从 GitHub Releases 下载
├── preferences/             # MTS 使用的存档（需从 .app 内复制过来）
├── saves/                   # MTS 使用的存档
└── SlayTheSpire.app/        # 原版游戏包
```

## CommunicationMod 后续配置

首次成功运行后，配置文件生成在 `~/.config/ModTheSpire/CommunicationMod.cfg`（或类似位置），需要编辑 `command=` 指向你的 AI agent 脚本。

Agent 通过 stdin/stdout 与游戏通信（JSON 协议），配套 Python 库：https://github.com/ForgottenArbiter/spirecomm

## Steam 创意工坊 ID 参考

| Mod | Workshop ID |
|-----|------------|
| ModTheSpire | 1605060445 |
| BaseMod | 1605833019 |
| StSLib | 1609158507 |
