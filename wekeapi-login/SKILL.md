---
name: wekeapi-login
description: wekeapi CLI login 模块：微信扫码登录、登录状态轮询、在线状态检查、免扫码快捷登录、退出登录，以及登录成功后的数据同步（好友/群组/公众号/企微好友同步到本地 SQLite）。当用户要求登录微信、获取登录二维码、检查微信在线状态、掉线重连、免扫码快捷登录、退出微信登录或同步微信数据时使用。
---

# login 登录

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token`；除 `login qr` 外需 `--app-id`（运行 `wekeapi init` 保存到 `~/.wekeapi/config.json` 后可免传，显式 flag 优先）。
- stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定（--data 透传、--timeout 等）见 wekeapi-core skill。

## 登录编排（按序执行：状态检查 → 免扫码重登 → 扫码登录 → 数据同步）

### 第零步：读取配置与在线状态检查（每次触发微信操作先走这步）

1. 读取 `~/.wekeapi/config.json`（`init` 保存的 token/appId；旧版 `~/.wekeapi.json` 会被自动兼容读取）：
   - 有 token：免传凭证，继续下一步。
   - **无 token：必须中断流程，向用户索要 AccessToken 并等待用户提供；拿到 token 之前不得执行任何后续步骤（不得取二维码、不得探测在线状态等）。** 用户提供后执行 `wekeapi init --token <TOKEN>` 保存（自动化场景带 flag；API 地址编译期内置，无需收集 baseUrl）。若用户没有 token，引导其到官网（`env` 输出的 `siteURL`）申请，并保持等待状态，直到用户提供 token 才继续。
2. 已有 appId 时先检查在线状态，**不要直接扫码**：`$WEKEAPI login status --token T --app-id A`（默认不自动重连；确需重连时显式传 `--auto-connect`）
   - 返回 1（在线）：响应携带的账号个人信息已被 CLI 自动落库（wxid/appId 更新到配置、完整信息写入账号分库 `accounts` 表），直接执行用户操作，登录编排结束。
   - 返回 0（未登录）或 -1（离线）：config 中已有 appId，优先进入第一步免扫码快捷登录（不区分主动/被动退出）。
3. 无 appId（首次使用或配置被清空）：**必须中断流程，向用户询问并等待明确答复，二者其一确定后才可进入第二步扫码登录**：
   - 用户提供已有 appId → 用该 appId（可 `wekeapi init --app-id <APPID>` 保存）继续；
   - 用户没有 appId 或选择「使用新设备」→ 视为新设备登录（`login qr` 用 `--new-device`，首次不传 `--app-id` 由系统自动生成），进入第二步扫码登录。

### 第一步：免扫码快速登录（未登录/离线时优先尝试）

1. `$WEKEAPI login quick --token T --app-id A`：免扫码快捷登录，手机弹窗点确认即可；成功返回新的 `qrCodeUUID`。
2. 用返回的 `qrCodeUUID` 每 5 秒轮询 `$WEKEAPI login check --token T --app-id A --qr-code-uuid U`：
   - status=2 → 登录成功 → 执行第三节「数据同步」→ 执行用户操作。
   - `login quick` 返回 -24（新设备登录后掉线、风控下线等被动退出不支持 quick），或轮询超时/失败 → 转第二步扫码登录。
3. config 已有 appId 且离线（0/-1）时一律优先 quick，无需区分主动/被动退出；仅当 quick 返回 -24 或轮询超时/失败，才转第二步扫码登录。

### 第二步：扫码登录（无 appId 或快速登录失败时）

#### 2.1 收集登录参数（缺一不可，向用户逐项确认后才能进入 2.2）

1. 登录平台 `--device-type`：`Mac` / `iPad`，二选一，必须询问。
2. 自动验证 `--verify-type`（仅平台为 Mac 时询问）：`auto` / `manual`；平台为 iPad 时不询问，固定 `manual`（iPad 大概率需手动人脸认证，不支持自动验证）。
3. 代理模式 `--proxy-type`：`local`（本地代理）/ `self`（自有代理）/ `platform`（平台代理），三选一，必须询问，**优先推荐 local（本地代理）**——向用户展示选项时把 local 放首位并标注「推荐」，仅当用户明确选择其他方式或 local 不可用（无法安装本地代理工具）时才用 self/platform；并按所选模式收集 `--proxy-info`：
   - local：本地代理 ID（形如 `pzqn-xxxx-xxxx`，CLI 无命令可查询）需用户安装官方本地代理工具并运行后才能拿到。运行 `$WEKEAPI env`，从 `proxyDownloads` 取当前环境对应平台（按用户操作系统匹配）的 `url` 展示给用户（Linux 直接展示 `note` 中的安装命令），待用户下载并运行工具取得代理 ID 后再向其索要。
   - self：向用户索要代理 Url（http(s) 地址）。
   - platform：按「用户选择微信常用地 → 匹配城市代码 → 检测可用性 → 用城市码取二维码」流程执行（详见 wekeapi-proxy skill「平台代理选择流程」）：
     1. 采集省市：**让用户选择**，提醒文案用「选择微信常用地」，并向用户说明微信常用地通常指长期居住地或手机号归属地。两步采集、不可跳过：先 `$WEKEAPI proxy provinces` 取省份列表展示选项（选项展示不完整时提示用户手动输入省名）；**选完省后必须继续选该省对应的市**——`$WEKEAPI proxy cities --province <所选省>` 取该省城市列表展示选项（选项展示不完整时提示用户手动输入市名）。
     2. 匹配城市代码：`$WEKEAPI proxy cities --province <所选省>` 与所选市名（去掉"省/市"后缀）精确匹配取 `data[0].code`；城市不在对照表（如 上海）→ 提示切换代理方式或手选其他城市。
     3. 检测可用性：`$WEKEAPI proxy check-city --token T --app-id A --city-code <code>`（配置无 appId 时改用 `--app-id=` 传空 appId，实测结果一致），按 `data` 分支：
        - `data=true`：该城市有可用代理 → 用该城市代码作为 `--proxy-info` 进入 2.2 获取二维码。
        - `data=false`：**提示用户当前所在城市没有可用代理，请切换其他代理方式（local/self），终止 platform 流程**。
   - 用户消息中已明确的信息直接采用，不重复询问；只补问缺失项。

#### 2.2 获取二维码并轮询（仅当 2.1 三项信息全部齐备）

1. `$WEKEAPI login qr --token T --device-type <Mac|iPad> --verify-type <auto|manual> --proxy-type <local|self|platform> --proxy-info <代理ID|代理Url|城市代码> --out qr.jpg`
   - 无 appId（用户选择「使用新设备」）时按全新设备取码（`--new-device`，或首次不传 `--app-id` 由系统自动生成）；配置中已有 appId 时默认复用它（读配置），确需按全新设备取二维码用 `--new-device`。**必须记录返回 JSON 中的 `appId` 与 `qrCodeUUID`**。
   - **`login qr` 不保存 appId 到配置**：appId 只在登录成功（`login check` 返回 status=2）后才自动保存到 `~/.wekeapi/config.json`（stderr 有提示）。
   - 登录成功前的所有调用（`login check` 轮询、二维码超时后重新 `login qr`）必须显式传同一 `--app-id`（首次登录用 qr 返回的 appId），否则会被当作新设备重新生成 appId，容易触发风控。
   - 设备类型绑定在 appId 上：携带已有 appId 再取二维码时，`--device-type` 不改变设备类型（手机弹窗仍显示 appId 首次生成时的设备类型）。要更换设备类型必须按全新设备登录：配置中已有 appId 时用 **`--new-device`**（等价于显式空值 `--app-id=`），视为新设备登录、有风控风险，先向用户确认。**不要写 `--app-id ""`**：Windows PowerShell 5.1 会丢弃空字符串参数，导致 `--app-id` 吞掉后一个 flag（如 `--out`）并把垃圾值发给服务端报 `-50 服务异常`。
   - **展示二维码必须最快完成（高优先级）**：`login qr` 命令返回后，必须在**下一次回复中立即**用 markdown 图片语法展示二维码（`![qr](file:///qr.jpg绝对路径)`），让用户在聊天中直接看到并扫码；禁止任何中间步骤——不得加载其他 skill、不得渲染 widget、不得先做任何其他工具调用或解释；展示二维码之前不要启动轮询。
2. 每 5 秒轮询：`$WEKEAPI login check --token T --app-id A --qr-code-uuid U`
   - `data.status`：0 未扫码 / 1 已扫码 / 2 已登录 / 3 失败 / 10 需验证 / 11 待手机点确认。
   - status 2：停止轮询，appId 已保存到配置，CLI 自动补调一次在线状态检测采集个人信息（wxid 落配置、完整信息落库）→ 执行第三节「数据同步」→ 执行用户操作。
   - status 3：报错终止；status 10：触发身份验证，接口返回验证二维码，需账号持有人用专用扫码验证工具验证，提示用户。
   - 轮询超时（二维码过期）：用同一 `--app-id` 重新执行 2.1 取新二维码。

## 第三节：登录成功后数据同步（固定步骤，任何登录路径成功后必经）

- 触发时机：扫码登录或快捷登录轮询到 status=2 后、执行用户原操作之前，必须先完成数据同步。
- 命令：`$WEKEAPI sync all`（可选 `--types labels,friends,chatrooms,ghs,wework`、`--db <路径>`，通常全部省略）。
- 缺省全量同步五类数据到本地 SQLite（**每账号一库**，默认 `~/.wekeapi/data-<wxid>.db`，wxid 取配置；配置缺 wxid 会 exit 2——先登录成功一次或显式传 `--db`；旧版库 `~/.wekeapi/data.db` / `~/.wekeapi.db` 存在且新库不存在时自动复制迁移，旧文件保留）：`labels` 标签 / `friends` 好友 / `chatrooms` 群组 / `ghs` 公众号 / `wework` 企微好友。
- **标签最先同步**（labels 表，appId 归属与 contacts 一致，全量 upsert + 删除本地多余）：标签是联系人 `labels` 列（逗号分隔标签 ID）的字典数据，先于联系人入库以便关联查询；`--types` 选中 labels 时无论位置均最先执行。
- 同步只需 token/appId（通讯录接口无需 wxid），自动读取配置，无需显式传参；本地数据以 appId 作为归属键入库。
- stdout 输出同步摘要 JSON（各类型 Fetched/Added/Deleted/Updated 与 dbPath）；进度日志走 stderr。
- 退出码：0 全部成功 / 1 存在类型同步失败 / 2 本地错误（参数/数据库）。退出码 1 时读取 stdout 摘要，向用户报告失败类型，可用 `--types <失败类型>` 单独重试一次；退出码 2 时检查参数与磁盘，不要反复重试。
- 群组只做新增/更新不删除（接口只返回保存到通讯录的群）；好友自动过滤系统账号（filehelper、fmessage 等）。
- 同步完成（或失败已报告）后才视为登录编排结束，继续执行用户原操作。

## 命令表

| 命令 | 说明 | 参数 |
|---|---|---|
| `login qr` | 获取登录二维码 | **`--device-type`** Mac/iPad、**`--verify-type`** manual/auto、**`--proxy-type`** local/self/platform、**`--proxy-info`**（local 传代理ID，需用户安装本地代理工具获取，下载链接见 2.1 / self 传代理Url / platform 传城市代码，须经用户选择微信常用地与可用性检测，流程见 2.1 与 wekeapi-proxy skill）；`--new-device` 按全新设备取码（忽略配置 appId，等价 `--app-id=`）；`--out` 保存二维码图片 |
| `login check` | 轮询检测登录状态 | **`--qr-code-uuid`** |
| `login status` | 检查在线状态（1 在线 / -1 离线 / 0 未登录）；在线时自动记录账号信息（wxid/appId → 配置，完整个人信息 → 账号分库 `accounts` 表） | `--auto-connect` 掉线时是否自动重连（可选，默认不重连） |
| `login quick` | 快捷登录（免扫码，手机点确认） | — |
| `login logout` | 主动退出登录 | — |
| `sync all` | 登录成功后数据同步：标签（最先）/好友/群组/公众号/企微好友 → 本地 SQLite（第三节） | `--types`（缺省全量）、`--db`（默认 `~/.wekeapi/data-<wxid>.db`，wxid 取配置）均可省略 |

## 注意

- appId 即设备身份：设备类型（Mac/iPad）在 appId 首次生成时绑定，之后传 `--device-type` 不再改变；换设备类型需按新设备登录（`--new-device` 或 `--app-id=`）。
- appId 仅在登录成功（status=2）后落盘 `~/.wekeapi/config.json`；登录成功前调用 `login check` 或重取二维码必须显式传 `--app-id`。
- 账号信息记录：统一以在线状态检测响应为数据源——`login status` 在线时自动记录；`login check` 登录成功后 CLI 自动补调一次 status 完成同样记录（appId 先落盘配置）。个人信息（昵称/别名/头像）upsert 到该账号分库 `~/.wekeapi/data-<wxid>.db` 的 `accounts` 表，配置只存 wxid，查完整信息（如显示昵称）用 sqlite3 查 `accounts` 表。
- deviceType 优先 Mac（支持自动验证，成功率约 90%）；连续 2 次自动验证失败后改用 manual，勿再重试 auto。
- 如需保留 PC 登录机会可选 iPad，但大概率需手动人脸认证。
