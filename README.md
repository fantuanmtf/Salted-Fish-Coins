# Salted Fish Coins (咸鱼币)

Salted Fish Coins is a cryptocurrency project based on the Monero codebase.

这是一个基于 [Monero](https://github.com/monero-project/monero) 源码的加密货币项目，
目前运行在自己的开发网络 **snacknet** 上（名称是临时的，后续会改）。

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

- New network type `SNACKNET` (own address prefixes `99/100/101`, own P2P/RPC
  ports `48080/48081/48082`, own network ID, own genesis block, data directory
  `~/.saltedfish/snacknet`)
- Own hard fork schedule (`snacknet_hard_forks`: v1 at the genesis block, v16
  from block 2, so the chain starts on the current protocol version instead of
  replaying years of historical forks)
- No hardcoded checkpoints on snacknet (upstream mainnet/testnet/stagenet
  checkpoints are not applied, neither on the daemon nor in the wallet)
- No seed nodes and no DNS checkpoints on snacknet: nodes only connect to peers
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

## Running snacknet

Create a wallet and note the address it prints:

```sh
./build/bin/saltedfish-wallet-cli --snacknet --generate-new-wallet ~/my-sfc-wallet \
    --password "your password" --offline
```

Start a node and mine to that address:

```sh
./build/bin/saltedfishd --snacknet --offline --fixed-difficulty 1 \
    --start-mining <your snacknet address> --mining-threads 1 \
    --data-dir ~/.saltedfish/snacknet
```

Check the chain height:

```sh
curl -s http://127.0.0.1:48081/json_rpc -d \
  '{"jsonrpc":"2.0","id":"0","method":"get_info"}' | python3 -m json.tool
```

Then open the wallet, let it scan and check the balance:

```sh
./build/bin/saltedfish-wallet-cli --snacknet --wallet-file ~/my-sfc-wallet \
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

## Status

Early development. snacknet is a private/development network: there are no
public seed nodes yet, and the genesis block currently uses a placeholder
coinbase transaction (the genesis allocation is not spendable by the project).
