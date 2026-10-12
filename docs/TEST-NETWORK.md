# 附录：创建测试网 / 测试链

主网（zakonet）之外，想在不动真链的前提下做实验，有两条路。

## A. regtest（单机即时链，最快）

saltedfishd 支持 Monero 的 regtest（fakechain）模式：单机运行、无需对等节点、随叫随出块。

    saltedfishd --regtest --offline --fixed-difficulty 1 \
      --data-dir /tmp/sfc-regtest --non-interactive --hide-my-port

（`--offline` 是单机 regtest 的关键：`generateblocks` 要求节点自报"已同步"，而开着网络却
没有对等节点的节点永远到不了那个状态；`--offline` 直接标记就绪。）

出块用 regtest 专属的 `generateblocks` RPC（`--start-mining` 在 fakechain 上不触发，
见下）：

    curl -s http://127.0.0.1:18081/json_rpc -d \
      '{"jsonrpc":"2.0","id":"0","method":"generateblocks",
        "params":{"amount_of_blocks":1,"wallet_address":"<4 开头的主网格式地址>","starting_nonce":0}}'

也可以直接 `saltedfishd` 交互命令行里执行 `generateblocks 1 <地址>`。

要点：

- regtest 只接受**主网 / 硬网格式的地址**（前缀 18/19/42）。zakonet 的地址（前缀 99）会被
  `Wrong address prefix` 拒绝——随便生成一个普通钱包地址即可，链上币没有意义。
- `--fixed-difficulty 1` 让任意哈希都能出块；去掉则走正常难度调整。
- **已知限制（本分支实测）**：`--start-mining` 与 `start_mining` RPC 在 fakechain 上不会
  真正开始挖矿（上游遗留；`core::on_synchronized()` 在 fakechain 上没有走到挖矿启动路径）。
  regtest 上请改用 `generateblocks`（本项目的功能测试正是这么跑的，出块正常）。
  想长时间持续挖矿、或要一条别人也能连的链，用下面的自定义测试网（见 B）。

## B. 独立测试网（自定义创世块，可对外）

想要一条「有自己的创世块、自己的网络 ID、别人也能连」的测试链：

1. 在 `src/cryptonote_config.h` 新增一个网络段（照抄 `namespace zakonet` 的写法）：
   - 自己的地址前缀、端口、`NETWORK_ID`（独一无二的 UUID）；
   - 自己的 `GENESIS_TX` / `GENESIS_NONCE`；
   - 在 `get_config()` 的 switch 里注册。
2. 在 `src/hardforks/hardforks.cpp` 为该网络写硬分叉表，版本号与代码支持的版本对齐
   （参考 `mainnet_hard_forks[]` 表尾的最新版本；本项目 zakonet 的做法是
   `{1, 1, 0, <时间>}, {<最新版本>, 2, 0, <时间>}`）。
3. 编译后用它自己的网络参数起节点，`--fixed-difficulty 1` 挖出第一批块。
4. 想让别人连：按 [SEED-NODE-GUIDE.md](SEED-NODE-GUIDE.md) 做洋葱种子，或直接暴露
   P2P / 受限 RPC 端口。

（备注：zakonet 网络本身就是从 Monero stagenet 的创世改造出来的，流程一模一样。）
