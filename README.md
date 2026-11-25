# NodeStone Compass

**NodeStone Compass Protocol（NCP）** —— 一套基于 BitTorrent v2 的开放资源分发协议，以及 NodeStone 官方基于该协议维护的元数据源。

当前版本：**v0.10.0** · 许可证：[Apache-2.0](LICENSE)

## 文档

* [总览及最小实现](docs/OVERVIEW.md)：术语、整体流程、预配置、NSFID 编码与最小可用实现
* [客户端推荐实现](docs/minecraft/CLIENT_RECOMMENDED_IMPLEMENTATION.md)：调用约定、批量下载调度、WebSeed 引擎适配与本地缓存清理
* [服务端推荐实现](docs/minecraft/SERVER_RECOMMENDED_IMPLEMENTATION.md)：大版本打包策略、Packer 打包规范、WebSeed 字段配置与 NSCL 发布流程
* [协议规范](docs/PROTOCOL_SPECIFICATION.md)：`nsci` 与 `torrents` 分离设计、`latest.json` 字段表、BT v2 种子结构与完整示例
