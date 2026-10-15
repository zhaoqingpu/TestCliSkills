---
name: wekeapi-proxy
description: wekeapi CLI proxy 模块：微信设备代理管理（省市列表与城市代码查询、获取当前代理、设置/修改代理、检查城市是否有可用代理）。当用户要求查询或设置微信设备代理、选择平台代理城市、查省市/城市代码、检查代理城市可用性时使用。
---

# proxy 代理

## 前置

- 二进制在项目 `.trae/skills/wekeapi-core/assets/bin/` 下按 OS 选择（Windows: `wekeapi-windows-amd64.exe` / Linux: `wekeapi-linux-amd64` / macOS Intel: `wekeapi-darwin-amd64` / macOS ARM: `wekeapi-darwin-arm64`），下文以 `$WEKEAPI` 代指。
- `provinces`/`cities` 为本地查询命令，**无需 `--token`/`--app-id`**；其余命令需 `--token` 与 `--app-id`（运行 `wekeapi init` 保存配置后可免传，显式 flag 优先）。
- stdout 恒为 JSON；退出码 0 成功 / 1 API 业务错误 / 2 本地错误。
- 其余全局约定见 wekeapi-core skill。

## 命令表

| 命令 | 说明 | 参数 |
|---|---|---|
| `proxy provinces` | 获取平台代理可用的省份列表（内置官方对照表，30 省级区域） | — |
| `proxy cities` | 获取城市代码列表（336 个城市；平台代理 proxyInfo 传城市代码） | `--province` / `--city` 精确过滤，可组合；无匹配退出码 2 |
| `proxy get` | 获取设备当前代理信息 | — |
| `proxy set` | 设置/修改设备代理 | **`--proxy-type`** local/self/platform、**`--proxy-info`** |
| `proxy check-city` | 检查城市是否有可用代理（返回 `data`：true-有 / false-无） | **`--city-code`** |

## 平台代理选择流程（proxyType=platform 时，按序执行）

1. 采集省市（**让用户选择**，提醒文案用「选择微信常用地」，并向用户说明：微信常用地通常指长期居住地或手机号归属地）：
   - 第一步选省：`proxy provinces` 取省份列表（30 省级区域）展示选项让用户选；选项展示不完整时，提示用户直接手动输入省名。
   - 第二步选市：**选完省后必须继续选该省对应的市（省→市两步采集，不可跳过）**：`proxy cities --province <所选省>` 取该省城市列表展示选项让用户选；选项展示不完整时，提示用户直接手动输入市名。
2. 匹配城市代码：省、市名去掉「省/市/自治区」后缀，与 `$WEKEAPI proxy cities --province <所选省>` 结果精确匹配取 `code`（如 河南→开封 4102）；城市不在平台对照表（如 上海）→ 提示平台不支持该城市，请切换代理方式（local/self）或手选其他城市重测。
3. 检测可用性：`$WEKEAPI proxy check-city --token T --app-id A --city-code <code>`，按 `data` 分支处理：
   - `data=true`：该城市有可用代理 → 用该城市代码作为 proxyInfo（`login qr --proxy-info <code>` 或 `proxy set --proxy-info <code>`）继续。
   - `data=false`：**当前城市没有可用代理 → 提示用户切换其他代理方式（local/self），终止 platform 流程**。
4. （在线设备改代理时）设置代理：`$WEKEAPI proxy set --token T --app-id A --proxy-type platform --proxy-info <code>`。

**app-id 说明**：配置中已有 appId 时免传（自动读取）；**首次登录前无 appId** 时，用 `--app-id=` 显式传空 appId 调用 `check-city`（接口实测接受，检测结果与携带真实 appId 一致），不要为了检测先跑 `login qr`。（等号空值语法避免 shell 传空字符串；**不要写 `--app-id ""`**——Windows PowerShell 5.1 会丢弃空参数，导致 `--app-id` 吞掉后面的 flag 并报 `-50`。）

## 注意

- `--proxy-info` 与类型对应：local 传本地代理ID / self 传代理 Url / platform 传城市代码（用 `proxy cities` 查询，不要凭记忆硬编码）。
- 城市名需与对照表完全一致（如 `广东`、`深圳`，不带"省/市"后缀）；上海不在平台对照表内。
- 账号在线时代理立即生效；离线则下次登录/重连后生效。
- 新代理网络不通返回 `-51: 网络连接异常`。
- 登录时（`login qr`）的 `--proxy-type/--proxy-info` 取值规则与此模块相同。

## 示例

```bash
# 让用户选择微信常用地（省 → 市），取省份与城市列表展示选项
$WEKEAPI proxy provinces
$WEKEAPI proxy cities --province 河南

# 首次登录前无 appId：用 --app-id= 显式传空 appId 检测开封(4102)是否有可用代理
$WEKEAPI proxy check-city --token T --app-id= --city-code 4102

# 检测可用后设置平台代理（data=true 时）
$WEKEAPI proxy set --token T --app-id A --proxy-type platform --proxy-info 4102
```
