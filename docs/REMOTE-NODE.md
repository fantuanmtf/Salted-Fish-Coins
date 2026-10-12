# 使用公共远程节点（Remote Node）

不想自己同步区块链？可以直接把自己的钱包连到我们运行的公共节点上。

**节点只通过 Tor 洋葱访问**（服务器 IP 不暴露）。本机需要能连 Tor（SOCKS 端口默认 9050）。

## 节点信息

| 项目 | 值 |
|---|---|
| 公共 RPC（只读受限模式） | `imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:8100` |
| P2P 引导种子（同一个洋葱） | `imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:48080` |
| 网络参数 | `--zakonet` |

## 钱包怎么连

命令行钱包（经本机 Tor 的 SOCKS 代理）：

    saltedfish-wallet-cli --zakonet \
      --daemon-address imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:8100 \
      --proxy 127.0.0.1:9050

程序集成（wallet-rpc）同理加 `--proxy 127.0.0.1:9050`：

    saltedfish-wallet-rpc --zakonet \
      --daemon-address imbl4uqckegdurnznxn2o74yq4pxw7ftylatymqcbtgfo5qihvd7ojyd.onion:8100 \
      --proxy 127.0.0.1:9050 --rpc-bind-port 48083 --wallet-dir ~/wallets

连接后先 `refresh` 一次即可看到余额。（已实测：钱包经该洋葱 + `--proxy` 刷新正常。）

## 这个节点能做什么 / 不能做什么

- **能**：查询链高度与区块、扫描并同步钱包、构造与广播交易（日常钱包功能全都能用）。
- **不能**：挖矿控制、连接管理、日志级别等管理命令——节点以 restricted（只读受限）模式对外，
  和 Monero 生态的公共节点是一样的做法。
- **隐私提醒**：用别人的远程节点时，节点运营者能看到你的扫描进度（看不到你的私钥与具体转账
  ——钱包扫描与签名都在本地完成）；但经 Tor 连接时连你的 IP 都看不到。对隐私敏感请自己跑
  节点，见 [SEED-NODE-GUIDE.md](SEED-NODE-GUIDE.md)。

## 常见问题

- **连不上 / 超时**：确认本机 Tor 正常（`127.0.0.1:9050` 可达）、`--proxy` 参数已加；
  节点偶尔维护重启，稍等重试。洋葱端口是首次连接时较慢属正常（Tor 建链）。
- **余额没有变化**：先执行 `refresh`；收到的转账要 10 个块后才能花（挖矿产出的币要等 60 块）。
- **想自己做一个同样的公共节点**：见 [SEED-NODE-GUIDE.md](SEED-NODE-GUIDE.md)。
