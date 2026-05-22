# 杀戮尖塔 AI Agent 完整配置手册

## 快速恢复索引

```
关键命令：tail -30 /tmp/sts_agent_debug.log          # 查看 agent 运行日志
关键命令：cat ~/sts_lessons.txt                        # 查看累积学习教训
关键命令：python3 -c "import py_compile; py_compile.compile('/Users/longyuhan/sts_agent.py', doraise=True)"  # 检查语法
启动方式：双击桌面 /Users/longyuhan/Desktop/杀戮尖塔_Mod.command
```

---

## 1. 游戏与 Mod 文件结构

### 游戏根目录
```
/Users/longyuhan/Library/Application Support/Steam/steamapps/common/SlayTheSpire/
```

### 关键文件树
```
SlayTheSpire/
├── ModTheSpire.jar              # Steam Workshop 版 (v3.30+, 10.2MB)
├── ModTheSpire.jar.bak          # GitHub 旧版备份 (v3.6.3, 1.3MB)
├── jre -> SlayTheSpire.app/Contents/Resources/jre   # 符号链接（必须！）
├── mods/
│   ├── BaseMod.jar              # Steam Workshop 版 (v5.56.0, 7.6MB)
│   ├── BaseMod.jar.bak          # GitHub 旧版备份 (v5.5.0, 1.3MB)
│   ├── StSLib.jar               # Steam Workshop 版 (v2.12.0)
│   └── CommunicationMod.jar     # GitHub Releases v1.2.1 (无 Workshop 版)
├── preferences/                 # MTS 运行时存档（从 .app 内复制过来的）
├── saves/                       # MTS 运行时存档
├── SlayTheSpire.app/            # 原版游戏包（不要修改！）
│   └── Contents/Resources/
│       ├── preferences/         # 原版存档（已复制到根目录 preferences/）
│       ├── saves/               # 原版存档（已复制到根目录 saves/）
│       ├── desktop-1.0.jar      # 游戏主 JAR
│       └── jre/                 # 游戏自带的 Java 运行时
├── sendToDevs/logs/             # 游戏日志
└── 杀戮尖塔_Mod.command         # 桌面快捷方式的源文件
```

### Steam 创意工坊 ID

| Mod | Workshop ID | 本地路径 |
|-----|------------|---------|
| ModTheSpire | 1605060445 | `workshop/content/646570/1605060445/ModTheSpire.jar` |
| BaseMod | 1605833019 | `workshop/content/646570/1605833019/BaseMod.jar` |
| StSLib | 1609158507 | `workshop/content/646570/1609158507/StSLib.jar` |

Workshop 根目录：`~/Library/Application Support/Steam/steamapps/workshop/content/646570/`

---

## 2. 配置文件

### CommunicationMod 配置
**路径：** `/Users/longyuhan/Library/Preferences/ModTheSpire/CommunicationMod/config.properties`

```properties
command=/opt/anaconda3/bin/python3 /Users/longyuhan/sts_agent.py --character THE_SILENT --api-key <火山引擎API-KEY>
runAtGameStart=true
```

### ModTheSpire 配置
**路径：** `/Users/longyuhan/Library/Preferences/ModTheSpire/ModTheSpire.properties`

### 桌面启动器
**路径：** `/Users/longyuhan/Desktop/杀戮尖塔_Mod.command`

```bash
#!/bin/bash
GAME_DIR="$HOME/Library/Application Support/Steam/steamapps/common/SlayTheSpire"
cd "$GAME_DIR"
./SlayTheSpire.app/Contents/Resources/jre/bin/java -jar ModTheSpire.jar
```

---

## 3. Agent 代码演进史

### 阶段一：SimpleAgent（纯规则，毫秒级）
- **文件：** 无独立文件，直接 import spirecomm 自带的 `SimpleAgent`
- **特点：** 硬编码优先级列表，无 AI，速度快但策略固定
- **问题：** 中文游戏时卡牌名不匹配导致崩溃

### 阶段二：纯 LLM Agent（每步都问 LLM）
- **特点：** 每步决策都调 API，6-8 秒/步，太慢
- **问题：** 响应慢、token 消耗巨大、容易卡住

### 阶段三：Hybrid v1（选择性 LLM）
- **特点：** 关键决策用 LLM，其余 SimpleAgent
- **问题：** LLM 超时时卡 30 秒（8 秒超时 × 3 次重试）

### 阶段四：Hybrid v2 "Race Mode"（当前版本）
- **核心机制：** LLM 竞速 SimpleAgent，5 秒超时，永不卡住
- **逻辑：**
  1. SimpleAgent 立即给出答案（备用）
  2. 关键决策时 LLM 在后台线程启动
  3. 5 秒内 LLM 返回 → 用 LLM 的
  4. 超时 → 直接用 SimpleAgent 的，不等待
- **LLM 介入场景：** CARD_REWARD、BOSS_REWARD、SHOP_SCREEN、REST、SHOP_ROOM
- **死后学习：** 每局结束 LLM 复盘，教训累积到 `~/sts_lessons.txt`

---

## 4. 当前 Agent 完整代码

**路径：** `/Users/longyuhan/sts_agent.py`

### 依赖
```bash
pip install openai spirecomm
# spirecomm 需从 GitHub 安装:
# pip install git+https://github.com/ForgottenArbiter/spirecomm.git
```

### Python 环境
- 解释器：`/opt/anaconda3/bin/python3` (Python 3.12)
- site-packages：`/opt/anaconda3/lib/python3.12/site-packages`

### spirecomm 关键模块
```
/opt/anaconda3/lib/python3.12/site-packages/spirecomm/
├── ai/
│   ├── agent.py          # SimpleAgent 规则引擎
│   └── priorities.py     # 卡牌/遗物/路线优先级
├── communication/
│   ├── action.py         # 所有 Action 类（PlayCardAction 等）
│   └── coordinator.py    # 与 CommunicationMod 的 stdin/stdout 通信
└── spire/
    ├── game.py           # Game 状态类
    ├── card.py           # Card 类
    ├── character.py      # PlayerClass, Monster, Intent
    ├── screen.py         # ScreenType, 各 Screen 子类
    ├── relic.py          # Relic 类
    ├── potion.py         # Potion 类
    ├── power.py          # Power 类
    └── map.py            # Map, Node 类
```

### 架构设计

```
CommunicationMod (游戏)
    │ JSON via stdin/stdout
    ▼
spirecomm.Coordinator (通信层)
    │ Game object
    ▼
HybridAgent.get_next_action_in_game()
    │
    ├─ _is_critical() → False ──▶ _get_fast_action() → SimpleAgent (0ms)
    │
    └─ _is_critical() → True  ──▶ _race_llm()
                                      │
                                      ├─ thread: _do_llm_call() → 火山引擎 API
                                      └─ main:   join(5s timeout)
                                          │
                                          ├─ LLM returned → use LLM action
                                          └─ timed out    → use SimpleAgent action
    │ Action object
    ▼
spirecomm.Coordinator.send_message(command_string)
    │
    ▼
CommunicationMod (执行指令)
```

### 关键类和方法

| 类/方法 | 作用 |
|---------|------|
| `HybridAgent.__init__()` | 初始化 OpenAI client + SimpleAgent |
| `get_next_action_in_game()` | 主回调，每帧游戏状态更新时调用 |
| `_is_critical()` | 判断是否需要用 LLM |
| `_get_fast_action()` | 调用 SimpleAgent，毫秒级 |
| `_race_llm()` | 线程竞速，5 秒超时 |
| `_do_llm_call()` | 实际 API 调用 + 解析 |
| `_format_critical_state()` | 游戏状态 → 文本 prompt |
| `_parse_llm_response()` | LLM 文本 → Action 对象 |
| `_do_postmortem()` | 死后复盘，学习教训 |
| `get_next_action_out_of_game()` | 主菜单 → 自动开新局 |
| `handle_error()` | 游戏报错时回调 |

---

## 5. 日志与调试

### Agent 运行日志
```
/tmp/sts_agent_debug.log
```
每条日志格式：
```
[HH:MM:SS] FAST: PlayCardAction (screen=NONE, floor=23)
[HH:MM:SS] LLM: 2.1s, tokens=256
[HH:MM:SS] LLM RESP: CARD_REWARD Footwork
[HH:MM:SS] LLM WON: CardRewardAction
```

### LLM 累积教训
```
~/sts_lessons.txt
```
格式：每局复盘后追加，最多保留 3000 字符

### 游戏日志
```
.../SlayTheSpire/sendToDevs/logs/SlayTheSpire.log
```
搜索 CommunicationMod 相关：
```bash
grep -i "communication\|shutting.*child" .../SlayTheSpire/sendToDevs/logs/SlayTheSpire.log
```

---

## 6. 安装过程中犯过的错误（避免踩坑）

1. **GitHub 版 ModTheSpire 和 BaseMod 版本过旧**
   - GitHub MTS v3.6.3，CommunicationMod 需要 v3.18.1+
   - GitHub BaseMod v5.5.0，Workshop 版是 v5.56.0
   - **正确做法：** 从 Steam 创意工坊订阅，复制 JAR

2. **终端路径空格问题**
   - `Application Support` 中有空格，必须用引号包裹
   - 错误：`bash ~/Library/Application Support/...`
   - 正确：`bash "$HOME/Library/Application Support/Steam/..."`

3. **MTS 找不到 desktop-1.0.jar 和 JRE**
   - 脚本没有 `cd` 到游戏目录
   - 需要创建 `jre -> SlayTheSpire.app/Contents/Resources/jre` 符号链接

4. **游戏存档"丢失"**
   - 原版存档在 `.app/Contents/Resources/preferences/`
   - MTS 存档在游戏根目录 `preferences/`
   - 需要从 .app 内复制到根目录

5. **修改 .app 导致 Steam 崩溃**
   - 不要修改 `.app/MacOS/SlayTheSpire` 二进制文件
   - 使用 .command 文件或终端脚本启动

6. **CommunicationMod 默认不启动 agent**
   - 需要在 config.properties 中加 `runAtGameStart=true`

7. **纯 LLM 的 API 超时导致卡住**
   - 每次 API 调用 6-8 秒 + 3 次重试 = 30 秒卡顿
   - **解决方案：** Race Mode，5 秒超时直接跳过

---

## 7. 快速操作指南

### 启动游戏
双击桌面：**杀戮尖塔_Mod.command**

### 更换角色
编辑 `/Users/longyuhan/Library/Preferences/ModTheSpire/CommunicationMod/config.properties`，修改 `--character` 参数：
- `THE_SILENT` — 猎人
- `IRONCLAD` — 铁甲战士
- `DEFECT` — 机器人

### 重启 Agent（不重启游戏）
游戏内 Mod 设置 → CommunicationMod → 点击 "(Re)start external process"

### 清空学习教训
```bash
rm ~/sts_lessons.txt
```

### 查看 API 消耗
```bash
tail -5 /tmp/sts_agent_debug.log
# 查看 total_tokens 字段
```

### 切换模型
修改 config.properties 中的 `--model` 参数，或用环境变量

### 纯 SimpleAgent 模式（无 LLM）
修改 config.properties，去掉 `--api-key` 参数。Agent 检测到无 API key 时会用 SimpleAgent。

---

## 8. API 配置

### 火山引擎 Ark API
- **API Key：** 在 config.properties 中通过 `--api-key` 传入
- **Base URL：** `https://ark.cn-beijing.volces.com/api/v3`
- **当前模型：** `doubao-seed-2-0-pro-260215`（豆包 Seed 2.0 Pro）
- **推理关闭：** `extra_body={"thinking": {"type": "disabled"}}`（已内置）
- **SDK：** `openai` Python 包（兼容 OpenAI 格式）

### 环境变量（备选方案）
```bash
export ARK_API_KEY="your-key-here"
```

---

## 9. CommunicationMod 通讯协议要点

- Agent 必须先发送 `ready\n` 到 stdout
- 游戏状态以 JSON 形式通过 stdin 发给 agent
- Agent 通过 stdout 发送指令
- 卡牌名称必须用游戏的内部英文名（如 "Footwork" 而非 "灵动步法"）
- 游戏语言必须设为 English，否则卡名不匹配
- 指令格式：`play <index>` `end` `proceed` `choose <name>` 等
