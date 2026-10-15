---
name: wekeapi-contact
description: wekeapi CLI contact 模块：微信通讯录（好友列表/简要/详细信息、通讯录同步、搜索联系人、添加/同意/删除好友、备注、仅聊天权限、企微好友、手机通讯录）。当用户要求查好友、搜索微信号、加好友、删好友、改备注、查企微联系人、上传手机通讯录，或需要把昵称/备注/微信号解析成 wxid、查询本地联系人数据库（含 sqlite3 查询 ~/.wekeapi/data-*.db）时使用。
---

# contact 通讯录

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）；stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定见 wekeapi-core skill。

## 本地联系人查询（解析 wxid 优先走这里）

登录后 `sync all`（见 wekeapi-login skill）已把通讯录同步到本地 SQLite（**每账号一库**，默认 `~/.wekeapi/data-<wxid>.db`，wxid 见 `~/.wekeapi/config.json` 的 `wxid` 字段）。需要把微信号/昵称/备注解析成 wxid 时，**先用 sqlite3 查本地库，查不到再调远端接口**，避免多余 API 调用：

- **关键：`contact_id` 才是联系人的 wxid**（好友/群/公众号/企微 ID 都在这一列）。**`wx_id` 是数据归属键（sync 写入时为 appId，同一账号所有行相同），查询时不要选它、更不要把它当联系人 wxid 用。**
- **标签字典**：`contacts.labels` 存的是逗号分隔的标签 ID；标签名在**同库 `labels` 表**（`sync all` 已最先同步，`wx_id` 同为 appId），按 `label_id` 关联即可把标签 ID 翻译成名称。
- **编码红线（仅 Windows 生效；macOS/Linux 的 shell 与 sqlite3 均为 UTF-8，中文字面量可直接写、无需转码，`char()` 写法亦兼容）：SQL 内禁止出现中文字面量，一律用 `char(码点1,码点2,...)` 代替。** 原因：Windows 下 PowerShell 把中文参数传给 sqlite3.exe 时按系统 ANSI 代码页（GBK）转码，而库内是 UTF-8，字节不一致导致 LIKE **静默查不到、不报错**。做法：把关键词逐字符转成十进制 Unicode 码点填入 `char()`，例：「超红」→ `char(36229,32418)`（超=36229，红=32418）。
- **强制两步法（仅 Windows；禁止跳步、禁止手算码点）**：
  1. 先用 PowerShell 把关键词转成十进制码点（传中文给 powershell.exe 是 Unicode 无 GBK 问题，红线只针对 sqlite3.exe）。**命令必须用下方无变量写法**：agent 的 shell 本身常是 PowerShell，嵌套调用 `powershell -Command "..."` 时外层会把 `$k`/`$_` 当变量展开吞掉导致语法错误：

     ```bash
     powershell -Command "[int[]][char[]]'庆普' -join ','"
     ```

  2. 把输出（如 `24198,26222`）填入 `char(...)` 再套用下方 SQL 模板。**码点只能来自该命令输出，心算/查表容易错一位且无法察觉。**
- 结果判读：按红线写的 SQL 若返回空，先确认码点来自第 1 步输出、`contact_id`/`wx_id` 没选反列，再考虑联系人真不存在或库未同步（`sync all` 刷新后重查）；**不要把空结果直接当「联系人不存在」**。
- **多字段一次匹配（默认用这条）**：关键字一次 OR 匹配 8 个字段——`contact_id` wxid、`alias` 微信号、`nickname` 昵称、`remark` 备注、`signature` 签名、`wording_name`/`wording_id` 企微企业、`raw_json` 详情原始 JSON（含手机号、地区等兜底信息），尽量一条 SQL 就命中（下例码点 `36229,32418` 对应关键词「超红」；Windows 用时替换成强制两步法第 1 步的输出，macOS/Linux 直接写中文字面量）：

  ```bash
  sqlite3 ~/.wekeapi/data-<wxid>.db "WITH kw AS (SELECT '%'||char(36229,32418)||'%' AS p) SELECT contact_id, contact_type, alias, nickname, remark, signature FROM contacts, kw WHERE contact_id LIKE p OR alias LIKE p OR nickname LIKE p OR remark LIKE p OR signature LIKE p OR wording_name LIKE p OR wording_id LIKE p OR raw_json LIKE p;"
  ```

- 结果多时收窄：明确找好友加 `AND contact_type=1`（2-群聊 / 3-公众号 / 4-企微好友）；优先取 `nickname`/`remark`/`alias` 与关键词**完全相等**的行。
- 精确匹配变体（关键词恰好是完整昵称/备注/微信号时用，零误报；`contact_id` 通常为 ASCII，wxid 可直接写字面量）：

  ```bash
  sqlite3 ~/.wekeapi/data-<wxid>.db "SELECT contact_id, contact_type, alias, nickname, remark FROM contacts WHERE nickname=char(36229,32418) OR remark=char(36229,32418) OR alias=char(36229,32418);"
  ```

- 本地未命中再走远端：`contact search --contact-info <微信号/手机号>`（结果用于加好友；若已是好友，响应中的 V3 即其 wxid）；或 `contact list` 取 ID 后用 `contact brief`/`detail` 批量拉信息。
- 本地库非空但明显缺数据（如远端刚加的好友），可先 `sync all` 刷新再查。

## 命令表

| 命令 | 说明 | 参数 |
|---|---|---|
| `contact list` | 通讯录列表（好友/群/公众号，仅返回 ID） | `--contact-type`：1-好友 / 2-已保存群 / 3-公众号；省略则返回全部。**不支持 4**（企微好友请用 `contact wework-list`） |
| `contact brief` | 批量获取简要信息 | **`--wxids`** |
| `contact detail` | 批量获取详细信息 | **`--wxids`** |
| `contact sync` | 强制同步通讯录到平台并更新缓存 | — |
| `contact search` | 搜索联系人（结果用于加好友） | **`--contact-info`** 微信号/手机号 |
| `contact add-friend` | 主动添加好友 | **`--scene`**、**`--v3`**、**`--v4`**、**`--verify-content`**；`--chatroom-id`（群聊来源时） |
| `contact accept-friend` | 同意好友申请 | **`--msg-base64`**（取自回调消息） |
| `contact delete-friend` | 删除好友 | **`--wxid`** |
| `contact set-remark` | 设置好友备注 | **`--wxid`**、**`--remark`** |
| `contact set-chat-only` | 设置好友仅聊天 | **`--wxid`**、**`--only-chat`** |
| `contact wework-list` | 企微好友列表 | — |
| `contact wework-info` | 批量获取企微好友信息 | **`--wxids`** |
| `contact wework-search` | 搜索企微联系人 | **`--contact-info`** |
| `contact wework-add` | 添加企微好友 | **`--wxid`**、**`--v1`**（均来自搜索结果）；`--veryfy-content` |
| `contact wework-delete` | 删除企微好友 | **`--wxid`** |
| `contact wework-set-remark` | 设置企微好友备注 | **`--wxid`**、**`--remark`** |
| `contact phone-list` | 手机通讯录里的微信好友 | — |
| `contact phone-upload` | 上传手机通讯录 | **`--op-code`** 1-上传/2-删除、**`--mobiles`** |

## 注意

- `list`/`brief` 从平台缓存读取：登录后平台自动同步一次；遇到"通讯录同步进行中"错误等一会再调；新联系人在平台没更新时调 `sync` 强制更新。
- `list` 只返回保存到通讯录的群聊；未保存的群需从消息推送自行判断并记录。
- `add-friend` 的 `--scene`：1-QQ 2-邮箱 3-微信号 4-QQ好友 13-通讯录 14-群聊 15-手机号 30-二维码；`v3`/`v4` 来自 `contact search` 或群成员详情（`chatroom member-detail`）。若已是好友，search 结果的 V3 即好友 wxid。
- 建议账号在线 3 天后再调 `add-friend`；添加成功会推送一条含 V3 的回调消息，可用于判断是否成功。
- **`contact list --contact-type 4` 会返回 `-11 参数错误`**：该接口不支持企微类型；企微好友请用 `contact wework-list` / `contact wework-info`。
- **备注无法清空**：`set-remark` 的平台接口对空备注返回 `-11 参数错误`；且在 Windows PowerShell 下无法向 `--remark` 传空串（空参被丢弃、`--remark=` 又被必填校验拒绝）。需要清空备注请在微信客户端操作。
- **仅聊天无读回**：`set-chat-only` 返回 `code:0`，但 `brief`/`detail` 不返回仅聊天字段，无法用 API 二次确认，请以返回码为准。
