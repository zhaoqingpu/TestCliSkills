---
name: wekeapi-message
description: wekeapi CLI message 模块：微信消息发送（文本/图片/视频/文件/语音/链接/小程序/Emoji/名片/AppMsg）、消息转发与撤回、下载收到的媒体（图片/视频/语音/文件/Emoji/CDN），以及通过 wekeapi-msg-sync 查询消息记录。当用户要求发微信消息、群发、转发消息、@群成员、撤回消息、查询聊天记录或下载消息里的附件时使用。
---

# message 消息

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）；stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- `--to-wxid` 好友、群通用（群 ID 形如 `12345@chatroom`）。其余全局约定见 wekeapi-core skill。
- 用户给的是微信号/昵称/备注而非 wxid 时，先用 `$WEKEAPI data contacts --keyword <关键词>` 解析（`data.contacts[].id` 即 wxid），查不到再调远端接口；**不要再用 sqlite3 查本地数据库**。
- **消息由 wekeapi-msg-sync 统一存储**：服务接收消息回调并落库，主 CLI 不再写本地 messages 表。查询聊天记录用 `$WEKEAPI data messages`（`--conversation-id S:<wxid>|R:<roomid>`、`--keyword`、`--start-time`/`--end-time`、`--direction 0|1|2`、`--page`/`--limit`），返回 `data.messages[]`（含 `new_msg_id`/`from`/`to`/`room_id`/`content`/`create_time`/`revoked`）。

## 发送

| 命令 | 说明 | 参数 |
|---|---|---|
| `message send-text` | 发文本 | **`--to-wxid`**、**`--content`**；`--ats` 群@成员（逗号分隔，@全部填 `all`，内容需含"@昵称"） |
| `message send-voice` | 发语音（wav/SILK，≤60 秒） | **`--to-wxid`**、**`--file`/`--url`** |
| `message send-image` | 发图片（≤20MB，jpg/png/bmp/webp/tiff） | **`--to-wxid`**、**`--file`/`--url`** |
| `message send-video` | 发视频（≤200MB） | **`--to-wxid`**、**`--file`/`--url`** |
| `message send-file` | 发文件（≤200MB） | **`--to-wxid`**、**`--file`/`--url`**、**`--file-name`** 含扩展名 |
| `message send-link` | 发链接 | **`--to-wxid`**、**`--title`**、**`--link-url`**、**`--file`/`--url`** 缩略图（≤5MB）；`--desc` |
| `message send-miniapp` | 发小程序（先发给自己取接收消息的 xml） | **`--to-wxid`**、**`--xml`/`--xml-file`** |
| `message send-emoji` | 发 Emoji | **`--to-wxid`**、**`--emoji-md5`**、**`--emoji-size`** |
| `message send-card` | 发名片 | **`--to-wxid`**、**`--card-wxid`**、**`--card-nickname`**；`--card-alias` |
| `message send-appmsg` | 发 AppMsg（视频号/聊天记录/音乐等） | **`--to-wxid`**、**`--xml`/`--xml-file`**；提供 ori* 参数则按转发发送 |
| `message revoke` | 撤回消息（newMsgId/createTime 取自发送响应） | **`--to-wxid`**、**`--new-msg-id`**、**`--create-time`** |

## 转发（同一内容发多人：先 send-* 发第一人，再 forward-*，效率更高）

**技巧**：把发送接口的完整响应（stdout 原始 JSON）存文件，再 `forward-* --to-wxid 下一人 --data @resp.json` 即可。CLI 会**自动解包**响应包裹（取其中的 `data`）并**自动补全 API 必需的 ori 字段**：
- `oriMsgId ← newMsgId`
- `oriMsgSourceBase64 ← msgSourceBase64`
- `oriMsgTo ← 响应中的 toWxid`（原接收人）
- `oriMsgFrom ← 当前账号 wxid`（取配置 `~/.wekeapi/config.json`）

因此通常**只需再补 `--to-wxid` 指定新接收人**（显式 flag 优先于 `--data`）。若响应用 `jq .data` 只存了内层对象，也同样可用。

| 命令 | 数据来源 | 关键参数 |
|---|---|---|
| `message forward-image` | send-image 的响应 | **`--to-wxid`** + CDN 字段（`--file-id --file-md5 --aes-key --big-img-size --mid-img-size --thumb-img-width/height/size`；ori 自动补全） |
| `message forward-image-msg` | 收到图片消息的 xml | **`--to-wxid`**、**`--xml`/`--xml-file`**；ori 字段可选 |
| `message forward-video` | send-video 的响应 | **`--to-wxid`** + CDN 字段（`--file-id --file-md5 --aes-key --new-md5 --crc32 --file-size --play-length --raw-file-id/md5/aes-key/size --thumb-width/height/size`；ori 自动补全） |
| `message forward-video-msg` | 收到视频消息的 xml | **`--to-wxid`**、**`--xml`/`--xml-file`** |
| `message forward-file` | send-file 的响应 | **`--to-wxid`**、**`--file-name`** + CDN 字段（`--file-id --file-md5 --aes-key --file-crc --file-size`；ori 自动补全） |
| `message forward-file-msg` | 收到文件消息的 xml | **`--to-wxid`**、**`--xml`/`--xml-file`** |
| `message forward-link` | send-link 的响应 | **`--to-wxid`**、**`--title`**、**`--link-url`** + CDN 字段（`--aes-key --thumb-url --thumb-md5 --thumb-width/height/size --req-id`；ori 自动补全） |
| `message forward-link-msg` | 收到链接消息的 xml | **`--to-wxid`**、**`--xml`/`--xml-file`** |
| `message forward-miniapp` | 收到小程序消息的 xml | **`--to-wxid`**、**`--xml`/`--xml-file`** + ori 字段 |

> ori 字段 = `--ori-msg-id/--ori-msg-from/--ori-msg-to/--ori-msg-source`（原消息 newMsgId、发送/接收人 wxid、MsgSource，自动 base64）。除 `forward-miniapp`（来自接收消息的 xml）外，基于发送响应的 `forward-image/video/file/link` 均会自动补全，无需手填。完整字段含义见 `$WEKEAPI message forward-image --help`。

## 下载（均支持 `--out <路径>` 直接保存文件）

| 命令 | 说明 | 参数 |
|---|---|---|
| `message download-voice` | 下载语音 | **`--new-msg-id`**、**`--xml`/`--xml-file`**、**`--format`** 1-SILK/2-WAV、**`--from-wxid`** |
| `message download-image` | 下载图片 | **`--type`** 1-原图/2-常规/3-缩略、**`--xml`/`--xml-file`**；无对应规格返回 -33 |
| `message download-video` | 下载视频 | **`--type`** 1-原视频/2-常规/3-缩略、**`--xml`/`--xml-file`** |
| `message download-file` | 下载文件（xml 中 type=6 的消息） | **`--xml`/`--xml-file`** |
| `message download-emoji` | 下载 Emoji（URL 7 天有效） | **`--xml`/`--xml-file`** |
| `message download-cdn` | 无 xml 时用 CDN 字段下载 | **`--type`** 1-高清图/2-常规图/3-缩略图/4-视频/5-文件、**`--file-id`**、**`--aes-key`**、**`--file-size`** |

## 示例

```bash
# 群里发文本并@所有人
$WEKEAPI message send-text --token T --app-id A --to-wxid 12345@chatroom --content '@all 开会了' --ats all

# 发图片给第一人，用响应转发给其他人（高效群发）
$WEKEAPI message send-image --token T --app-id A --to-wxid wxid_a --file ./photo.jpg > resp.json
$WEKEAPI message forward-image --token T --app-id A --to-wxid wxid_b --data @resp.json

# 下载收到的图片原图
$WEKEAPI message download-image --token T --app-id A --type 1 --xml '<msg><img .../></msg>' --out ./dl.jpg
```

## 注意

- **发送限流**：短时间内连续发送会返回 `-12 调用超限`。两两发送之间建议间隔十余秒，遇 `-12` 等待后重试。
- **撤回窗口**：微信撤回仅在约 2 分钟内有效，超时返回 `-52`（后端 `[-430]`）。需要撤回时请"先发后立即撤"（拿到 newMsgId/createTime 后马上调 `revoke`）。
- **目标不校验**：向不存在或非法的 `--to-wxid` 发送文本，服务端不做校验，仍返回 `code:0`（不会报业务错误）；请自行确保目标 wxid/群 ID 正确。
- **语音格式**：`send-voice` 仅支持 wav / 原生 SILK，其它格式返回 `-30 不支持的格式`。
- 消息存储由 wekeapi-msg-sync 负责，查询用 `$WEKEAPI data messages`；发送/转发命令本身不再写本地数据库。
