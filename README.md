# olink-plugins

Olink 进程型插件示例与生态入口（伞仓）。

各插件为**独立仓库**，本仓通过 [git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) 钉住具体 commit。

**License:** [MIT OR Apache-2.0](./LICENSE-MIT)

## Clone

```bash
git clone --recurse-submodules https://github.com/huan-xi/olink-plugins.git
# 已 clone 则：
git submodule update --init --recursive
```

只关心某一个插件时，可直接 clone 对应子仓。

## 插件列表

| 本地目录 | domain | 独立仓库 |
|---|---|---|
| `xgimi/` | `xgimi` | https://github.com/huan-xi/olink-plugin-xgimi |
| `dreame/` | `dreame` | https://github.com/huan-xi/olink-plugin-dreame |
| `speaker-cast/` | `speaker_cast` | https://github.com/huan-xi/olink-plugin-speaker-cast |

## 开发文档

见 [docs/plugin-development.md](./docs/plugin-development.md)。

## Host 说明

Olink Host（`olink-server` / 插件网关）**未**随本仓开源。联调需要自有 Olink 环境。插件与 Host 通过本机 Unix Domain Socket 上的 HTTP 通信。

## Disclaimer

第三方云 / LAN 协议实现可能受厂商服务条款约束，使用请自行评估合规风险。
