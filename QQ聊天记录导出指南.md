# QQ NT (Mac) 聊天记录导出指南

本文档描述如何从 Mac 版 QQ NT 提取并导出指定好友的聊天记录。

---

## 一、环境信息

| 项目 | 路径/值 |
|------|---------|
| QQ 应用 | `/Applications/QQ.app/Contents/MacOS/QQ` |
| QQ 版本 | 6.9.88-44725（wrapper.node MD5: `ea4f2fd398866d5b7c78c339bb4a2ff0`）|
| 加密数据库 | `~/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/nt_qq_*/nt_db/nt_msg.db` |
| Frida 脚本 | `~/qce-native/hook_nt_key.js` |
| 解密脚本 | `~/decrypt_qq_db.py` |
| 导出脚本 | `~/export_qq_to_txt.py` |
| 我的 QQ 号 | 2487751130 |
| 我的昵称 | KAKA |

**依赖工具：**
- Frida 17.9+（已安装：`/Library/Frameworks/Python.framework/Versions/3.13/bin/frida`）
- sqlcipher 3.53+（已安装：`/opt/homebrew/bin/sqlcipher`）
- Python 3 + pycryptodome（`pip3 install pycryptodome`）

---

## 二、操作流程

### 步骤 1：提取数据库密钥

QQ NT 本地数据库使用自定义 `QQ_NT_DB` 格式加密，需要通过 Frida 注入 QQ 进程截获密钥。

**执行命令：**

```bash
# 先关闭正在运行的 QQ
pkill -f "QQ.app/Contents/MacOS/QQ"

# 用 Frida spawn 模式启动 QQ，在数据库打开前安装 hook
frida -f /Applications/QQ.app/Contents/MacOS/QQ \
  -l ~/qce-native/hook_nt_key.js \
  --runtime=v8 \
  2>&1 | tee /tmp/frida_key_output.txt
```

**预期输出：**

```
[+] wrapper.node at 0x...
[+] Hooking nt_sqlite3_key_v2 at 0x...
[+] Hooking nt_sqlite3_key at 0x...
[+] Hooks installed. Waiting for database key access...

========== KEY (v1) #1 ==========
Key length: 16
Key: }>y+iir8*vVZ{)9D
Hex: 7d3e792b696972382a76565a7b293944
=======================================
```

**需要记录的值：** `Hex` 字段（32 位十六进制字符串），即数据库解密密钥。

> **注意：** 密钥在 QQ 版本未更新时通常不变。如果解密失败，说明 QQ 已更新，需要重新执行步骤 1 获取新密钥。如果 `hook_nt_key.js` 中的偏移量失效（出现 "access violation" 或 key 未出现），说明 `wrapper.node` 已更新，届时需要重新逆向寻找 `nt_sqlite3_key` 函数偏移。

> **已知密钥（截至 2026-05-20）：** `7d3e792b696972382a76565a7b293944`

---

### 步骤 2：解密数据库

```bash
python3 ~/decrypt_qq_db.py \
  ~/Library/Containers/com.tencent.qq/Data/Library/Application\ Support/QQ/nt_qq_*/nt_db/nt_msg.db \
  /tmp/nt_msg_decrypted.db \
  "<步骤1获取的Hex密钥>"
```

**解密算法说明（已内置于 decrypt_qq_db.py）：**

- 文件头部 1024 字节为 QQ 自定义头（`SQLite header 3\0` + `QQ_NT_DB` 魔数）
- 跳过头部后，前 16 字节为 PBKDF2 salt
- 密钥派生：`PBKDF2-SHA512(passphrase, salt, 4000 iterations)` → 32 字节 AES-256 key
- 每 4096 字节为一页，使用 AES-256-CBC 解密
- 每页末尾 48 字节为保留区，其中最后 16 字节为 IV
- 第一页解密后替换为标准 SQLite 头（`SQLite format 3\0`）

---

### 步骤 3：导出指定好友聊天记录

**列出所有好友：**

```bash
python3 ~/export_qq_to_txt.py
```

**导出指定好友（支持 QQ 号或昵称）：**

```bash
python3 ~/export_qq_to_txt.py "2901346539"     # 按QQ号
python3 ~/export_qq_to_txt.py "甲贺忍无"         # 按昵称
python3 ~/export_qq_to_txt.py "THEO"           # 按昵称（模糊匹配）
```

**输出文件：** `~/Desktop/QQ聊天记录_<昵称>.txt`

---

## 三、数据库列映射

QQ NT 使用混淆列名，关键列映射如下：

### c2c_msg_table（私聊消息表）

| 列名 | 含义 | 说明 |
|------|------|------|
| 40001 | msg_id | 消息 ID（大整数） |
| 40002 | msg_seq | 消息序列号（**非**时间戳） |
| 40010 | chat_type | 聊天类型（1=好友） |
| 40020 | sender_uid | 发送者 UID |
| 40021 | peer_uid | 聊天对象 UID |
| 40027 | msg_type | 消息类型标志 |
| 40033 | sender_uin | 发送者 QQ 号 |
| **40050** | **msg_time** | **消息时间戳**（Unix 秒） |
| 40090 | alt_text | 备用文本字段 |
| 40093 | alt_text2 | 备用文本字段 |
| 40800 | msg_content | 消息内容（Protobuf BLOB） |

### nt_uid_mapping_table（UID 映射表）

| 列名 | 含义 |
|------|------|
| 48901 | 内部 ID |
| 48902 | UID（如 `u_PuzbAxoLwLdPaG_QW4UNVw`） |
| 1002 | QQ 号（UIN） |

### recent_contact_v3_table（联系人表）

| 列名 | 含义 |
|------|------|
| 40021 | 联系人 UID |
| 40030 | 联系人 QQ 号 |
| 40090 | 备注名（自定义） |
| 40093 | 自己的 QQ 昵称 |
| 40094 | 联系人的 QQ 昵称 |

---

## 四、密钥提取的技术原理

`hook_nt_key.js` 通过 Frida 动态注入 QQ 进程，在 `wrapper.node` 模块中找到 `nt_sqlite3_key` 和 `nt_sqlite3_key_v2` 函数（硬编码偏移），安装拦截器。

当 QQ 启动时打开加密数据库，会调用这些函数传入密钥。Frida 拦截器捕获函数参数（`pKey` 指针 + `nKey` 长度），从内存中读取原始密钥字节，转换为十六进制字符串输出。

**当前有效偏移（QQ 6.9.88-44725 / wrapper.node arm64）：**

| 函数 | wrapper.node 内偏移 |
|------|-------------------|
| `nt_sqlite3_key_v2` | `0x2e24700` |
| `nt_sqlite3_key` | `0x2e246bc` |

如果 QQ 更新导致偏移失效，可用 `hook_all_sqlite.js` 临时替代（hook 系统 `libsqlite3.dylib` 的导出函数），但该方法不一定能捕获密钥（QQ 使用静态链接的 SQLite）。

---

## 五、备用方案（如果 Frida hook 偏移失效）

如果 QQ 更新导致硬编码偏移失效，可通过以下方式恢复：

1. 使用 `nm` 或 `objdump` 反汇编 `wrapper.node`，搜索 `sqlite3_key` 调用
2. 定位 `nt_sqlite3_key_v2(db, zDbName, pKey, nKey)` 函数（ARM64 签名：x0=db, x1=zDbName, x2=pKey, x3=nKey）
3. 更新 `hook_nt_key.js` 中的偏移量
4. 或者等待 qce-native 工具更新

---

## 六、脚本文件清单

| 文件 | 用途 |
|------|------|
| `~/qce-native/hook_nt_key.js` | Frida 密钥提取脚本 |
| `~/qce-native/hook_all_sqlite.js` | Frida 备用提取脚本（扫描所有模块导出） |
| `~/decrypt_qq_db.py` | QQ_NT_DB 解密脚本 |
| `~/export_qq_to_txt.py` | 聊天记录导出脚本（支持指定好友） |

---

## 七、快速操作卡片

```bash
# === 一键完整流程 ===

# 1. 提取密钥（QQ 会重启）
frida -f /Applications/QQ.app/Contents/MacOS/QQ \
  -l ~/qce-native/hook_nt_key.js \
  --runtime=v8 2>&1 | tee /tmp/frida_key.txt
# 从输出中找到 "Hex: 7d3e792b696972382a76565a7b293944" 记下这个值

# 2. 解密数据库（用上一步的 hex key）
python3 ~/decrypt_qq_db.py \
  ~/Library/Containers/com.tencent.qq/Data/Library/Application\ Support/QQ/nt_qq_*/nt_db/nt_msg.db \
  /tmp/nt_msg_decrypted.db \
  "7d3e792b696972382a76565a7b293944"

# 3. 导出（替换为实际好友QQ号或昵称）
python3 ~/export_qq_to_txt.py "甲贺忍无"
```

---

## 八、我的 QQ 号

222

