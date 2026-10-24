# UDP + TCP DNS 服务器（M6 + M7b）—— `scone dns`

> Scone DNS 面的规范规格。
> 任何文档/代码分歧 = bug（修复事实来源）。

## 概览

**DNS 服务器**是中继的可选面：它在 **UDP 与
TCP** 上（RFC 7766、同一端口）回答针对 Scone 名称、
且**只来自对照链验证过的数据**的 DNS 查询，并为
其余一切提供**可选的递归回退**（`addr:port`）。
它不拥有链、不拥有存储、不拥有 DHT：存在性/过期
决策经**权威 DNS 快照**（每个已应用区块之后发布
的 RAM 视图 —— 见下文）、记录获取经中继
（进程）、走内部 `RelayCommand`/`ResolveLocal`
通道（进程内、一次往返）—— 与 CLI 相同的已验证
路径。

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Sharded cache, 16 ways (qname)  │
                    │  └────────────────────────────────┘
                    │  Scone-shaped name?
                    │  ├─ YES → authoritative DNS snapshot:
                    │  │        unknown/expired → NXDOMAIN (RAM,
                    │  │        zero round trips into the core)
                    │  │        → apex search by suffix
                    │  │        → internal `resolve_local` (relay)
                    │  │           chain + verified record (hash,
                    │  │           owner, sequence) → answers
                    │  │        apex without a published record → NODATA
                    │  └─ NO → upstream fallback (if configured)
                    │           otherwise REFUSED
                    │
                    │  response budget:
                    │    UDP → 512 bytes (no EDNS); overflow =
                    │          cut at the last fitting record + TC=1 →
                    │          the client retries over TCP
                    │    TCP → 64 KiB (2-byte length prefix,
                    │          RFC 1035 §4.2.2): full set
```

## 启动

```bash
scone relay --dns 127.0.0.1:5353            # DNS surface only
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + recursive fallback
```

中继在 stdout 上宣告生效地址（机器可读契约、
如同 `p2p:`）：套接字一绑定就输出一条
`dns: <addr>` 行。

验证客户端：`dig <name> --dns 127.0.0.1:5353
--qtype A`（A、AAAA、TXT、MX、NS、CNAME、ANY）。

## 解析路径（权威 = 链、始终如此）

0. **权威 DNS 快照**：由中继核心在每个已应用
   区块**之后**（push 而非 poll）以及启动时发布的
   不可变视图
   （`Arc<HashMap<DomainId, {valid_until, has_record}>>`、
   `tokio::sync::watch` 馈送）。存在性/过期决策 ——
   NXDOMAIN、过期域名 —— 经此快照从 RAM 提供：
   一次 `Arc` 克隆 + 一次 map 查找、**绝不进入
   中继的单所有者运行循环**。完整路径
   （`RelayCommand`/`ResolveLocal`）仍保留给真正的
   记录获取（DHT 拉取）、得益于缓存而罕见。快照
   绝不携带记录数据：只有存在性与过期。
   - **O(touched) 写时复制增量**：map 是
     `Arc<HashMap>`；tip 延长发布区块的增量
     （区块自身的交易点名它们能改变的每个域名
     —— Register/Assign/Update/Renew/TransferDomain
     —— 加上在 push 时求值的确定性过期 GC 移除）
     经 `Arc::make_mut`：只有在读者仍持有前一
     视图时才克隆 map。启动与重组全量重建快照
     （一次分页的后端遍历）。
   - 新鲜度：区块应用期间读者可能看到滞后几百
     毫秒的快照 —— 与状态 RPC 相同的滞后；DNS
     缓存 TTL 不受影响（应答按其来源保持权威）。
1. **分片缓存**：[`ShardedCache`] —— 16 个独立的
   `Cache` 分片、按 `hash(qname)` 分键（qtype 共享
   分片：按名称的局部性）。一个热名称只与 1/16
   的流量竞争、而不是在数千 qps 下排在单一
   mutex 之后。全局边界（默认 100,000 条 / 64 MiB、
   `--dns-cache-limit` / `--dns-cache-max-mem`）按
   分片分摊（已记录的最坏情况：1 + N/shards 的
   因子）。每条条目的**真实** rdata 大小被记账
   （每条约 128 字节开销）、FIFO 驱逐在两个边界
   中先到者触发 —— 即使面对满尺寸条目
   （`MAX_DHT_RECORD_LEN`）内存上限依然成立。
   TTL 60 s（权威）/ 截断到 600 s（回退）。负缓存
   （NXDOMAIN/NODATA）遵循相同规则；每条条目携带
   其来源（权威或回退）使命中时 AA 位保持诚实。
   **绝不缓存 SERVFAIL**（瞬态本地失败）**与
   REFUSED**（配置状态）：两者在每次查询时重新
   计算。容量处 FIFO 驱逐。**热路径（受
   Cloudflare/1.1.1.1 启发）**：
   - 共享 `Arc<CacheEntry>` 条目：命中是一次引用
     计数递增、**绝不**是 rdata 的深拷贝（`put`
     替换 Arc —— 无共享变更；TTL 仍在分片锁下
     检查）；
   - **无分配查找**：两级 map（`Box<str>` qname →
     qtype → 条目）、查询借用的 `&str` 直接探测
     （`Borrow<str>`）—— 旧的 `(String, u16)` 元组键
     在**每次**查找上都分配新 `String`；
   - **紧凑连续记录**：每条条目一个 `Box<[u8]>`
     缓冲（每条记录 `u16 rdlen ‖ u16 rtype ‖
     u32 ttl ‖ rdata`）而非每条记录一个 `Vec` ——
     内存预算精确追踪所分配的内容；
   - **输出上的 RFC 1035 名称压缩**：权威应答的
     owner 是指向回显问题的 2 字节压缩指针
     （`0xC00C`）—— 问题绝不被每条记录完整重
     编码。
2. **Scone 形状？** 严格预过滤：标签
   `[a-z0-9-_]`（1..63）、≥ 2 个标签、TLD 1..63
   字节 `[a-z0-9-]`（见 `TldName::MAX_LEN`）、总长
   ≤ 253 —— **且 TLD 在 ICANN 根之外**
   （`is_icann_tld`、冻结的 IANA 列表 + RFC 2606：
   `com`、`fr`、`org`…… 永不属于 Scone）。不可能
   是 Scone 名称的名称（如 `www.foo_bar`、TLD 中
   的下划线）进入回退；可能是 Scone 名称的名称
   （`absent.uip`）是链的事务：未注册 → 权威
   NXDOMAIN、即使没有上游。ICANN TLD
   （`example.com`）在配置时被转发给上游、否则
   REFUSED —— 绝不被 Scone 权威解析。
3. **Apex 搜索**：后缀从最长到最短（最少
   2 个标签）。第一个链上状态为中继所知的后缀 =
   apex；记录集服务整个 qname（含子名 —— 隐式
   通配行为、未决事项）。
4. **`resolve_local`（内部通道）**：单次往返、
   绝非 DHT 等待者。链上状态（`registered`、
   所有者、序列号）+ **重新验证**过的本地 DHT
   缓存记录（所有者、序列号、
   record_hash == 链上承诺）。不合格的记录绝不容
   答 —— 可用性 ≠ 权威。
   - 未注册 → NXDOMAIN；
   - 已注册但无有效记录 → NODATA（NOERROR、0
     条应答）；
   - 已验证记录 → 按 qtype 过滤的应答。

## 编解码器（RFC 1035、严格子集）

- 查询：12 字节头部、**一个** question、
  QCLASS = IN、**无压缩**（指针 = 拒绝）。标签转
  小写并验证。QR=1、QDCOUNT≠1、无效标签 →
  **沉默**（垃圾一文不花）。
- 应答：QR|RD（回显）|RA、**只在权威应答上
  AA**（链、新鲜或缓存 —— 绝不在回退或错误
  RCODE 上）、question 逐字拷贝、权威应答带
  **RFC 1035 压缩的 owner**（指向回显问题的
  2 字节指针 `0xC00C`）；回退上游回复按原样
  提供（绝不重编码）。
- 服务的类型：A、AAAA、CNAME、NS、MX、TXT、
  **PTR、SRV、CAA**（+ 按原样重编码的 Unknown；
  共 9 种原生类型、rdata 按 RFC 编码 —— 
  1035/2782/8659；严格 CAA：仅 critical 位 0x80、
  小写标签）。A/AAAA 查询包含集合中的 CNAME
  （别名追踪）。ANY 查询（255）：整个集合被
  编解码器过滤。
- 限制（M7b）：**应答预算取决于传输** —— UDP
  上 512 字节（经典 RFC 1035 §2.3.4 载荷、无
  EDNS）、TCP 上 65,535。超出预算的集合在最后一
  条放得下的记录处截断、带 **TC=1**（诚实截断、
  RFC 1035 §4.2.1）：客户端经 TCP 重试（RFC
  7766）、在那里获得完整集合。大于 UDP 预算的
  上游回复被重编码截断（TC=1）而不是按原样
  交付；TCP 上逐字节通过。截断也适用于缓存
  命中（预算在应答时应用、而非缓存填充时）。
- RCODE：0 NOERROR（带或不带应答）、2 SERVFAIL
  （本地失败）、3 NXDOMAIN（权威）、5 REFUSED
  （无上游的非 Scone）。

## 递归回退

- 上游 `addr:port`（UDP）、按序尝试、每上游
  2 s 超时、套接字与上游 **connect**（内核过滤：
  来自任何其他源的分组绝不被接受）。上游响应
  只有在**与查询匹配**时才被接受：相同 TXID、
  QR=1、question（QNAME+QTYPE+QCLASS）逐字节
  拷贝。不合格的响应 → SERVFAIL、且什么都不
  缓存。
- 已验证的上游响应按原样放行；其应答以来源
  `fallback` 缓存（绝不设 AA、即使在缓存命中
  上）、TTL 截断到 600 s。上游错误 RCODE
  （NOERROR/NXDOMAIN 之外）不被缓存。
- **默认私有**：无上游 = Scone 形状之外的名称
  REFUSED。

## DHT 负缓存（防放大）

一次**未应答的 DHT 拉取**（记录在链上被期望 ——
`record_hash` 已承诺 —— 但不在本地缓存也不在
被查询的对等节点处）把该名称在
`DHT_NEGATIVE_TTL_SECS`（30 s）内停入有界负缓存
（FIFO、上限 `DHT_NEGATIVE_CAP` = 4,096 个名称）：
TTL 内对同一名称的重复解析被**抑制**（计数、
绝不重新拉取）。扫描器 —— 或其所有者从未发布
的热门名称 —— 不能再把一次 DNS 查询放大成对
每个网格对等节点的每次查询一条 `GetRecord`。
负条目在 TTL 到期、记录到达（成功的拉取写入
真正的缓存）或被挤出 FIFO 上限时消亡。

暴露的计数器（`scone_dht_negative_hits_total`、
`_expired_total`、`_capped_total` —— 见
`relay.md` 的"指标"）。

## TCP（M7b、RFC 7766）

与 UDP 相同的端口（在 `--dns` 地址上的第二个 TCP
绑定）。传输遵循 RFC 1035 §4.2.2：每条消息以
**2 字节**大端长度为前缀。

- **查询**：前缀长度 ≥ 1 且 ≤ 512 字节（DNS 查询
  放得进经典载荷 —— RFC 7766 §6；不服务 EDNS）。
  越界 → 立即关闭连接。
- **应答**：65,535 字节预算（2 字节前缀的最大
  值）—— 对 UDP 太大的集合在此**完整**提供、
  TC=0。
- **流水线**：每连接多条查询按序串行服务
  （一次写 = 一条带前缀的应答）。
- **读取超时**：每次读取（前缀或主体）5 s ——
  沉默的客户端被断开、其槽位释放。
- **并发连接**：最多 64（信号量）；第 65 个被
  立即拒绝（关闭连接）、绝不排队 —— 任务或内存
  无无界增长。
- 读/写/超时错误：debug 日志 + 干净关闭。**在
  网络输入上绝不 panic。**
- 垃圾（无法理解的查询）得到沉默但不关闭
  连接：下一条（有效的）查询仍被服务。

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → authoritative → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (UDP had answered TC=1 → the client retries here)
```

## 边界与护栏

| 护栏 | 值 |
|---|---|
| UDP 数据报（查询/应答） | 4096 字节（超出拒绝） |
| UDP 应答预算（超出 TC=1） | 512 字节（无 EDNS） |
| TCP 查询最大 | 512 字节（超出关闭） |
| TCP 应答最大 | 65,535 字节 |
| 并发 TCP 连接 | 64（信号量、立即拒绝） |
| TCP 读取超时 | 5 s |
| 缓存 | 默认全局边界 100,000 条 / 64 MiB、分摊到 16 个分片（`--dns-cache-limit` / `--dns-cache-max-mem`） |
| 权威 TTL | **按记录**（`record.ttl`、边界 1..=2,592,000 s；历史常量 60 s 只对服务应答保留） |
| 回退 TTL（截断） | 600 s |
| 并行 UDP 查询 | 4,096（`MAX_INFLIGHT_UDP`、信号量；超出：丢弃、客户端重试）—— 可调 `--dns-max-inflight`（最小 1） |
| DHT 负缓存 | 30 s（`DHT_NEGATIVE_TTL_SECS`）、FIFO 上限 4,096 个名称 |
| 每上游超时 | 2 s |

遵守的项目规则：网络数据上绝不 panic；处处严格
解码；服务器不触碰链、不触碰存储、不触碰
DHT（中继仍是唯一的验证权威）。

## 测试

- 单元（`scone-network::dns`）：严格编解码器
  （QR、压缩、标签拒绝）、权威应答、NODATA、
  NXDOMAIN、无上游 REFUSED、对 UDP mock 的回退、
  有界缓存、TXT 分块、上游解析；M6 修正：AA 位
  诚实（权威 vs 回退 vs 错误、含缓存命中）、
  超大集合的 TC=1 截断、上游响应拒绝（TXID 或
  question 不匹配）、SERVFAIL/REFUSED 不缓存；
  分片缓存：分片分布、分摊的全局边界、热名称
  上 4 个并发 worker；热路径：无分配查找、紧凑
  条目、owner 压缩。
- M7b：按传输的预算（UDP 512 字节 → TC=1、
  TCP → 完整集合）、缓存命中的截断、真实
  `run_tcp`（集合 > 512 完整提供、2 条查询的
  流水线、超长查询 → 关闭、沉默客户端 → 5 s
  超时、同一连接上垃圾然后有效查询）、UDP 上
  超过 512 的上游回复重编码 TC=1 / TCP 上逐字
  节通过。
- 真实集成（`tests/dns_server.rs`）：完整中继
  （含 devnet 生产）+ 注册 + UpdateDomain +
  PutRecord、然后真实 UDP 查询：已验证的 A
  （AA 位）、TXT、NODATA AAAA、NXDOMAIN、经
  mock 的回退、再查询、经 apex 的子名。
- 二进制 E2E（`crates/scone/tests/dns_cli.rs`）：
  真实的 `scone` 二进制 —— `relay --dns`、
  `identity generate`、`domain register`、
  `domain update --file`、然后在真实 UDP 端口上
  `dig`（dnsutils）（已验证的 A、TXT、NXDOMAIN、
  REFUSED）。

## 未决事项

- 无 EDNS（UDP 预算保持 512 字节；TC=1 截断 +
  TCP 覆盖 devnet 需求）。
- 记录集不携带按记录名称：整个集合服务 apex
  的每个子名（隐式通配）。按记录寻址需要扩展
  `DnsRecord`（协议）。
- 回退转发原始查询：恶意上游可以回答任何东西
  （未验证的响应 —— 这是标准的递归模型；
  Scone 名称**绝不**经过它）。

## GeoDNS（地理路由）

> 规范规格。任何文档/代码分歧 = bug。

### 内嵌 GeoIP 表（自动）

每个二进制都内嵌一张编译进中继的 **IP → 国家
映射表**：**无需配置**、地理路由处处自动生效。

- **数据来源**：**DB-IP Lite**（"IP to Country
  Lite"）、**CC BY 4.0** 授权 —— 可在署名前提下
  再分发：`https://db-ip.com`（每月整理；此前：
  经 `scripts/gen-geo-table.py` 的 RIR 授权、已
  移除）。量化单元格 **/20 v4 与 /24 v6**（DB-IP
  的原生粒度、约 130 万个范围、超出解析器的
  内存界限）—— 约 96,500 条规则。生成日期在
  表头（`assets/embedded-geo.table`）。
- **校验和**：表的 SHA-256 在编译期计算
  （`build.rs`）、**启动时重新验证**（不匹配 =
  类型化拒绝 —— 被篡改的二进制绝不容答地理
  应答）并在启动时连同署名信息记录日志。该
  哈希保证"同一构建的每个节点共享同一张表"
  是可校验的、而非仅是假定。
- **再生成**：`scripts/geo-convert.py`（下载当月
  DB-IP Lite CSV、转换 —— 数字国家、大洲由
  ISO→M49 派生、Lite 中无 ASN/区域）。
- **粒度**：国家（数字 ISO 3166-1）+ 派生的
  大洲（UN M49）。无 ASN、无区域（Lite 中
  没有 —— ASN/区域需求等待内嵌管线的演进、
  绝非本地表）。
- **性能**：对排序范围做**二分查找**（索引在
  首次使用时按 v4/v6 族各建一次）—— 实测热
  约 95 ns/查找、全表随机访问约 50 µs（基准：
  `cargo test -p scone-network --release --test
  geo_bench -- --ignored`）。
- **中继自定位**：路由跟随**每次查询的源
  IP** —— 局域网中继自动把其本地客户端路由到
  正确的区域。
- **强制的表**（remove-geo-operator-table，
  2026-10-22）：内嵌表是**唯一来源** —— 不存在
  运营者导入或覆盖。**不存在运行时 geo 选项**：
  每个 `--geo-*` 标志都被 clap 硬性拒绝（地理
  区由构建强制）。有特定需求（ASN、区域）的
  运营者等待内嵌管线的演进、而非本地表。

### 概览

域名可以在其签名记录集中发布**地理路由**规则：
`GEOROUTE` 记录（Scone 私有类型 `32,768`、在
IANA 范围之外）。解析器以**由规则选出的端点
地址**（离**客户端**位置最近的作用域、其源 IP
经表派生）回答这些域名的 A/AAAA 问题、而不是
静态的 A/AAAA 记录。其余一切（无 GEOROUTE 的
域名、其他 qtype）走经典 DNS 路径、**逐字节
相同**。

### GeoDNS 指标

Prometheus 端点暴露 geo 应答拆分：

- `scone_dns_geo_answers_total` —— 已服务的
  GEOROUTE 应答总数；
- `scone_dns_geo_by_continent_total{continent="<M49>"}`
  —— 按大洲；
- `scone_dns_geo_by_country_total{country="<ISO>"}`
  —— 按国家（受标准自身约束：≤ 9 个大洲、
  ≤ 250 个国家）。

### GEOROUTE 记录（线上）

一条 GEOROUTE = 一个**作用域** + ≤ 32 个**端点**
（解码时规范排序：priority 升序、weight 降序、
target 升序；重复 = 拒绝）：

```text
GeoScope   = 0x00 (world)
           | 0x01 continent.v (UN M49 code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (ISO 3166-1 numeric code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO 3166-2 subdivision, u32:
                                publisher-defined number, compared
                                as is — no local table)
           | 0x04 asn.v       (AS number, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalized 16-byte IP prefix: IPv4 = address in the
              first 4 bytes + ZERO tail, len ≤ 32;
              IPv6 = 16 free bytes, len ≤ 128; host bits ALWAYS
              zero — a non-normalized prefix is rejected)
GeoTarget  = 0x00 octets[4]  (binary IPv4 address)
           | 0x01 octets[16] (binary IPv6 address)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

验证边界（解码）：`priority > 0`、
`weight ≤ 100`（`weight = 0` = 不活跃端点、绝不被
选中）、TTL 如任何记录在 1..=2,592,000 s 内。
本地表不认识的国家/大洲在线上**被允许**：路由
只是回退到更高的作用域。

**线上向后兼容（v1 → v2）**：v1 字节**不变** ——
v1 作用域（世界/大洲/国家）保持其判别值
`0x00..=0x02` 与精确编码；三个新作用域取后续
判别值（`0x03..=0x05`）、处于与规范排序一致的
全序中（`World < Continent < Country < Region <
Asn < IpPrefix`）。遇到判别值 ≥ `0x03` 的 v1
解码器本就严格失败（记录被拒绝）；v1 产生的
字节仍被相同解码（见 `geo_known_bytes` 测试、
已冻结）。

### 选择（解析）

1. **作用域** —— 递减的特异性、**6 级**：
   **最长匹配 IP 前缀 → 精确 AS → 精确国家 →
   精确区域 → 大洲 → 世界**。
   - *IpPrefix*：在前缀匹配客户端 IP（按定位器
     所知：`client_ip`、否则已知的来源前缀）
     的路由中、**最长**（`len` 最大）者胜；
     `len` 相同时、规范序中的第一个（`routes`
     切片的顺序、编码时排序）。一个行政区划
     比前缀/AS（它们标识确切网络）更不特定、
     也比国家（不透明编号、只有发布者与定位器
     共享同一编号时才可靠）更不特定、但比
     大洲更特定（它包含于其国家、故更包含于
     其大洲）。
   - *Asn*：客户端的精确 AS 号。
   - *Country*：精确国家。
   - *Region*：精确行政区划（不透明标识符）。
   - *Continent*：由定位器提供**或从国家经
     本地表派生**（`continent_of_country`）；
     表不认识的国家回退到显式大洲或世界。
   - *World*。
     v1 的 [`GeoLocation`]（region/asn/prefixes/
     client_ip 未设）产生恰好 v1 的行为：国家
     → 大洲 → 世界。无匹配路由（包括没有
     World 作用域）→ 经典 DNS 行为、如同
     GEOROUTE 不存在。
2. **Priority** —— 类 SRV 语义：只有**最小
   priority**组是可选的；更高的 priority 是
   **故障转移**层级。
3. **Weight** —— 在当选组内做**确定性加权**
   抽取：以 SplitMix64 对 nonce（解析器的查询
   计数器）做稳定哈希、对照累计权重。无 RNG：
   相同 nonce 的两次查询给出相同结果、递增的
   nonce 按权重比例分布。`weight = 0` = 不
   活跃、绝不被选中（即使单独 —— 移到下一
   层级）。
4. **签名顺序、零探测** —— 解析器**绝不**评判
   端点的可用性：当选范围内的每个活跃端点
   （weight > 0）都是候选，最小层级的头部由
   加权抽取选出，其余的**按签名记录集顺序**
   充当故障转移（顺序就是故障转移机制）。
   端点可达性是端点运营者的事、绝不是解析器
   的事。

### 应答

- 单一地址：**IPv4 则 A、IPv6 则 AAAA** —— 按
  qtype 严格按族过滤（A 的问题**绝不**回答
  IPv6、反之亦然）。
- TTL = 源 GEOROUTE 记录的（解码时受一般 TTL
  规则约束）。
- 设置 AA（权威应答 —— GEOROUTE 与任何其他
  记录一样是已验证的签名记录）。
- 过滤后空应答（无正确族的候选、该区域无
  路由）→ 解析器继续其经典
  路径（通常是 NODATA：NOERROR、0 条应答）；
  错误 rcode 遵循一般规则（绝不缓存）。
- 集合中共存的经典 A/AAAA 记录**优先于** geo
  （geo 只在该 qtype 缺少线上应答时适用）。

### 缓存：显式决策

geo 路由决策**绝不**在 `(qname, qtype)` 下
缓存：它取决于解析器的客户端作用域、这样的键
会把美国路由提供给台湾客户端。geo 应答在每次
查询时重算（成本：一次 select_scope + 加权
抽取、µs 级）并绕过缓存。经典的 (qname, qtype)
缓存对其余一切保持不变 —— 包括 geo 未做决策时
产生的 NODATA。错误 rcode 保持绝不缓存。

### NS 委派（`ns` 记录）—— 所有者选择由谁来服务

域名的所有者为每个域名选择服务模式：

- **DHT（默认）** —— 无 NS 记录时，Scone 网络
  服务链上 A/AAAA（中继 `--dns`、或经导出的
  NSD 桥接）；
- **独立 NS** —— 所有者添加 `ns <ip-or-name>`
  记录（他们自己的 nameserver、直接 IP 或
  主机名）：该域名被**委派** —— 网络停止服务
  其内容（A/AAAA → NODATA + NS 权威）、任何
  解析者跟随所有者的 NS。界限：**每个域名
  12 个去重的 NS**（所有形式、按值去重 ——
  映射现实世界注册局的配额上限）。移除一个
  NS 释放一个槽位；移除全部则恢复 DHT 模式。

**不存在健康检查**：端点可用性是端点运营者
的事 —— 解析器按记录顺序原样服务签名记录集
（记录顺序就是故障转移机制）。

### 安全

- GEOROUTE 记录与其他一样是签名记录：在使用
  前对照链验证（所有者、序列号、record_hash）。
  **没有验证就没有网络信任** —— 未验证的
  DHT GEOROUTE 绝不容答。
- 客户端位置来自**自动发现的线路**：中继经由
  其外部地址（STUN/AutoNAT/UPnP）自定位、由
  构建强制的内嵌表解析 —— 局域网客户端（私有
  源 IP）继承该位置、互联网客户端由其自身 IP
  解析、未知则回退到 World。绝不来自网络
  GeoIP API（无查询泄露、无依赖）、也绝不来自
  运营者声明（各区域由构建强制）。
- 抽取是确定且本地的：无秘密、不要求不可预
  测性（安全在于记录签名、而非抽取）。

### 完整示例（参考规格）

`www.example.scone` 发布（apex
`example.scone`、整个集合服务每个子名）：

| 作用域 | 端点 | Priority | Weight |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

期望行为：

- **US** 客户端 → 10.1.0.1（≈70%）或
  10.1.0.2（≈30%）；只有当没有任何 p10 端点
  活跃（所有 weight = 0）时才到达 p20 层级 →
  此时 **10.2.0.1**；
- **TW** 客户端 → 10.3.0.1（≈80%）或
  10.4.0.1（≈20%）；
- **FR** 客户端（250）→ 集合中无 FR 或 EU 路由
  → WORLD → **10.0.0.1**；
- 对仅 IPv6 路由的 **A** 问题 → 空应答（A 中
  绝不出现 IPv6）；
- 无适用规则的域名 → 相同问题、经典行为
  （NODATA：GEOROUTE 无线上 A rdata）。

### CLI 选项（`scone relay`）

| 选项 | 默认 | 角色 |
|---|---|---|
| （无 —— GeoDNS 常开、内嵌表） | 开 | 按源 IP 自动路由 |
| `--geo-ip-table`、`--geo-config`、`--geo-country/…`、`--health-*` | — | （已移除 —— 硬性破坏：地理区由构建强制、解析器不做任何健康检查；不存在运行时标志） |

#### 内嵌表是唯一来源

GeoIP 表**由构建强制**（DB-IP Lite、量化 —— 见
上方内嵌表一节）：无法导入任何运营者表。
解析：**DNS 查询的源 IP**（按客户端：UDP
`recv_from` / TCP `peer_addr`）按 16 字节对照
内嵌表比较；**最长匹配前缀**的规则胜（平局时
文件顺序中的第一个）。

- **私有/不可路由的源**（RFC1918、ULA
  fc00::/7、链路本地、回环）：客户端位于中继
  的网络之内 —— 它继承中继自身发现的位置；
- **表中缺失的公共源**：World（不做 geo 切分
  —— 绝不给出错误应答）。

**严格性**为任何未来的表表面保留：无效行是
携带违规行号的类型化启动错误；绝无被静默
忽略的行、绝无部分表。

#### 选项优先级

- **定位器**：内嵌表**强制**地理定位 —— 无
  运营者覆盖。中继从其外部地址发现自己
  的位置（STUN/AutoNAT/UPnP 由该表解析）；
  局域网客户端继承该位置、互联网客户端由其
  源 IP 解析、未知则回退到 World。

示例（无需任何 geo 标志、geo 路由即生效）：

```bash
scone relay --dns 127.0.0.1:5353
```


## DoH / DoT（加密传输）—— 经外部前端

Scone DNS 服务器说**纯 UDP 与 TCP**（RFC 1035 /
7766）—— 这是原生范围、完整且经过测试。到
*终端客户端*的加密传输（DoH —— DNS-over-HTTPS、
RFC 8484；DoT —— DNS-over-TLS、RFC 7858）**委托
给标准前端**：Scone 提供数据、前端提供传输。

为什么这样设计：原生 DoH 服务器要求节点内带
完整的 HTTPS 栈（证书、续期、HTTP/2）—— 在
有浏览器端最终用户之前是零收益的大运维面。
现有前端做得更好也更久；节点保持最小。

### 示例：unbound 在 Scone 中继前做 DoH

`/etc/unbound/unbound.conf`：

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) on 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # the Scone relay (--dns 127.0.0.1:5353)
```

证书（`scone.pem`/`scone.key`）像任何 TLS 服务
一样获取 —— 例如解析器域名上的
certbot/acme.sh。DoH 客户端随后指向
`https://dns.example.tld/dns-query`。

### 示例：knot-resolver 做 DoT

`/etc/knot-resolver/kresd.conf`：

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### 前端增加什么

- **客户端→前端信道的加密**（敌意段：用户的
  网络）；前端→中继段保持本地（同机或可信
  网络）。
- **一个额外的前端侧缓存**（unbound/knot 也有
  缓存 —— 无害、Scone TTL 仍是上界）。
- **一个稳定端点**（IPv4/IPv6、域名）而中继
  移动。

### 前端不做什么

- 它不*验证*任何东西：中继服务的每个应答在
  被缓存前已对照链验证（见"解析路径"）。前端
  转发已权威的应答。
- 它不取代去中心化：DoH 前端是*被运营的*服务
  —— 解析器运营者的选择、而非协议的。每个
  Scone 中继保持可经原生 UDP/TCP 独立使用。

### 未来的原生实现（已记录、未承诺）

原生 DoH 传输（`--doh ADDR`）仍可设想：线上
格式相同（RFC 8484 复用二进制 DNS 消息）、它
将是 POST/GET 解码 + TLS 代理（caddy/nginx 处理
证书）之后的 HTTP/2、节点本身保持无 TLS。无进
行中的工作 —— 需求将随最终用户而来。
