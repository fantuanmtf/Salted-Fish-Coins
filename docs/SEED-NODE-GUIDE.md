# 附录：同步区块链 & 自建节点 / 种子节点

## 0. 准备二进制

从源码构建（见仓库 README），网络参数一律加 `--zakonet`。节点默认数据目录 `~/.saltedfish/zakonet`。

## 1. 先解决「链从哪来」（重要，先读这段）

Salted Fish Coins 继承 Monero 的 P2P 设计：**通过 Tor 洋葱 / I2P 连接不能同步区块链**。
匿名网络连接只放行三种消息——握手、定时同步、交易广播——区块相关命令会被 P2P 层的过滤器
拦掉：

- 源码：`src/p2p/net_node.cpp` 的 `is_filtered_command()`（非 public 区只放行上述三类）；
- 源码：`src/cryptonote_protocol/cryptonote_protocol_handler.inl` 中
  `// No chain synchronization over hidden networks (tor, i2p, etc.)`；
- 上游文档：[docs/ANONYMITY_NETWORKS.md](https://github.com/monero-project/monero/blob/master/docs/ANONYMITY_NETWORKS.md)。

这是上游的刻意设计（防女巫/带宽考量），不是本项目的 bug。所以新节点的**第一次链同步**需要
以下方式之一：

1. **直接拷贝链数据**（当前最简单，链还很小）：向节点运营者获取数据目录（LMDB），解包到
   `~/.saltedfish/zakonet/`，确认结构为 `zakonet/lmdb/data.mdb`。
2. **走非洋葱通道**：如果你有官方节点的登录权限，可开 SSH 隧道把它的 P2P 端口转到本地：

       ssh -L 127.0.0.1:48090:127.0.0.1:48080 user@服务器

   然后让节点用 `--add-priority-node 127.0.0.1:48090 --allow-local-ip` 连它——隧道属于
   「公网区」（public zone），同步不受限制。
3. 未来版本可能放开匿名网络同步（需修改上述两处过滤器并让全网升级，尚未实施）。

同步完之后，节点就可以长期通过洋葱与网络保持连接了（交易广播/节点发现不受限制）。

## 2. 运行普通节点

    saltedfishd --zakonet --data-dir ~/.saltedfish/zakonet \
      --add-peer imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:48080 \
      --tx-proxy tor,127.0.0.1:9050,10

`--tx-proxy tor,127.0.0.1:9050,10` 表示经本机 Tor 的 SOCKS 端口连洋葱对端。

## 3. 做一个种子节点（自己的洋葱地址，隐藏 IP）

1. 安装 tor，在 `/etc/tor/torrc` 加：

       HiddenServiceDir /var/lib/tor/zakonet/
       HiddenServicePort 48080 127.0.0.1:48084

   重启 tor，`cat /var/lib/tor/zakonet/hostname` 就是你的 .onion 地址。

2. 启动节点，**关键参数是 `--anonymous-inbound`**：

       saltedfishd --zakonet --data-dir /var/lib/zakonet \
         --p2p-bind-ip 127.0.0.1 --p2p-bind-port 48080 \
         --anonymous-inbound $(cat /var/lib/tor/zakonet/hostname):48080,127.0.0.1:48084 \
         --tx-proxy tor,127.0.0.1:9050,10 --hide-my-port --non-interactive

   不写 `--anonymous-inbound` 的后果：tor 转发进来的连接会被当成普通（clearnet）连接，
   对端一共享 tor 区 peerlist 就会被 `sent peerlist from another zone` 掐线——每个连接只
   活 15~75 秒，种子等于废的。加上它，节点才知道「从 127.0.0.1:48084 进来的是 tor 区对端」。

3. 把 `你的onion:48080` 分享出去；别人用
   `--add-peer 你的onion:48080 --tx-proxy tor,127.0.0.1:9050,10` 连你。

## 4. 开机自启（systemd 示例）

    # /etc/systemd/system/zakonet-node.service
    [Unit]
    Description=zakonet node (tor hidden-service inbound)
    After=network-online.target tor.service
    Wants=network-online.target tor.service

    [Service]
    User=你的用户名
    ExecStart=/usr/local/bin/saltedfishd --zakonet --data-dir /var/lib/zakonet \
      --p2p-bind-ip 127.0.0.1 --p2p-bind-port 48080 \
      --anonymous-inbound 你的onion.onion:48080,127.0.0.1:48084 \
      --tx-proxy tor,127.0.0.1:9050,10 --hide-my-port --non-interactive
    Restart=always
    RestartSec=10

    [Install]
    WantedBy=multi-user.target

tor 本身用系统服务（`tor.service`）即可；确保 `HiddenServicePort` 指向上面的 127.0.0.1:48084。

## 5.（可选）对钱包用户提供公共 RPC

把**只读受限 RPC**开到公网（**不要**把完整 RPC 暴露公网——它含管理命令且无认证）：

    --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port <端口> --confirm-external-bind

之后别人可以 `saltedfish-wallet-cli --zakonet --daemon-address 你的IP:<端口>`，用法见
[REMOTE-NODE.md](REMOTE-NODE.md)。
