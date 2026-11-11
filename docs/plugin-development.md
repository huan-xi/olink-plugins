# Olink 插件开发指南

面向插件作者：说明进程型插件如何接入 Host、如何通信、需要实现哪些能力，以及本机联调与打包流程。

## 上位规格说明

完整 Host 侧规格（包格式、Supervisor、DeviceBridge、前端运行时、存储、安全打包）随 Olink 宿主产品维护，**不在本公开仓**。

本指南仅描述插件作者可依赖的公开契约：子进程模型、UDS HTTP、`X-Plugin-Token`、`plugin.json`、打包与联调流程。实现细节以你使用的 Host 版本为准。

参考实现：本仓 submodule 下的 `xgimi/`、`dreame/`、`speaker-cast/`。

---

## 1. 架构与运行原理

### 1.1 定位

Olink 插件是**可安装的子进程扩展**，不是 `.so` 进程内加载：

- 每个插件一个稳定 **`domain`**（如 `xgimi`、`dreame`），同时作为包 ID、磁盘目录、前端路由前缀、设备 `did` 前缀。
- 后端由 Host（`olink-plugin-host`）安装、拉起、监督；插件与 Host **只通过 Unix Domain Socket 上的 HTTP** 通信（无端口分配）。
- 设备进入现有链路：`DeviceBridge → DeviceManager → MappingEngine → HAP/Matter`。插件**不自建** HAP/Matter，也不直接碰 EventBus。
- 可选 React 前端：包内 ESM，由 `olink-front` 动态 `import()`，复用宿主 `/plugin-runtime/*`（同一套 React / UI）。

```text
浏览器 / Console
    │  /api/plugins/* 、/api/plugins/proxy/{domain}/*
    │  /plugin-assets/{domain}/{version}/*
    ▼
olink-server
    └── PluginHost（Integration）
            ├── PackageStore / Install
            ├── ProcessSupervisor          ──spawn──▶  插件子进程
            ├── Host UDS HTTP（host.sock） ◀──IPC──▶   插件 UDS HTTP（{domain}.sock）
            ├── DeviceBridge
            └── Gateway（管理 API + assets + proxy）
```

### 1.2 为何子进程 + UDS HTTP

| 选择 | 作用 |
|---|---|
| 独立进程 | 崩溃隔离、独立日志、可重启；与主仓语言/依赖解耦 |
| UDS | 本机本用户、无端口冲突；路径权限 0600 / 目录 0700 |
| HTTP/JSON | 契约清晰、易调试；双向都是「HTTP over UnixStream」 |
| Token | 每次启动注入 `OLINK_PLUGIN_TOKEN`；双向带 `X-Plugin-Token` |

### 1.3 启动时序

```text
用户启用插件（ui_plugins.enabled）且允许启动
  → Host 准备目录与 socket 路径，生成 token
  → spawn 插件二进制（独立进程组，stdin 接管，stdout/err → plugin.log）
  → Host 轮询插件 GET /health，直至就绪或超时
  → 插件 bind 自己的 sock 后，向 Host POST /olink/v1/hello（feature 协商）
  → 成功后挂 DeviceBridge；插件可 register / report
  → 若协商到 schedules，Host 按 manifest 注册定时任务，到点调插件 /olink/v1/trigger
```

停止顺序：注销该 domain 设备 → `POST /olink/v1/shutdown` → SIGTERM → SIGKILL（进程组）→ 清理 sock。

崩溃重启：指数退避；连续失败会 `auto_disable`，需用户手动再启。

### 1.4 双 Socket

| Socket | 谁监听 | 谁连接 | 典型路径 |
|---|---|---|---|
| 插件 sock | 插件 | Host（health / read / write / trigger / shutdown）；浏览器经 Host proxy | `{data_path}/run/{domain}.sock` |
| Host sock | Host | 插件（hello / register / report / available …） | `{data_path}/run/host.sock` |

路径过长（macOS/Linux 限制）时，Host 会退到 `/tmp/olink-…/run`。启动前会 unlink 残留 sock。

### 1.5 环境变量（Host 注入）

| 变量 | 说明 |
|---|---|
| `OLINK_PLUGIN_DOMAIN` | domain |
| `OLINK_PLUGIN_VERSION` | 当前包版本 |
| `OLINK_PLUGIN_SOCK` | 插件应 bind 的 UDS 路径 |
| `OLINK_PLUGIN_HOST_SOCK` | Host IPC UDS 路径 |
| `OLINK_PLUGIN_TOKEN` | 本次启动 token（每次更换） |
| `OLINK_PLUGIN_DATA_DIR` | 唯一可写业务目录 `plugins/{domain}/data` |
| `OLINK_PLUGIN_CACHE_DIR` | 可清理缓存 |
| `OLINK_PLUGIN_LOG_DIR` | 日志目录（Host 也写 `plugin.log`） |
| `OLINK_PLUGIN_PACKAGE_DIR` | 当前包目录（只读） |
| `OLINK_PLUGIN_HOST_VERSION` | Host 版本（若注入） |

`cwd` 一般为 `DATA_DIR`。`runtime.env` 可加业务变量；`OLINK_*` / `PATH` / `LD_*` 等保留名由 Host 覆盖，插件不可劫持。

### 1.6 Domain 隔离（硬约束）

| 维度 | 规则 |
|---|---|
| domain | `^[a-z][a-z0-9_]{1,31}$`；不可与内置保留名冲突（mijia/hap/matter/…） |
| 设备 did | `{domain}.{device_key}`；`device_key`：`^[A-Za-z0-9_-]{1,64}$` |
| 前端路由 | `/plugins/{domain}/*` |
| HTTP 业务 | `/api/plugins/proxy/{domain}/*` → 插件 UDS（需登录）；**不能**代理 `/health` 与 `/olink/v1/*` |
| 磁盘 | `{data_path}/plugins/{domain}/{data,cache,logs,package}` |

---

## 2. 开发联调与打包

### 2.1 推荐：`dev_roots`（本机跳过安装包）

Host 与插件源码同机时，可直接指向源码目录，无需 `pack.sh` / 安装。

在 Host `data/application.toml`：

```toml
[plugins]
dev_mode = true
allow_unsigned_local = true
dev_roots = { xgimi = "/Users/huanxi/project/olink-project/olink-plugins/xgimi", dreame = "/Users/huanxi/project/olink-project/olink-plugins/dreame" }
# 可选：前端 Vite 热更新
# dev_assets = { xgimi = "http://127.0.0.1:5174" }
```

```bash
cd olink-plugins/xgimi && cargo build
# 管理页启用该 domain 后：改代码 → cargo build → 重启插件
```

注意：

- 同 domain 若已正式安装，dev_roots 生效时安装路径会报 `PLUGIN_DEV_ROOT_ACTIVE`（不要混用）。
- 细节见各插件 `DEV.md` 与规格 `docs/superpowers/specs/2026-11-09-olink-plugin-dev-roots-design.md`。

### 2.2 UI 联调要点

- 生产：宿主加载 `/plugin-assets/{domain}/{version}/entry.js`。
- 开发：`dev_assets` 可指向 Vite；或改完 `ui/entry.js` 后硬刷新。
- `entry.js` **必须**用宿主绝对路径导入共享依赖，例如：

```js
import { createElement, useState } from '/plugin-runtime/react.js'
import { usePluginContext } from '/plugin-runtime/plugin-sdk.js'
```

不要写裸 `from 'react'`。`/plugin-runtime/*` 由 **olink-front** 提供。

业务请求走 `pluginFetch` → `/api/plugins/proxy/{domain}/...`（需登录），不要直连插件 UDS。

### 2.3 打包安装

```bash
cd olink-plugins/<plugin>
./scripts/pack.sh                 # → dist/{domain}-{version}.olinkpkg
```

包结构（简要）：

```text
*.olinkpkg (zip)
├── plugin.json
├── bin/{os}-{arch}/plugin        # 如 linux-x86_64、macos-aarch64
├── ui/entry.js                   # 可选
├── mappings/                     # 可选（input 等）
└── resources/                    # 可选
```

管理台：预检 `POST /api/plugins/inspect` → 确认权限 → `POST /api/plugins/install` → 设置页启用 → `start`。

安装默认**不自动启用**；`autostart` 仅在「已启用」且 Host 重启时自动拉起。

---

## 3. IPC 接口（UDS HTTP）

协议主版本：`api.protocol = 1`（破坏性变更另开 `/olink/v2`）。  
鉴权：请求头 `X-Plugin-Token: {OLINK_PLUGIN_TOKEN}`（两端都要）。  
JSON；失败时 Host 侧常见包络：`{ "ok": false, "error": "...", "message": "..." }`。

### 3.1 插件必须监听（Host → 插件）

| 方法 | 路径 | 作用 | 何时需要 |
|---|---|---|---|
| GET | `/health` | 就绪探活 | **始终** |
| POST | `/olink/v1/shutdown` | 优雅退出 | **始终** |
| POST | `/olink/v1/read` | 读属性 | 设备型 + `device.attrs` |
| POST | `/olink/v1/write` | 写属性 | 设备型 + `device.attrs` |
| POST | `/olink/v1/trigger` | 定时/手动任务 | 声明了 schedules 且 Hello 含 `schedules` |

另可实现任意业务 REST（如 `/state`、`/accounts`），经 Host proxy 给前端；**不要**占用 `/health` 与 `/olink/v1/*` 给业务。

#### `POST /olink/v1/read`

```json
{ "device_key": "sw1", "attr_ids": ["power-state"], "force": false }
```

响应示例：`{ "ok": true, "values": { "power-state": true } }`。

#### `POST /olink/v1/write`

```json
{ "device_key": "sw1", "attr_id": "power-state", "value": true }
```

响应示例：`{ "ok": true, "applied_value": true }`。  
按属性 `write.policy`：`optimistic` 时 Host 会先用 `applied_value` 更新缓存；权威值仍以后续 `report` 为准。

#### `POST /olink/v1/trigger`

```json
{ "job_id": "refresh", "manual": false }
```

响应：`{ "ok": true }`。计时器只在 Host/core Scheduler；插件不要自起等价 timer。

#### `POST /olink/v1/shutdown`

响应后尽快退出；Host 随后可能 SIGTERM/SIGKILL。

### 3.2 插件调用 Host（插件 → Host）

| 方法 | 路径 | feature | 作用 |
|---|---|---|---|
| POST | `/olink/v1/hello` | （握手，无门闸） | 协议与 feature 协商；启动必调 |
| POST | `/olink/v1/register` | `device.register` | 注册设备 |
| POST | `/olink/v1/unregister` | `device.register` | 注销设备 |
| POST | `/olink/v1/report` | `device.attrs` | 属性上报 |
| POST | `/olink/v1/available` | `device.attrs` | 在线/离线 |
| POST | `/olink/v1/log` | `log` | 结构化日志（若启用） |

未 Hello 成功前，除插件侧 `/health` 外，Host 对 `/olink/v1/*` 返回 403。路径所需 feature 未协商生效 → 403 `FEATURE_NOT_NEGOTIATED`。

#### Hello

请求：

```json
{
  "domain": "xgimi",
  "plugin_version": "0.1.3",
  "protocol": 1,
  "features": ["device.register", "device.attrs", "schedules"],
  "features_required": ["device.register", "device.attrs"]
}
```

响应：

```json
{
  "host_version": "…",
  "domain": "xgimi",
  "protocol": 1,
  "features": ["device.register", "device.attrs", "schedules"],
  "features_rejected": []
}
```

规则摘要：

- `features`：插件声明自己实现的能力；与 Host 求交得到生效集。
- `features_required`：任一 Host 不满足 → Hello 失败，进程应退出。
- **省略** `features` / `features_required` 时，默认视为 `["device.register","device.attrs"]`。
- **服务型插件**（无设备，如 speaker-cast）必须**显式**传 `features`（例如 `["log"]`）与 `features_required`（可 `[]`），切勿省略，否则会落到设备默认并要求 DeviceBridge。

#### register

```json
{
  "device_key": "sw1",
  "schema_id": "switch",
  "name": "Demo Switch",
  "manufacturer": "Olink",
  "model": "demo.switch",
  "services": [
    {
      "service_type": "switch",
      "attributes": [
        {
          "attr_type": "power-state",
          "access": "readwrite",
          "write": { "policy": "optimistic", "verify_capable": false }
        }
      ]
    }
  ]
}
```

- `schema_id` 必须存在于系统 SchemaRegistry（通用 device_type）。
- `services` 可省略：Host 按 schema 自动补全 required 属性。
- `access`：`readonly` / `writeonly` / `readwrite`（小写）。
- `write.policy`：`report` | `optimistic` | `verify`；缺省用 manifest `runtime.default_write_policy`。

#### report

```json
{
  "reports": [
    { "device_key": "sw1", "attr_id": "power-state", "value": true, "ts": 1730000000000 }
  ]
}
```

`ts` 为毫秒时间戳；上报是状态权威来源。

#### available

```json
{ "device_key": "sw1", "available": false }
```

进程退出时 Host 会将该 domain 设备置为不可用（保留注册）。

### 3.3 Feature 与权限对照

| id | 含义 | 插件侧 | Host 侧 |
|---|---|---|---|
| `device.register` | 设备登记 | 调 register/unregister | 建 DeviceBridge 设备 |
| `device.attrs` | 属性读写报 | 实现 read/write；调 report/available | 转发 Mapping/HAP 读写 |
| `schedules` | 定时任务 | 实现 trigger | Scheduler → trigger |
| `log` | 插件日志 IPC | 调 `/olink/v1/log` | 落盘/查询 |
| `lifecycle` | 激活/刷新（规划中） | activate/deactivate/refresh | DeviceLifecycleCmd |

`plugin.json` 的 `permissions` 声明安装时授权；敏感 feature（`device.register` / `device.attrs` / `log`）必须同时出现在 `permissions`，否则安装失败。  
`capabilities` 仅市场/展示标签，**不参与** Hello。

---

## 4. 插件需要提供的能力

按插件类型组合；不是每个插件都要做满。

### 4.1 所有进程插件（最小集）

| 能力 | 作用 |
|---|---|
| 合法 `plugin.json` | 声明 domain、runtime、api.transport=`uds`、protocol、permissions |
| 平台二进制 `bin/{os}-{arch}/plugin` | Host 按当前 OS/ARCH 解析 `runtime.entry` |
| 监听 `OLINK_PLUGIN_SOCK` | 提供 `/health`、`/shutdown` |
| Hello | 与 Host 协商协议与 features；失败则退出 |
| 数据落盘只用 `DATA_DIR` | Host 负责备份/恢复/统计；无 Host 托管 SQL |

### 4.2 设备型插件（接入灯/扫地机等）

| 能力 | 作用 |
|---|---|
| `device.register` + `device.attrs` | 把第三方设备变成 Olink 设备，进入映射与 HomeKit/Matter |
| register 时选对 `schema_id` | 复用系统通用 device_type → 输出映射，无需自写 HAP |
| read / write | 响应 Core/映射发起的读写下发 |
| report / available | 推送状态与在线；决定 HAP 特征是否更新、是否离线 |
| 合理 `write.policy` | `optimistic`（默认）、`report`（等设备推送）、`verify`（写后读回） |
| 可选 `schedules` | 轮询/巡检由 Host 触发，避免插件内多套定时器 |

可选：包内 `mappings/`（自定义 input 等）、`resources/capability/`、proxy 业务 API（账号登录、房间列表等）。

### 4.3 服务型插件（无设备桥）

例如音箱投送：不注册设备，只提供本机服务 + 可选 UI。

| 能力 | 作用 |
|---|---|
| 显式 Hello features | 避免默认设备 features；常用 `["log"]` + required `[]` |
| 业务 HTTP + proxy | Console 配置/状态 |
| 可选 UI ESM | 设置页/侧栏 |

### 4.4 可选前端

| 能力 | 作用 |
|---|---|
| `ui.entry` ESM | 在 `/plugins/{domain}` 下挂路由，与宿主同壳 |
| `routes.menu` | 侧栏入口 |
| 复用 `/plugin-runtime` 与 `@olink-home/ui` peer | 统一视觉与 React 单例 |

### 4.5 `plugin.json` 字段速查

```json
{
  "domain": "xgimi",
  "name": "极米",
  "version": "0.1.3",
  "description": "…",
  "publisher": "olink",
  "runtime": {
    "type": "process",
    "entry": "bin/{os}-{arch}/plugin",
    "default_write_policy": "optimistic"
  },
  "ui": {
    "entry": "ui/entry.js",
    "css": "ui/entry.css",
    "runtime": 1,
    "module": "routes",
    "olink_ui_peer": "0"
  },
  "api": {
    "transport": "uds",
    "protocol": 1,
    "features": ["device.register", "device.attrs", "schedules"],
    "features_required": ["device.register", "device.attrs"]
  },
  "permissions": ["device.register", "device.attrs", "storage.domain"],
  "schedules": [
    {
      "id": "refresh",
      "kind": "interval",
      "every_secs": 15,
      "timeout_secs": 10,
      "overlap": "skip",
      "enabled": true
    }
  ],
  "autostart": true
}
```

已知 `permissions`：`device.register`、`device.attrs`、`log`、`storage.domain`、`net.outbound`。

---

## 5. 如何调试

### 5.1 推荐工作流（本机）

1. Host 开 **debug 构建**（`dev_mode` 在发行版会被忽略）。
2. `application.toml` 配好 `dev_roots`（可选 `dev_bins` / `dev_assets`），见 §2.1。
3. `cargo build` 插件 → 管理页启用 domain → `POST /api/plugins/{domain}/start`（或 Console 启停）。
4. 改代码后：`cargo build` → **重启插件**（不必重启整个 olink）：

```bash
# 需登录态 Cookie / Token；本地可把 Authorization 换成自己的
curl -X POST "http://127.0.0.1:5512/api/plugins/xgimi/restart"
```

5. 看日志确认 Hello / register / 业务错误（§5.2）。

`dev_bins` 可指向固定二进制路径，例如：

```toml
[plugins]
dev_mode = true
allow_unsigned_local = true
dev_roots = { xgimi = "/ABS/PATH/olink-plugins/xgimi" }
dev_bins = { xgimi = "/ABS/PATH/olink-plugins/xgimi/target/debug/olink-plugin-xgimi" }
```

### 5.2 日志与运行状态

| 来源 | 路径 / API | 用途 |
|---|---|---|
| 插件 stdout/err | `{data_path}/plugins/{domain}/logs/plugin.log` | 主日志（Host 轮转，约 10MB×3） |
| 尾部 API | `GET /api/plugins/{domain}/logs?tail=200` | Console / 脚本拉最近行（上限 2000） |
| 运行状态文件 | `{data_path}/plugin-state/{domain}.json` | `running`、`pid`、`accepted_features`、`last_error`、`restart_count`、`source=dev_root` |
| 状态 API | `GET /api/plugins/{domain}/state` | 同上信息的 HTTP 视图 |
| 定时任务 | `GET /api/plugins/{domain}/schedules`；`POST .../schedules/{id}/run` | 看下次/上次结果；手动触发一轮 |

```bash
# 跟日志
tail -f "$DATA_PATH/plugins/xgimi/logs/plugin.log"

# 看是否在跑、Hello 生效 features
cat "$DATA_PATH/plugin-state/xgimi.json" | jq .
```

插件进程默认注入 `OLINK_PLUGIN_LOG_LEVEL=info`。业务日志请打到 stdout/stderr（进 `plugin.log`），不要写到 Host 的 `plugin-state/`。

### 5.3 探活与 proxy

插件 sock 路径见 `plugin-state` 同级约定，一般为 `{data_path}/run/{domain}.sock`（过长时可能在 `/tmp/olink-…/run/`）。

```bash
# 健康检查（无需 token）
curl --unix-socket "$DATA_PATH/run/xgimi.sock" http://localhost/health

# 经 Host 反代调业务 API（需登录；不要用 proxy 打 /health 或 /olink/v1/*）
curl -H "Authorization: Bearer …" \
  "http://127.0.0.1:5512/api/plugins/proxy/xgimi/state"
```

`/olink/v1/*` 需 `X-Plugin-Token`（每次启动更换，在插件进程环境里）。本机可从进程环境取出：

```bash
PID=$(jq -r .pid "$DATA_PATH/plugin-state/xgimi.json")
# macOS
ps eww -p "$PID" | tr ' ' '\n' | grep OLINK_PLUGIN_TOKEN
# 或：lsof -p "$PID" 看 sock；token 用于手工 curl Host sock
curl --unix-socket "$DATA_PATH/run/host.sock" \
  -H "X-Plugin-Token: $TOKEN" \
  -H "content-type: application/json" \
  -d '{"domain":"xgimi","plugin_version":"0.1.0","protocol":1,"features":["device.register","device.attrs"],"features_required":["device.register","device.attrs"]}' \
  http://localhost/olink/v1/hello
```

（正常运行中再次 Hello 一般不需要；仅用于确认 Host IPC / token 绑定。）

### 5.4 设备与映射

1. 确认 Hello `accepted_features` 含 `device.register` / `device.attrs`。
2. 插件 `register` 成功后，在设备列表看 `did = {domain}.{device_key}`。
3. HomeKit/Matter 无特征：先查 `schema_id` 是否系统已有；再查属性上报是否走到 `report`（日志 + 属性历史）。
4. 写不生效：看 Host 是否调到插件 `/olink/v1/write`（插件日志），以及 `write.policy` 是否等 `report`。
5. 手动跑一轮轮询：`POST /api/plugins/{domain}/schedules/refresh/run`（id 以 manifest 为准）。

### 5.5 前端调试

- DevTools → Network：确认 `entry.js` / `entry.css` 来自 `/plugin-assets/...` 或 `dev_assets` 源；`/plugin-runtime/*.js` 必须是 JS 而非 HTML。
- 仍报裸 `react`：Disable cache 后硬刷新；Host 对带版本 assets 曾下 `immutable`。
- 业务请求失败：看是否走 `/api/plugins/proxy/{domain}/...`、插件是否 running、响应是否 502/503。
- `olink-front` 开发态：`pnpm dev`；若 runtime facade 异常，重启 front。

### 5.6 IDE 断点（可选）

1. `cargo build` 出 debug 二进制，`dev_bins` 指向该路径。
2. Host 拉起插件后，用 `plugin-state` 里的 `pid` attach（LLDB / CodeLLDB / rust-lldb）。
3. 或临时在插件里 `std::process::id()` 打日志，确认 attach 目标。

注意：Host 用独立进程组 + 文件日志；attach 不要杀错父进程。调试完记得去掉死循环/长 sleep，以免触发 startup timeout 或 health 失败重启。

### 5.7 常见启动失败顺序

```text
1. 二进制是否存在（dev_roots/dev_bins / package entry）
2. sock 是否 bind 成功（权限、路径长度、残留文件）
3. GET /health 是否在 startup_timeout 内 200
4. Hello 是否成功（protocol / features_required / token / domain 绑定）
5. register 是否 schema 合法
→ 全程对照 plugin.log + plugin-state.last_error
```

---

## 6. 新建插件建议路径

1. 复制 `xgimi/`（设备型）或参考 `speaker-cast/`（服务型）/ `olink-plugin-demo`（最小 IPC）。
2. 改 `plugin.json` 的 `domain` / `name` / `description`；改 `Cargo.toml` 包名与 bin 名。
3. 实现：读 env → bind sock → `/health` → Hello →（可选）register / 业务循环。
4. 本机用 `dev_roots` 联调；稳定后 `pack.sh` 安装验证。
5. 需要 UI 时写 `ui/entry.js`，只从 `/plugin-runtime/*` 导入。

最小启动骨架逻辑（概念）：

```text
读 OLINK_PLUGIN_* 
→ bind OLINK_PLUGIN_SOCK，挂 /health + /olink/v1/*
→ POST host:/olink/v1/hello
→ （设备型）POST register，循环 report / 处理 read|write|trigger
→ 收到 shutdown 后退出
```

---

## 7. 排障速查

| 现象 | 排查 |
|---|---|
| `PLUGIN_STARTUP_TIMEOUT` | 插件未及时 `/health` 或未 Hello；看 `plugins/{domain}/logs/plugin.log` |
| `PLUGIN_FEATURES_UNAVAILABLE` | `features_required` 超出 Host；或服务型误用默认设备 features |
| `FEATURE_NOT_NEGOTIATED` | 调了未协商 feature 对应的路径 |
| `PLUGIN_SCHEMA_UNKNOWN` | `schema_id` 不在系统 schemas |
| proxy 502/503 | 插件未运行或 UDS 断；确认已 start |
| UI 裸 `react` 报错 | entry 未改成 `/plugin-runtime/...`；或浏览器缓存了旧 entry |
| 正式安装与 dev 冲突 | 关掉其一；读 `PLUGIN_DEV_ROOT_ACTIVE` |
| `auto_disabled` | 短时频繁崩溃；看 `disabled_reason` / `restart_count`，修完后手动 start |
| health 通但设备不上 | Hello features、register 日志、`schema_id`、Console 设备列表 did |

日志：`{data_path}/plugins/{domain}/logs/plugin.log`；运行状态：`{data_path}/plugin-state/{domain}.json`（插件不可写）。
