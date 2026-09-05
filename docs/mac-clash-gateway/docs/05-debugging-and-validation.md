# 调试与验证

## 分层排查顺序

按热点、Mihomo TUN、主 PF、子规则、state、数据面的顺序排查。前一层不成立时，不要通过反复 apply 掩盖问题。

## 测试一 热点和 DHCP

目的：确认故障是否发生在 Apple Internet Sharing，而不是代理层。

```bash
ifconfig bridge100
arp -an -i bridge100
```

预期：`bridge100` 存在且有 `192.168.2.1`；客户端获得 `192.168.2.x`，路由器为 `192.168.2.1`。

异常解释：接口不存在表示热点未建立；地址不同表示固定参数不匹配；客户端无地址通常应先检查 Internet Sharing 和 DHCP，不要先加载 PF 转发。

## 测试二 Mihomo 和 TUN

目的：确认代理进程存在，并找到当前 TUN，不依赖历史接口名。

```bash
pgrep -x verge-mihomo
ifconfig | grep -B2 -A3 '198\.18\.0\.1'
```

预期：进程存在，且恰好一个 `utunX` 拥有 `198.18.0.1`。

异常解释：没有结果时 watcher 应 `WAIT` 或 `DISABLED`；多个接口时脚本拒绝 apply，避免选错路径。

## 测试三 主 PF 挂载点

目的：判断 named anchor 是否真的连接到数据路径。

```bash
sudo pfctl -sr -vv | grep -E 'clash-forward|com\.apple\.internet-sharing'
sudo grep -n 'clash-forward' /etc/pf.conf
```

预期：内核主规则和磁盘文件都有 `anchor "clash-forward"`，位置在 Apple filter anchor 之前。

异常解释：子规则存在但主挂载点缺失时，子规则计数会一直为 0。先修复主文件并安排重启验收，不要反复加载子 anchor。

## 测试四 刷新预检查

目的：一次回答主 anchor、热点、进程、TUN 和语法是否全部满足。

```bash
sudo /usr/local/sbin/clash-forward-refresh --check
```

预期：`CHECK OK`。

异常解释：

- 10：主 anchor 缺失。
- 11：`bridge100/192.168.2.1` 未就绪。
- 12：`verge-mihomo` 未运行。
- 13：`198.18.0.1` 对应接口数量不是 1。
- 14：接口名不是 `utun` 加数字。
- 15：PF 模板语法或宏验证失败。

## 测试五 规则命中和 state

目的：区分“规则没有加载”“规则未被调用”和“旧 state 绕过新规则”。

```bash
sudo pfctl -a clash-forward -s rules -v
sudo pfctl -ss -vv | grep -B2 -A6 '192\.168\.2\.' | head -150
```

预期：两条规则存在；手机访问后 Packets、Bytes 和 States 增长。

异常解释：规则为空表示 fail-open 或尚未 apply；规则存在且计数始终为 0，检查主 anchor 和旧 state；只有 UDP 为 0 可能只是应用未使用 QUIC，不单独判失败。

## 测试六 DHCP 和 mDNS 不进入 TUN

目的：排除原方案最危险的回归。

```bash
TUN_IF="$(ifconfig | awk '/^[a-zA-Z0-9_.-]+:/{i=$1;sub(":","",i)} $1=="inet" && $2=="198.18.0.1"{print i; exit}')"
sudo tcpdump -ni "$TUN_IF" -vvv 'udp port 67 or udp port 68 or udp port 5353'
```

预期：手机正常使用约 10 秒后 `0 packets captured`。

异常解释：出现 DHCP 或 mDNS 包说明 bypass 或匹配边界失效，应立即 `--disable`。不要把 `packets received by filter` 误读为捕获结果。

## 测试七 数据面链路

目的：直接证明热点客户端流量进入动态发现的 TUN。

```bash
sudo tcpdump -ni "$TUN_IF" -c 10 -vvv 'net 192.168.2.0/24 and (tcp or udp)'
```

预期：出现 `192.168.2.x` 的双向 TCP 或 UDP 包；Fake-IP 模式下看到 `198.18.x.x` 目标是正常现象。

异常解释：PF 计数增长但 TUN 无包时，应检查抓包接口是否仍是当前 TUN；两者都无包则回到主 anchor 和 state 排查。

## 测试八 Clash 生命周期自愈

目的：证明 TUN 消失不会让热点整体中断，并在恢复后自动接管。

步骤：退出 Clash，等 10 至 20 秒，确认日志出现 `DISABLING`、`DISABLED` 或 `WAIT`；确认 anchor 为空，国内普通网络可用。重新启动 Clash，不执行人工 apply，确认日志出现 `APPLYING` 和 `ACTIVE`，国内外网络恢复。

异常解释：TUN 消失后仍保留 `route-to` 是严重故障，应人工 `--disable`；恢复后停在 `ERROR-HOLD` 时读取预检查输出和错误日志，修正原因后重启 watcher。

## 测试九 Internet Sharing 生命周期自愈

目的：证明 `bridge100` 重建不会留下指向失效拓扑的规则。

步骤：保持 Clash 运行，关闭 Internet Sharing，确认 `bridge100` 消失且 anchor 清空；重新打开热点，不执行人工 apply，确认 watcher 自动 `ACTIVE`，手机取得动态地址，计数增长。

## 测试十 整机重启

目的：同时验证 `/etc/pf.conf` 持久化、system LaunchDaemon、启动时序等待和数据面恢复。

重启后禁止先运行 `--apply`、`pfctl -f` 或手工清 state。检查：

```bash
sudo launchctl print system/local.clash-forward-watch | grep -E 'state =|pid =|runs =|last exit'
sudo pfctl -sr -vv
ifconfig bridge100
ps aux | grep -i '[m]ihomo'
ifconfig | grep -B2 -A3 '198\.18\.0\.1'
sudo tail -80 /var/log/clash-forward-watch.log
sudo pfctl -a clash-forward -s rules -v
```

预期：daemon 运行；主 anchor 存在；热点和 TUN 就绪或 watcher 先等待；最终 `ACTIVE`；手机三类网站可用；安全抓包为 0；数据面抓包为双向。

## 快速故障矩阵

| 现象 | 最可能层次 | 首个检查 |
| --- | --- | --- |
| 手机拿不到 `192.168.2.x` | Internet Sharing 或 DHCP | `ifconfig bridge100` |
| 国内可用 国外不可用 anchor 为空 | 正常 fail-open | watcher 日志和 Mihomo |
| 规则存在但计数为 0 | 主挂载点或旧 state | `pfctl -sr -vv` 和 `pfctl -ss -vv` |
| Clash 退出后手机全部断网 | fail-open 失败或 Apple 热点故障 | 立即 `--disable` |
| 日志每 10 秒都 apply | 幂等判断失败 | `rules_match_tun` 和实际规则文本 |
| `ERROR-HOLD` | 同一拓扑 apply 失败 | refresh `--check` 和错误日志 |
| DHCP 或 mDNS 出现在 TUN | bypass 回归 | 立即 `--disable` 并核对 anchor 文件 |
