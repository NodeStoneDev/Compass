# NodeStone Compass Protocol (NCP) v0.7.0 — 协议详细定义

## 1. 为什么把 `nsci` 和 `torrents` 分开？

在 `latest.json`（NSCL）中，`nsci.raw_kv` 保存 `NSFID -> torrenthash`，而 `assets.torrents` 保存 `torrenthash -> delivery`，不把它们合并成一张表主要有四个原因：

1. **避免冗余**：`delivery` 包含下载地址或内嵌载荷，体积较大；而 `nsci` 中经常有多个 `NSFID`（例如同一个大版本下多个小版本的 `<asset_index>.json`，或者同一个文件的不同标识）指向同一个种子。分开存放可以大幅减小 `latest.json` 的体积。
2. **全局唯一且可自校验**：以**种子 Hash**（BT v2 InfoHash，或 `raw_file` 的 SHA-256）作为关联 Key，天然具备全局唯一性和校验能力。
3. **方便本地缓存管理**：本地种子直接按 `.nodestone/torrents/<torrent_hash>.torrent` 存放。客户端从 `nsci` 查到 Hash 后就能直接定位并校验本地种子文件，不需要再去读 `delivery`。
4. **支持多镜像源差异化投递**：NodeStone Compass 允许多个镜像源提供服务。同一个种子在不同镜像源里的 `Torrent Hash` 完全一致，但各自的 `delivery`（比如种子下载 URL、或者是否直接内嵌）可以根据源自身情况灵活配置。

## 2. NSCL（`latest.json`）字段说明

`latest.json` 的顶层及嵌套字段定义如下：

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :---: | :--- |
| `filetype` | `string` | 是 | 固定为 `"nscl/v0.7"`。 |
| `namespace` | `string` | 是 | 命名空间，须与客户端预配置的 `<provider.namespace>` 一致（如 `"minecraft"`）。 |
| `revision` | `integer` | 是 | 单调递增的修订号。客户端必须拒绝比本地已缓存版本更小的 `revision`。 |
| `generatedBy` | `string` | 是 | 生成该清单的节点标识，格式为 `<worker>`（如 `"compass-worker-hk-01"`）。 |
| `generateDate` | `string` | 是 | 生成时间，采用 ISO 8601 UTC 格式（如 `"2026-11-21T12:40:34Z"`）。 |
| `nsci.raw_kv` | `object` | 是 | `NSFID -> Hash` 映射字典。v0.7.0 仅使用 `url:<Base64>` 作为 Key；Hash 为 64 位小写十六进制 BT v2 InfoHash（当对应项为 `raw_file` 模式时，为该目标文件的 SHA-256）。 |
| `assets.torrents` | `object` | 是 | `Torrent Hash -> { delivery }` 字典，描述每个种子的获取方式。 |
| `delivery.integrated` | `object` | 否 | 内嵌载荷：`{ "type": "torrent" \| "raw_file", "payload": "<Base64>" }`，`type` 缺省为 `"torrent"`。<br>**慎用警告**：仅限用于 `version_manifest_v2.json` 等高频变化、体积仅数 KB 的顶层单文件入口（内嵌 `.torrent` 或直出单文件），严禁内嵌常规多文件种子，以免导致 `latest.json` 体积膨胀。 |
| `delivery.url` | `list<string>` | 否 | `.torrent` 种子文件的 HTTP(S)/FTP 下载地址列表。 |
| `delivery.magnetic` | `string` | 否 | BT v2 磁力链接（`magnet:?xt=urn:btmh:1220<64-hex>...`）。声明支持该字段的客户端**必须实现 BEP-52 `hash request` 扩展**。 |

## 3. 种子打包与底层协议规范

### 1. BitTorrent v2（BEP-52）与 `hash request` 强制要求

* NCP 种子统一采用 **BitTorrent v2（BEP-52）** 格式，种子 Hash（InfoHash v2）为 Bencode 编码后 `info` 字典的 SHA-256 值（64 位小写十六进制字符串）。
* **磁力链接（`delivery.magnetic`）强制要求**：在 BitTorrent 协议中，BEP-9（`ut_metadata`）扩展只能从 Peer 拉取 `info` 字典，不包含顶层的 `piece layers`。而 BT v2 中任何大于 16 KiB 的文件都必须依赖 `piece layers` 来校验分块。因此，**任何声明支持 `delivery.magnetic` 的客户端，其底层 BT 引擎必须完整实现 BEP-52 的 `hash request` / `hashes` 扩展消息**，以便从 Peer 动态拉取 Merkle 树分层哈希并在校验通过后持久化保存。

### 2. 种子目录结构约定（Packer 规范）

1. **`info["file tree"]` 第一级固定为主机名**：
   无论是单文件种子还是跨多个官方域名的多文件种子，文件在 `info["file tree"]` 中的相对路径 `rel_path`（由各级字典 Key 用 `/` 拼接）**第一级目录固定为官方源的完整主机名（`<host>[:<port>]`），后续层级为 URL 路径**：
   * 单文件示例：`https://piston-meta.mojang.com/mc/game/version_manifest_v2.json` 在 `file tree` 中的路径固定为：
     `piston-meta.mojang.com/mc/game/version_manifest_v2.json`
   * 多文件跨域名示例：`https://libraries.minecraft.net/com/mojang/logging/1.1.1/logging-1.1.1.jar` 在 `file tree` 中的路径固定为：
     `libraries.minecraft.net/com/mojang/logging/1.1.1/logging-1.1.1.jar`
2. **协议头字段 `ns-schemes`**：
   种子顶层使用 NCP 专用字段 `ns-schemes` 声明协议头列表（缺省为 `["https://"]`）。由于普通 BT 客户端如果按原生 BEP-19 解析顶层 `url-list`，会把 `info["name"]` 拼到域名位置导致无效的 HTTP 请求，因此**打包器建议不在顶层设置标准 `url-list`**（若设置，SDK 在将种子挂载到底层 BT 引擎前应先清空 `url-list` 并禁用引擎自带的 BEP-19 WebSeed）。

### 3. 资源分包约定（以 `minecraft` 命名空间为例）

服务端打包 Minecraft 资源时，推荐将版本核心文件与通用资源分拆打包：

* **顶层清单种子**：包含 `version_manifest_v2.json`（通常在 `delivery.integrated` 中内嵌）。
* **版本核心种子（`minecraft-core-<ver>`）**：包含该版本的 `<version>.json`、`client.jar` 及该版本依赖的全部 `libraries`。
* **大版本 Assets 种子（`minecraft-assets-<major>`）**：如 `assets-1.20`、`assets-1.21`，包含对应的 `<asset_index>.json` 及该大版本下的全部 `objects`。
* `latest.json` 的 `nsci.raw_kv` 只需收录 `version_manifest_v2.json`、各版本 `<version>.json` 和各 `<asset_index>.json` 的 `NSFID -> Torrent Hash` 映射。

## 4. `latest.json` 完整示例

```json
{
  "filetype": "nscl/v0.7",
  "namespace": "minecraft",
  "revision": 1086,
  "generatedBy": "compass-worker-hk-01",
  "generateDate": "2026-11-21T12:40:34Z",
  "nsci": {
    "raw_kv": {
      "url:aHR0cHM6Ly9waXN0b24tbWV0YS5tb2phbmcuY29tL21jL2dhbWUvdmVyc2lvbl9tYW5pZmVzdF92Mi5qc29u": "8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4",
      "url:aHR0cHM6Ly9waXN0b24tbWV0YS5tb2phbmcuY29tL3YxL3BhY2thZ2VzLzE3YzU0ZWIzM2I1ZDRhOGY5YzA0ZWIxMjM0NTY3ODkwYWJjZGVmMDEvMS4yMS40Lmpzb24=": "3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b",
      "url:aHR0cHM6Ly9waXN0b24tbWV0YS5tb2phbmcuY29tL3YxL3BhY2thZ2VzL2EzZjFiMmM0ZDVlNmY3MDg5MWIyYzNkNGU1ZjZhN2I4YzlkMGUxZjIvMTkuanNvbg==": "9c1185a5c5e9fc54612808977ee8f548b2258d31667a8b1a1c3f84982f3c4e5a"
    }
  },
  "assets": {
    "torrents": {
      "8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4": {
        "delivery": {
          "integrated": {
            "type": "torrent",
            "payload": "ZDEwOmNyZWF0ZWQgYnlxMTk6Tm9kZVN0b25lIFBhY2tlciB2MjEzOmNyZWF0aW9uIGRhdGVpMTc5NTI2NDgzNGU0OmluZm9kOTpmaWxlIHRyZWVkMjM6dmVyc2lvbl9tYW5pZmVzdF92Mi5qc29uZDA6ZDY6bGVuZ3RoaTEyNDUwZTExOnBpZWNlcyByb290MzI6..."
          }
        }
      },
      "3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b": {
        "delivery": {
          "url": [
            "https://compass.nodestone.io/v07/minecraft/torrents/3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b.torrent"
          ],
          "magnetic": "magnet:?xt=urn:btmh:12203a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b&dn=minecraft-core-1.21.4"
        }
      },
      "9c1185a5c5e9fc54612808977ee8f548b2258d31667a8b1a1c3f84982f3c4e5a": {
        "delivery": {
          "url": [
            "https://compass.nodestone.io/v07/minecraft/torrents/9c1185a5c5e9fc54612808977ee8f548b2258d31667a8b1a1c3f84982f3c4e5a.torrent"
          ],
          "magnetic": "magnet:?xt=urn:btmh:12209c1185a5c5e9fc54612808977ee8f548b2258d31667a8b1a1c3f84982f3c4e5a&dn=minecraft-assets-1.21"
        }
      }
    }
  }
}
```

## 5. 未来预留与演进计划

1. **`sha256:<Hex>` 格式的 NSFID**：
   目前 BT v2 `info["file tree"]` 中记录的 `pieces root` 是按 16 KiB 分块计算的 Merkle 树根哈希，并不等于文件整体的线性 SHA-256。因此在 v0.7.0 中，`latest.json` 及种子展开过程暂不使用 `sha256:` 前缀，仅在协议层预留该格式，供后续内容寻址扩展使用。
2. **正则表达式格式的 `nsci` 路由规则**：
   后续考虑在 `nsci` 下引入正则匹配表，用于进一步清理索引、减小 `latest.json` 体积。正则规则仅在查找入口时做路由匹配使用，**不作为 NSFID 使用**，命中或展开后写入本地缓存的仍必须是真实的 `NSFID -> hash` 映射。
