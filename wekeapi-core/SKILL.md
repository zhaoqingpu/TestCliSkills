---
name: wekeapi-core
description: wekeapi CLI 共享基础：二进制位置、全局参数（--token/--app-id）、退出码与调用约定，数据同步服务（wekeapi-msg-sync）与 data 查询命令，并路由到 login/contact/chatroom/label/message/proxy 各模块 skill。任何 wekeapi 微信任务开始前、或不确定该用哪个模块 skill 时使用。
---

# wekeapi CLI 共享基础

wekeapi CLI 操作微信（wekeapi 微信协议 API）。全部参数走命令行，stdout 恒为 JSON。本 skill 是各模块 skill 的共享基础；命令详情见对应模块 skill。

## 二进制

预编译二进制打包在本 skill 的 `assets/bin/` 下（项目内路径 `.trae/skills/wekeapi-core/assets/bin/`），**无需源码、无需构建**。按 OS/架构选择（下文以 `$WEKEAPI` 代指）：

| OS/架构 | 路径（相对本 skill 目录） |
|---|---|
| Windows x64 | `assets/bin/wekeapi-windows-amd64.exe` |
| Linux x64 | `assets/bin/wekeapi-linux-amd64` |
| macOS Intel | `assets/bin/wekeapi-darwin-amd64` |
| macOS ARM (M 系列) | `assets/bin/wekeapi-darwin-arm64` |

Linux/macOS 首次使用前执行一次 `chmod +x <对应路径>`。若 macOS 报「无法打开，因为无法验证开发者」（Gatekeeper 拦截），执行 `xattr -d com.apple.quarantine <对应路径>` 清除隔离属性后重试。若系统 `PATH` 中已有 `wekeapi`，优先使用 PATH 中的版本。

同一 `assets/bin/` 目录下还带有各平台的 `wekeapi-msg-sync-<os>-<arch>[.exe]`（数据同步服务），主 CLI 会自动发现并启动，无需手工指定路径。

### 数据同步服务（wekeapi-msg-sync）

联系人/消息/群聊等业务数据由 `wekeapi-msg-sync` 服务统一接收回调、同步并存储，主 CLI 不再维护本地业务数据库。

- **无需手工启动**：`login check` 登录成功（status=2）后，CLI 会自动把 token/appId/baseURL/wxid 写入服务目录的 `wekeapi-msg-sync.json`，并启动服务；服务已启动时先关闭再启动（重新全量同步）。
- **服务目录查找顺序**：`data` 命令的 `--msg-sync-dir` > 环境变量 `WEKEAPI_MSG_SYNC_HOME` > `~/.wekeapi/config.json` 的 `msgSyncDir`（`init --msg-sync-dir` 可设置）> 从当前目录/可执行文件目录向上查找 `wekeapi-msg-sync` 目录。
- **查询用 `data` 命令**（见下文），服务未运行时先 `$WEKEAPI data start`。
- **数据库与端口可配置**：默认 SQLite（服务目录下 `wekeapi_msg_sync.db`）、端口 `3006`；用 `$WEKEAPI data config --db sqlite --db-path <文件名> --port <端口>` 或 `$WEKEAPI data config --db mysql --db-dsn '<连接串>' --port <端口>` 设置，改后 `$WEKEAPI data sync` 重启生效；不带参数只查看当前配置（不写文件）。
- **排查**：`$WEKEAPI data status` 查看同步状态与数据量；服务日志在服务目录的 `wekeapi-msg-sync.log`。

## 初始化配置（推荐）

`$WEKEAPI init --token <AccessToken> [--app-id <appId>]`：将凭证保存到 `~/.wekeapi/config.json`，之后所有命令自动读取，可免传 `--token`/`--app-id`（显式 flag 优先于配置文件）。AccessToken 请到官网申请（官网地址运行 `$WEKEAPI env` 查 `siteURL`）；无配置时引导用户申请 AccessToken 并执行本命令保存，不要让用户逐次重复提供 token。API 地址由编译环境内置，init 不收集、用户无需提供 baseUrl。

**两种运行方式**：
- 交互向导（用户在终端直接运行 `wekeapi init`）：提示输入 AccessToken（已有配置时回车保留）；提示走 stderr。
- 自动化（skill/agent，stdin 非交互终端）：CLI 不会提示。**收集顺序固定：向用户索取 AccessToken（官网为 `env` 输出的 `siteURL`），执行 `init --token <TOKEN>` 保存**。

- **环境信息（编译期内置）**：二进制区分线上（prod，默认编译）/测试（test，`go build -tags test`）两个环境。运行 `$WEKEAPI env`（无需凭证）查看：`env` 环境名、`baseURL` 默认 API 地址、`siteURL` 官网、`proxyDownloads` 各平台本地代理工具下载地址（Linux 含一键安装命令）。**skill 中任何地址都从这里现查，不要硬编码**。
- **本地数据目录**：主 CLI 只保存配置文件 `~/.wekeapi/config.json`（`token`/`appId`/`baseURL`/`wxid`/可选的 `msgSyncDir`），**不再维护本地业务数据库**。`wxid` 在登录成功/`login status` 返回个人信息时自动记录；联系人、消息、群聊与标签由 `wekeapi-msg-sync` 服务存储（默认 SQLite，可配置 MySQL），服务配置位于服务目录的 `wekeapi-msg-sync.json`，由登录成功自动写入。旧版 `~/.wekeapi.json` 仍可只读兼容，首次保存后迁移到 `~/.wekeapi/config.json`。
- `--token` 必填（AccessToken 请到官网申请，`$WEKEAPI env` 查 `siteURL`）；`--app-id` 首次登录可为空，省略时保留配置中已有值，显式清空用 `--app-id=`（等号空值语法；**不要写 `--app-id ""`**），新设备登录也可用 `login qr --new-device`。
- `--base-url` 一般无需传：API 地址由编译环境内置（`$WEKEAPI env` 查 `baseURL`），仅临时指向其他地址时才显式传入。
- appId 仅在登录成功（`login check` 返回 status=2）后自动写入配置（连同 wxid），`login qr` 阶段不保存；后续命令可用 `login status` 先检查在线状态再操作。
- 输出 JSON 中 token 已脱敏；配置文件权限 0600。

## 全局参数（所有命令可用）

| 参数 | 说明 | 默认值 |
|---|---|---|
| `--token` | 访问凭证 AccessToken（**必填**，映射 `X-Access-Token` 请求头；可 `init` 保存后免传） | — |
| `--app-id` | 设备 appId（除 `login qr` 外必填，自动写入 body 的 `appId`；可 `init` 保存后免传） | — |
| `--base-url` | API 基础地址（一般无需传，编译环境内置） | 编译环境决定（`$WEKEAPI env` 查询） |
| `--timeout` | HTTP 超时，大文件上传调大如 `--timeout 10m` | `60s` |
| `--pretty` | 缩进输出 JSON | 关闭 |
| `--data` | 透传完整 JSON body（字符串或 `@文件路径`），与具名 flag 合并，**具名 flag 优先** | — |

- 凭证由用户提供（AccessToken 请到官网申请，`$WEKEAPI env` 查 `siteURL`）；若用户已通过 `init` 保存配置则免传 `--token`/`--app-id`，显式 flag 优先于配置文件；无配置时先引导用户申请并执行 `init` 保存，不要让用户逐次重复提供 token。
- 不确定参数时先运行 `$WEKEAPI <模块> <命令> --help`，help 内含必填项、枚举值与注意事项。

## 输出与退出码

- stdout：仅输出 API 返回的原始 JSON `{"code":0,"message":"OK","data":{...}}`；stderr：错误信息与提示。
- 退出码：`0` 成功 / `1` API 业务错误（stdout 仍输出 JSON，含 `code`/`message`）/ `2` 本地错误（缺参数、网络失败、文件读写失败等）。
- 用退出码判断结果，不要解析 stderr；退出码 1 时读 stdout JSON 的 `code`/`message` 决定重试或终止。

## 参数便利规则

- 列表参数逗号分隔：`--wxids a,b`。
- **显式空值一律用 `--flag=`（等号后为空）**，如 `--app-id=`、`--base-url=`。**禁止写 `--flag ""`**：Windows PowerShell 5.1 调用原生 exe 时会丢弃空字符串参数，`--flag ""` 实际只剩裸 `--flag`，会把它后面那个 flag 当值吞掉（曾导致 appId 变成 `--out`，服务端报 `-50 服务异常`；CLI 现已对以 `-` 开头的 appId 值直接报本地错误提示）。需要 JSON 类参数（`--data`）时，Windows 下改用 `--data @文件路径`——PowerShell 还会剥离参数中的内嵌双引号，直接写 JSON 会解析失败。bash / zsh / PowerShell 7.3+ 不受影响。
- **新设备登录**：`login qr --new-device` 忽略配置里的 appId，按全新设备取二维码（等价于 `--app-id=`），不依赖 shell 传空字符串。
- 媒体文件：`--file <本地路径>`（自动 base64）或 `--url <http(s)地址>` 二选一。
- XML 参数：`--xml '<msg>...</msg>'`（自动 base64）或 `--xml-file <路径>` 二选一。
- 下载类与二维码类命令支持 `--out <路径>` 直接保存文件。
- 复杂字段（数组/对象）可 `--data '{"key":[...]}'` 或 `--data @resp.json` 透传。

## 数据查询优先（联系人解析）

- 登录成功后数据由 `wekeapi-msg-sync` 服务统一存储；用户给出的是微信号/昵称/备注等关键字而非 wxid 时，**先用 `data contacts` 查询解析，查不到再调远端接口**：

  ```bash
  $WEKEAPI data contacts --keyword <昵称/备注/微信号/wxid> [--type friend] [--limit 20]
  ```

- `data` 命令输出与其他命令一致：`{"code":0,"message":"OK","data":{...}}`；`data.contacts[].id` 即 wxid。
- 其他数据查询：`data messages`（消息）、`data rooms`（群聊）、`data status`（同步状态与数据量）。
- 数据库与服务端口：`data config`（不带参数查看当前值；`--db sqlite --db-path <文件名>` 或 `--db mysql --db-dsn <连接串>` 加 `--port` 修改），改后 `data sync` 重启生效。
- 服务未运行时先 `$WEKEAPI data start`；需要触发全量同步时 `$WEKEAPI data sync`（重启服务，启动后自动全量同步）。
- 数据可能因平台同步延迟而缺失（如刚加的好友），可先 `data sync` 刷新再查；**不要把空结果直接当「联系人不存在」**。

## 模块路由

| 任务 | 使用 skill |
|---|---|
| 初始化配置（保存 token/appId 到 `~/.wekeapi/config.json`） | wekeapi-core（`init` 命令） |
| 查看内置环境信息（环境名、内置 API 地址、官网、代理工具下载地址，无需凭证） | wekeapi-core（`env` 命令） |
| 扫码登录 / 登录状态轮询 / 在线检查 / 快捷登录 / 退出登录 / 登录成功自动启动数据同步服务并配置数据库与端口 | wekeapi-login |
| 查询联系人/消息/群聊/同步状态（`data contacts/messages/rooms/status`）、设置数据库与端口（`data config`）、启动/停止/重启数据同步服务 | wekeapi-core（`data` 命令） |
| 通讯录 / 好友增删查 / 搜索加好友 / 备注 / 企微好友 / 手机通讯录 | wekeapi-contact |
| 群聊（建群、拉人踢人、群主管理员、公告、群名、二维码、拍一拍、解散退出） | wekeapi-chatroom |
| 标签（列表、增删、给好友打标签） | wekeapi-label |
| 发消息（文本/媒体/链接/小程序/名片）、转发、撤回、下载收到的媒体 | wekeapi-message |
| 设备代理（省市代码查询、查看、设置、城市可用性） | wekeapi-proxy |

群 ID 形如 `12345@chatroom`；发送目标 `--to-wxid` 好友、群通用。
