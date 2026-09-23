# workbuddy-account-migrate

> WorkBuddy 切换账号后对话记录不见了？一键恢复。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: macOS | Windows | Linux](https://img.shields.io/badge/Platform-macOS%20%7C%20Windows%20%7C%20Linux-blue.svg)](https://github.com/xiaoliuzhuan666/workbuddy-account-migrate)
[![Python 3.8+](https://img.shields.io/badge/Python-3.8+-green.svg)](https://www.python.org/)
[![Version 1.6.3](https://img.shields.io/badge/Version-1.6.3-brightgreen.svg)](https://github.com/xiaoliuzhuan666/workbuddy-account-migrate)

**[English](#english) | [中文](#chinese)**

---

<h2 id="chinese">中文</h2>

### 你是不是遇到了这个问题？

WorkBuddy 切换账号 / 重新登录 / 换了腾讯云身份后，**之前的对话记录全没了**？长期记忆、MCP 连接器配置也看不到了？

**数据其实没丢**——它们还在磁盘上，只是 WorkBuddy 用 `user_id` 做了账号隔离，新账号的 UI 看不到旧账号的数据。

本工具一键把旧账号的数据合并到当前登录账号，**对话记录、记忆、连接器全部恢复可见**。

```
切换账号前：                       切换账号后：
┌──────────────────┐              ┌──────────────────┐
│  账号 A           │              │  账号 B           │
│  26 个对话 ✅     │    ──→      │  26 个对话 ❌     │ ← UI 看不到了
│  13KB 记忆 ✅     │              │  13KB 记忆 ❌     │ ← 文件还在磁盘上
│  17 个 MCP ✅     │              │  17 个 MCP ❌     │
└──────────────────┘              └──────────────────┘
                                         │
                                    运行迁移脚本
                                         │
                                         ▼
                                  ┌──────────────────┐
                                  │  账号 B           │
                                  │  26 个对话 ✅     │ ← 合并到当前账号
                                  │  13KB 记忆 ✅     │ ← 追加去重
                                  │  17 个 MCP ✅     │ ← 深度合并
                                  └──────────────────┘
```

### 功能特性

| 特性 | 说明 |
|:---|:---|
| ✅ 交互式向导 | 运行即用，列出所有账号，手动选择目标/源账号，无需知道 user_id |
| ✅ 跨平台路径适配 | v1.4：storage.json 路径自动适配 macOS / Windows / Linux |
| ✅ 国内版 / 国际版 | 交互式向导可选版本，或 `--intl` 参数指定国际版（`~/.workbuddy-ai`） |
| ✅ Session 对话记录迁移 | 修改 SQLite 数据库中的 `user_id` 字段，对话记录全部回归 |
| ✅ Memory 长期记忆合并 | 追加式去重合并，不会丢失当前账号已有记忆 |
| ✅ Connector MCP 连接器合并 | JSON 深度合并，目标账号已有配置保留不动 |
| ✅ 自动备份 + 回滚 | 迁移前自动备份数据库、记忆、连接器，支持一键回滚 |
| ✅ WAL 安全处理 | 迁移前后执行 SQLite checkpoint，确保数据持久化 |
| ✅ 登录态权威识别 | v1.6.3：以 `account-snapshot.json` 的 `primary.uid`（客户端真实登录态，**左侧面板按它过滤**）为权威，`storage.json` 降为兜底；两来源不一致时强烈提示显式传 `--target` |
| ✅ 迁移结果验证 | v1.3：UPDATE 后验证源 user_id 归零，确认迁移成功 |
| ✅ 云端通道映射重置 | v1.6.3：清掉旧账号的 `edge-sync` 映射行，对话按新账号通道重新上传（回滚自动还原）；`--keep-cloud-mapping` 可关闭 |
| 🧰 强制重登辅助脚本 | `scripts/force-relogin.sh`：客户端反复自动登录到错账号时逼它弹登录界面（附带实测局限说明） |
| ✅ 零依赖 | 仅需 Python 3.8+，无第三方包 |

### 快速开始

```bash
git clone https://github.com/xiaoliuzhuan666/workbuddy-account-migrate.git
cd workbuddy-account-migrate
python3 scripts/migrate.py
```

> 受限 shell / 沙箱环境里 `git clone` 可能报「目标路径已存在」，改用 tarball：
> ```bash
> mkdir -p workbuddy-account-migrate && cd workbuddy-account-migrate
> curl -sSL https://codeload.github.com/xiaoliuzhuan666/workbuddy-account-migrate/tar.gz/refs/heads/main \
>   | tar xz --strip-components=1
> ```

运行效果：

```
======================================================================
WorkBuddy 版本选择
======================================================================

  1. 国内版（数据目录 ~/.workbuddy）
  2. 国际版（数据目录 ~/.workbuddy-ai）

请选择 WorkBuddy 版本（输入序号，默认 1）:
```

选择版本后进入账号选择：

```
======================================================================
WorkBuddy 账号迁移向导
======================================================================

请选择迁移方向：先选【目标账号】（接收数据），再选【源账号】（被迁移）

  序号   user_id                                  Sessions     Memory   Connectors
  ------------------------------------------------------------------------
  1      abc12345-6789-...                              18     13.0KB  17mcp/6conn
  2      def67890-1234-...                               7      5.1KB  17mcp/4conn

请选择【目标账号】（接收数据的账号，输入序号）: 1
```

输入序号即可，全程不需要知道 user_id。

**其他模式：**

```bash
# 仅诊断 — 查看所有账号数据分布
python3 scripts/migrate.py --diagnose

# 指定源账号迁移（高级用户）
python3 scripts/migrate.py --source <USER_ID>

# 显式指定目标账号（不依赖当前登录态推断，v1.4 新增）
python3 scripts/migrate.py --source <USER_ID> --target <USER_ID>

# 国际版（数据目录 ~/.workbuddy-ai）
python3 scripts/migrate.py --intl
python3 scripts/migrate.py --intl --diagnose
python3 scripts/migrate.py --intl --source <USER_ID>

# 显式指定数据目录（优先级高于 --intl）
python3 scripts/migrate.py --dir ~/.workbuddy-ai

# 进程检测"查不出来"时按「客户端已关闭」谨慎继续
#   与 --force 的区别：真检测到客户端仍在运行时，它照样拦截
python3 scripts/migrate.py --assume-clients-closed

# 回滚到指定备份
python3 scripts/migrate.py --rollback <TAG>

# 迁移完成后自动重启 WorkBuddy 客户端（macOS），会话列表立即刷新
python3 scripts/migrate.py --source <USER_ID> --yes --restart
```

> **国内版 vs 国际版**：目录结构完全一致，区别在于**数据目录位置**与**登录态来源**：
>
> | | 国内版 | 国际版 |
> |:---|:---|:---|
> | 数据目录 | `~/.workbuddy/` | `~/.workbuddy-ai/` |
> | 登录态权威来源 | 目录内 `storage/skeleton/account-snapshot.json` → `primary.uid`（平台 `storage.json` → `genie.userId` 仅兜底） | 目录内 `storage/skeleton/account-snapshot.json` → `primary.uid`（**不读**平台 `storage.json`） |
>
> 因此同一台机器上装了两个版本时，脚本会按**数据目录**决定读哪份登录态：
> 指向国际版目录时不会去读国内版的 `storage.json`（否则会把国际版数据迁到一个国际版里不存在的账号下）。
> **目录优先级**：`--dir` > `--intl` > 自动探测（`~/.workbuddy-ai` 存在且非空时判为国际版，否则国内版）。交互式向导还会让你确认一次版本。

### 备份目录布局

> ⚠️ v1.6 起**单对话迁移**的备份挪到了 `migrate_backups/session/` 子目录，
> 与 `scripts/migrate.py` 的整账号备份（直接放 `migrate_backups/` 根下）分开——
> 两者的 `meta.json` 格式不同，混在一个目录里会让 `--backups` 列出、
> `--rollback` 撞上 `KeyError`。
>
> | 路径 | 内容 | 回滚命令 |
> |:---|:---|:---|
> | `~/.workbuddy/migrate_backups/<TAG>/` | 整账号备份 | `scripts/migrate.py --rollback <TAG>` |
> | `~/.workbuddy/migrate_backups/session/<TAG>/` | 单对话备份 | `scripts/migrate_session.py --rollback <TAG>` |
>
> **旧位置的单对话备份仍然能被 `--backups` / `--rollback` 找到**（两个目录都会扫），
> 但如果外部脚本里硬编码了备份路径，请注意这次布局变化。
>
> `--backup-dir`（**只有 `migrate_session.py` 有这个参数**，`migrate.py` 没有）
> 的语义是「**额外**加入一个搜索根」而不是「限定只搜它」：
> 新建的备份写进 `<指定目录>/session/`（同样套一层命名空间，避免你把自定义目录
> 指向 `migrate_backups` 时和整账号备份挤在一起），查找 / 回滚时仍会回退扫描
> 上面两个标准目录——命中的备份不在你指定的目录里时，脚本会显式告警，
> 避免误滚了不相干的旧备份。

### 退出码

下表两个脚本共用，但 **`3` 只有 `migrate.py` 会产生**（整账号迁移才有"源账号空"这个概念；
单对话脚本遇到源/目标版本目录没数据时是 `1`）。

| 退出码 | 含义 |
|:---:|:---|
| `0` | 确实迁移 / 回滚了东西 |
| `1` | 出错（含：`migrate_session.py` 的源/目标版本目录里根本没有库 = 版本选错） |
| `2` | 「客户端必须关闭」检测拦截（检测到客户端仍在运行，或检测不可信且未给 `--force` / `--assume-clients-closed`） |
| `3` | **仅 `migrate.py`**：无数据可迁，本次未做任何改动。涵盖两种情况——源账号确实没有 session / memory / connector；或源有数据但与目标完全一致、合并后没有任何新增内容 |

> ⚠️ 自动化脚本不要把「退出码 0」当成「一定迁了东西」，也不要把「3」当成失败：
> 它表示本次没有产生任何改动。判读建议：`0` → 成功，`3` → 跳过，其他 → 失败。
>
> ⚠️ **破坏性变更**：v1.6.1 之前，源账号没数据时 `migrate.py` 返回 `0`。
> 如果你在 CI / 自动化里按 `rc == 0` 判定成功，升级后会看到原本"成功"的场合变成 `3`
> ——这不是新出现的失败，只是以前把"跳过"报成了"成功"。

### 迁移内容

| 数据类型 | 存储位置 | 隔离方式 | 是否迁移 | 迁移策略 |
|:---|:---|:---|:---:|:---|
| Session 对话记录 | `workbuddy.db` sessions 表 | `user_id` 字段 | ✅ | UPDATE user_id |
| 长期记忆 Memory | `~/.workbuddy/memory/{uid}_memory.md` | 按文件名 | ✅ | 追加去重合并 |
| Connector 连接器配置 | `~/.workbuddy/connectors/{uid}/mcp.json` | 按子目录 | ✅ | JSON 深度合并 |
| **云端通道映射** | `edge-sync-mapping*.db` 的 `msg_channel` | 按 `convmsg:{uid}` 记账 | ✅ **v1.6.3 新增** | 删除旧账号的映射行，让 EdgeSync 按新账号通道重传（回滚时从备份还原） |
| todos / tasks | `todos/{sessionId}.json`、`tasks/{sessionId}/` | 无隔离（按 sessionId） | ❌ | 不用迁；面板看不到任务是客户端只读当前 session 内存导致的 |
| Skills 技能 | `~/.workbuddy/skills/` | 无隔离 | ❌ | 全局共享，无需迁移 |
| Automations 定时任务 | `workbuddy.db` automations 表 | 无 user_id | ❌ | 全局共享，无需迁移 |
| Settings / MCP / Plugins | 全局配置文件 | 无隔离 | ❌ | 全局共享，无需迁移 |
| inspiration | `~/.workbuddy/inspiration/{uid}/` | 按 uid 子目录 | ❌ | 需要时手动 `mv` 到目标 uid 目录 |
| security | `~/.workbuddy/security/{uid}/` | 按 uid 子目录 | ❌ | 安全检测模块的加密库，**不要动** |
| storage/user-\<uid\>* | `~/.workbuddy/storage/` | 按 uid 子目录 | ❌ | 客户端 UI 偏好等，未迁移 |

> **为什么要管「云端通道映射」**：本地 `user_id` 改对只是让**本机**看得到；对话在云端仍挂在旧账号的 `convmsg:{旧uid}` 通道下，EdgeSync 会认为"已同步过"而不重传 —— 结果换台设备登录新账号时看不到这些历史。v1.6.3 起迁移会自动清掉旧账号的映射行（删前整库备份），回滚时自动还原。

### 单对话跨版本迁移（v1.6.0）

上面是「整个账号」的迁移。如果你只想把**某一个对话**从国内版搬到国际版（或反过来），用另一个脚本：

```bash
python3 scripts/migrate_session.py                 # 交互式向导，一步到位
python3 scripts/migrate_session.py --list          # 先看看国内版有哪些对话
python3 scripts/migrate_session.py --from domestic --to intl --session-id <SESSION_ID>
```

**与整账号迁移的区别**

| | `migrate.py` | `migrate_session.py` |
|:---|:---|:---|
| 范围 | 整个账号（全部对话 + 记忆 + 连接器） | **一个对话** |
| 版本 | 同一版本内 | **支持国内 ⇄ 国际** |
| 默认语义 | **归属转移**（`UPDATE sessions.user_id`，源账号不再看到这些对话；记忆/连接器是合并） | **移动（源删除）**，可 `--mode copy` |

> ⚠️ `migrate.py` 的"合并"只针对 Memory / Connectors：**对话是改 `user_id` 转移归属**，
> 迁移后源账号下就看不到它们了（数据没被删除，只是归属变了）。想两边都保留请改用
> `scripts/migrate_session.py --mode copy`。
>
> ⚠️ `migrate.py` 只在**一个数据目录内**工作（`--intl` 只是把目标目录切到 `~/.workbuddy-ai`，
> 不是"跨版本搬数据"）。它**不搬** `projects/{slug}/*.jsonl` 对话正文与 `tasks/`：
> 同一数据目录内这些内容本来就按 session 共享、不需要搬；但如果你想把这些对话搬到
> **另一个版本**的数据目录，必须用 `scripts/migrate_session.py`——否则目标版本里
> 只会多出一条 session 行，正文为空（打开是空对话）。
>
> **`--mode copy` 的语义**（migrate_session.py）：无论源对话属于哪个账号，copy 都会**保留源**，
> 在目标账号下克隆出一份新对话。此前跨账号 + copy 走的是改 `user_id`（归属转移），
> 源账号会丢失该对话，与"保留源"矛盾，已修正。

**⚠️ 迁移与回滚前都必须关闭两个版本的 WorkBuddy 窗口**，脚本会检测并拒绝执行（`--rollback` 同样检测——回滚也要改库 + 删文件，客户端在跑时内存缓存会把回滚结果覆盖回去）。原因：数据还在 WAL 里没落盘、客户端内存缓存会覆盖你的写入。

**一个对话实际包含哪些东西**（少一样客户端就显示异常）：

| 数据 | 位置 | 说明 |
|:---|:---|:---|
| session 行 | `workbuddy.db` sessions 表 | 跨库插入，`user_id` 改写为目标版本账号 |
| 用量行 | `session_usage` 表 | token 统计 |
| 工作区登记 | `workspaces` 表 | 否则客户端找不到路径 |
| **对话正文** | `projects/{slug}/{id}.jsonl` | **不复制的话对话是空的** |
| 工具结果 | `projects/{slug}/{id}/tool-results/*.txt` | 大工具输出外溢目录，缺失会丢内容 |

**冲突处理**（目标已存在时询问，并展示差异帮你判断）：

```
⚠️  目标版本已存在【标题相同】但 ID 不同的对话
  原因：标题一致但 id 不同，很可能是同一段对话被迁移过一次，
       再次迁移会在客户端里出现两条看起来一样的对话。

  指标          目标现有（将被覆盖）        源（将写入）
  ─────────────────────────────────────────────────────────
  ★ 最后活动    09-10 08:26                09-10 15:02
  ★ 消息数      5 条（我 5 / AI 0）         26 条（我 3 / AI 23）
  ★ 对话大小    453 B · 5 行               605.6 KB · 140 行
    最后提问    老的提问内容                …
  ─────────────────────────────────────────────────────────
  → 源比目标新 6 小时 36 分钟，消息多 21 条，内容远超目标（约 1369 倍）
  → 建议：覆盖（源更新且更完整）

  请确认 [y] 覆盖 / [s] 不覆盖（跳过该对话） / [n] 不操作（取消）:
```

- **硬冲突**（ID 相同）：`覆盖` / `不操作`
- **软冲突**（标题相同、ID 不同）：`覆盖` / `不覆盖` / `不操作`
- 覆盖时始终以**源的 ID** 写入并删除目标那条旧记录，保证正文文件名与 ID 一致

**同版本复制**（`--from` 与 `--to` 相同）

同一版本内有两种语义，脚本会自动判断：

| 情况 | 行为 |
|:---|:---|
| 源对话属于**别的账号** | 只把 `user_id` 改到当前账号（归属转移） |
| 源对话**已属于当前账号** | 克隆出一条新对话：新的 session id，标题加「（副本）」 |

克隆会一并处理三件容易漏掉的事：正文文件按新 id 改名、正文内部每条消息的
`"sessionId"` 全部改写为新 id、工具结果目录 `tool-results/` 一起复制。
原对话保持不变，回滚只删副本、不动原对话。

**参数**

| 参数 | 说明 | 默认 |
|:---|:---|:---|
| `--from` / `--to` | 源/目标版本 `domestic`\|`intl` | `domestic` |
| `--list` | 列出源版本的对话（配合 `--query` 过滤） | - |
| `--query` | 按标题 / 工作目录 / id 过滤列表 | - |
| `--session-id` | 对话 id（支持前缀） | - |
| `--mode` | `move`（迁移后删源）/ `copy`（保留源并克隆一份到目标账号） | **`move`** |
| `--on-conflict` | `ask`/`skip`/`overwrite`/`newer`（无终端询问时 `ask` 降级为 `skip`） | `ask` |
| `--target-uid` | 手动指定目标版本的 user_id（不传则从 account-snapshot 推断）。**形态不像 UUID 时会先警告**（拼错会把对话挂到不存在的账号下，表现同样是"迁移成功但对话消失"）；非交互模式下还需再确认一次 | - |
| `--dry-run` | 只打印计划不写盘（不会再弹冲突询问） | 关 |
| `--yes` | 非交互模式：跳过「确认执行 / 确认回滚」这类询问。**冲突处理不受它控制**——由 `--on-conflict` 决定，`ask` 在无终端时降级为 `skip` | 关 |
| `--force` | 跳过"客户端必须关闭"检测 | 关 |
| `--assume-clients-closed` | 比 `--force` 温和：只在**进程检测本身失败**（查不出结果）时按「客户端已退出」继续；**真检测到客户端仍在运行时照样拦下** | 关 |
| `--backups` / `--rollback TAG` | 查看备份 / 回滚（`--rollback` 支持标签前缀） | - |
| `--full` | 与 `--rollback` 配合：整库恢复（库 + 正文 + 任务）。**必须**与 `--rollback` 同用，单独给会报错退出 | 关 |
| `--backup-dir` | 备份目录（**仅本脚本有**；`migrate.py` 无此参数）：新备份写进 `<该目录>/session/`；查找/回滚时作为**额外**搜索根，仍会回退扫描两个版本的标准目录 | - |

> 未给 `--session-id` 时进入交互式向导；此时 `--mode` / `--yes` / `--dry-run` / `--target-uid` / `--backup-dir` / `--assume-clients-closed` / `--on-conflict` / `--from` / `--to` / `--query` 同样会透传（此前在向导路径被静默丢弃）。

回滚精确到单条，不影响其他对话：`python3 scripts/migrate_session.py --rollback <TAG>`。

> ⚠️ **平台说明**：单对话迁移的**完整跨版本链路仅 Windows 实测通过**（Windows 11 + Python 3.13）。
> macOS 已部分验证（2026-09-21）：`--list` 在国内版真实数据 fixture 上工作正常（含中文名渲染、
> 大小统计），脚本本身是跨平台的（路径走 pathlib、进程检测 Windows 用 `tasklist`、其他平台用
> `ps`），但跨版本迁移全流程在 macOS / Linux 未经实测，欢迎提 Issue 反馈。

### 工作原理

**Step 1：自动诊断** — 从数据库、Memory 文件、Connector 目录三个来源自动发现所有账号。当前登录账号的判定顺序（v1.6.3）：**① `{数据目录}/storage/skeleton/account-snapshot.json` 的 `primary.uid`** → ② `storage.json` 的 `genie.userId` → ③ DB 中 session 数最多的 user_id。三者与 daemon 日志里的面板 uid 会一起打印出来，不一致时明确告警。
> v1.4~v1.6.2 曾以 `storage.json` 为唯一权威，但国内版实测它与客户端真实登录态**可能长期是两个不同的 uid**，导致迁移方向每次判错、迁完左侧列表仍然空白（2026-09-22 实例：用户为此来回折腾 6 次）。**判定口径已改为「客户端登录态优先」**；也别用"最新 session"推断——旧账号切换前的最后一条 session 可能比当前账号更新。

**Step 2：安全备份** — 迁移前自动备份到 `~/.workbuddy/migrate_backups/{timestamp}_{uid前8位}/`（同一秒重复迁移会自动加序号，不会覆盖前一份备份）

**Step 3：执行迁移** — Session 用 `UPDATE user_id`，Memory 按 **`memoryBlock` 语义块**去重后追加（结构化 Memory；无 `RAW_JSON` 块的旧格式才退回按行去重），Connector JSON 深度合并

**Step 4：持久化 + 验证** — 迁移后执行 WAL checkpoint 确保数据落盘，验证源 user_id 归零确认迁移成功

**Step 5：重启提示** — 提示重启 WorkBuddy 客户端，UI 刷新缓存后数据可见

### 兼容性

| 平台 | 状态 |
|:---|:---|
| WorkBuddy 国内版 (Windows) | ✅ 已测试（Windows 11 + Python 3.13，数据目录 `~/.workbuddy/`） |
| WorkBuddy 国际版 (Windows) | ✅ 已测试（v1.5，数据目录 `~/.workbuddy-ai/`，使用 `--intl` 参数） |
| WorkBuddy 国内版 (macOS) | ⚠️ 理论支持，**未实测**（`~/Library/Application Support/...` 路径逻辑沿用跨平台实现） |
| WorkBuddy 国内版 (Linux) | ⚠️ 理论支持，**未实测**（`XDG_CONFIG_HOME` 路径） |
| WorkBuddy 国际版 (macOS / Linux) | ⚠️ 理论支持，**未实测** |
| CodeBuddy CLI | ❌ 不适用（见下方说明） |

> ⚠️ 本项目**目前仅 Windows 实测通过**（Windows 11 + Python 3.13）。上面标"未实测"的平台
> 走的是同一套 pathlib 路径推导与进程检测（Windows 用 `tasklist`、其他平台用 `ps`），
> 但没有任何实测记录，欢迎提 Issue 反馈结果。

> **国内版 vs 国际版**：国内版数据目录为 `~/.workbuddy/`，国际版为 `~/.workbuddy-ai/`。迁移工具默认操作国内版，加 `--intl` 参数操作国际版。交互式向导会提示选择版本。

**为什么不支持 CodeBuddy CLI？** CodeBuddy CLI 的记忆按项目维度隔离（`~/.codebuddy/memories/{project-id}/`），对话记录按 `{sessionId}.jsonl` 独立文件存储，不依赖 `user_id` 过滤，**不存在账号切换后数据丢失的问题**。如果你是 CodeBuddy 用户遇到类似问题，欢迎提 Issue。

### 安全规则

1. **必须先备份** — 迁移前自动创建备份，不可跳过
2. **源 ≠ 目标** — 防止自我覆盖
3. **Memory 追加不覆盖** — 不会丢失当前账号已有记忆
4. **Connector 深度合并** — 保留目标账号已有配置
5. **迁移后重启** — WorkBuddy 客户端有内存缓存
6. **备份 7 天可清** — 手动删除即可

### 回滚

```bash
# 整账号备份（migrate.py）
ls ~/.workbuddy/migrate_backups/          # 国内版（国际版在 ~/.workbuddy-ai/migrate_backups/）
python3 scripts/migrate.py --rollback 20260525170000_abc12345

# 单对话备份（migrate_session.py，v1.6 起放在 migrate_backups/session/）
python3 scripts/migrate_session.py --backups
python3 scripts/migrate_session.py --rollback 20260922000000_domestic2intl_12345678

# 整库恢复（migrate_session.py）：把迁移波及到的库整体还原到迁移那一刻
python3 scripts/migrate_session.py --rollback 20260922000000_domestic2intl_12345678 --full
```

> ℹ️ **`--full` 的还原范围**：整库恢复会把迁移**波及到**的数据库整体还原到迁移那一刻
> （库 + 正文 + 任务），所以迁移之后新增的对话与消息会一并丢失，确认前脚本会明确提示。
> `copy` 模式（`--mode copy`）下源版本是用户特意保留的、迁移本身没改动过它，
> `--full` 也**不会**还原源侧（打印 ⏭️ 说明跳过），只还原目标版本。

> ⚠️ **备份目录含个人数据，用完请及时清理**。备份里可能有：
> `workbuddy.db`（整库快照）、`mcp.json` / `connector-states.json`（可能含 token）、
> `.master.key`、`{uid}_memory.md`、以及对话正文 `*.jsonl` 与 `tool-results/`。
> 确认不再需要回滚后，建议**移入回收站**而不是直接删除，例如：
>
> ```bash
> # 先看清有哪些（两个版本各自一个目录）
> ls -la ~/.workbuddy/migrate_backups/ ~/.workbuddy-ai/migrate_backups/
> # macOS：移入废纸篓
> trash ~/.workbuddy/migrate_backups/20260525170000_abc12345
> # Linux：移入回收站
> gio trash ~/.workbuddy/migrate_backups/20260525170000_abc12345
> # Windows PowerShell：移入回收站
> #   Add-Type -AssemblyName Microsoft.VisualBasic
> #   [Microsoft.VisualBasic.FileIO.FileSystem]::DeleteDirectory(
> #     "$env:USERPROFILE\.workbuddy\migrate_backups\20260525170000_abc12345",
> #     'OnlyErrorDialogs', 'SendToRecycleBin')
> ```
>
> 迁移脚本**不会**自动清理旧备份 —— 什么时候不再需要回滚由你自己判断。

### 竞品对比

| 项目 | 定位 | 同平台账号切换 | Session 迁移 | Memory 迁移 |
|:---|:---|:---:|:---:|:---:|
| **本项目** | 同平台账号切换数据合并 | ✅ | ✅ | ✅ |
| [ai-memory-sync](https://github.com/supercrzy/ai-memory-sync) | 跨设备记忆同步 | ❌ | ❌ | ✅ |
| [claw-migrate](https://github.com/citriac/claw-migrate) | 跨平台记忆迁移 | ❌ | ❌ | ✅ |
| [workbuddy-manager](https://github.com/starsss0416/workbuddy-manager) | 本地会话管理 | ❌ | ✅ 本地 | ❌ |

**本项目填补的空白**：跨平台迁移和跨设备同步都有人做了，但**同平台账号切换后的数据合并**是唯一没人覆盖的场景。

### FAQ

**Q: WorkBuddy 切换账号后对话记录 / 历史记录真的没丢吗？**

A: 没丢。数据文件全部还在磁盘上，只是 UI 按 `user_id` 过滤导致看不到。本工具把这些数据合并到当前账号下即可恢复可见。

**Q: 迁移后旧账号数据还在吗？**

A: Session 的 `user_id` 被改为新账号，所以在旧账号的 UI 下不可见了。Memory 和 Connector 的源文件仍然保留，可手动清理。

**Q: 支持双向迁移吗？**

A: 支持。从 B 迁到 A 后，可以登录 B 再执行 `--source <A的user_id>`，或者用 `--target` 直接指定目标账号、无需切换登录。Memory 按语义块（`memoryBlock`）去重、Connector 按 key 合并，反向迁移不会产生重复内容。注意：反向迁移会把 A 名下**所有** session 一起迁走（包括 A 原有的）；如果只是想撤销上一次迁移，用 `--rollback` 回滚更干净。

**Q: 支持 CodeBuddy CLI 吗？**

A: 暂不支持。CodeBuddy CLI 不存在账号切换数据丢失的问题。详见上方「兼容性」章节。

**Q: macOS / Linux 可以用吗？**

A: 路径层面支持。v1.4 起 storage.json 路径已按平台自动适配（macOS `~/Library/Application Support/...`、Windows `%APPDATA%`、Linux `XDG_CONFIG_HOME`），v1.6.1 起登录态改以数据目录内的 `account-snapshot.json` 为准，不再依赖平台 `storage.json` 的具体路径。

**但只有 Windows 真正实测过**（Windows 11 + Python 3.13），macOS / Linux 属于"代码跨平台、没在本体上跑过"。进程检测在 Windows 用 `tasklist`、其他平台用 `ps`，后者完全没有实测覆盖。详见「兼容性」章节。

**Q: 在 WorkBuddy / AI 助手的会话里运行脚本报错 `PermissionError: [Errno 13]` / mkdir 异常？**

A: 部分 AI 助手的会话内 Shell 会通过 `PYTHONPATH` 注入沙箱 shim（如 sitecustomize.py），劫持所有 Python 进程的文件操作——备份目录已存在时 `mkdir(exist_ok=True)` 也会抛异常，迁移还没开始就崩（2026-09-20 实战踩坑，SKILL.md 有记录）。解法是剥掉该变量运行（脚本仅用标准库，不需要它）：

```bash
env -u PYTHONPATH python3 scripts/migrate.py
```

### 项目结构

```
workbuddy-account-migrate/
├── README.md                              # 本文档
├── LICENSE                                # MIT 许可证
├── .gitignore                             # 排除敏感文件
├── SKILL.md                               # WorkBuddy Skill 描述符
├── TOPICS.md                              # 专题索引（给 Skill / AI 检索用）
├── scripts/
│   ├── migrate.py                         # 整账号迁移（同版本内）
│   └── migrate_session.py                 # 单对话迁移（支持跨版本，v1.6）
├── tests/
│   ├── prepare_fixture.py                 # 构造临时测试 fixture（只读复制真实数据）
│   └── run_tests.py                       # 端到端 + 单元级测试（用例数随 fixture 内容浮动，以运行末尾输出为准）
└── references/
    └── data_isolation_map.md              # 数据隔离全景图
```

> 测试全部在临时 fixture 中运行，不会触碰真实数据目录。
> `python3 tests/run_tests.py` 即可复现全部验证。
>
> ⚠️ **完整跨版本测试链仅 Windows 实测通过**（Windows 11 + Python 3.13）。fixture 复制的是本机
> 真实 WorkBuddy 数据，跨版本测试用例要求本机**同时有国内版和国际版**数据（fixture 缺某版目录
> 时会如实跳过并报告）。macOS 实测：仅有国内版数据时 fixture 只能造出 domestic 一半，
> `run_tests.py` 会因缺少国际版库中止——这是环境限制而非脚本缺陷。

### 贡献

- Bug 报告 / 功能请求 → [Issues](https://github.com/xiaoliuzhuan666/workbuddy-account-migrate/issues)
- 代码贡献 → 提交 PR，请确保无硬编码的 user_id 或 Token
- Windows / Linux 实测反馈 → 欢迎 Issue

### 已知限制

| 限制 | 说明 |
|:---|:---|
| Memory 多语义块 | 结构化 Memory 迁移是**追加**一个 `RAW_JSON` 块。WorkBuddy 客户端是否合并读取多个块未经验证——若客户端只读首块，迁移过去的记忆在文件里存在但 UI 不显示。脚本追加后会提示你去客户端确认 |
| 非 Windows 进程检测 | 客户端"必须关闭"检测在 Windows 用 `tasklist` 实测有效；macOS / Linux 走 `ps`，Electron 应用的进程名可能是包名，存在漏报，必要时用 `--force` 并自行确认 |
| 软冲突多条同名 | 目标里有多条同标题对话时，脚本**只处理其中一条**（会打印其余的 id 与标题提醒），剩下的需要在客户端里手动清理 |
| 会话 `cwd` 为空 | 极少数会话记录里 `cwd` 为空，正文目录只能靠源侧目录名回退；若连这也取不到，脚本会中止而不是静默放错位置 |
| 正文 id 改写范围 | 只改写 `"sessionId":"..."` 字段值。消息正文里引用到的旧 id（日志、路径）保持原样——那是用户可见内容，不应被改 |

### 更新日志

#### v1.6.3 (2026-09-22)

**修复：目标账号会判成"另一个账号"，迁移"成功"但左侧列表依旧空白**

- **判定口径翻转**：`get_current_user_id()` 改为 **`account-snapshot.json` → `primary.uid` 优先**，`storage.json` 的 `genie.userId` 降为第二优先级、DB 会话数兜底。原因见下
- **踩到的坑**：国内版 `storage.json` 记的账号（扩展侧）与客户端真实登录态可以是两个不同 uid，且**长期不一致**。工具当时按 `storage.json` 选目标，每次都把数据并到「面板看不到」的那个账号，用户重启后依然是空列表，来回试了 6 次
- **diagnose 重排**：并列打印四个信号 —— 客户端登录态(account-snapshot) / 扩展侧记录(storage.json) / daemon 最近一次 `listSessions` 的 uid / DB 各账号会话数；不一致时直接给出「迁移务必带 `--target`」的结论
- **migrate 新增 Phase 4.5 一致性检查**：目标账号 ≠ 客户端登录态时，明确列出两条补救路径（切账号看 / 回滚后加 `--target` 重跑），不再只提示"迁移完成"
- **迁移前告警**：打印目标账号时同时打印客户端登录态，`--target` 与登录态不符会先警告（`--target` 是手动指定时提示"确认有意为之"）
- **备份 meta 补字段**：`meta.json` 除 `target_uid` 外新增 `source_uid`、`client_login_uid`、`storage_json_uid`、`session_counts` —— 事后复盘"当时到底登在哪个账号"全靠它
- **建议命令带 `--target`**：`--diagnose` 的迁移建议按数据量排序，并直接输出含 `--target` 的完整命令（0 数据的账号不再生成命令）
- **修 daemon 日志解析**：嵌套 JSON 的引号是转义的（`\"userId\":\"...\"`），原正则匹配不到面板 uid

**（沿用 v1.6.2）沙箱 shim 与 `--restart`**：脚本启动自动剥离 `PYTHONPATH`；`--restart` 迁移后自动重启客户端。

**新增：云端通道映射重置（`edge-sync-mapping*.db`）**

- 本地 `user_id` 改对只让**本机**看得到；对话在云端仍挂在 `convmsg:{旧uid}` 通道下，EdgeSync 认为"已同步过"不会重传 → **换台设备登录新账号看不到这些历史**
- 迁移时自动删除旧账号的映射行（只删映射、不碰对话内容，删前整库备份到 `<备份>/edge-sync/`），下次启动客户端由 EdgeSync 重新上传
- `--rollback` 会一并还原映射库；`--keep-cloud-mapping` 可跳过本步骤

**新增：`scripts/force-relogin.sh`（强制重登辅助）**

- 客户端反复自动登录到错账号时，移走 `storage/skeleton/account-snapshot.json` 逼它弹登录；`--restore` 还原，默认检测客户端是否已退出
- 脚本头部如实写明实测局限：**能拿到正确账号，但挡不住约 1 分钟后的自动回切**（回切源头在加密凭据层），因此长期方案是把数据并到客户端实际登录的账号

**交互向导标记目标账号**

- 账号列表给「客户端登录态」那一行加 `← 客户端登录态（面板按它过滤）` 标记，并提示**目标通常就选它**，避免人工选错方向
- 迁移前打印目标账号时同时打印客户端登录态；两者不符先告警

**测试与文档**

- `tests/run_tests.py`：本机缺国际版数据时显式跳过（退出码 0），不再抛 `sqlite3 "unable to open database file"` traceback
- README/SKILL 补「迁移边界」清单（todos / inspiration / security / storage/user-* 为何不迁）、「登录态有两个来源」章节、tarball 安装方式


#### v1.6.2 (2026-09-22)

**修复：WorkBuddy 会话内运行被沙箱 shim 劫持导致迁移崩溃**

- WorkBuddy 会话的 Bash 里运行时，注入的 `PYTHONPATH` 指向沙箱 shim（sitecustomize.py）会劫持 `Path.mkdir`：即使传 `exist_ok=True`，目录已存在也抛 `PermissionError EEXIST`，迁移在备份阶段就崩溃（托管 Python 和系统 Python 都中招）
- 现在脚本启动时自动剥离 `PYTHONPATH` 并 re-exec 自身（等价于 `env -u PYTHONPATH python3 migrate.py ...`，但无需记住特殊用法）；所有 `mkdir` 处保留 `exists()` 先判断作为双保险

**新增：`--restart` 迁移完成后自动重启客户端**

- `python3 migrate.py --source <UID> --yes --restart`：迁移/回滚完成后延迟数秒自动退出并重新拉起 WorkBuddy（macOS），左侧会话列表立即刷新，不用手动重启
- 采用后台延迟执行（脱离进程组），脚本先输出完整结果再触发重启；在 WorkBuddy 会话内调用时当前 AI 会话会中断，属预期行为。Windows / Linux 提示手动重启

#### v1.6.1 (2026-09-21)

**修复：`migrate_session.py` 行为与文档不符 / 静默失败**（全部改动来自 [@bukall](https://github.com/bukall)，PR #5）

- `--mode copy` 跨账号时不再退化成"改 `user_id` 转移归属"：copy 一律保留源，克隆一份归属到目标账号
- 会话 `cwd` 为空时不再把正文静默写到 `projects/` 根目录（客户端按 `projects/<slug>/<id>.jsonl` 找，
  放根目录等于迁移成功却打不开）。现在按「行 cwd → 会话画像 cwd → 源正文所在目录名」三级回退，
  仍无法确定则中止并提示回滚
- 列表大小统计改为递归累加（`--list` 这一处漏改，`tool-results/` 仍被算成 ~4KB）
- 顶层补上 `sqlite3.Error` 分支：跨库插入撞上目标库新增的 NOT NULL 无默认值列时，
  给出"用 --rollback 回滚"的可操作提示，而不是 traceback
- 软冲突覆盖：被删掉的那条目标对话的 `session_usage` 现在会一起备份与回滚
- 客户端进程检测失败不再静默当成"已关闭"（要求显式 `--force`）；非 Windows 额外用
  `ps -eo args=` 匹配完整命令行，避免 Electron 包名漏报
- 正文 id 改写只动 `"sessionId":"..."` 字段值：以前整行 replace 会把消息正文里
  恰好出现同串 id 的文本（日志、路径）一起改坏

**修复：`migrate.py` 的数据安全与解析问题**

- 备份数据库改用 sqlite backup API（带 WAL），不再 `shutil.copy2` 主库文件
  ——客户端没退出时后者拿到的是陈旧快照；失败时退回文件复制并明确告警
- `PRAGMA wal_checkpoint` 的 busy 标志现在会判断：checkpoint 没做完时不再宣称"验证通过"
- Memory 结构化迁移改为与目标里**所有**已有 `memoryBlock` 比对，重复执行不再重复追加同一块
- `_get_storage_json_path()` 改为受 `WORKBUDDY_MIGRATE_HOME` / `--dir` 约束，
  不再去读真实机器的平台 storage.json；`STORAGE_JSON` 为 `None` 时不再直接 `open()`
- `get_connector_info()` 显式按 utf-8 读取 `mcp.json`（中文配置此前被静默吞掉显示 0 个 server）
- user_id 判定改用 UUID 形态匹配，不再"目录名含连字符就算账号"
- `--rollback` 支持 `--yes` 跳过确认；与 `--source` 等参数同时给出时明确报错，不再静默优先

#### v1.6.1 (2026-09-21)

**修复：`migrate_session.py` 行为与文档不符 / 静默失败**

- `--mode copy` 跨账号时不再退化成"改 `user_id` 转移归属"：copy 一律保留源，克隆一份归属到目标账号
- 会话 `cwd` 为空时不再把正文静默写到 `projects/` 根目录（客户端按 `projects/<slug>/<id>.jsonl` 找，
  放根目录等于迁移成功却打不开）。现在按「行 cwd → 会话画像 cwd → 源正文所在目录名」三级回退，
  仍无法确定则中止并提示回滚
- 列表大小统计改为递归累加（`--list` 这一处漏改，`tool-results/` 仍被算成 ~4KB）
- 顶层补上 `sqlite3.Error` 分支：跨库插入撞上目标库新增的 NOT NULL 无默认值列时，
  给出"用 --rollback 回滚"的可操作提示，而不是 traceback
- 软冲突覆盖：被删掉的那条目标对话的 `session_usage` 现在会一起备份与回滚
- 客户端进程检测失败不再静默当成"已关闭"（要求显式 `--force`）；非 Windows 额外用
  `ps -eo args=` 匹配完整命令行，避免 Electron 包名漏报
- 正文 id 改写只动 `"sessionId":"..."` 字段值：以前整行 replace 会把消息正文里
  恰好出现同串 id 的文本（日志、路径）一起改坏

**修复：`migrate.py` 的数据安全与解析问题**

- 备份数据库改用 sqlite backup API（带 WAL），不再 `shutil.copy2` 主库文件
  ——客户端没退出时后者拿到的是陈旧快照；失败时退回文件复制并明确告警
- `PRAGMA wal_checkpoint` 的 busy 标志现在会判断：checkpoint 没做完时不再宣称"验证通过"
- Memory 结构化迁移改为与目标里**所有**已有 `memoryBlock` 比对，重复执行不再重复追加同一块
- `_get_storage_json_path()` 改为受 `WORKBUDDY_MIGRATE_HOME` / `--dir` 约束，
  不再去读真实机器的平台 storage.json；`STORAGE_JSON` 为 `None` 时不再直接 `open()`
- `get_connector_info()` 显式按 utf-8 读取 `mcp.json`（中文配置此前被静默吞掉显示 0 个 server）
- user_id 判定改用 UUID 形态匹配，不再"目录名含连字符就算账号"
- `--rollback` 支持 `--yes` 跳过确认；与 `--source` 等参数同时给出时明确报错，不再静默优先

**修复：高危数据安全问题（第三轮）**

- 备份 `meta.json` 改为**增量落盘**：每个破坏性步骤（覆盖删旧对话、复制正文、move 删源行、
  删源文件）之后立即写盘。以前只在最后写一次，中途失败会让磁盘上的 meta 缺
  `override_deleted` / `source_deleted` / `copied_to` → 精确回滚静默漏项
- 跨版本迁移的 `projects/<slug>/` 目录名改为**直接沿用源侧真实目录名**（那是客户端按 cwd
  实际建出来的），`cwd_to_slug()` 只作兜底。自己推的规则一旦与客户端不一致，
  正文会落到客户端不扫描的目录 → "迁移成功却打不开"
- 软冲突覆盖：先把数据库行 commit 成功，**再**删目标文件。以前反着做，commit 失败会留下
  "行还在、正文没了"的不一致
- 回滚 / `--full` 整库恢复时会一并清掉 `workbuddy.db-wal` / `-shm`：只覆盖主库的话，
  SQLite 下次打开会把与新主库不匹配的旧 WAL 重放上去，回滚可能无效甚至数据错乱。
  `migrate.py --rollback` 也补上了"整库覆盖会抹掉迁移后新增数据"的警告

**修复：`--intl` 与 Connector 合并的静默失效**

- `--intl` 时不再读平台 `storage.json`：它是国内版登录态文件，机器上同时装两个版本时会把
  国内版 uid 当成国际版当前账号，导致国际版数据被迁到一个不存在的账号下（表现为对话全部消失）。
  国际版一律走 `account-snapshot.json`
- `migrate_connectors()` 的"深度合并"以前只看顶层 key：`mcp.json` 顶层只有 `mcpServers`，
  目标一旦已有该 key 就被判"无新增"整体跳过，一个 server 都合并不进去。
  现在先 `deep_merge_dict` 再比较合并前后差异

**修复：其他一致性与健壮性**

- 进程检测排除脚本自身（仓库目录名含 `workbuddy`，脚本命令行会命中关键字，
  否则用户被"检测到客户端正在运行"无条件拦住，只能加 `--force` 关掉整项检查）
- `snapshot_db()` 改为只读打开源库 + 异常时关闭连接（与 `_backup_db()` 实现统一）
- `mkdir` 补 `parents=True`（备份目录 / 目标 connectors 目录 / tasks 目录）
- memory 文件名也做 UUID 校验（之前只修了 connectors 目录）
- 前缀匹配 `id LIKE ?` 加 `ESCAPE`：用户粘贴的 id 含 `%` / `_` 不再被当通配符
- `_wal_checkpoint()`：非 WAL 库返回的 `-1` 不再被当成"有进程占锁"
- `--target` 不带 `--source`、`--generate-commands` 不带 `--restore-tasks` 时明确报错，
  不再静默忽略；`--target` 非 UUID 形态时告警
- Phase 4 验证顺带校验 Memory / Connector，并对"静默跳过"给出警示
- 删除死代码 `_extract_memory_block()`；`render_diff(kind=)` 现在真的用上了
- `_rewrite_session_id()` 失败时清理 `.tmp`，不留残缺文件

**修复：第 6 轮审计（回滚守卫 / 静默失败 / 文档一致）**

- 跨版本复制正文改为**逐文件登记** `copied_to`：第 2 个及以后文件失败时，先前已复制到
  目标侧的文件不会被精确回滚漏删（此前会留下孤儿正文；同版本克隆路径早已修过，跨版本漏改）
- `_remove_db_sidecars()` 返回成败，`--full` 整库回滚在 `-wal` / `-shm` 删不掉（客户端仍持句柄）
  时**中止**，而不是照样覆盖主库
- `migrate.py` 回滚改为「备份里确实有 `workbuddy.db` 才清边车并覆盖」：传错标签（例如只含
  memory 的备份）不再白删未落盘的 WAL（那是永久丢失）；`rollback()` 的 `meta` 先初始化，
  消除「无 meta.json」兼容分支的潜在 `NameError`
- `migrate.py` 整账号备份 `meta.json` 改**原子写**；备份目录名同秒冲突自动加序号，
  不再覆盖前一份备份的 meta
- `migrate.py` 补上「客户端必须关闭」检测（新增 `--force`），与 `migrate_session.py` 对齐；
  迁移中途抛异常时打印备份标签与回滚命令，`migrate_connectors()` 的 `json.load` 不再裸抛
- `--rollback <TAG>` 拒绝含路径分隔符 / `..` / 绝对路径的标签（此前会被拼进备份目录
  并用于 rmtree / copytree 的目标推导）
- `--dry-run` 不再弹冲突询问（管道输入时 EOFError 会被当成"已取消"，计划根本不打印）
- 4 处「验证命中 n 行」改为命中数不符即告警（此前写入没生效也照样显示 ✅）；
  move 删源文件 / 源 tasks 目录包 `try/except OSError`，不再裸抛 traceback
- `_is_self_process()` 改为按**路径成分**匹配：不再因为路径里恰好含 `migrate.py`
  （第三方同名脚本、备份副本）就把进程当成自己、静默关掉客户端检测
- 跨版本目标目录**始终按源侧 slug**（写进库里的 cwd 是源的值）；覆盖时清理目标侧
  「源里没有」的同 id 附属文件，避免新旧混搭
- 交互向导透传 `--mode` / `--dry-run` / `--yes` / `--target-uid` / `--backup-dir`
  （此前在"未给 `--session-id`"时被静默丢弃）；删除从未使用的死参数 `dir_arg`
- 登录态来源口径统一：`migrate.py` 的 `get_current_user_id()` 改为与
  `migrate_session.get_current_uid()`、`references/data_isolation_map.md` 一致 ——
  **account-snapshot.json 优先，平台 storage.json 兜底**。此前 `migrate.py` 把平台
  storage.json 排在前面，同一台机器上两个脚本可能解析出不同的"当前账号"；
  两者不一致时会打印告警
- 文档：参数表补齐 `--list/--query/--target-uid/--yes/--full/--backup-dir`；兼容性表去掉
  「国内版 (macOS) ✅ 已测试」（与"仅 Windows 实测"矛盾）；项目结构补 `TOPICS.md`

**修复：第 7～8 轮复核（去重复实现 / 回滚健壮性 / 文案对齐）**

- **进程检测不再有两份实现**：`migrate_session.py` 里那套几乎逐行复制的
  `_is_self_process` / `_client_display_name` / `find_running_clients` /
  `require_clients_closed` 已删除（连 `require_clients_closed` 的签名都和 `migrate.py`
  不一样），改为对 `legacy.*` 的薄委托 —— 符合「单对话脚本 import 复用 migrate.py」的
  仓库约定；顺带删掉因此不再使用的 `import csv` / `import subprocess`
- `migrate_session.py --rollback` 的 `meta.json` 读取加兜底：半截损坏时给可操作提示
  而不是 traceback（而且这段在"要不要回滚"的确认询问之前，崩在这里用户连提示都看不到）
- `migrate.py --rollback` 在 `meta.json` **存在但解析失败**时也打印「无法确定目标账号，
  Memory / Connectors 会被跳过、请改用带 meta.json 的备份」；该说明统一提到确认询问**之前**
- `--dry-run` 计划里的备份落点改为与实现一致（`migrate_backups/session/`，给了
  `--backup-dir` 时以它为准），此前只打印 `migrate_backups/`
- 文档：`--yes` 的说明改为「跳过确认询问；冲突处理由 `--on-conflict` 决定，
  无终端时 `ask` 降级为 `skip`」（原写"跳过所有确认"，与实现不符）；
  回滚章节补充「备份目录含个人数据，请及时移入回收站清理」及跨平台清理命令
- `migrate.py` 的「源账号无任何数据」提示补上**本次使用的数据目录**与换版本方式
  （`--intl` / `--dir`）：两个版本都装时自动探测会选国际版，用户在国内版里明明有数据
  却只看到一句"无任何数据"（当时的退出码还是 0，v1.6.1 起改为 3），
  很容易误判成"真没东西可迁"
- 测试：补 `migrate.py` **主流程端到端**用例（造源账号 → 改 `user_id` → 校验备份
  `meta.json` → 整库回滚 → 无数据提示）。此前 `migrate.py` 只有纯函数单测，
  主流程一次都没真正跑过（第 6 轮审计 R6-50 指出的缺口）
- 测试：补两个**零覆盖的用户入口** —— `--backups` 备份列表，以及**交互式向导**
  （`--dry-run` 驱动走完全流程并断言未改动任何数据，另验证 `--mode` 透传：
  第 8 轮刚改过向导的参数透传，此前没有任何回归网）
- 测试：补 `--restore-tasks` / `--generate-commands` 用例 —— 这是仓库里最后一个
  **没被任何用例跑过的写路径**（造 pending 任务 + 伪造"当前 session"，断言恢复产物
  落在正确目录且带 `restored_from_session` 元数据；`--generate-commands` 只打印不写盘）

**修复：第 9 轮审计（目标账号裁决 / 静默失败 / 回滚完整性）**

- **登录态冲突不再默默返回一个"可能错的"目标账号**：`account-snapshot.json` 与平台
  `storage.json`（或与 DB 推断）不一致时，只读场景沿用「account-snapshot 优先」并告警；
  `migrate()` 这类破坏性路径必须先让用户裁决（新增 `uid_conflicts()` / `_decide_target_uid()`），
  `--yes` 下无法裁决则退出 1，并把两条可用的 `--target` 命令打印出来
- 路径解析：`APPDATA` 为空或非绝对路径、`XDG_CONFIG_HOME` 为空串时不再把 `storage.json`
  解析成**当前工作目录**（`Path("")` 就是 cwd）——回退到 `home/AppData/Roaming`；
  `WORKBUDDY_MIGRATE_HOME` 生效时也不去读真实机器的 XDG
- 新增 `--assume-clients-closed`：进程检测**失败**时按「客户端已全部退出」谨慎继续；
  真检测到客户端仍在运行照样拦截（比 `--force` 一把关掉整项检查温和）
- 源账号无数据时退出码改为 **3**（无数据可迁、未做任何改动），不再与「迁移成功」的 0
  混用；Phase 5「什么都没迁」同口径；新增「退出码」章节
- `_migrate_intra()` 核对 `UPDATE` 的命中行数：命中 0 行时回滚并报错，不再给出成功反馈
- `deep_merge_dict()` 补齐空值语义（空壳 `args: []` / `env: {}` / 空串不再把源配置挡在门外）；
  合并冲突逐条列出来，不再被静默吞掉后还报「无新增内容」。**没有改成源覆盖目标**——
  README / SKILL 明确写着「目标已有配置保留不动」，改成源胜会造成目标侧数据丢失
- 软冲突覆盖的 `override_deleted` 改为在 commit **之前**落盘：中途崩掉时回滚不再漏还原旧对话
- `_atomic_write_text()` 的 `os.replace` 失败不再静默（清理 `.tmp` 后原样上抛），两个脚本同步；
  `migrate_connectors()` 写 `mcp.json` / `connector-states.json` 也改用原子写
- `_is_self_process()` 只在「裸脚本名」或「完整路径在本仓库下」时才认脚本名，
  第三方同名脚本不再被当成自己而静默关掉客户端检测
- `restore_tasks()`：推断当前 session 的 sqlite 连接进 `finally`（异常路径不再泄漏）、
  写任务文件改原子写；`get_task_stats()` 不再把损坏的任务文件当成「没有任务」
- `_migrate_cross()` 复制任务数据的 `rmtree` / `copytree` 收口 `OSError`
  （它是在 DB 已 commit 之后才失败的，必须给回滚指引而不是裸 traceback）
- `remove_path()` 返回成败，回滚的各调用点按返回值决定是否打印 ✅，不再悄悄进入半回滚态
- 找不到 `scripts/migrate.py` 时，即使给了 `--force` 也要求二次确认，不再零检测放行
- `--on-conflict=newer` 的平局不再判为覆盖（保守跳过）
- `--backups` 不再静默跳过 meta 损坏或属于另一工具的备份；`--backup-dir` 里找不到却命中
  标准目录时显式告警；单对话备份挪到 `migrate_backups/session/` 的布局变化写进文档
- `migrate.py` 回滚的文件操作收口 `OSError`；收尾汇总改为按「这次真的写了什么」判断，
  不再把跳过报成已迁移
- 测试：两个脚本补上 UTF-8 输出包装（Windows GBK 控制台不再崩在第一个 emoji）；
  `prepare_fixture` 清理失败不再留下半新半旧的 fixture、连接也不再泄漏；
  helper 的 sqlite 连接统一登记收口；并补上本轮各项的回归用例

**修复：第 10 轮复核（`--full` 的还原范围 / 参数静默忽略 / 测试侧句柄）**

- `--full` 必须与 `--rollback` 同用：以前单独给 `--full`（没有 `--rollback`）会被静默丢弃，
  用户以为自己选了「整库恢复」，实际跑的还是不带 `--full` 的精确回滚，两侧数据按不同
  口径恢复后就对不上了；现在打印用法并以退出码 1 结束（与 `--target` 必须配 `--source`、
  `--generate-commands` 必须配 `--restore-tasks` 同口径）
- `--rollback --full` 在 `copy` 模式下不再还原源侧：`copy` 迁移本身没有改动过源版本的数据，
  而整库快照是无差别覆盖的，连源一起还原会把源版本迁移之后新增的对话 / 消息一起抹掉 ——
  用户选 `copy` 恰恰是为了保住源。现在按 `source_deleted` 判断（同版本迁移时两个库是同一个、
  备份里只会有 `snapshot_src.db`，那一份仍照还原），跳过时打印 ⏭️ 说明，
  并把「无法恢复」的报错区分成真缺失与主动跳过，不再让 skip 看起来像备份损坏
- 测试：`run_tests.py` 里残留的裸 `sqlite3.connect` 收进 `try/finally`、或登记到
  `_OPEN_CONNS` 由 `atexit` 兜底关闭 —— `pick_test_session()` 中间有多条 `sys.exit(1)`
  分支，以前会跳过末尾的 `close()`，句柄带着库锁留到下一次 `build()`
- 测试：新增用例 —— `copy` 模式 `--full` 回滚后源库逐字节未变（并以「目标侧确实撤掉了」
  反证回滚真的执行了）、`--full` 不带 `--rollback` 必须报错

**修复：第 13 轮复审（TTY 挂死的回归 / 整账号目标账号护栏 / 残留与口径）**

- **修掉第 12 轮引入的回归：真终端里跑测试会挂在 `input()` 上**。新的「检测到客户端
  先问一次」判据是 `sys.stdin.isatty()` —— 那是真实终端的属性，把 stdout 重定向到
  文件并不影响。于是 `python tests/run_tests.py` 会卡在一个没人看的提问上
  （`... 2>&1` 又是好的）。现在进程内统一钉死 `_stdin_can_prompt=False`，
  并新增 2.12a 覆盖提问行为本身（此前这条新分支零测试）
- **整账号迁移不再静默把「DB 反推」的账号当目标**：`migrate()` 用的是
  `get_current_user_id()`（丢掉了来源），两份登录态都读不到时会拿「DB 里 session 数
  最多」当目标 —— 典型机器上那是旧账号，而整账号迁移是全量改 `user_id`，错了就是
  「迁移成功、整个账号的对话都看不见了」。现在强告警 + 要求显式确认（`--yes` 视为
  已确认），且不再把它标成"当前登录"。单对话脚本早有告警，唯独这条最大爆炸半径的
  路径没有
- **`migrate.py` 的异常收口补齐**：只有 `_run_migrate()` 包了 `RuntimeError`，
  `--rollback` / `--restore-tasks` 抛异常时仍是提示 + 一整段 traceback。现在 `main()`
  统一收口，与 `migrate_session.py` 的三收口对齐
- **非法备份标签不再自相矛盾**：以前先打「❌ 标签不合法」紧接着又打
  「❌ 找不到备份: ../../etc」。新增 `find_backup()` 返回 `(路径, reason)`
- **`table_columns()` 的白名单不再用 `assert`**：`python -O` 会把它整段剥掉，
  且 `AssertionError` 顶层不捕。改为抛 `ValueError`
- **只读位收口补到 `migrate.py`**：connectors / memory 的备份与还原仍是裸
  `copytree` / `copy2`，只读文件会传染到备份产物上
- **fixture 硬杀残留不再永远躺在 `%TEMP%`**：独立目录化解决了交叠互删，但进程被硬杀
  时 atexit 不跑，残留的**真实对话副本**再也没人路过。现在建 fixture 时顺手清掉
  超过 3 天的同命名旧目录
- `--list-tasks` 与 `--source` / `--target` / `--restore-tasks` 同给时会被静默丢掉
  （dispatch 是 elif 链），补互斥检查
- `_uid_src_text()` 不再本地抄一份 `migrate.uid_source_label()` 的映射（两份表必然漂移）
- 同版本回滚在校验失败时也会记进汇总（以前裸打印「✅ 回滚完成」）
- 测试：atexit 顺序改成「先关 sqlite 连接、再删 fixture」（Windows 上开着的句柄会让
  rmtree 报 WinError 145）；用例编号现在**源码顺序也等于执行序**

**修复：第 12 轮审计（护栏死代码 / 备份目录隔离 / 回滚不再假报成功）**

- **「只剩 DB 反推目标账号」的告警此前从未触发**：`get_current_uid()` 第二步调的
  `migrate.get_current_user_id()` 内部**自带 DB 兜底**，登录态文件都读不到时它会静默
  返回 DB 推断出的 uid，来源被打成 `legacy(...)`；唯一打 `db-majority` 标签的分支
  永远到不了，而告警判据正是看这个标签。于是最危险的那条路径一声不吭 —— 对
  domestic→intl（最常见方向）尤其致命，因为国际版按设计从不读平台 `storage.json`，
  目标只要缺 `account-snapshot.json`，uid 必然静默落到 DB 推断上（典型机器上那往往是
  旧账号）。现在新增 `migrate._resolve_current_uid()` 返回 `(uid, 来源)`，两侧共用；
  同版本路径 `_migrate_intra()` 以前根本没接这个告警，一并补上
- **检测到客户端仍在运行时改为询问用户**：以前被这条守卫拦下就是打印风险 + 直接返回
  `exit 2`，想继续只能重敲命令加 `--force`。现在在真终端里先问一次（输入 `y` = 当场
  确认接受风险并继续，回车 / 其他 = 取消）。非终端（CI / 管道 / 重定向）不询问、
  行为完全不变 —— 问不到人时 `input()` 要么立刻 EOF 要么把进程挂住
- **自定义 `--backup-dir` 也套 `session/` 命名空间**：单对话备份本来统一写
  `migrate_backups/session/` 与整账号备份分开，但给了 `--backup-dir` 时是平铺写入的，
  指向 `migrate_backups` 就会两种备份混在一个目录下。现在自定义根也写进 `<root>/session/`
- **任务去重指纹在「没有 `id` 字段」的任务上互相碰撞**（第 11 轮幂等化引入的行为回归）：
  指纹是 `(来源 session, 任务 id)`，老格式 / 手工改过的任务没有 `id` 时同会话多条任务
  指纹都退化成 `(sid, "")`，第一条写完第二条起就被误判「已恢复过」跳过。现在缺 `id` 时
  回退用标题判重
- **删除失败不再假报成功**：`remove_path()` 现在只返回 `False` 不抛 `OSError`，调用方
  残留的 `except OSError` 全是死分支，软冲突删目标旧文件那条失败照样打印 ✅。另外
  失败信息会打两遍（`remove_path` 自己一段、调用方一段），新增 `quiet=` 参数收敛
- **`--assume-clients-closed` 在降级分支也认**：「拿不到 migrate.py」的分支提示了这个
  开关却只认 `--force`，照提示做了照样被拦
- **只读位传染收口到任务数据**：`copy_path` 早补了写权限，但 `create_backup` 的任务备份、
  迁移时的任务复制、回滚还原任务仍裸用 `shutil.copytree`。新增 `copy_tree()` 全面替换；
  `remove_path()` 删除失败时先抹只读位再重试一次
- **回滚不再无条件报「✅ 完成」**：回滚里凡是「失败但不能崩栈」的步骤以前都沉默过去，
  收尾照样打 ✅ —— 半回滚是最难排查的状态。新增 `_report_rollback()` 列出没做到的步骤
  与续做方法；四处裸 `json.loads` 改走 `_load_backup_json()`（备份损坏时不再崩栈）
- **提示不再叠三段**：备份失败会把同一件事说三遍，现在原因只由顶层打印一次；uid 来源的
  技术标签不再直接透给用户
- **`--session` / 向导参数不再静默丢弃**：`--session` 只被任务恢复路径读取，不带
  `--restore-tasks` 时静默忽略；向导里 `--on-conflict` 三处被写死成 `ask`，
  `--from` / `--to` / `--query` 直接丢弃
- **dry-run 两端同口径**：向导里的 dry-run 以前照样被「必须关闭客户端」拦下，而显式
  `--session-id` 的 dry-run 跳过检测
- 测试基建：fixture 每次运行用独立目录（`wbmigrate-fixture-<pid>-<随机>`），交叠运行
  不再互相删对方的库；`prepare_fixture` 不再在 import 期 `Path.home()` 崩溃；
  `table_columns()` 补表名白名单断言；用例编号重排为严格等于执行序

**修复：第 11 轮审计（幂等 / 提前中止的可操作性 / 两侧脚本对称性）**

- **`--restore-tasks` 现在幂等**：以前重复执行会把同一批历史任务再写一份（id 递增），
  同一个待办在面板里出现多次。根因是 `TASKS_DIR` 里也放着当前 session 自己的任务，
  被 `get_task_stats()` 一起扫出来 → 上一轮的输出成了下一轮的输入。现在先定恢复目标、
  源侧排除它，并按 `metadata` 里早就在写的「来源 session + 来源任务 id」去重
  （没有 id 字段的老任务按标题判重），跳过时会写明是因哪一条已存在而跳过
- **孤儿附属文件照样清理**：目标侧清多余文件的判据原来是 `if dst_row:`，而最需要清理的
  恰恰是「上次迁移崩在中间、库行回滚了但正文留在盘上」——没有库行 = 被整段跳过。
  不敢无条件删是因为备份也挂在同一判据里，现将 `dst_files` 备份移出（无条件备份），
  清理改成 always，并按 `remove_path()` 返回值决定是否打印 ✅
- **`migrate_session.py` 支持 `--assume-clients-closed`**：以前只有 `migrate.py` 有这个
  开关，单对话路径想绕过「检测失败」只能用 `--force` 把整项检查关掉，两个脚本不对称
- **`--target-uid` 增加形态校验**：不像 UUID 时先警告（拼错会把对话挂到不存在的账号下，
  表现同样是"迁移成功但对话消失"），非交互模式还要再确认一次
- **目标账号推断与小脚本口径统一**：`get_current_uid()` 以前把「DB 里 session 数最多」
  排在第二位，而 `migrate.py` 排在最后 —— 典型机器恰恰是旧账号 session 更多，跨版本
  迁移会直接拿它当 `--target-uid`（即"迁过去就看不见了"那条路径）。现在共用同一套
  优先级，DB 推断降级为最后手段并明确告警提示用 `--target-uid`
- **失败不再裸 traceback**：克隆时改写 `sessionId` 的 `OSError`、move 删源文件失败，
  以前一个穿透顶层（`RuntimeError` / `sqlite3.Error` 之外没人接），一个假报 ✅；
  现在前者转成带回滚指引的 `RuntimeError`，后者按返回值判定
- **`migrate.py:create_backup()` 失败会自清理**：以前中途失败留下没有 `meta.json` 的
  半成品目录（既不能回滚、又在 `--backups` 里留一条永远找不到的记录）
- **`main()` 补 `RuntimeError` 收口**：WAL 被占用 / 备份失败这类可预见中止，以前先打印
  人话提示紧接着甩一整段 traceback，提示被淹没（`migrate_session.py` 顶层早有收口）
- **`--diagnose` 与其它动作参数同给时不再静默丢弃**（对称 `--rollback` 的既有检查）
- **Windows 输出不用二次包装**：`import migrate` 会无条件给 stdout 套一层 UTF-8 wrapper，
  已被包装过的情况下旧 wrapper 被 GC 会关掉共用 buffer，之后所有 `print` 变成
  `ValueError: I/O operation on closed file`（在自己的单元测试里 import 时必现）
- **`collect_info` 不再因脏时间戳崩整条 `--list`**（`int()` → 容错 `_to_int`）
- **备份标签校验下沉到 `load_backup()`**：原先只写在 `rollback()` 里，绕过它直接调用的
  路径可以用 `../../xxx` 拼到备份目录之外
- `SKILL.md` 的手动迁移示例改成**参数化查询**（`?` 占位符），不再示范 f-string 拼 SQL
- 测试：`prepare_fixture.build()` 中途失败时自动清理半截 fixture（含真实数据的子集）；
  两条静默空转 / 静默通过的用例现在要么真跑、要么明说"未执行"

#### v1.6.0 (2026-09-10)

**新增：`scripts/migrate_session.py` — 单对话跨版本迁移**（本节主体来自 [@bukall](https://github.com/bukall)，PR #5）

只迁移**指定的一个对话**，并支持**国内版 ⇄ 国际版**双向：

- 默认 `move`（迁移后删除源版本中的该对话），可选 `--mode copy` 保留源
- 迁移单元完整：**session 行 + `session_usage` + `workspaces` 登记 + `projects/*.jsonl` 对话正文**。只搬数据库行是不够的，正文不在数据库里，漏了对话就是空的
- 跨版本迁移自动把 `user_id` 改写为目标版本当前登录账号，否则目标版本里依然看不到
- 冲突分级询问：
  - 硬冲突（ID 相同）→ 覆盖 / 不操作
  - 软冲突（标题相同、ID 不同，多为重复迁移）→ 覆盖 / 不覆盖 / 不操作
  - 询问时展示差异对比（最后活动时间、消息数、对话大小、工具调用、token 用量、最后提问）并给出覆盖建议
- 覆盖时始终以**源的 ID** 写入并删除目标那条旧记录，保证正文文件名与 ID 一致
- 备份精确到单条，回滚不影响其他对话；`--dry-run` 可先预览
- 安全：迁移前检测客户端是否运行，**未关闭则拒绝执行**（WAL 未落盘 + 内存缓存会覆盖写入）
- 新增 `tests/`：`prepare_fixture.py` 从真实数据只读复制出临时 fixture，`run_tests.py` 提供一整套端到端 + 单元级用例（用例数随本机 fixture 内容浮动，实际数量看运行末尾的「结果」行），全程在临时目录运行

**修复：正文含 `tool-results/` 目录时备份直接崩溃**

- 大工具输出会被外溢到 `projects/{slug}/{id}/tool-results/*.txt`（一个与会话同名的**目录**）。
  备份阶段对目录调用 `shutil.copy2()` 在 Windows 上抛 `PermissionError: [Errno 13]`，
  整个迁移中断。现在文件与目录统一走 `copy_path()` / `remove_path()`
- 同一根因还波及迁移复制、move 删源、回滚还原、软冲突清理旧记录四处，一并修复
- 对话大小统计改为递归累加，此前 `tool-results/` 被算成 0，显示的体积偏小
- 列表里区分显示「N 个文件 + N 个目录（tool-results）」，不再让人误以为多出异常项

**修复：同版本选 copy 时提示「无需迁移」却什么也没做**

- `_migrate_intra` 原先只实现「改 `user_id`」一种语义，源对话已属于当前账号时无从可改就空转
- 现在自动按**克隆**处理：生成新 session id，标题加「（副本）」，复制正文与任务数据
- 克隆会改写正文中每条消息的 `"sessionId"`（否则副本内部仍指向原对话）、
  并按新 id 重命名正文文件与 `tool-results/` 目录
- 回滚按 `kind=session_clone` 单独处理，**只删副本、不动原对话**
  （走通用回滚分支会按原 id 删行，把原始对话一起删掉）
- 备份中途失败会自清理，不再残留没有 `meta.json` 的半成品目录

**改进：原有 `scripts/migrate.py`**

- 当前账号识别：国内版继续以平台 `storage.json` 的 `genie.userId` 为权威；国际版使用数据目录内的 `storage/skeleton/account-snapshot.json` → `primary.uid`（跨平台路径统一，不依赖 `%APPDATA%` 探测，来自 [@fhjowe](https://github.com/fhjowe)，PR #6）；两者都取不到时回落 DB 中 session 数最多的 `user_id`
- 回滚安全性：备份 `meta.json` 缺失导致 `target_uid` 为空时，跳过 Memory / Connectors 恢复。原先路径会退化成整个 `connectors/` 目录并被 `rmtree` **删光所有账号的连接器配置**
- 回滚完整性：Connectors / Memory 的恢复不再要求目标当前必须存在，只要备份里有就恢复。原先迁移后清理过目录就恢复不了
- Memory 迁移在 `memory/` 目录不存在时自动创建，不再报错

#### v1.5.0 (2026-09-09)

**国内版 / 国际版双版本支持**

- **新增**：支持 WorkBuddy 国际版（数据目录 `~/.workbuddy-ai/`），通过 `--intl` 参数或交互式向导选择
- **改进**：交互式向导新增版本选择步骤，展示两个版本的路径区别
- **默认行为**：不加参数时自动探测数据目录（`~/.workbuddy-ai` 存在且非空判为国际版，否则国内版），`--intl` / `--dir` 可显式指定

#### v1.4.0 (2026-08-06)

**跨平台支持 + 当前账号识别修复**（感谢 [@yuren238](https://github.com/yuren238)，PR #1）

- **跨平台**：storage.json 路径自动适配 macOS / Windows / Linux，不再硬编码 macOS 路径
- **Bug 修复**：当前账号识别改为以 storage.json 为权威来源，DB 按 session 数最多做辅助验证。此前用"最新 session"推断，旧账号的最后一条 session 可能比当前账号更新，导致误把旧账号当成当前账号
- **新增**：`--target` 参数，可手动指定目标账号，无需切换登录
- **改进**：交互式向导改为手动选择目标/源账号，避免自动推断错误
- **Bug 修复**：Windows GBK 编码终端下 emoji 输出导致 UnicodeEncodeError 崩溃

#### v1.3.0 (2026-05-26)

**关键修复：账号切换后 storage.json 中 genie.userId 未同步，导致迁移被静默跳过**

- **Bug 修复**：`get_current_user_id()` 改为多源交叉验证——同时从 DB 最新 session 和 storage.json 读取 user_id，不一致时警告并优先使用 DB 值。此前仅依赖 `genie.userId`，账号切换后可能过时，导致 source=target 迁移被跳过。
- **Bug 修复**：`migrate_sessions()` 迁移后增加 WAL checkpoint + 验证源 user_id 归零。此前修改可能因 WAL 未落盘而在客户端重启后丢失。
- **文档更新**：SKILL.md 新增 AI 手动迁移最佳实践、3 条新踩坑记录。

#### v1.2.0 (2026-05-25)

- 新增历史任务恢复（`--list-tasks`、`--restore-tasks`）
- 新增交互式向导模式
- 新增 `--generate-commands` 生成 TaskCreate 命令

#### v1.1.0 (2026-05-25)

- 首次公开发布
- Session、Memory、Connector 迁移
- 自动备份 + 回滚

### License

[MIT](LICENSE) © 2026

---

<h2 id="english">English</h2>

### The Problem

After switching accounts in WorkBuddy (Tencent Cloud AI assistant desktop app), **all your previous conversation history, long-term memory, and MCP connector configs disappear from the UI**. The data is still on disk — just hidden by `user_id` isolation.

This tool merges old account data into your current account with a single command.

### Quick Start

```bash
git clone https://github.com/xiaoliuzhuan666/workbuddy-account-migrate.git
cd workbuddy-account-migrate
python3 scripts/migrate.py
```

Interactive wizard — pick your edition (domestic or international), then select accounts by number.

**Other modes:**

```bash
python3 scripts/migrate.py --diagnose                            # Diagnose only
python3 scripts/migrate.py --source <USER_ID>                    # Specify source account
python3 scripts/migrate.py --source <USER_ID> --target <USER_ID> # Pin the target too (no login switch)
python3 scripts/migrate.py --intl                                # International edition (~/.workbuddy-ai)
python3 scripts/migrate.py --intl --diagnose                     # Diagnose international edition
python3 scripts/migrate.py --intl --source <USER_ID>             # Migrate within the international edition
python3 scripts/migrate.py --dir ~/.workbuddy-ai                 # Explicit data dir (beats --intl)
python3 scripts/migrate.py --assume-clients-closed               # Continue when process *detection fails*
                                                                 # (still blocks if a client is detected)
python3 scripts/migrate.py --rollback <TAG>                      # Rollback to a backup
```

> **Domestic vs International**: not just a different data directory — the **login source differs too** (both editions treat `storage/skeleton/account-snapshot.json` as authoritative; only the domestic edition falls back to the platform `storage.json`, and the international edition never reads it). Domestic uses `~/.workbuddy/`, international uses `~/.workbuddy-ai/`. Directory resolution order: `--dir` > `--intl` > auto-detect (`~/.workbuddy-ai` non-empty means international). See the note under **Compatibility** for details.

### Backup layout

Since v1.6 the two scripts keep their backups apart — the `meta.json` formats differ, so mixing them makes `--backups` list garbage and `--rollback` crash with `KeyError`:

| Path | Contents | Roll back with |
|:---|:---|:---|
| `~/.workbuddy/migrate_backups/<TAG>/` | whole-account backup | `scripts/migrate.py --rollback <TAG>` |
| `~/.workbuddy/migrate_backups/session/<TAG>/` | single-session backup | `scripts/migrate_session.py --rollback <TAG>` |

Backups in the old location are still found (both directories are scanned). `--backup-dir` (**only `migrate_session.py` has it — `migrate.py` does not**) adds an **extra** search root rather than restricting the search: new backups are written to `<dir>/session/` (the same namespace, so pointing a custom dir at `migrate_backups` can't mix them with whole-account backups), lookups still fall back to the two standard directories, and a hit outside your directory is reported explicitly.

### Exit codes

Both scripts share this table, **except `3`, which only `migrate.py` can return** — "nothing to migrate" is a whole-account concept. The single-session script exits `1` when the source or target edition has no DB at all.

| Code | Meaning |
|:---:|:---|
| `0` | Something was actually migrated / rolled back |
| `1` | Error (including: `migrate_session.py` found no DB in the source or target edition — wrong edition) |
| `2` | Blocked by the "clients must be closed" check (a client is running, or detection was unreliable and no `--force` / `--assume-clients-closed` was given) |
| `3` | **`migrate.py` only**: nothing to migrate, nothing was changed. Covers two cases — the source account really has no session / memory / connector, *or* it has data that was already identical to the target, so the merge produced nothing new |

> ⚠️ Do not treat exit code `0` as "data was definitely moved" in automation, and do not treat `3` as a failure: it means nothing was changed. Rule of thumb: `0` → success, `3` → skipped, anything else → failure.
>
> ⚠️ **Breaking change**: before v1.6.1 `migrate.py` returned `0` when the source account had no data. If your automation checks `rc == 0`, runs that used to "succeed" now return `3` — that is not a new failure, it is a long-standing skip finally being reported honestly.

### What Gets Migrated

| Data | How | Strategy |
|:---|:---|:---|
| Session history | SQLite `user_id` field | UPDATE to new account |
| Long-term Memory | `~/.workbuddy/memory/{uid}_memory.md` | Append + deduplicate |
| MCP Connectors | `~/.workbuddy/connectors/{uid}/mcp.json` | JSON deep merge |
| Cloud channel mapping | `edge-sync-mapping*.db` → `msg_channel` | v1.6.3: drop the old account's rows so EdgeSync re-uploads under the new account (restored on rollback; `--keep-cloud-mapping` opts out) |
| Skills | `~/.workbuddy/skills/` | ❌ — global, no isolation, nothing to migrate |
| Automations | `workbuddy.db` → `automations` table | ❌ — no `user_id` column, global |
| Settings / MCP / Plugins | global config files | ❌ — global |

Not migrated (no account isolation): `todos/`, `tasks/`, `skills/`, automations, settings. Per-uid dirs left alone: `inspiration/{uid}/`, `security/{uid}/`, `storage/user-{uid}*`. See the Chinese section for the full boundary table.

### Features

- 🧙 Interactive wizard (pick target & source accounts by number, no user_id needed)
- 🖥️ Cross-platform: data-dir paths resolved via pathlib, platform `storage.json` auto-detected for macOS / Windows / Linux (v1.4)
- 🌍 Domestic / International edition: interactive wizard prompts for edition, or use `--intl` for `~/.workbuddy-ai`
- 💾 Automatic backup before every migration, one-command `--rollback` afterwards
- 🔒 Safe: append-only memory, deep-merge connectors (existing target config is kept), WAL checkpoint before & after
- 🔍 Authoritative login detection: `storage/skeleton/account-snapshot.json` inside the data dir first, platform `storage.json` as fallback (domestic edition only), DB session-count as cross-check (both scripts share this order since v1.6.1)
- ✅ Post-migration verification (source user_id must be zero)
- 🪶 Zero dependencies (Python 3.8+ only)

### How it works

1. **Auto-diagnose** — discover every account from the DB, memory files and connector directories. The current account comes from `storage/skeleton/account-snapshot.json` → `primary.uid` inside the data dir (authoritative, edition-aware); the domestic edition additionally falls back to the platform `storage.json` → `genie.userId`, and the DB's highest session-count user_id is a cross-check. Conflicts prefer the login source and warn. (Never "latest session": an old account's last session can be newer than the current account's.)
2. **Safe backup** — into `~/.workbuddy/migrate_backups/{timestamp}_{first-8-of-uid}/` before anything is written (same-second reruns get a numeric suffix instead of overwriting the previous backup).
3. **Migrate** — sessions via `UPDATE user_id`; memory appended after deduplicating by semantic `memoryBlock` (line-based dedup only for the old format without `RAW_JSON`); connectors deep-merged.
4. **Persist + verify** — WAL checkpoint, then a fresh read-only connection re-checks that the source user_id is down to zero.
5. **Restart prompt** — restart the client so its in-memory cache is refreshed.

### Compatibility

- ✅ WorkBuddy Domestic edition — Windows (tested: Win 11 + Python 3.13)
- ✅ WorkBuddy International edition — Windows (tested: data dir `~/.workbuddy-ai/`, use `--intl`, v1.5)
- ⚠️ WorkBuddy Domestic / International edition — macOS / Linux (paths adapted in v1.4, **untested**)
- ❌ CodeBuddy CLI (not needed — memory is isolated per project at `~/.codebuddy/memories/{project-id}/` and sessions are per-`{sessionId}.jsonl` files, with no `user_id` filter, so switching accounts loses nothing)

> ⚠️ **Only Windows is actually tested** (Windows 11 + Python 3.13). Everything marked "untested"
> shares the same pathlib-based path handling and process detection (`tasklist` on Windows, `ps`
> elsewhere), but there is no test record — issue reports welcome.

> **Domestic vs International**: not just a different data directory — the **login source differs too**.
> Both editions write `storage/skeleton/account-snapshot.json` (authoritative); the domestic edition
> additionally has the platform `storage.json` as a fallback, while the **international edition never
> reads it** (that file holds the domestic login, and reading it would make `--intl` migrate data
> under an account that does not exist there). Directory priority: `--dir` > `--intl` > auto-detect
> (`~/.workbuddy-ai` wins when present and non-empty).

### Single-session cross-edition migration

To move **one conversation** between editions (domestic ⇄ international), use the second script:

```bash
python3 scripts/migrate_session.py                                    # interactive wizard
python3 scripts/migrate_session.py --list --from domestic             # list what's there first
python3 scripts/migrate_session.py --from domestic --to intl --session-id <ID>
```

**How it differs from `migrate.py`**

| | `migrate.py` | `migrate_session.py` |
|:---|:---|:---|
| Scope | whole account (all conversations + memory + connectors) | **one conversation** |
| Editions | within a single edition | **domestic ⇄ international** |
| Default semantics | **ownership transfer** (`UPDATE sessions.user_id`; the source account stops seeing them — nothing is deleted). Memory / connectors are merged | **move** (source deleted), or `--mode copy` to keep it |

> ⚠️ `migrate.py` works **inside one data directory** (`--intl` only switches which directory is the target — it does not move data across editions) and it does **not** carry `projects/{slug}/*.jsonl` or `tasks/` along, because within one directory those are shared per session anyway. To move conversations to the *other* edition you must use `migrate_session.py`, otherwise the target only gets a session row with an empty transcript.
>
> ⚠️ **`--mode copy`**: whatever account the source belongs to, `copy` keeps the source and produces a separate conversation owned by the target. (Cross-edition this is a copy into the other edition's DB under the same session id; same-edition it is a clone with a **new** session id. It used to degrade into a `user_id` reassignment across accounts, which lost the conversation from the source account — contradicting "copy keeps the source".)

- **Close both WorkBuddy clients first** — the script refuses to run otherwise (WAL not flushed + in-memory cache would overwrite your changes). This applies to `--rollback` as well: rollback also rewrites the DB and deletes files.
- If the target already has the conversation, you get a diff (last activity / message count / size / last prompt) plus a recommendation, then a choice:
  - **hard conflict** (same ID) → overwrite / cancel
  - **soft conflict** (same title, different ID) → overwrite / don't overwrite / cancel
  - overwriting always writes under the **source's** ID and removes the target's old row, so the transcript filename matches the ID.

**What one conversation actually consists of** (miss one and the client misbehaves):

| Data | Where | Note |
|:---|:---|:---|
| session row | `workbuddy.db` → `sessions` | inserted cross-DB, `user_id` rewritten to the target account |
| usage stats | `session_usage` | token counters |
| workspace entry | `workspaces` | without it the client can't resolve the path |
| **transcript** | `projects/{slug}/{id}.jsonl` | **without it the conversation opens empty** |
| tool results | `projects/{slug}/{id}/tool-results/*.txt` | spilled large outputs; missing = missing content |

**Same-edition runs** (`--from` equals `--to`): if the conversation belongs to another account, `move` just reassigns `user_id`; if it already belongs to the current account (or you passed `--mode copy`) it is **cloned** — new session id, title suffixed with 「（副本）」, every in-transcript `"sessionId"` rewritten, transcript and `tool-results/` renamed to the new id. The original is never touched, and rollback deletes only the copy.

**Parameters**

| Flag | Meaning | Default |
|:---|:---|:---|
| `--from` / `--to` | source / target edition (`domestic` \| `intl`) | `domestic` |
| `--list` / `--query` | list conversations (optionally filtered by title / cwd / id) | - |
| `--session-id` | conversation id (prefix accepted) | - |
| `--mode` | `move` (delete source) / `copy` (keep source) | **`move`** |
| `--on-conflict` | `ask` / `skip` / `overwrite` / `newer` (without a TTY `ask` degrades to `skip`) | `ask` |
| `--target-uid` | pin the target edition's user_id (otherwise inferred from `account-snapshot.json`). **Warns first if the value doesn't look like a UUID** — a typo would attach the conversation to a non-existent account, which looks exactly like "migrated fine but the conversation vanished"; in non-interactive mode it asks for one more confirmation | - |
| `--dry-run` | print the plan, write nothing (no conflict prompt either) | off |
| `--yes` | skip confirmation prompts — **conflict handling is governed by `--on-conflict`, not by this flag** | off |
| `--force` | skip the "clients must be closed" check | off |
| `--assume-clients-closed` | gentler than `--force` and accepted by both scripts: continues only when process **detection itself fails**; still blocks when a client is actually detected running | off |
| `--backups` / `--rollback TAG` | list backups / roll back (prefix accepted) | - |
| `--full` | with `--rollback` only: whole-DB restore (DB + transcript + tasks). On its own it **errors out** | off |
| `--backup-dir` | backup dir (**this script only** — `migrate.py` has no such flag): new backups go to `<dir>/session/`; also an **extra** search root when looking up / rolling back | - |

Without `--session-id` you get the interactive wizard, which passes `--mode` / `--yes` / `--dry-run` / `--target-uid` / `--backup-dir` / `--assume-clients-closed` / `--on-conflict` / `--from` / `--to` / `--query` through.

**Rollback** (single-session backups — for `migrate.py` whole-account backups use `migrate.py --rollback <TAG>`)

```bash
python3 scripts/migrate_session.py --backups
python3 scripts/migrate_session.py --rollback 20260922000000_domestic2intl_12345678
python3 scripts/migrate_session.py --rollback <TAG> --full     # whole-DB restore
```

Rollback is precise to that one conversation — other conversations are untouched.

> ℹ️ **`--full` scope**: the whole-DB restore puts the databases the migration **actually touched** back to the pre-migration state (DB + transcript + tasks), so anything created after the migration is lost — the script says so before you confirm. In `copy` mode the source edition was deliberately kept and was never modified, so `--full` does **not** restore it (it prints a ⏭️ line saying it skipped); only the target is restored.

> ⚠️ **Backups contain personal data** — a `workbuddy.db` snapshot, `mcp.json` / `connector-states.json` (possibly tokens), `.master.key`, `{uid}_memory.md`, plus transcripts and `tool-results/`. Move them to the recycle bin rather than deleting them outright once you no longer need to roll back. The scripts never clean up old backups for you.

### Known limitations

| Limitation | Detail |
|:---|:---|
| Multiple memory blocks | Structured memory is **appended** as one `RAW_JSON` block. Whether the client merges several blocks is unverified — if it reads only the first one, migrated memory exists on disk but stays invisible in the UI. The script prompts you to check |
| Non-Windows process detection | the "clients must be closed" check uses `tasklist` on Windows (tested); on macOS / Linux it uses `ps`, where Electron apps may report a package name — false negatives are possible, so verify yourself if you need `--force` |
| Several same-titled targets | with more than one same-title conversation in the target, the script handles **one** of them (it prints the other ids/titles) — clean up the rest in the client |
| Empty session `cwd` | a few rows have no `cwd`; the transcript directory then falls back to the source-side directory name, and if that fails the script aborts instead of silently writing to the wrong place |
| Transcript id rewriting | only the `"sessionId":"..."` **field value** is rewritten. Old ids referenced inside message text (logs, paths) are left alone — that is user-visible content |

### FAQ

**Q: Is my history really not lost after switching accounts?**

A: Right — the files are still on disk, the UI just filters by `user_id`. Merging them into the current account makes them visible again.

**Q: Can I migrate back?**

A: Yes. Login to B again and run `--source <A's user_id>`, or use `--target` and skip the login switch. Memory is deduplicated by semantic block (`memoryBlock`) and connectors merge by key, so reverse runs don't duplicate. Note that a reverse run moves **all** sessions owned by A (including A's own) — to undo one specific migration, `--rollback` is cleaner.

**Q: Is the old account's data still there after a migration?**

A: Sessions are reassigned (`user_id` changed), so they are invisible under the old account — nothing was deleted. Memory and connector source files are kept as-is; delete them yourself if you want.

**Q: CodeBuddy CLI?**

A: Not supported, and not needed — see **Compatibility**.

**Q: Does it work on macOS / Linux?**

A: Supported at the path level. Paths have been adapted per platform since v1.4 (macOS `~/Library/Application Support/...`, Windows `%APPDATA%`, Linux `XDG_CONFIG_HOME`), and since v1.6.1 login state comes from `account-snapshot.json` inside the data dir, so it no longer depends on where the platform `storage.json` lives.

**Only Windows is actually tested**, though (Windows 11 + Python 3.13) — macOS / Linux are "cross-platform code, never run here". Process detection uses `tasklist` on Windows and `ps` elsewhere, and the `ps` path has zero test coverage. See **Compatibility**.

### Project structure

```
workbuddy-account-migrate/
├── scripts/
│   ├── migrate.py            # whole-account migration (within one edition)
│   └── migrate_session.py    # single-session migration (cross-edition, v1.6)
├── tests/
│   ├── prepare_fixture.py    # builds a temp fixture (read-only copy of real data)
│   └── run_tests.py          # end-to-end + unit tests (count varies with your fixture)
└── references/
    └── data_isolation_map.md # data-isolation map
```

Tests run entirely inside the temp fixture and never touch your real data directory: `python3 tests/run_tests.py`. ⚠️ The test scripts are **Windows-only in practice** (Win 11 + Python 3.13): the fixture is copied from your real data, whose paths are Windows-shaped, and a machine without WorkBuddy installed can't build a fixture at all.

### Changelog

#### v1.6.3 (2026-09-22)

**Fixed: the target account was resolved to the wrong uid — migration "succeeds" but the sidebar stays empty**

- `get_current_user_id()` now prefers **`account-snapshot.json` → `primary.uid`** (the client's real login, which is what the session list filters by). `storage.json`'s `genie.userId` drops to second priority, DB session count is the last resort
- Why: on the domestic edition those two sources can hold **two different uids** for a long time. The tool used to pick `storage.json`, so every run merged data into the account the UI never reads — the user retried 6 times and still saw an empty sidebar
- `--diagnose` now prints all four signals side by side (client login / storage.json / last daemon `listSessions` uid / per-account session counts) and states the `--target` conclusion explicitly
- `migrate()` gained a Phase 4.5 consistency check: when the target ≠ client login it lists the two recovery paths instead of just printing "done"
- Backup `meta.json` now records `source_uid`, `client_login_uid`, `storage_json_uid`, `session_counts` for post-mortems
- Fixed daemon-log parsing: nested JSON escapes its quotes (`\"userId\":\"...\"`), so the previous regex never matched
- **New: cloud channel mapping reset** — `edge-sync-mapping*.db` rows still point at the old account's channel, so EdgeSync believes the conversations are already synced and never re-uploads them; the migration now deletes only those rows (full DB backed up first, restored by `--rollback`, opt out with `--keep-cloud-mapping`)
- **New: `scripts/force-relogin.sh`** — moves `storage/skeleton/account-snapshot.json` aside to force a fresh login; its header documents the measured limitation (works, but the client may switch back after ~1 minute, so merging data is the durable fix)
- Interactive wizard now tags the client-login account with `← 客户端登录态（面板按它过滤）`
- `tests/run_tests.py` skips gracefully (exit 0) when the international edition is absent
- Docs: migration boundary list, two-login-sources section, tarball install fallback

#### v1.6.2 (2026-09-22)

**Fixed: sandbox-shim hijack crashing runs inside WorkBuddy sessions**

- When run from a WorkBuddy session's Bash, the injected `PYTHONPATH` points to a sandbox shim (sitecustomize.py) that hijacks `Path.mkdir`: even with `exist_ok=True`, an existing directory raises `PermissionError EEXIST`, crashing the migration at the backup phase (both the managed and the system Python are affected)
- The script now strips `PYTHONPATH` on startup and re-executes itself (equivalent to `env -u PYTHONPATH python3 migrate.py ...` without having to remember it); all `mkdir` call sites keep an `exists()` pre-check as a second line of defense

**Added: `--restart` to auto-restart the client after migration**

- `python3 migrate.py --source <UID> --yes --restart`: after migration/rollback, automatically quits and relaunches WorkBuddy (macOS) after a short delay, refreshing the session list immediately — no manual restart needed
- Implemented as a detached background job: the script prints its full output first, then triggers the restart; when invoked inside a WorkBuddy session, the current AI session will be interrupted (expected). Windows / Linux print a manual-restart reminder

#### v1.6.1 (2026-09-21)

**Fixed: `migrate_session.py` behavior/docs mismatches & silent failures** (all changes by [@bukall](https://github.com/bukall), PR #5)

- `--mode copy` across accounts no longer degrades to "reassign `user_id`": copy always keeps the source and clones one into the target account
- Sessions with an empty `cwd` no longer silently write the transcript into the `projects/` root (the client looks it up at `projects/<slug>/<id>.jsonl` — the migration would "succeed" but the conversation would never open). Now a three-level fallback (row `cwd` → session profile `cwd` → source transcript's directory name), and it aborts with a rollback hint if still undeterminable
- `--list` size stats now recurse into directories (previously `tool-results/` was under-counted as ~4KB)
- Top-level `sqlite3.Error` handler added: when a cross-edition insert hits a new NOT NULL column without default in the target DB, you get an actionable "rollback with --rollback" message instead of a traceback
- Soft-conflict overwrite: the deleted target conversation's `session_usage` rows are now backed up and rolled back too
- Client process detection no longer treats a failed check as "client is closed" (explicit `--force` required); non-Windows platforms additionally match `ps -eo args=` against full command lines, avoiding Electron bundle-name misses
- Transcript id rewriting now only touches `"sessionId":"..."` field values — a naive full-line replace used to corrupt message text that happened to contain the same id string (logs, paths)

**Fixed: data-safety & parsing issues in `migrate.py`**

- DB backup now uses the sqlite backup API (includes WAL data) instead of `shutil.copy2` on the main DB file — the latter captured a stale snapshot when the client was still running; falls back to file copy with an explicit warning
- `PRAGMA wal_checkpoint`'s busy flag is now checked: no more claiming "verification passed" when the checkpoint didn't complete
- Structured memory migration compares against **all** existing `memoryBlock`s in the target — repeated runs no longer append the same block twice
- `_get_storage_json_path()` respects `WORKBUDDY_MIGRATE_HOME` / `--dir` and no longer reads the real machine's platform storage.json; `STORAGE_JSON` being `None` no longer leads to a bare `open()`
- `get_connector_info()` reads `mcp.json` as UTF-8 explicitly (Chinese configs were silently swallowed, showing 0 servers)
- user_id detection now uses UUID-shape matching instead of "directory name contains a hyphen"
- `--rollback` accepts `--yes` to skip confirmation; combining it with `--source` etc. now errors out explicitly instead of silently prioritizing rollback


#### v1.6.1 (2026-09-21)

**Fixed: round-13 review (TTY hang regression / whole-account target guard / leftovers)**

- **Fixed a regression introduced in round 12: running the test suite on a real terminal hung on `input()`.** The new "ask once when a client is detected" branch keys off `sys.stdin.isatty()` — a property of the *real terminal*, unaffected by redirecting stdout to a file. So `python tests/run_tests.py` blocked on a question nobody was watching (`... 2>&1` worked fine). In-process calls now pin `_stdin_can_prompt` to `False`, and new case 2.12a covers the asking behaviour itself (that branch had zero coverage)
- **Whole-account migration no longer silently adopts a DB-inferred target uid.** `migrate()` used `get_current_user_id()`, which discards the source: with no login-state file readable it fell back to "most sessions in the DB" — typically the *old* account — and since this path reassigns `user_id` for an entire account, getting it wrong looks like "migration succeeded but every conversation vanished". It now warns hard and requires explicit confirmation (`--yes` counts as confirmed), and no longer labels the guessed uid "current login". The single-session script already warned; this was the one path with the largest blast radius that didn't
- **`migrate.py` now funnels exceptions at the top level.** Only `_run_migrate()` wrapped `RuntimeError`, so `--rollback` / `--restore-tasks` still dumped a traceback right after their human-readable hint. `main()` now matches `migrate_session.py`'s three-way handling
- **An invalid backup tag no longer contradicts itself** ("❌ invalid tag" followed by "❌ backup not found: ../../etc"). New `find_backup()` returns `(path, reason)`
- **`table_columns()`'s table-name whitelist no longer uses `assert`** — `python -O` strips it entirely, and `AssertionError` isn't caught at the top level. Now raises `ValueError`
- **Read-only handling reached `migrate.py`**: the connector / memory backup and restore calls were still bare `copytree` / `copy2`, so read-only files propagated into backup artifacts
- **A hard-killed fixture no longer sits in `%TEMP%` forever.** Per-run directories fixed cross-run clobbering but lost self-healing: when the process is killed, atexit never runs and the leftover copy of **real conversations** is never visited again. Building a fixture now sweeps stale same-named directories older than 3 days
- `--list-tasks` silently dropped `--source` / `--target` / `--restore-tasks` (the dispatch is an `elif` chain); added the same mutual-exclusion check `--rollback` and `--diagnose` already had
- `_uid_src_text()` no longer keeps a local copy of `migrate.uid_source_label()`'s mapping (two copies of one table always drift)
- Same-edition rollback now records a failed verification in the summary instead of printing "✅ done" unconditionally
- Tests: atexit order is now "close sqlite connections, then delete the fixture" (an open handle makes `rmtree` fail with WinError 145 on Windows); case numbers now match source order as well as execution order

**Fixed: round-12 audit (dead guardrail / backup-dir isolation / rollback no longer claims success)**

- **The "target uid was only guessed from the DB" warning never fired.** `get_current_uid()` step 2 calls `migrate.get_current_user_id()`, which *already* has a DB fallback: when no login-state file can be read it silently returns the DB-inferred uid with the source label `legacy(...)`, so the only branch that emits the `db-majority` label was unreachable — and that label is exactly what the warning checks. The most dangerous path was therefore silent. It bites hardest on domestic→intl (the common direction): the international edition never reads the platform `storage.json` by design, so a target missing `account-snapshot.json` always falls back to DB inference, which on a typical machine is the *old* account. `migrate._resolve_current_uid()` now returns `(uid, source)` and both scripts share it; `_migrate_intra()` (same-edition path) never wired the warning up at all and does now
- **Being blocked by the "client still running" guard now asks first.** It used to print the risks and return `exit 2`, so continuing meant re-typing the command with `--force`. On a real TTY it now asks once (`y` = accept the three risks and continue, Enter/anything else = cancel). Non-interactive contexts (CI, pipes, redirection) are **not** asked and behave exactly as before — with nobody to answer, `input()` either hits EOF immediately or hangs the process
- **A custom `--backup-dir` now gets the same `session/` namespace.** Single-session backups always went to `migrate_backups/session/`, separate from `migrate.py`'s whole-account backups — but with `--backup-dir` they were written flat, so pointing it at `migrate_backups` mixed both kinds in one directory. Custom roots now write to `<root>/session/` too
- **Task de-duplication fingerprints collided for tasks with no `id`** (a regression from the round-11 idempotency work): the fingerprint is `(source session, task id)`, so for legacy/hand-edited tasks without `id` every task in a session degraded to `(sid, "")` and everything after the first was wrongly skipped as "already restored". Falls back to the subject when `id` is missing
- **Deletion failures no longer report success.** `remove_path()` returns `False` instead of raising `OSError`, which left dead `except OSError` branches at the call sites; the soft-conflict branch that deletes the target's old files still printed ✅ on failure. Failure text was also printed twice — new `quiet=` flag lets callers print their own, more contextual wording
- **`--assume-clients-closed` is now honoured on the degraded branch** (where `migrate.py` can't be found). The hint mentioned the flag but only `--force` was accepted, so following the hint still got you blocked
- **Read-only bit no longer spreads through task data.** `copy_path` already restored write permission, but the task backup in `create_backup`, the task copy in the migration paths and the task restore on rollback still used bare `shutil.copytree`. New `copy_tree()` replaces all of them; `remove_path()` now clears the read-only bit and retries once before giving up
- **Rollback no longer unconditionally prints "✅ done".** Every step that can fail without being fatal used to be swallowed and the summary still claimed success — a half-rollback is the hardest state to diagnose. New `_report_rollback()` lists what did not happen and how to resume; four bare `json.loads` calls went through `_load_backup_json()` so a corrupt backup no longer crashes mid-rollback
- **Failure messages no longer stack three deep** (backup failure was reported by three layers); internal uid-source labels are no longer printed straight at the user
- **`--session` and wizard arguments are no longer silently dropped.** `--session` is only read by the task-restore path and was ignored without `--restore-tasks`; the wizard hardcoded `on_conflict="ask"` at three call sites and discarded `--from` / `--to` / `--query`
- **`--dry-run` is symmetric now**: the wizard's dry-run was still blocked by the "clients must be closed" guard while an explicit `--session-id` dry-run skipped it
- Test infra: each run gets its own fixture directory (`wbmigrate-fixture-<pid>-<random>`) so overlapping runs can't delete each other's DB; `prepare_fixture` no longer crashes at import time on `Path.home()`; `table_columns()` asserts the table name against a whitelist; case numbers were renumbered to strictly match execution order

**Fixed: rounds 7–8 review (duplicate implementation / rollback robustness / wording)**

- **Process detection is no longer implemented twice**: the near-verbatim copy in `migrate_session.py` (`_is_self_process` / `_client_display_name` / `find_running_clients` / `require_clients_closed` — whose signature even differed from `migrate.py`'s) was removed in favour of a thin delegation to `legacy.*`, matching the repo convention that the single-session script reuses `migrate.py` by import. The now-unused `import csv` / `import subprocess` went with it; tests stub `migrate.find_running_clients` (one seam instead of two)
- `migrate_session.py --rollback` now reads `meta.json` defensively: a truncated file yields an actionable message and exit code 1 instead of a traceback — and this used to happen *before* the "roll back?" prompt, so the user never even saw the prompt
- `migrate.py --rollback` prints the "cannot determine the target account → Memory / Connectors are skipped → use a backup with a complete meta.json" explanation for **all three** cases (missing / corrupt / no `target_uid`), moved ahead of the confirmation prompt
- The `--dry-run` plan now reports the real backup location (`migrate_backups/session/`, or `--backup-dir` when given) instead of just `migrate_backups/`
- Docs: `--yes` is described as "skips confirmation prompts; conflict handling is governed by `--on-conflict`, and `ask` degrades to `skip` without a TTY" (it used to claim "skips all confirmations"); the rollback section warns that backup directories contain personal data and gives per-platform recycle-bin commands
- `migrate.py`'s "source account has no data" message now also prints **which data directory was used** and how to switch (`--intl` / `--dir`): with both editions installed the auto-detection picks the international one, so a user with data in the domestic edition saw only "no data" (exit code 0) and could easily conclude there was nothing to migrate
- Tests: added an **end-to-end case for the `migrate.py` main flow** (create a source account → reassign `user_id` → verify the backup `meta.json` → whole-DB rollback → the no-data hint). `migrate.py` previously had pure-function unit tests only and its main flow had never actually been run (the gap flagged as R6-50 in the round-6 audit)
- Tests: covered two previously untested user entry points — the `--backups` listing and the **interactive wizard** (`--dry-run` drives the whole flow and asserts nothing was written; a second run verifies `--mode` is passed through, which round 8 had just fixed without any regression net)
- Tests: covered `--restore-tasks` / `--generate-commands` — the last **write path in the repo that no test had ever run** (seeds a pending task plus a fake "current session", then asserts the restored file lands in the right directory carrying `restored_from_session` metadata; `--generate-commands` only prints)

**Fixed: data-safety and rollback-integrity issues**

- Cross-edition transcript copy now registers each file in `meta["copied_to"]` **as it goes**: if the 2nd (or later) file fails, the files already copied are still tracked and get deleted by precise rollback (previously they were left behind as orphan transcripts; the same-edition clone path had already been fixed, the cross-edition one had not)
- `--full` whole-DB rollback now **aborts** when `-wal` / `-shm` cannot be removed (client still holding the DB) instead of overwriting the main DB anyway — otherwise SQLite replays the stale WAL and the rollback silently fails
- `migrate.py` rollback only clears sidecar files and overwrites the DB when the backup **actually contains** `workbuddy.db`; a wrong tag (e.g. a memory-only backup) no longer deletes an un-checkpointed WAL (which would be permanent data loss)
- `migrate.py` whole-account backup `meta.json` is now written atomically; backup directories that collide within the same second get a numeric suffix instead of overwriting the previous backup's meta
- `--rollback <TAG>` (both scripts) rejects tags containing path separators, `..` or absolute paths — they used to be concatenated into the backup directory and reused as rmtree/copytree targets
- `migrate.py` gained the "client must be closed" process check (new `--force` flag), aligning it with `migrate_session.py`; **both scripts now check on rollback too** (rollback also rewrites the DB and deletes files)
- `migrate_session.py` `--rollback` now requires the clients to be closed as well (previously only the migration path was guarded)
- `--dry-run` no longer pops the conflict prompt (with piped stdin an EOFError was treated as "cancelled" and the plan was never printed)
- The four "verified N rows" messages now warn when the count does not match (previously a write that silently did nothing still showed ✅); `move` deleting source files / the source tasks directory is wrapped in `try/except OSError` instead of raising a bare traceback
- Cross-edition target directory **always uses the source-side slug** (the `cwd` written into the DB is the source's); on overwrite, target-only sibling files for the same id are cleaned up so old and new data don't mix
- Overwrite no longer leaves the target with a stale transcript: files present in the target but absent from the source are removed (they are backed up in `dst_files/` and restored on rollback)
- The interactive wizard now passes `--mode` / `--dry-run` / `--yes` / `--target-uid` / `--backup-dir` through (they were silently dropped when `--session-id` was omitted); removed the never-used `dir_arg` parameter
- Current-account resolution is now consistent across both scripts and the docs: **`storage/skeleton/account-snapshot.json` first, platform `storage.json` as fallback**, with a warning when the two disagree. `migrate.py` previously put platform `storage.json` first, so the two scripts could resolve different "current accounts" on the same machine

**Fixed: `migrate_session.py` behaviour contradicting the docs / silent failures**

- `--mode copy` no longer degrades into a `user_id` reassignment across accounts: `copy` always keeps the source and clones a copy owned by the target account
- An empty session `cwd` no longer silently writes the transcript into the `projects/` root (the client looks in `projects/<slug>/<id>.jsonl`; the root means "migration succeeded but the conversation won't open"). Three-level slug fallback, aborting with a rollback hint if none resolves
- List size reporting recurses into directories (`--list` was missed, so `tool-results/` still counted as ~4 KB)
- Soft-conflict overwrite now backs up and restores the deleted target conversation's `session_usage`
- A failed client-process detection is no longer silently treated as "client closed" (explicit `--force` required); on non-Windows `ps -eo args=` is used to match the full command line, avoiding Electron package-name misses
- In-transcript id rewriting only touches the `"sessionId":"..."` field value — a whole-line replace used to corrupt user-visible message text that happened to contain the same id string (logs, paths)

**Fixed: `migrate.py` data safety and parsing**

- DB backup uses the sqlite backup API (WAL included) instead of `shutil.copy2` on the main DB file — the latter produced a stale snapshot while the client was running; falls back to file copy with an explicit warning
- The `PRAGMA wal_checkpoint` busy flag is now interpreted: an unfinished checkpoint is no longer reported as "verified"
- Structured Memory migration compares against **all** existing `memoryBlock`s, so repeated runs don't append the same block twice
- `get_connector_info()` reads `mcp.json` as explicit UTF-8 (Chinese configs used to be swallowed silently and shown as 0 servers)
- user_id detection uses UUID-shape matching instead of "directory name contains a hyphen"
- `--rollback` supports `--yes`; combining it with `--source` and friends now errors out instead of silently taking priority

**Fixed: high-risk data-safety issues (round 3)**

- Backup `meta.json` is written **incrementally** after every destructive step (override deletion, transcript copy, `move` source-row deletion, source-file deletion). Previously it was written once at the end, so a mid-way failure left `override_deleted` / `source_deleted` / `copied_to` missing on disk → precise rollback silently skipped items
- The cross-edition `projects/<slug>/` directory name now reuses the **source-side real directory name** (the one the client actually created from the cwd), with `cwd_to_slug()` only as fallback — a self-invented rule that disagrees with the client drops the transcript into a directory the client never scans
- Soft-conflict overwrite commits the DB rows **before** deleting the target files (the reverse order could leave "row present, transcript gone" if the commit failed)
- Rollback / `--full` restore clears `workbuddy.db-wal` / `-shm` first: overwriting only the main DB lets SQLite replay an old WAL, which can make the rollback ineffective or corrupt data. `migrate.py --rollback` also warns that a whole-DB restore wipes data created after the migration

**Fixed: silent failures with `--intl` and connector merging**

- `--intl` no longer reads the platform `storage.json` (a domestic-edition login file): on a machine with both editions installed it would take the domestic uid as the international current account, migrating international data under an account that does not exist there (all conversations appear to vanish). The international edition always uses `account-snapshot.json`
- `migrate_connectors()` "deep merge" used to look only at top-level keys: `mcp.json` has just one (`mcpServers`), so once the target had it the whole file was skipped and not a single server was merged. It now deep-merges first and then compares before/after

**Fixed: other consistency and robustness issues**

- Process detection excludes the script itself (the repo directory is named `workbuddy`, so the script's own command line matched the keyword, blocking users unconditionally unless they disabled the whole check with `--force`)
- `snapshot_db()` opens the source read-only and closes connections on error (unified with `_backup_db()`)
- `mkdir` gained `parents=True`; memory filenames are UUID-validated too; prefix matching `id LIKE ?` uses `ESCAPE`; `_wal_checkpoint()` no longer treats a non-WAL `-1` as "locked by a process"
- `--target` without `--source` and `--generate-commands` without `--restore-tasks` now error instead of being ignored; a non-UUID `--target` warns
- Phase 4 verification also checks Memory / Connectors and flags silent skips
- Removed dead code `_extract_memory_block()`; `_rewrite_session_id()` cleans up its `.tmp` file on failure

**Fixed: round-6 audit (rollback guards / silent failures / doc consistency)**

- Login-source wording unified across `migrate.py`, `migrate_session.py` and `references/data_isolation_map.md`
- `_is_self_process()` matches by **path components**: a process counts as the script itself only when its command line is a bare script name or a path inside this repo — a third-party script named `migrate.py`, or a backup copy, no longer silently disables the client check
- Docs: parameter table completed (`--list` / `--query` / `--target-uid` / `--yes` / `--full` / `--backup-dir`); compatibility table no longer claims "domestic (macOS) ✅ tested" (contradicted "Windows only"); project structure lists `TOPICS.md`
- Tests: case numbering kept contiguous; the fixture no longer copies `connectors/` (contains tokens/credentials); a new case forces the **last** transcript file to fail and asserts no orphan files remain after rollback

**Fixed: round-9 audit (target-account arbitration / silent failures / rollback integrity)**

- Conflicting login sources no longer silently yield a possibly-wrong target account: read-only paths keep "account-snapshot wins" plus a warning, while the destructive `migrate()` path now makes the user arbitrate (`uid_conflicts()` / `_decide_target_uid()`); under `--yes`, with no way to decide, it exits 1 and prints the two usable `--target` commands
- Path resolution: an empty / non-absolute `APPDATA` or an empty `XDG_CONFIG_HOME` no longer resolves `storage.json` to the **current working directory** (`Path("")` == cwd) — it falls back to `home/AppData/Roaming`, and `WORKBUDDY_MIGRATE_HOME` never reads the real machine's XDG
- New `--assume-clients-closed`: continues cautiously when process *detection fails*, and still blocks when a client is actually detected (gentler than `--force`, which disables the whole check)
- "Source account has no data" now exits **3** (nothing was changed) instead of `0`, so automation can no longer mistake "nothing to migrate" for "migrated"; the Phase-5 "nothing was actually migrated" path uses the same code, and there is a new **Exit codes** section
- `_migrate_intra()` verifies the `UPDATE` row count — 0 rows hit now rolls back and errors out instead of reporting success
- `deep_merge_dict()` learned empty-value semantics (shell values such as `args: []` / `env: {}` / `""` no longer block the source config) and reports merge conflicts one by one instead of swallowing them and claiming "nothing new". It still does **not** let the source overwrite the target: the docs promise "existing target config is kept", and flipping that would lose target-side data
- Soft-conflict overwrite writes `override_deleted` to disk **before** the commit, so a crash mid-way no longer makes rollback skip restoring the old conversation
- `_atomic_write_text()` no longer swallows a failed `os.replace` (it cleans up `.tmp` and re-raises); both scripts aligned, and `migrate_connectors()` writes `mcp.json` / `connector-states.json` atomically too
- `_is_self_process()` (further narrowed): a process now counts as the script only when its command line is a **bare** script name or a path inside this repo — a third-party script named `migrate.py` is no longer mistaken for the tool itself, which used to silently disable the client check
- `restore_tasks()`: the connection used to infer the current session is closed in `finally` (no leak on error paths), task files are written atomically, and `get_task_stats()` no longer reports a corrupt task file as "no tasks"
- `_migrate_cross()` wraps the task-data `rmtree` / `copytree` in `OSError` handling — it fails *after* the DB commit, so it must print rollback guidance rather than a traceback
- `remove_path()` returns success and rollback call sites only print ✅ when it actually worked: no more silent half-rolled-back states
- When `scripts/migrate.py` cannot be imported, a second confirmation is required even with `--force`, instead of running with zero detection
- `--on-conflict=newer` no longer treats a tie as "overwrite" (it conservatively skips)
- `--backups` no longer silently skips backups whose `meta.json` is broken or that belong to the other tool; a hit outside `--backup-dir` warns explicitly; the `migrate_backups/session/` layout change is documented
- `migrate.py` rollback wraps its file operations in `OSError` handling, and the closing summary reports what was **actually** written instead of counting skips as migrated
- Tests: both scripts force UTF-8 output, so a Windows GBK console no longer crashes on the first emoji; `prepare_fixture` no longer leaves a half-built fixture when cleanup fails and no longer leaks connections; test helpers register their sqlite connections for `atexit` cleanup; regression cases were added for the items above

**Fixed: round-10 review (`--full` scope / silently ignored flags / test-side handles)**

- `--full` must be combined with `--rollback`: passing it alone used to be silently dropped, so you thought you had asked for a whole-DB restore while the precise rollback ran instead — and the two restore to different scopes. It now prints usage and exits 1 (same treatment as `--target` without `--source` and `--generate-commands` without `--restore-tasks`)
- `--rollback --full` no longer restores the source side in `copy` mode: a `copy` migration never wrote to the source (it opens it read-only), and the whole-DB snapshot overwrites indiscriminately, so restoring it would wipe everything the source edition gained after the migration — while `copy` exists precisely to keep the source. The decision now follows `source_deleted` (same-edition migrations share one DB and only ever produce `snapshot_src.db`, which is still restored), and skipping prints a ⏭️ line so it can't be mistaken for a broken backup
- Tests: the remaining bare `sqlite3.connect` calls in `run_tests.py` are wrapped in `try/finally` or registered for `atexit` cleanup — `pick_test_session()` has several `sys.exit(1)` branches that used to skip the trailing `close()`, leaving a handle (and its lock) behind for the next `build()`
- Tests: new cases — after a `copy` `--full` rollback the source DB is byte-identical (with "the target side really was undone" as proof the rollback ran), and `--full` without `--rollback` must error

**Fixed: round-11 audit (idempotency / actionable aborts / script symmetry)**

- **`--restore-tasks` is now idempotent**: running it twice used to write every historical task a second time under a new id, so the same todo appeared twice in the panel. Root cause: the current session's own tasks live under `TASKS_DIR` too, so the previous run's output became the next run's input. The restore target is now resolved first and excluded as a source, and duplicates are detected via the `metadata` already written (`restored_from_session` + `restored_from_task_id`)
- **Orphaned attachments are cleaned up**: the "delete leftover target-side files" step was gated on `if dst_row:`, but the case that needs it most is exactly when there is *no* row — a migration that died halfway left its transcript on disk. It couldn't be unconditional because the backup was gated the same way; `dst_files` are now backed up unconditionally, cleanup always runs, and ✅ is printed based on `remove_path()`'s return value
- **`migrate_session.py` gained `--assume-clients-closed`**: only `migrate.py` had it, so the single-session path had to reach for `--force` (which disables the whole check) just to work around a failed detection — now both scripts are symmetric
- **`--target-uid` is validated**: a value that doesn't look like a UUID warns first (a typo attaches the conversation to a non-existent account, which looks exactly like "migrated fine but the conversation vanished"), and asks one more time in non-interactive mode
- **Target-uid inference unified**: `get_current_uid()` ranked "most sessions in the DB" second while `migrate.py` ranked it last — and on a typical machine the *old* account has more sessions, which is precisely the "migrate away and it disappears" path. Both now share one priority order, with the DB guess demoted to last resort and an explicit warning to use `--target-uid`
- **No more naked tracebacks**: an `OSError` while rewriting `sessionId` during a clone (the new row is already committed at that point) is converted to a `RuntimeError` with rollback guidance; deleting source files now reports failure when `remove_path()` returns `False` instead of printing ✅
- **`migrate.py:create_backup()` cleans up after itself**: a mid-way failure used to leave a directory with no `meta.json` — impossible to roll back, yet permanently listed by `--backups`
- **`main()` now catches `RuntimeError`**: foreseeable aborts (WAL busy, backup failure) printed a human message and then dumped a full traceback on top of it, burying the message
- **`--diagnose` no longer silently discards other action flags** (mirrors the existing `--rollback` exclusivity check)
- **No double-wrapping stdout on Windows**: importing `migrate` unconditionally wrapped stdout in UTF-8; when something already had, the discarded wrapper took the shared buffer with it on GC and every later `print` raised `ValueError: I/O operation on closed file`
- **`collect_info` no longer crashes an entire `--list` on a dirty timestamp** (tolerant `_to_int`)
- **Backup-tag validation moved down into `load_backup()`**: it lived only in `rollback()`, so any caller that skipped that function could escape the backup root with `../../xxx`
- `SKILL.md`'s manual migration example now uses **parameterised queries** (`?` placeholders) instead of demonstrating f-string SQL construction
- Tests: `prepare_fixture.build()` cleans up a half-built fixture if it dies midway; two cases that used to no-op silently now either really run or say they didn't

#### v1.6.0 (2026-09-10)

**Single-session cross-edition migration (domestic ⇄ international)** (this section's work by [@bukall](https://github.com/bukall), PR #5)

- **New**: `scripts/migrate_session.py` — migrate one conversation between editions
- **New**: carries `session_usage`, `workspaces` and the `projects/*.jsonl` transcript along (DB row alone = empty conversation)
- **New**: conflict prompts — hard conflict (same ID) offers 2 choices, soft conflict (same title, different ID) offers 3, both with a side-by-side diff and an overwrite recommendation
- **New**: `move` by default, `copy` optional; per-session backup, rollback touches nothing else
- **Improved**: `migrate_session.py` resolves the current account from `storage/skeleton/account-snapshot.json` inside the data dir (edition-aware, cross-platform) — `migrate.py` only switched to that order in v1.6.1, see below
- **Safety**: refuses to run while a WorkBuddy client is running
- **Tests**: new `tests/` with fixture builder + 86 end-to-end checks, all in a temp dir

**Fixed: backup crashed when the transcript included a `tool-results/` directory**

- Large tool outputs spill to `projects/{slug}/{id}/tool-results/*.txt` — a **directory** named after the session.
  `shutil.copy2()` on it raised `PermissionError: [Errno 13]` on Windows and aborted the whole migration.
  Files and directories now go through a shared `copy_path()` / `remove_path()`.
- Same root cause affected migration copy, `move` source deletion, rollback restore and soft-conflict cleanup — all fixed.
- Size reporting now recurses into directories (previously `tool-results/` counted as 0).

**Fixed: same-edition `copy` said "nothing to migrate" and did nothing**

- `_migrate_intra` only implemented the "reassign `user_id`" case; when the session already belonged to the current account there was nothing to reassign, so it bailed out.
- It now clones: new session id, title suffixed with 「（副本）」, transcript and task data copied.
- The clone rewrites every in-transcript `"sessionId"` and renames the transcript / `tool-results/` to the new id.
- Rollback handles `kind=session_clone` separately — it removes only the copy, never the original.
- A failed backup now cleans itself up instead of leaving a half-written directory.

**Changes to the existing `migrate.py`:**

- Current account detection now uses `storage/skeleton/account-snapshot.json` (edition-aware, cross-platform — by [@fhjowe](https://github.com/fhjowe), PR #6)
- Rollback safety: if `meta.json` is missing and `target_uid` is empty, Memory/Connectors restore is skipped — the path would otherwise degrade to the whole `connectors/` dir and `rmtree` **every account's config**
- Rollback completeness: Connectors/Memory are restored whenever the backup has them, even if the target no longer exists
- Memory migration creates `memory/` when missing instead of crashing

#### v1.5.0 (2026-09-09)

**Domestic / International edition support**

- **New**: support for WorkBuddy International edition (data directory `~/.workbuddy-ai/`) via `--intl` flag or interactive wizard selection
- **Improved**: interactive wizard now prompts for edition choice with path details
- **Default**: without any flag, the data directory is auto-detected (non-empty `~/.workbuddy-ai` wins); `--intl` / `--dir` pin it explicitly

#### v1.4.0 (2026-08-06)

**Cross-platform support + current-account detection fix** (thanks [@yuren238](https://github.com/yuren238), PR #1)

- **Cross-platform**: storage.json path auto-adapts to macOS / Windows / Linux (no more hardcoded macOS path)
- **Bug fix**: current account is now detected from storage.json as the authoritative source, with the DB's most-frequent user_id as a cross-check. Previously the "latest session" heuristic could misidentify a stale old account as the current one
- **New**: `--target` flag to explicitly set the target account without switching logins
- **Improved**: interactive wizard now asks for target and source accounts explicitly, avoiding auto-inference errors
- **Bug fix**: emoji output no longer crashes on Windows GBK/CP936 terminals (UnicodeEncodeError)

#### v1.3.0 (2026-05-26)

**Critical fix: Migration was silently skipped due to stale user_id**

- **Bug fix**: `get_current_user_id()` now uses multi-source cross-validation — reads from both DB latest session and `storage.json`, warns when inconsistent, prioritizes DB value. Previously relied solely on `genie.userId` which could be stale after account switch, causing `source == target` and migration being skipped.
- **Bug fix**: `migrate_sessions()` now performs WAL checkpoint after UPDATE (not just before), and verifies source user_id is zero. Previously, modifications could be lost on client restart due to unflushed WAL logs.
- **SKILL.md**: Added AI manual migration best practices, 3 new troubleshooting entries.
- **README**: Updated feature list, workflow description, and version badge.

#### v1.2.0 (2026-05-25)

- Added task history recovery (`--list-tasks`, `--restore-tasks`)
- Added interactive wizard mode
- Added `--generate-commands` for TaskCreate tool

#### v1.1.0 (2026-05-25)

- Initial public release
- Session, Memory, Connector migration
- Auto-backup + rollback support

### License

[MIT](LICENSE) © 2026
