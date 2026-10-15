---
name: wekeapi-chatroom
description: wekeapi CLI chatroom 模块：微信群管理（创建群、邀请/添加成员、踢人、群主转让、管理员设置、群公告、群名/群备注/群内昵称、群二维码、拍一拍、邀请确认、解散/退出/保存群）。当用户要求建微信群、拉人进群、踢人、改群名、发群公告、转让群主或解散群时使用。
---

# chatroom 群聊

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）；stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 群 ID 形如 `12345@chatroom`。其余全局约定见 wekeapi-core skill。

## 命令表

**建群与成员进出**

| 命令 | 说明 | 参数 |
|---|---|---|
| `chatroom create` | 创建微信群 | **`--wxids`** 至少 2 个 |
| `chatroom members` | 获取群成员列表 | **`--chatroom-id`** |
| `chatroom member-detail` | 获取群成员详情（仅微信成员；企微成员用 contact wework-info） | **`--chatroom-id`**、**`--member-wxids`** |
| `chatroom add-friend` | 添加群成员为好友（发申请） | **`--chatroom-id`**、**`--member-wxid`**、**`--verify-content`** |
| `chatroom invite` | 邀请好友进群 | **`--chatroom-id`**、**`--wxids`** |
| `chatroom add-member` | 添加好友进群（data 为成功添加的 wxid 列表） | **`--chatroom-id`**、**`--wxids`** |
| `chatroom scan-join` | 扫二维码进群 | **`--qr-code-info`**（解析二维码得到的 URL） |
| `chatroom accept-join` | 同意进群邀请 | **`--url`**（回调消息中的 URL） |
| `chatroom approve-invite` | 同意成员邀请他人进群的申请（群主/管理员） | **`--msg-base64`**、**`--new-msg-id`** |
| `chatroom remove-member` | 移出群成员（群主/管理员） | **`--chatroom-id`**、**`--wxids`** |

**群主与管理员**（仅群主可操作）

| 命令 | 说明 | 参数 |
|---|---|---|
| `chatroom transfer-owner` | 转让群主 | **`--chatroom-id`**、**`--wxid`** 新群主 |
| `chatroom add-admin` | 添加管理员（最多 3 个） | **`--chatroom-id`**、**`--wxids`** |
| `chatroom remove-admin` | 移除管理员 | **`--chatroom-id`**、**`--wxids`** |
| `chatroom dissolve` | 解散群聊 | **`--chatroom-id`** |

**群信息与互动**

| 命令 | 说明 | 参数 |
|---|---|---|
| `chatroom qrcode` | 获取群二维码 | **`--chatroom-id`**；`--out` 保存图片 |
| `chatroom set-announcement` | 设置群公告 | **`--chatroom-id`**、**`--content`** |
| `chatroom get-announcement` | 获取群公告 | **`--chatroom-id`** |
| `chatroom set-invite-confirm` | 开/关邀请确认 | **`--chatroom-id`**、**`--invite-verify-set`** 1-开 0-关 |
| `chatroom rename` | 修改群名称 | **`--chatroom-id`**、**`--chatroom-name`** |
| `chatroom set-remark` | 修改群备注 | **`--chatroom-id`**、**`--chatroom-remark`** |
| `chatroom set-my-nickname` | 修改我在群内昵称 | **`--chatroom-id`**、**`--nick-name-in-chatroom`** |
| `chatroom pat` | 群拍一拍（对方 5 秒后收到） | **`--chatroom-id`**、**`--to-wxid`** |
| `chatroom quit` | 退出群聊 | **`--chatroom-id`** |
| `chatroom save` | 保存群到通讯录 | **`--chatroom-id`** |
| `chatroom unsave` | 从通讯录移除群 | **`--chatroom-id`** |

## 示例

```bash
# 建群并拉人
$WEKEAPI chatroom create --token T --app-id A --wxids wxid_a,wxid_b
$WEKEAPI chatroom invite --token T --app-id A --chatroom-id 12345@chatroom --wxids wxid_c

# 设置群公告
$WEKEAPI chatroom set-announcement --token T --app-id A --chatroom-id 12345@chatroom --content "周五开会"
```

## 注意

- **保存/解散与列表缓存**：`save` / `unsave` / `dissolve` 成功后，`contact list --contact-type 2` 会因平台缓存未即时刷新而可能与实际不一致（刚 save 的群看不到、已 dissolve 的群仍在列表）。此时执行 `contact sync` 强制同步（或稍等片刻）后再查即可，不代表操作失败。
- **群主/管理员权限**：`rename` / `set-remark` / `set-my-nickname` / `set-announcement` / `set-invite-confirm` / `transfer-owner` / `add-admin` / `remove-admin` / `remove-member` / `dissolve` 等需要群主或管理员权限。
- **成员详情**：`member-detail` 仅支持微信成员；企微成员请用 contact 模块的 `contact wework-info`。
