# NodeStone Compass Protocol (NCP) v0.7.0 — 客户端推荐实现

本文档在《概览与最小实现》的基础上，说明实际工程接入时推荐的批量下载调度、缓存未命中自愈流程以及本地缓存清理策略。

## 1. 三条基本原则

1. **本地缓存存的必须是真实的 `NSFID -> hash` 映射**：不管是来自 `latest.json` 的入口记录，还是从种子 `file tree` 展开出来的子文件记录，Key 都必须是符合规范的 `url:<Base64>`（或未来启用的 `sha256:<Hex>`），Value 为 64 位小写十六进制哈希。
2. **种子落盘与 `file tree` 展开必须视为同一事务**：首次下载某个 `.torrent` 文件并落盘时，必须同步把它的 `file tree` 展开写入本地持久化 NSCI 缓存数据库，两边都成功才算种子获取完成。
3. **正常流程下 `nsGetFile` / `nsGetFilesBatch` 不会在本地 NSCI 未命中**：按照协议流程，请求任何子文件前，必定已经拉过 `latest.json` 并加载过对应的版本入口种子（`<version>.json` 或 `<asset_index>.json`），里面的子文件一定已经展开进了本地缓存。一旦出现未命中，说明前置流程被跳过或本地缓存状态不一致，不能静默忽略。

## 2. 批量获取优化（`nsGetFilesBatch`）

启动器解析完 `<version>.json` 和 `<asset_index>.json` 后，通常要一次性下载成百上千个 `libraries` 和 `assets` 小文件。如果逐个调用 `nsGetFile` 会产生大量重复查库和调度开销，推荐优先实现并使用批量接口 `nsGetFilesBatch(nsfidList, contextEntryNSFIDs=None)`：

1. **批量查库**：一次性在本地 NSCI 缓存中查询 `nsfidList` 中所有项的 `(torrenthash, rel_path, orig_url)`。如果有任何一个 NSFID 未命中，先执行第 3 节的自愈流程；自愈后仍有未命中项则中止并报错。
2. **按种子分组**：把命中的文件按 `torrenthash` 聚合为 `Map<torrenthash, List<(NSFID, rel_path, orig_url)>>`。
3. **按种子批量提交下载**：对每个去重后的 `torrenthash` 调用一次 `nsGetTorrent`（通常在前一步下载 `<version>.json` 和 `<asset_index>.json` 时种子就已经在本地了），把该组内的目标文件列表一次性提交给 BT 引擎设置文件优先级，并配合 SDK 的 WebSeed 调度器并发或顺序拉取。
4. **交付结果**：全部完成后返回 `Map<NSFID, LocalFilePath>`，或者通过完成回调每下完一个文件通知一次上层启动器。

## 3. 缓存未命中的排查与四步自愈

实际接入时，很多第三方启动器自己带有本地文件缓存（比如 `.minecraft/versions/<ver>/<ver>.json` 或 `assets/indexes/<id>.json` 已经在硬盘上了）。如果用户删过 `.nodestone/` 目录、旧启动器中途刚接入 NCP SDK，或者上次展开种子时意外退出，启动器就会跳过下载 `<version>.json` 和 `<asset_index>.json`，直接读本地已有的 JSON 去调 `nsGetFile` / `nsGetFilesBatch` 请求里面的 `libraries` 和 `assets`，导致本地 NSCI 缓存查不到记录。

遇到未命中时，客户端推荐按顺序执行以下四步自愈：

### 第一步：URL 规范化重算

把未命中的 `url:<Base64>` 解码回原始 URL，检查是否存在协议或域名大写、带了默认端口（如 `:443`）、或者转义格式不一致的问题。按规范把 `scheme` 和 `host` 转为小写、去掉默认端口后重新编码出 `NSFID`，再查一次本地 NSCI 缓存。

### 第二步：按上下文入口（`contextEntryNSFIDs`）补拉入口种子并展开

启动器在调用 `nsGetFile(aNSFID, contextEntryNSFIDs)` 或 `nsGetFilesBatch(nsfidList, contextEntryNSFIDs)` 下载某个版本的依赖文件时，推荐在 `contextEntryNSFIDs` 参数里带上当前任务的入口文件 NSFID（也就是当前版本 `<version>.json` 和 `<asset_index>.json` 的 NSFID）。

当子文件未命中时，SDK 先检查 `contextEntryNSFIDs` 里的入口种子是否已经在本地展开过；如果没有，直接调 `nsGetTorrent` 把对应的版本核心种子和大版本 Assets 种子拉下来并执行 `file tree` 展开。只要传入了 `contextEntryNSFIDs`，哪怕 `.nodestone/` 目录刚被彻底清空，这一步也能直接把所有子文件索引补齐。

### 第三步：扫描本地未展开的种子与刷新 `latest.json`

* 如果调用方没传 `contextEntryNSFIDs`，先扫描本地 `.nodestone/torrents/` 目录，把已经下载到本地、但还没在 NSCI 数据库里标记完成展开的 `.torrent` 文件补跑一次 `file tree` 展开。
* 如果还是未命中，且距离上次拉取 `latest.json` 已经超过冷却间隔（例如 60 秒），就从 `provider.urls` 重新拉取一次最新 `revision` 的 `latest.json` 更新入口映射（应对 Mojang 刚刚发布新版本的场景），然后再重试一次第二步。

### 第四步：明确报错并由上层决定是否回退直连

如果走完前三步还是查不到（比如启动器传入了一个根本不在官方源里的第三方文件 URL），不要静默吞掉错误，应当向上层抛出带有 `(orig_url, nsfid, local_revision)` 的未命中异常，由启动器记录告警日志，并由启动器决定是中止任务还是回退到普通的 HTTPS 直连下载。

## 4. 本地缓存管理与清理建议

本地 NSCI 缓存采用「重复即覆盖」，主要是为了处理 Mojang `version_manifest_v2.json` **固定 URL 但内容频繁更新**的情况——它的 URL 不变（NSFID 不变），但每次 Mojang 发新版后，它在 `latest.json` 里指向的种子 Hash 会变。相比之下，**已发布的游戏版本文件（`<version>.json`、`client.jar`、`libraries`、`assets`）内容是固定不变的**。

针对这两类数据的特点，推荐按如下方式管理本地缓存：

### 1. 持久化表结构与来源标记

建议使用轻量级本地数据库（如 SQLite，存放在 `.nodestone/cache/nsci.db`）持久化 NSCI 缓存。每条记录保存 `(nsfid, torrenthash, rel_path, orig_url, source_type)` 五个字段，其中 `source_type` 用来区分记录来源：

* `"entry"`：来自 `latest.json` 的 `nsci.raw_kv` 顶层入口记录。
* `"derived"`：来自种子 `info["file tree"]` 展开生成的派生子文件记录。

```python
import base64

def expand_filetree_to_db(torrent: dict, torrenthash: str, db_conn):
    file_entries = parse_btv2_file_tree(torrent["info"]["file tree"])
    schemes = torrent.get("ns-schemes") or torrent.get("url-list") or ["https://"]
    if isinstance(schemes, str):
        schemes = [schemes]

    records = []
    for rel_path in file_entries:
        for raw_scheme in schemes:
            scheme = raw_scheme.rstrip("/") + "//" if raw_scheme.endswith(":/") else (
                raw_scheme if raw_scheme.endswith("://") else "https://"
            )
            orig_url = f"{scheme}{rel_path}"
            b64 = base64.b64encode(orig_url.encode("utf-8")).decode("ascii")
            nsfid = f"url:{b64}"
            records.append((nsfid, torrenthash, rel_path, orig_url, "derived"))

    db_conn.executemany(
        "INSERT OR REPLACE INTO nsci_cache (nsfid, torrenthash, rel_path, orig_url, source_type) VALUES (?, ?, ?, ?, ?)",
        records
    )
    db_conn.commit()
```

### 2. 过期可变入口种子的自动清理

每次拉取新 `revision` 的 `latest.json` 并覆盖旧的 `"entry"` 记录后，检查被替换掉的 `old_torrenthash` 在数据库中还剩多少条 `source_type == "entry"` 的记录（即 `"entry"` 引用计数）：

* 一旦某个 `old_torrenthash` 的 **`"entry"` 引用计数归零**（例如旧版 `version_manifest_v2.json` 的种子已经被新版替代），应立即停止该种子的做种任务，按 `torrenthash` 删掉它在数据库里展开出的所有 `"derived"` 记录，并删除本地对应的 `.torrent` 文件和临时数据块。

### 3. 已发布版本不可变种子的按需清理

* 各个已发布版本的种子（版本核心种子与大版本 Assets 种子）默认随版本文件保留做种。
* 如果用户需要控制磁盘占用，可以按 LRU（最近最少使用）策略，或者对照启动器本地已安装的版本列表，定期清理超过指定天数未使用的旧版本做种任务、`.torrent` 文件及对应的 `"derived"` 缓存记录（`.minecraft` 目录下的游戏本体文件仍由启动器自己管理，不受影响）。
