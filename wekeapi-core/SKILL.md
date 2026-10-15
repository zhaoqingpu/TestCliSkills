---
name: wekeapi-core
description: wekeapi CLI 共享基础：二进制位置、全局参数（--token/--app-id）、退出码与调用约定，并路由到 login/contact/chatroom/label/message/proxy 各模块 skill。任何 wekeapi 微信任务开始前、或不确定该用哪个模块 skill 时使用。
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

## 初始化配置（推荐）

`$WEKEAPI init --token <AccessToken> [--app-id <appId>]`：将凭证保存到 `~/.wekeapi/config.json`，之后所有命令自动读取，可免传 `--token`/`--app-id`（显式 flag 优先于配置文件）。AccessToken 请到官网申请（官网地址运行 `$WEKEAPI env` 查 `siteURL`）；无配置时引导用户申请 AccessToken 并执行本命令保存，不要让用户逐次重复提供 token。API 地址由编译环境内置，init 不收集、用户无需提供 baseUrl。

**两种运行方式**：
- 交互向导（用户在终端直接运行 `wekeapi init`）：提示输入 AccessToken（已有配置时回车保留）；提示走 stderr。
- 自动化（skill/agent，stdin 非交互终端）：CLI 不会提示。**收集顺序固定：向用户索取 AccessToken（官网为 `env` 输出的 `siteURL`），执行 `init --token <TOKEN>` 保存**。

- **环境信息（编译期内置）**：二进制区分线上（prod，默认编译）/测试（test，`go build -tags test`）两个环境。运行 `$WEKEAPI env`（无需凭证）查看：`env` 环境名、`baseURL` 默认 API 地址、`siteURL` 官网、`proxyDownloads` 各平台本地代理工具下载地址（Linux 含一键安装命令）。**skill 中任何地址都从这里现查，不要硬编码**。
- **本地数据目录**：配置文件与同步数据库统一存放在 `~/.wekeapi/` 目录下。`config.json` 配置中的 `wxid` 字段是账号标识（登录成功/`login status` 返回个人信息时自动记录），**完整账号信息（昵称/alias/手机号/uin/头像）不进配置**，存到账号分库的 `accounts` 表。**数据库每账号一库**，文件名含 wxid：`~/.wekeapi/data-<wxid>.db`（含 `contacts` 联系人表、`messages` 消息表——发送/接收共用，`direction` 1-发送/2-接收、`accounts` 本账号信息表）；`login status` / 登录成功会自动把个人信息 upsert 到 `accounts`，查完整信息用 `sqlite3 ~/.wekeapi/data-<wxid>.db "SELECT * FROM accounts"`。旧版单文件 `~/.wekeapi.json` 与旧版数据库（`~/.wekeapi/data.db`、`~/.wekeapi.db`）存在时 CLI 自动读取/复制迁移到新位置（旧文件保留不删除），无需手工处理。
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

## 本地数据优先（联系人解析）

- 登录成功后 `sync all`（wekeapi-login skill）会把标签（最先）与通讯录同步到本地 SQLite（每账号一库，默认 `~/.wekeapi/data-<wxid>.db`）。
- 用户给出的是微信号/昵称/备注/手机号等关键字而非 wxid 时，**先查本地库解析（注意：联系人 wxid 在 `contact_id` 列，`wx_id` 是数据归属键、存 appId），用多字段一次匹配的 SQL 尽量一条命中，查不到再调远端接口**。
- **硬性门槛：对本地库写 SQL 前必须先加载 wekeapi-contact skill 的「本地联系人查询」一节，用其 SQL 模板与平台相关编码规则（Windows 强制两步法转码点；macOS/Linux 直接中文），禁止凭表结构自行写 SQL。** 仅知道表结构不够——编码规则在模板里。
- **编码红线（仅 Windows 生效；macOS/Linux 的 shell 与 sqlite3 均为 UTF-8，中文字面量可直接写）：对所有 `~/.wekeapi/data-*.db` 的 SQL，禁止出现中文字面量，关键词一律转成 `char(码点1,码点2,...)`。** Windows 下 PowerShell 按 GBK 把中文传给 sqlite3.exe，库内是 UTF-8，中文字面量会让 LIKE **静默返回空、不报错**——本地查不到 ≠ 联系人不存在，先怀疑编码再下结论。

## 模块路由

| 任务 | 使用 skill |
|---|---|
| 初始化配置（保存 token/appId 到 `~/.wekeapi/config.json`） | wekeapi-core（`init` 命令） |
| 查看内置环境信息（环境名、内置 API 地址、官网、代理工具下载地址，无需凭证） | wekeapi-core（`env` 命令） |
| 扫码登录 / 登录状态轮询 / 在线检查 / 快捷登录 / 退出登录 / 登录成功后数据同步（`sync all`） | wekeapi-login |
| 通讯录 / 好友增删查 / 搜索加好友 / 备注 / 企微好友 / 手机通讯录 | wekeapi-contact |
| 群聊（建群、拉人踢人、群主管理员、公告、群名、二维码、拍一拍、解散退出） | wekeapi-chatroom |
| 标签（列表、增删、给好友打标签） | wekeapi-label |
| 发消息（文本/媒体/链接/小程序/名片）、转发、撤回、下载收到的媒体 | wekeapi-message |
| 设备代理（省市代码查询、查看、设置、城市可用性） | wekeapi-proxy |

群 ID 形如 `12345@chatroom`；发送目标 `--to-wxid` 好友、群通用。
