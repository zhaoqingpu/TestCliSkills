---
name: wekeapi-login
description: wekeapi CLI login 模块：微信扫码登录、登录状态轮询、在线状态检查、免扫码快捷登录（调用后必须提示用户在手机上点确认登录）、退出登录，以及登录成功后的数据服务接管（自动写 wekeapi-msg-sync.json 并启动/重启 wekeapi-msg-sync，触发全量同步）。当用户要求登录微信、获取登录二维码、检查微信在线状态、掉线重连、免扫码快捷登录、退出微信登录或同步微信数据时使用。
---

# login 登录

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- 调用需 `--token`；除 `login qr` 外需 `--app-id`（运行 `wekeapi init` 保存到 `~/.wekeapi/config.json` 后可免传，显式 flag 优先）。
- stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定（--data 透传、--timeout 等）见 wekeapi-core skill。

## 登录编排（按序执行：状态检查 → 免扫码重登 → 扫码登录 → 数据同步服务接管）

### 第零步：读取配置与在线状态检查（每次触发微信操作先走这步）

1. 读取 `~/.wekeapi/config.json`（`init` 保存的 token/appId；旧版 `~/.wekeapi.json` 会被自动兼容读取）：
   - 有 token：免传凭证，继续下一步。
   - **无 token：必须中断流程，用 `AskUserQuestion` 弹框向用户索要 AccessToken，等用户在弹框里提供后再继续；拿到 token 之前不得执行任何后续步骤（不得取二维码、不得探测在线状态等）。** 弹框选项：「粘贴 AccessToken（选 `Other` 输入）」/「还没有，去官网申请」；用户选后者时展示 `$WEKEAPI env` 输出的 `siteURL`，申请完成后再次弹框收集。**禁止在回复里让用户在聊天框打字回复 token。** 拿到后执行 `wekeapi init --token <TOKEN>` 保存（自动化场景带 flag；API 地址编译期内置，无需收集 baseUrl）。
2. 已有 appId 时先检查在线状态，**不要直接扫码**：`$WEKEAPI login status --token T --app-id A`（默认不自动重连；确需重连时显式传 `--auto-connect`）
   - 返回 1（在线）：CLI 已自动把 wxid/appId 写入配置，并把最新凭证写入 `wekeapi-msg-sync.json`，确保数据同步服务按最新凭证运行（配置未变化且服务已在运行时不会重启）；直接执行用户操作，登录编排结束。
   - 返回 0（未登录）或 -1（离线）：config 中已有 appId，优先进入第一步免扫码快捷登录（不区分主动/被动退出）。
3. 无 appId（首次使用或配置被清空）：**必须中断流程，向用户询问并等待明确答复，二者其一确定后才可进入第二步扫码登录**：
   - 用户提供已有 appId → 用该 appId（可 `wekeapi init --app-id <APPID>` 保存）继续；
   - 用户没有 appId 或选择「使用新设备」→ 视为新设备登录（`login qr` 用 `--new-device`，首次不传 `--app-id` 由系统自动生成），进入第二步扫码登录。

### 第一步：免扫码快速登录（未登录/离线时优先尝试）

1. `$WEKEAPI login quick --token T --app-id A`：免扫码快捷登录，成功返回新的 `qrCodeUUID`。
   - **必须立即提示用户到手机确认（高优先级）**：`login quick` 成功返回后，**在下一次回复的第一句用醒目文字提示用户**「已向你的微信推送登录确认，请现在打开手机微信，在弹出页点击「确认登录」，否则确认超时会失败」；不得静默直接轮询让用户不知道要做什么。
   - 提示同时告知：确认有效期有限，超时未点确认则本次快捷登录失败（会转扫码登录）；手机没有收到弹窗时可改用扫码登录（第二步）。
   - 提示发出后再开始轮询；轮询超过约 30 秒仍未见用户确认（status 仍为 0/11）时，再提醒一次用户点手机上的确认，不要每 5 秒刷屏。
2. 用返回的 `qrCodeUUID` 每 5 秒轮询 `$WEKEAPI login check --token T --app-id A --qr-code-uuid U`：
   - status=2 → 登录成功：CLI 自动记录 appId/wxid，并把凭证同步到 `wekeapi-msg-sync.json`、（重启）启动数据同步服务 → 确认服务就绪后执行用户操作。
   - status=11（待手机点确认）：**再次提示用户在手机上点击「确认登录」**，继续轮询。
   - `login quick` 返回 -24（新设备登录后掉线、风控下线等被动退出不支持 quick），或轮询超时/失败 → 转第二步扫码登录。
3. config 已有 appId 且离线（0/-1）时一律优先 quick，无需区分主动/被动退出；仅当 quick 返回 -24 或轮询超时/失败，才转第二步扫码登录。

### 第二步：扫码登录（无 appId 或快速登录失败时）

#### 2.1 收集登录参数（缺一不可，向用户逐项确认后才能进入 2.2）

> 本节所有问题（登录平台、验证方式、代理方式、代理 ID/Url、省市选择）一律用 `AskUserQuestion` 弹框按序收集（见 wekeapi-core「交互输入规范」），不要在聊天框里让用户输入；互不依赖的问题（如登录平台）可放同一弹框，有前后依赖的分开弹。

1. 登录平台 `--device-type`：`Mac` / `iPad`，二选一，必须弹框询问。
2. 自动验证 `--verify-type`（仅平台为 Mac 时询问）：`auto` / `manual`；平台为 iPad 时不询问，固定 `manual`（iPad 大概率需手动人脸认证，不支持自动验证）。
3. 代理模式 `--proxy-type`：`local`（本地代理）/ `self`（自有代理）/ `platform`（平台代理），三选一，必须弹框询问，**优先推荐 local（本地代理）**——向用户展示选项时把 local 放首位并标注「推荐」，仅当用户明确选择其他方式或 local 不可用（无法安装本地代理工具）时才用 self/platform；并按所选模式收集 `--proxy-info`：
   - local：本地代理 ID（形如 `pzqn-xxxx-xxxx`，CLI 无命令可查询）需用户安装官方本地代理工具并运行后才能拿到。运行 `$WEKEAPI env`，从 `proxyDownloads` 取当前环境对应平台（按用户操作系统匹配）的 `url` 展示给用户（Linux 直接展示 `note` 中的安装命令），待用户下载并运行工具取得代理 ID 后再弹框向其索要（选 `Other` 粘贴）。
   - self：弹框向用户索要代理 Url（http(s) 地址，选 `Other` 粘贴）。
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
   - status 2：停止轮询，appId 已保存到配置，CLI 自动补调一次在线状态检测记录 wxid，并把最新凭证同步到 `wekeapi-msg-sync.json`、（重启）启动数据同步服务 → 确认服务就绪后执行用户操作。
   - status 3：报错终止；status 10：触发身份验证，接口返回验证二维码，需账号持有人用专用扫码验证工具验证，提示用户。
   - 轮询超时（二维码过期）：用同一 `--app-id` 重新执行 2.1 取新二维码。

## 第三节：登录成功后的数据服务接管（固定步骤，任何登录路径成功后必经）

- 触发时机：扫码登录或快捷登录轮询到 status=2 后、执行用户原操作之前，必须先完成数据服务接管。
- CLI 已自动完成，**无需再执行旧的 `sync all`**：
  1. 把主配置中的 `token`/`appId`/`baseURL`/`wxid` 写入服务目录的 `wekeapi-msg-sync.json`；
  2. 若 wekeapi-msg-sync 已启动，先关闭再启动；未启动则直接启动（首次启动按默认配置：SQLite `wekeapi_msg_sync.db`、端口 `3006`）；
  3. 服务启动后立即执行一次全量同步（标签/好友/群组/公众号/企微好友）并按 `sync_interval` 周期差量同步。
- 服务目录查找顺序：`--msg-sync-dir`（`data` 命令组持久 flag）> 环境变量 `WEKEAPI_MSG_SYNC_HOME` > `~/.wekeapi/config.json` 的 `msgSyncDir`（`init --msg-sync-dir` 可设置）> 默认 `~/.wekeapi/msg-sync`；配置 `wekeapi-msg-sync.json`、日志 `wekeapi-msg-sync.log`、PID 与 SQLite 数据库都在该目录。
- 登录命令的 stdout 仍是登录接口 JSON；服务启动进度与结果提示走 stderr（前缀 `[msg-sync]`）。若启动失败（如找不到服务可执行文件），登录仍算成功，但需向用户报告警告并检查服务目录。

### 3.1 数据服务配置（登录成功后必须询问用户，按序弹框，不可跳过）

登录成功时服务已按默认值启动，**紧接着必须让用户设置数据库与服务端口**，确认后用 `data config` 改写配置、`data sync` 重启生效。

**全部用 `AskUserQuestion` 弹框按序收集（顺序固定：先数据库类型 → 选 MySQL 再问连接 URI → 最后端口），一次只弹一个问题，上一框答复后再弹下一框；禁止在聊天框里让用户输入。**

先查看当前配置（不带修改参数时**只读、不写文件**）：

```bash
$WEKEAPI data config
```

- 输出 `data.db`（sqlite/mysql）、`data.db_path`/`data.dsn`（DSN 已脱敏）、`data.port`。
- 若已是用户此前确认过的非默认配置（如 MySQL 或端口不是 3006），先弹一次框问「是否沿用当前配置（数据库 X，端口 Y）」（选项「沿用当前配置」/「重新配置」），选沿用则直接跳到 3.3。
- 不要替用户决定数据库类型。

**弹框 1：数据库类型（二选一，本框只问类型，不附带 URI/端口）**

| 选项 | 选定后的下一步 |
|---|---|
| `SQLite`（推荐，零部署，单机够用） | 直接进入弹框 2 问服务端口；**文件名固定默认 `wekeapi_msg_sync.db`（服务目录 `~/.wekeapi/msg-sync`），不再追问文件名** |
| `MySQL`（多端共享/已有 MySQL） | 弹框 2 先收集 MySQL 连接 URI，拿到后弹框 3 问服务端口 |

**弹框 2（根据上一框分支，不得合并或跳过）**

- 选 `MySQL`：弹框收集 **MySQL 连接 URI**（自由文本，选项「粘贴连接串（选 `Other`）」），问题描述里给示例：`mysql://user:pass@127.0.0.1:3306/dbname?charset=utf8mb4`；也接受 `user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4`、`jdbc:mysql://...`（服务自动剥离 `jdbc:` 前缀、转换 JDBC 参数）。库不存在时服务启动会**自动创建**（utf8mb4）并自动建表，账号需有 CREATE 权限；无建库权限时再提示用户手工建库。**拿到 URI 后再弹框 3 问端口。**
- 选 `SQLite`：**直接弹框问服务端口**（不进 URI 环节，也不问文件名）。

**弹框 3：服务端口**

- 默认 `3006`（选项给 `3006（默认）`）+ `Other` 自定义 1-65535 端口。端口被占用时服务启动会失败（日志见 `~/.wekeapi/msg-sync/wekeapi-msg-sync.log`），换端口重试即可。

### 3.2 写入配置并重启生效

```bash
# SQLite：只传端口，文件名保留默认 wekeapi_msg_sync.db（不必也不传 --db-path）
$WEKEAPI data config --db sqlite --port <端口>
# MySQL：连接串含特殊字符，PowerShell 用单引号包裹
$WEKEAPI data config --db mysql --db-dsn '<连接 URI>' --port <端口>
# 重启服务使配置生效（重启后自动全量同步）
$WEKEAPI data sync
```

- `data config` 只改数据库与端口，保留 token/appId/wxid 等字段；输出 `data.changed` 表示配置是否变化，`data.needRestart=true` 表示运行中的服务仍用旧配置。
- **切换数据库类型不迁移旧数据**：换成 MySQL 后需等重新全量同步完成；旧 SQLite 文件保留在 `~/.wekeapi/msg-sync`，可提示用户自行清理。
- Windows PowerShell 下连接串用单引号（`--db-dsn '...'`），避免 `$`/`@`/`&` 被解释；**不要写 `--db-dsn ""`**。
- 用户拒绝调整（明确表示用默认值）时，也至少确认一次端口，再按默认值跳过写入。

### 3.3 确认就绪

- 确认服务就绪：`$WEKEAPI data status`（返回 `data.status` 为 `idle`/`syncing`/`error`，并给出 `contacts_count`/`rooms_count`/`messages_count`；首次同步完成前可能为 `syncing`）。
- 后续查询数据用 `$WEKEAPI data contacts/messages/rooms`（见 wekeapi-core skill），不要再用 sqlite3 查本地数据库；服务未运行时可 `$WEKEAPI data start`，需要重新全量同步时用 `$WEKEAPI data sync`（等价 `data restart`）。
- 同步完成（或失败已报告）后才视为登录编排结束，继续执行用户原操作。

## 命令表

| 命令 | 说明 | 参数 |
|---|---|---|
| `login qr` | 获取登录二维码 | **`--device-type`** Mac/iPad、**`--verify-type`** manual/auto、**`--proxy-type`** local/self/platform、**`--proxy-info`**（local 传代理ID，需用户安装本地代理工具获取，下载链接见 2.1 / self 传代理Url / platform 传城市代码，须经用户选择微信常用地与可用性检测，流程见 2.1 与 wekeapi-proxy skill）；`--new-device` 按全新设备取码（忽略配置 appId，等价 `--app-id=`）；`--out` 保存二维码图片 |
| `login check` | 轮询检测登录状态 | **`--qr-code-uuid`** |
| `login status` | 检查在线状态（1 在线 / -1 离线 / 0 未登录）；在线时自动记录 wxid/appId，并确保数据同步服务按最新凭证运行 | `--auto-connect` 掉线时是否自动重连（可选，默认不重连） |
| `login quick` | 快捷登录（免扫码，手机点确认）；**调用成功后必须立即提示用户到手机微信点「确认登录」** | — |
| `login logout` | 主动退出登录 | — |
| `data status` | 登录后查看数据同步状态与数据量（wekeapi-msg-sync） | — |
| `data config` | 登录成功后设置数据同步服务的数据库（SQLite/MySQL）与端口（弹框按序收集：类型 → MySQL 连接 URI → 端口） | `--db` sqlite/mysql、`--db-path` SQLite 文件名（默认 `wekeapi_msg_sync.db`，登录流程不传）、`--db-dsn` MySQL 连接串、`--port`（默认 3006）；不带参数只查看当前配置 |

## 注意

- appId 即设备身份：设备类型（Mac/iPad）在 appId 首次生成时绑定，之后传 `--device-type` 不再改变；换设备类型需按新设备登录（`--new-device` 或 `--app-id=`）。
- appId 仅在登录成功（status=2）后落盘 `~/.wekeapi/config.json`；登录成功前调用 `login check` 或重取二维码必须显式传 `--app-id`。
- 账号信息记录：统一以在线状态检测响应为数据源——`login status` 在线时自动记录；`login check` 登录成功后 CLI 自动补调一次 status 完成同样记录（appId 先落盘配置）。配置只存 wxid/appId，业务数据（联系人/消息/群聊）由 wekeapi-msg-sync 服务存储，用 `data` 命令查询。
- deviceType 优先 Mac（支持自动验证，成功率约 90%）；连续 2 次自动验证失败后改用 manual，勿再重试 auto。
- 如需保留 PC 登录机会可选 iPad，但大概率需手动人脸认证。
