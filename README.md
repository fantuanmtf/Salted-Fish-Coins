# Salted Fish Coins (咸鱼币)

Salted Fish Coins is a cryptocurrency project based on the Monero codebase.

这是一个基于 [Monero](https://github.com/monero-project/monero) 源码的加密货币项目。
其正式网络（主网）名为 **zakonet**，目前处于早期实验阶段。

- 不想自己跑节点、只想用钱包 → 公共远程节点（仅 Tor）：
  [`docs/REMOTE-NODE.md`](docs/REMOTE-NODE.md)
  （`imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:8100`）
- 想自己同步链、跑节点、做种子节点 → [`docs/SEED-NODE-GUIDE.md`](docs/SEED-NODE-GUIDE.md)
- 想开一条测试链做实验 → [`docs/TEST-NETWORK.md`](docs/TEST-NETWORK.md)

## Fork declaration / Fork 声明

This project is a **fork of Monero**. It is not affiliated with, endorsed by, or
supported by The Monero Project.

- Upstream: <https://github.com/monero-project/monero>
- Forked from commit: `24272e4a3b2e812cc2a569d95b177ecef4b91e46` (2026-10-08)
- Upstream license: BSD 3-Clause, Copyright (c) 2014-2024, The Monero Project —
  see [`LICENSE-MONERO`](LICENSE-MONERO)
- This fork's license: BSD 3-Clause, Copyright (c) 2026, 饭团 —
  see [`LICENSE-SALTED-FISH-COINS`](LICENSE-SALTED-FISH-COINS)
- Upstream README: [`README-MONERO-UPSTREAM.md`](README-MONERO-UPSTREAM.md)

All original copyright notices in the source files are retained, as required by
the BSD 3-Clause license.

### What is changed relative to upstream Monero

- New network type `ZAKONET` (own address prefixes `99/100/101`, own P2P/RPC
  ports `48080/48081/48082`, own network ID, own genesis block, data directory
  `~/.saltedfish/zakonet`)
- Own hard fork schedule (`zakonet_hard_forks`: v1 at the genesis block, v16
  from block 2, so the chain starts on the current protocol version instead of
  replaying years of historical forks)
- No hardcoded checkpoints on zakonet (upstream mainnet/testnet/stagenet
  checkpoints are not applied, neither on the daemon nor in the wallet)
- No seed nodes and no DNS checkpoints on zakonet: nodes only connect to peers
  you add explicitly with `--add-peer`
- Rebranded binaries (`saltedfishd`, `saltedfish-wallet-cli`, …) and data
  directory (`~/.saltedfish`)

## Building

```sh
git clone git@github.com:fantuanmtf/Salted-Fish-Coins.git
cd Salted-Fish-Coins
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc) --target daemon simplewallet wallet_rpc_server
```

The binaries end up in `build/bin/`.

## Running zakonet

Create a wallet and note the address it prints:

```sh
./build/bin/saltedfish-wallet-cli --zakonet --generate-new-wallet ~/my-sfc-wallet \
    --password "your password" --offline
```

Start a node and mine to that address:

```sh
./build/bin/saltedfishd --zakonet --offline --fixed-difficulty 1 \
    --start-mining <your zakonet address> --mining-threads 1 \
    --data-dir ~/.saltedfish/zakonet
```

Check the chain height:

```sh
curl -s http://127.0.0.1:48081/json_rpc -d \
  '{"jsonrpc":"2.0","id":"0","method":"get_info"}' | python3 -m json.tool
```

Then open the wallet, let it scan and check the balance:

```sh
./build/bin/saltedfish-wallet-cli --zakonet --wallet-file ~/my-sfc-wallet \
    --daemon-address 127.0.0.1:48081 --trusted-daemon
[wallet]: refresh
[wallet]: balance
```

Notes:

- On a brand new chain (`--fixed-difficulty 1`) blocks are produced almost
  instantly. Stop the miner with `curl -s http://127.0.0.1:48081/stop_mining -d '{}'`
  before the chain grows unboundedly, or drop `--fixed-difficulty` to use the
  normal difficulty algorithm.
- RandomX mining wants ~2.5 GB of RAM per mining thread when the full dataset is
  enabled (`MONERO_RANDOMX_FULL_MEM=1`). Without it, mining falls back to the
  slow light mode, which still works.

## 引导节点（bootstrap seed）

zakonet 目前没有 DNS 种子，新节点靠下面这个**临时引导节点**入网。它只通过 Tor
提供 P2P，服务器 IP 不会暴露：

```
imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:48080
```

连接方式（本机需要跑 Tor，SOCKS 端口默认 9050）：

```sh
./build/bin/saltedfishd --zakonet \
    --add-peer imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:48080 \
    --tx-proxy tor,127.0.0.1:9050,10
```

也可以把这个地址写进 `~/.saltedfish/zakonet/saltedfish.conf`：

```
add-peer=imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:48080
tx-proxy=tor,127.0.0.1:9050,10
```

**重要（链同步）**：按 Monero 的设计，洋葱连接只能做节点发现与交易广播，**不能同步区块链**
（上游刻意过滤了区块相关 P2P 命令，本项目实测亦然）。新节点首次拿到链需要其它方式——从运营者
拷贝链数据、或走非洋葱通道，详见 [`docs/SEED-NODE-GUIDE.md`](docs/SEED-NODE-GUIDE.md) 第 1 节。

## 支持这个项目 / Buy Me a Coffee

如果这个项目对你有用，欢迎请作者喝杯咖啡：

- Monero (XMR)：
  `4B6wjJUUVwyARsbMd174PN6ttc4LkBC2qarKDA881R1F3pEUMxcWx8Ta1FzAGvcoGPhE`

（zakonet 的 SFC 还在开发中、没有价值，暂不收 SFC 打赏。）

## Status

Early development —— 实验性主网（zakonet 已上线运行，2026-10-12）：

- 官方种子节点（Tor 洋葱，见上）+ 公共远程节点（同一个洋葱的 `:8100` 端口，仅 Tor，见
  [`docs/REMOTE-NODE.md`](docs/REMOTE-NODE.md)）；
- 出块、钱包、链上转账均已实测通过（含三个钱包经由同一节点转账的冒烟测试）；
- 链上包含少量早期测试产出（含运营者保留用于后续测试的预挖），全部公开可查；
- 创世块使用占位 coinbase 交易（创世分配不可花费）。

注意：本网络有几处与上游行为不同的已知点——

1. **通过洋葱连接不能同步区块链**（上游设计，实测亦然）。新节点首次同步见
   [`docs/SEED-NODE-GUIDE.md`](docs/SEED-NODE-GUIDE.md) 第 1 节；
2. 本 fork 让节点直接打包池内交易（不依赖 Dandelion++ 的 stem→fluff 广播——小网络里该
   广播不可靠，会让交易永远无法确认）。实现见 `src/cryptonote_core/cryptonote_core.cpp`
   与 `src/cryptonote_core/tx_pool.cpp` 中 `mine_stem_txes` 相关改动。
