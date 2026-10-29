# CLI（`scone`）—— 日志与详细程度

> `scone` 二进制日志行为的规范。任何文档/代码
> 分歧 = bug（修复事实来源）。

## 原则

- **stdout 保留给命令输出**（`scone status`、
  `identity list`、`status` ……）。脚本可以重定向 stdout，
  而永远不会在里面发现日志 —— 管道保持可靠。公认的
  例外：`scone relay` 在监听器绑定后立即在
  stdout 上输出数据行
  `p2p: <multiaddr>`、`dns: <addr>` 和 `metrics: <addr>`
  （可用于 `--bootstrap` / 配置的数据，不是日志；
  各输出恰好一次，立即刷新）。
- **所有日志都走 stderr**（紧凑的 `tracing` 格式），包括
  中继的日志 —— 迁移之前中继就已经
  输出到 stderr 了。
- CLI 面向用户的错误消息（`scone: <error>`）留在
  stderr：这是 UX，不是日志。

## 级别与 `--verbose` / `-v`

| 调用方式 | 生效级别 |
|---|---|
| `scone <cmd>`（不带 `-v`） | `warn` |
| `scone relay`（不带 `-v`） | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …`（及更多） | `trace` |

`-v`/`--verbose` 是**全局且可重复**的标志：在子命令
之前或之后都被接受（`scone -vv identity list`
与 `scone identity list -vv` 等价）。

**为什么 `scone relay` 默认保持在 INFO**：中继是一个
守护进程 —— 它必须记录自己的活动（RPC/p2p 监听、
接受与产出的区块、连接的对等节点、拒绝事件）才能
被运维，而无需额外标志。相反，一次性命令保持
安静（WARN），以免污染终端或脚本。

## `RUST_LOG`（覆盖）

如果设置了 `RUST_LOG` 环境变量，它将**完全
取代**由 `-v` 推导出的级别（`env-filter` 语法）：

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## 环境配置（`.env`）

在解析参数之前，`scone` 从当前目录加载 `.env` 文件
（`main.rs` 中的 `dotenvy::dotenv()`）。文件
缺失 = 静默无操作，绝不是错误。加载的值成为
声明了环境变量的中继选项的基线
（`cli.rs` 中的 `env = "…"`）。

优先级（一个选项、三个来源）：

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- 命令行上传递的标志永远胜出；
- 否则使用变量（无论从 `.env` 加载还是设置在
  真实进程环境中 —— 同一个变量，同样的效果）；
- 否则使用 clap 的内建默认值。

可重复的变量（`SCONE_BOOTSTRAP`、`SCONE_DNS_UPSTREAM`）
会累积：`.env` 中每个值一行（dotenvy 追加）。

`SCONE_*` 变量的完整表格（网络、P2P 监听、
bootstrap、创世、UPnP、DNS 面、指标、拓扑
profile、锚节点）及各自的语义位于
[`relay.md`](relay.md#environment-configuration-env)；
带注释的模板是仓库根目录的 `.env.example`（复制
为 `.env`，绝不提交 —— `.env` 已被 gitignore）。

## 级别语义（代码约定）

- `error!`：失败（今日没有调用点 —— 网络失败被
  吞为 `warn!`，只有致命的本地失败才以
  `run()` 抛出的类型化错误呈现）；
- `warn!`：被拒之门外的恶意或敌意网络输入、
  从内存池驱逐的过期交易、出块期间丢弃的
  冲突交易、出站/入站失败、DHT 键不匹配；
- `info!`：区块接受/产出、RPC/p2p 监听、对等节点
  连接、中继启动（状态检查点）；
- `debug!`：内部细节（交易接受、对等节点断开、RPC
  往返、keystore 列出、`SCONE_KAD_DEBUG` 下的
  kad 事件）；
- `trace!`：保留给载荷（今日没有调用点）。

库 crate（`scone-network` ……）发出 `tracing` 宏但
**不初始化任何 subscriber**：初始化只存在于
`scone` 二进制中（在子命令分发之前 —— 中继从
启动起就需要它的日志）。


## 简单用法（cli-simplify-v1）

- **不需要 `--peer`**：命令默认与本地中继
  通信；`SCONE_PEER` 环境变量或 `--peer` 可覆盖。
- 当 keystore 只持有一个身份时**不需要
  `--identity`**（静默使用；多个 → 错误会
  列出它们）。
- 已移除（存根会提示替代品）：`show`、`submit`、
  `lookup`、`owned`。
- 新增：`scone domain renew <name>`、`scone domains`、
  `scone domain get|set`（单条记录，无文件）。

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLD 生命周期（cli-unified-lifecycle）

`scone tld register <tld>` 是一步到位的 TLD 入口，
与 `scone domain register` 对称 —— 先
读取链上状态，然后执行正确的动作：

- **空闲 TLD** → 注册，默认生而为**开放**（无标志：
  自助命名空间，1/3 的 PoW）。`--private` 以全额
  PoW 的价格选择自有命名空间，并带显式
  确认（经济决策）。`--yes` 跳过确认
  （脚本）。
- **属于你、90 天内到期** → 提议续期
  （绑定到当前到期时间的 PoW —— 防重用），确认后
  执行。
- **属于你、离到期尚远** → 无事可做（`tld renew --force`
  可强行续期）。
- **属于他人** → 带所有者的干净类型化错误。

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
scone tld renew mytld               # explicit renewal
```

同样地，对你拥有的、90 天内到期的名称执行
`scone domain register <name>` 会提议续期而不是失败。

## 记录委托（KSK/ZSK-lite）

域名的所有者可以授予**其他**密钥代表它签署
`domain update` 的权利 —— 记录委托集合（至多
3 把密钥，每次 `set` 全量替换）：

```bash
# 授权（由所有者签名；CLI 会获取命名空间的 CAS
# 世代并把交易绑定到它）：
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# 查看（无需身份）：
scone domain delegates list example.uip

# 吊销全部：
scone domain delegates clear example.uip --identity alice
```

委托随后签署 `domain update`，与所有者的操作完全
一样（`--identity deleg`）—— 链上交易的 `owner`
字段是**域名**的所有者，而签名来自委托。授权
是所有者独占的（委托无法管理该集合），会被
一次转移清空，且每次 `set`/`clear` 都使命名空间的
变异计数器递增（CAS 防重放 —— 见
`transactions.md`）：重放的 `set` 无法复活被吊销的
密钥。被吊销委托的更新在本地被拒绝（"既非
所有者也非记录委托"）。

## 高级命令

上面的默认值覆盖日常使用。面向脚本和 web/API
互操作，完整命令面仍然可用：

每个操作只有一条规范路径；以前的根级别别名（`show`、`lookup`、`owned`、`renew`）已被移除（remove-cli-aliases），从未真实存在的 `domain info` 提法也一并移除。

离线十六进制命令面（`tx build|sign|verify`、`submit tx`
子命令）已被移除（remove-tx-surface-v2）：它所服务的 M8b
操作如今是一等命令 ——
`scone tld open <tld>`（单向自助开放、为你
挖矿的开放 PoW）、`scone tld transfer <tld> <owner>`、
`scone tld revoke <tld>`、`scone domain assign <name> <owner>`
（TLD 所有者，私有 TLD）以及 `scone domain transfer <name>
<owner>`。每一条都是：读状态、签名、提交、txid。DNS 记录
解析移到了 `scone domain resolve <name>`（原 `record
get`）；十六进制发布通道（`record put`）已消失 ——
编辑通过 `domain set` / `domain update` 进行。
- `scone domain mine <name>` —— 只挖注册证明
  （无身份、无 keystore、不提交）：将
  `scone-domain-pow-v1` JSON `{type, name, network_id,
  epoch, proof}` 打印到 stdout。委托挖矿：另一台
  机器为你挖矿，你的密钥绝不离开你的机器。可以
  管道或重定向：`scone domain mine n.tld | scone domain
  register n.tld --proof -`（或 `--proof FILE`）。证明
  绑定名称与网络；新鲜度由节点在应用时强制
  执行（纪元有效窗口）。使用 `--mine-for <pubkey>`
  时，它改为为该受益者挖一个 POW-TO-OWNER 证明，
  并打印 `scone-domain-pow-to-v1` JSON `{type, name,
  network_id, epoch, pow_to, proof}` —— 只能由该确切
  密钥的 `register --to` / `renew --to` 消费（挑战
  内嵌该密钥；提交时不区分大小写校验）。
- Pow-to-owner 提交（无需 keystore —— `--to` 与
  `--identity` 互斥）：`scone domain register <name>
  --to <pubkey>` / `scone domain renew <name> --to
  <pubkey>` 以及 `scone tld register <tld> --to
  <pubkey>` / `scone tld renew <tld> --to <pubkey>`
  提交挑战绑定受益者密钥的证明；链上 `owner`
  重新计算为 `hash(pubkey_dest)`（续期绝不改变
  所有权）。不带 `--proof` 时，命令自己挖证明；
  带 `--proof -`/FILE 时消费
  `scone-domain-pow-to-v1` 载荷，并拒绝为其他
  受益者挖出的证明。管道风格：`scone domain
  mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`。
- `scone domain export <name> [--out FILE]` —— 以
  精确的 `update --file` 格式写出经链上校验的
  记录集（保证往返：export → 编辑 → update 重新提交
  同一集合）。不带 `--out`：stdout。`domain resolve` 使用
  相同的可读格式，因此 `resolve > zone.txt` 可以直接
  导入。没有文本语法的种类（GEOROUTE、未知类型）
  以注释形式的警告导出 —— 绝不静默丢弃。

`domain get` 还会打印 `domain_id` 和 `valid_until`。

## PoW 基准测试（`scone bench-pow`）

测量机器的 Argon2id 哈希率（与共识验证使用的
是同一个哈希），并把网络的难度换算成具体的
等待时间：

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

输出：总哈希率与每个 worker 的哈希率，以及每项
操作的预期平均时间 —— 转让/指派（固定费率）、
私有 TLD 开放，以及 TLD/域名注册下限（长度
曲线）。所选网络决定 Argon2id 参数（mainnet：
64 MiB、t=2 —— 真实成本；testnet/devnet：轻量
配置，毫秒级证明）。每个 worker 分配自己的
64 MiB Argon2id 通道 —— 请按可用内存调整
`--cores`。

## 构建：dev 与启用主网（mainnet-compile-gate）

默认构建**不含主网**：`scone relay --network
mainnet`（以及带 `SCONE_NETWORK=mainnet` 的注册服务）
立即失败并给出重新构建的提示。官方分发的
二进制以如下方式构建：

```bash
cargo build --release --features mainnet
```

`scone --version` 在此类构建上显示 `+mainnet`，
中继启动日志显示 `build: mainnet-enabled`。

默认的 `--network` 也随构建而定：在启用主网的
构建上是 `mainnet`（官方二进制让用户落在
生产网络上），否则为 `testnet`。显式的
`--network` 标志或 `SCONE_NETWORK` 环境变量永远
胜出（clap 顺序：标志 > 环境变量 > 默认值）。这是
一个运维护栏（误运行、dev/CI 二进制）—— 不是
密码学边界；针对恶意节点的防线仍然是
共识规则。
