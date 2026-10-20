---
name: wekeapi-contact
description: wekeapi CLI contact 模块：微信通讯录（好友列表/简要/详细信息、通讯录同步、搜索联系人、添加/同意/删除好友、备注、仅聊天权限、企微好友、手机通讯录）。当用户要求查好友、搜索微信号、加好友、删好友、改备注、查企微联系人、上传手机通讯录，或需要把昵称/备注/微信号解析成 wxid、查询 wekeapi-msg-sync 中的联系人数据时使用。
---

# contact 通讯录

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）；stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定见 wekeapi-core skill。

## 联系人数据查询（解析 wxid 优先走这里）

登录成功后，联系人数据由 `wekeapi-msg-sync` 服务统一存储。需要把微信号/昵称/备注解析成 wxid 时，**先用 `data contacts` 查询，查不到再调远端接口**，避免多余 API 调用：

```bash
$WEKEAPI data contacts --keyword <昵称/备注/微信号/wxid> [--type friend] [--limit 20]
```

- 输出为 CLI 统一信封 `{"code":0,"message":"OK","data":{...}}`；`data.contacts[].id` 就是联系人 wxid，`label_ids`/`labels` 为标签 ID 与名称。
- `--type` 可选：`friend`（好友）/`chatroom`（群）/`gh`（公众号）/`wework`（企微好友）/`all`（默认全部）；结果多时用来收窄。
- 优先取 `nickname`/`remark`/`alias` 与关键词**完全相等**的行；模糊结果再结合昵称/备注判断。
- 服务未运行：先 `$WEKEAPI data start`；刚加的好友可能因平台同步延迟而缺失，可 `$WEKEAPI data sync` 刷新后重查。
- 数据未命中再走远端：`contact search --contact-info <微信号/手机号>`（结果用于加好友；若已是好友，响应中的 V3 即其 wxid）；或 `contact list` 取 ID 后用 `contact brief`/`detail` 批量拉信息。
- **不要再用 `sqlite3` 查询本地数据库**（旧版本地库已废弃），也不需要 Windows 中文码点转码；HTTP/JSON 为 UTF-8，中文关键词直接传。

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

- `list`/`brief` 从平台缓存读取：登录后平台自动同步一次；CLI 已内置 -43「通讯录同步进行中」自动重试（每次等 15s、约 60s 窗口，stderr 提示进度）。**若重试后仍持续 -43（平台同步任务卡死，可持续数小时/数天），补救手段是重新登录**：`login logout` → `login quick`（手机点确认），让平台开启一次全新的通讯录同步任务——实测重登后 1 分钟内 `contact list` 即恢复。新联系人在平台没更新时调 `sync` 强制更新。
- `list` 只返回保存到通讯录的群聊；未保存的群需从消息推送自行判断并记录。
- `add-friend` 的 `--scene`：1-QQ 2-邮箱 3-微信号 4-QQ好友 13-通讯录 14-群聊 15-手机号 30-二维码；`v3`/`v4` 来自 `contact search` 或群成员详情（`chatroom member-detail`）。若已是好友，search 结果的 V3 即好友 wxid。
- 建议账号在线 3 天后再调 `add-friend`；添加成功会推送一条含 V3 的回调消息，可用于判断是否成功。
- **`contact list --contact-type 4` 会返回 `-11 参数错误`**：该接口不支持企微类型；企微好友请用 `contact wework-list` / `contact wework-info`。
- **备注无法清空**：`set-remark` 的平台接口对空备注返回 `-11 参数错误`；且在 Windows PowerShell 下无法向 `--remark` 传空串（空参被丢弃、`--remark=` 又被必填校验拒绝）。需要清空备注请在微信客户端操作。
- **仅聊天无读回**：`set-chat-only` 返回 `code:0`，但 `brief`/`detail` 不返回仅聊天字段，无法用 API 二次确认，请以返回码为准。
