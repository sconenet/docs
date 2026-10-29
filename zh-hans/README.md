# Scone 文档

> `docs/` 下的所有文档均为**规范性（NORMATIVE）**文档：必须能够在不阅读
> Rust 代码的情况下支持独立实现。任何文档与代码的
> 分歧都是一个 bug（确定唯一事实来源并
> 修复它）。

分为两部分：

- **`general/`** —— 工作原理：协议的概念与规则，
  无需阅读格式细节即可理解；
- **`technical/`** —— 面向开发者的详细规范：精确的
  二进制格式、哈希公式、边界限制、API 以及
  各组件的行为。

## `general/` —— 工作原理

| 文档 | 内容 |
|---|---|
| [architecture.md](general/architecture.md) | 权限（区块链）/ 可用性（DHT）/ 本地存储的分离；各 crate 的职责；DNS 解析流程；可演进性 |
| [naming.md](general/naming.md) | 命名规则：LDH 字母表、TLD、域名与子域名、规范化、`DomainId` 派生及十六进制表示 |

## `technical/` —— 开发者规范

| 文档 | 内容 |
|---|---|
| [protocol.md](technical/protocol.md) | 规范线上二进制格式：LEB128 变长整数、标识符、DNS 记录、区块、P2P 消息、限制、版本管理 |
| [transactions.md](technical/transactions.md) | 交易格式 v1：已签名形式（字段、线上顺序——RegisterDomain 携带名称、任意判别值、所有者派生、签名载荷 `SCONE-TX-SIG-V1`）以及四种无签名的 pow-to-owner 形式（挖出的工作量证明即是授权）、委托权威家族（绑定计数器：TransferDomain、运营者、记录委托）、变异计数器、验证规则 |
| [blockchain.md](technical/blockchain.md) | 区块链层：`TxId`、Merkle 树、`BlockHash`、创世区块、规范状态、区块验证、共识抽象 |
| [keystore.md](technical/keystore.md) | `.sconekey` 密钥文件格式（Argon2id + XChaCha20-Poly1305）、`scone-keystore` API、`identity` 命令 |
| [storage.md](technical/storage.md) | 本地 redb 持久化：表结构、原子增量、内存策略、信任模型、DoS 边界 |
| [relay.md](technical/relay.md) | 网络中继（`scone-network`）：libp2p 协议、节点发现（种子、目录、UPnP）、可达性（AutoNAT、circuit-relay、DCUtR）、内存池、同步、Kademlia DHT、本地拓扑、Prometheus 指标、DNS 快照、完整 CLI、`.env` 配置 |
| [dns.md](technical/dns.md) | UDP + TCP DNS 服务器（M6/M7b）：权威 DNS 快照、16 路分片热路径缓存（共享条目、无分配查找、紧凑记录、名称压缩）、RFC 1035 编解码器、TCP RFC 7766（512 字节 UDP / 64 KiB TCP 预算、TC 位）、DHT 负缓存、递归回退、边界限制 |
| [cli.md](technical/cli.md) | `scone` 二进制：日志与详细程度（stdout/stderr、`-v` 级别、`RUST_LOG`）、通过 `.env` 进行环境配置（优先级：标志 > `.env` > 默认值）、记录委托（`domain delegates set/clear/list`）、由委托签署的更新 |
| [security-model.md](technical/security-model.md) | 安全模型：委员会选举、种子防研磨（分析：无需 VRF）、歧义行为与罚没 |
| [audit-methodologie.md](technical/audit-methodologie.md) | 安全审计方法：分歧、事实来源、修正轮次 |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | 服务证明 RFC：运营者心跳、操作委托、在新鲜席位中选举委员会 |

## 交叉引用约定

内部文档和 Rust 代码注释通过从仓库根目录起的
> 绝对路径引用这些页面
>（例如 `/docs/en/technical/protocol.md`）。
