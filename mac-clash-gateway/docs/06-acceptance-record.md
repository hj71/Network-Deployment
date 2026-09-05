# P1 至 P4 验收记录

## 验收范围

验收在 2026-09-04 至 2026-09-05 完成。运行时观察到的 `<运行时 utunX>`、客户端 `<客户端 DHCP 地址>` 和 PID 只属于证据，不属于正式逻辑。

## P1 主 PF anchor 持久化

状态：PASS。

证据：

- `/etc/pf.conf` 和原 anchor 已备份。
- diff 仅在 Apple filter anchor 之前增加 `anchor "clash-forward"`。
- 临时文件和正式 `/etc/pf.conf` 均通过 `pfctl -n -f`。
- P1 没有用 `pfctl -f /etc/pf.conf` 刷新运行中的 Apple ruleset。
- P4 重启后主 ruleset 再次出现 `anchor "clash-forward"`，插入者为启动阶段进程。

## P2 动态规则刷新

状态：PASS。

证据：

- 正式模板限制 `bridge100`、`192.168.2.0/24`、IPv4 TCP 和 UDP，并排除 bypass 表。
- `--check` 同时确认主 anchor、`bridge100=192.168.2.1`、Mihomo、动态 TUN 和 PF 语法。
- `--apply` 动态识别到一次运行的 `<运行时 utunX>`，加载规则并执行全网段 state 清理。
- 一次实测清理了 49 条 state；正式脚本加载后计数增长到 TCP 1342 packets 和 41 states，UDP 2673 packets 和 34 states。
- TUN 上 DHCP 67/68 与 mDNS 5353 抓包为 0。

## P3 LaunchDaemon 和自愈

状态：PASS。

LaunchDaemon 证据：

- plist 通过 `plutil -lint`。
- 文件权限为 `root:wheel` 和 `0644`。
- `launchctl print` 显示 `type = LaunchDaemon`、`state = running`、程序路径正确、`KeepAlive` 生效。
- 标准错误日志为空。

Clash 生命周期证据：

- TUN 消失后日志出现 `DISABLING`、`DISABLED` 和 `WAIT`；anchor 为空，普通国内网络仍可用。
- Clash 恢复后 watcher 自动 `APPLYING` 和 `ACTIVE`，一次清理 92 条热点 state。
- 未人工加载 PF 时，规则计数恢复到 TCP 184 packets 和 7 states；手机国内、Google、YouTube 均可访问。

Internet Sharing 生命周期证据：

- 关闭热点后 `bridge100` 不存在，watcher 撤销规则。
- 重开后 watcher 自动 apply，并清理 3 条 state。
- 数据恢复后 TCP 22076 packets 和 55 states，UDP 4820 packets 和 31 states。
- DHCP 和 mDNS TUN 抓包继续为 0。

## P4 整机重启

状态：PASS。

基础设施证据：

- 重启后 LaunchDaemon `state = running`，`runs = 1`，无需人工 bootstrap 或 kickstart。
- 主 PF ruleset 从磁盘恢复 `anchor "clash-forward"`。
- `bridge100` 自动恢复 `192.168.2.1/24`，成员和接口处于 active。
- Mihomo 和 `198.18.0.1` TUN 自动出现。
- watcher 先记录 `WAIT: bridge100/192.168.2.1 not ready`，随后自动 `APPLYING`、`APPLY OK` 和 `ACTIVE`。

安全和数据面证据：

- 重启后 DHCP 67/68 和 mDNS 5353 在 TUN 上为 `0 packets captured`。
- 手机产生流量后，TCP 达到 35097 packets、28423258 bytes、30 states；UDP 达到 1018 packets、818441 bytes、7 states。
- 动态 TUN 抓包观察到热点客户端与 Mihomo Fake-IP 目标之间的双向 TCP 包。
- 手机关闭蜂窝网络后，国内网站、Google 和 YouTube 均正常。

## 最终验收矩阵

| 项目 | 结果 |
| --- | --- |
| Apple Internet Sharing 和热点 | PASS |
| `bridge100 = 192.168.2.1/24` | PASS |
| 客户端 DHCP 和 Apple NAT | PASS |
| 主 PF anchor 持久化 | PASS |
| 独立 `clash-forward` anchor | PASS |
| `quick route-to` TCP 和 UDP | PASS |
| 本地地址 DHCP 和 mDNS bypass | PASS |
| 动态 `198.18.0.1` 到 `utunX` | PASS |
| 客户端地址不写死 | PASS |
| 全网段动态 state 清理 | PASS |
| Clash 退出 fail-open 和重启恢复 | PASS |
| 热点关闭撤规则和重开恢复 | PASS |
| LaunchDaemon 开机运行 | PASS |
| Mac 重启无人工修复恢复 | PASS |
| TUN 双向手机流量 | PASS |

结论：Production Baseline v1.0 已正式冻结。
