---
name: wekeapi-label
description: wekeapi CLI label 模块：微信标签管理（获取标签列表、添加标签、删除标签、修改好友标签）。当用户要求管理微信标签或给好友打标签时使用。
---

# label 标签

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）；stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定见 wekeapi-core skill。

## 命令表

| 命令 | 说明 | 参数 |
|---|---|---|
| `label list` | 获取标签列表 | — |
| `label add` | 添加标签 | **`--labels`** 名称列表（逗号分隔） |
| `label delete` | 删除标签 | **`--label-id`** |
| `label set-friend` | 修改好友标签 | **`--wxid`**、**`--label-ids`** 该好友的全部标签（逗号分隔） |

## 注意

- 手机端增删标签不会自动同步到平台；手机上加了新标签后，先调 `label list` 同步再操作。
- 标签数据由 wekeapi-msg-sync 随联系人一起同步；用 `$WEKEAPI data contacts --keyword <关键词>` 查询时，返回的 `label_ids`/`labels` 已把标签 ID 翻译成名称，可直接用于确认好友现有标签。
- `set-friend` 需提交该好友的**全部**标签（全量覆盖，不是增量）。

## 示例

```bash
# 新增标签并给好友打上
$WEKEAPI label add --token T --app-id A --labels 客户,同事
$WEKEAPI label set-friend --token T --app-id A --wxid wxid_xxx --label-ids 101,102
```
