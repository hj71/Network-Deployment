# 架构设计

## 数据路径

```text
热点客户端 192.168.2.x
          |
          v
bridge100 192.168.2.1
          |
          v
PF 主规则集
          |
          +--> 本地 私网 组播 保留地址 --> Apple Internet Sharing --> <上游接口>
          |
          +--> 公网 TCP UDP --> clash-forward quick route-to
                                      |
                                      v
                         动态 utunX 198.18.0.1
                                      |
                                      v
                              Mihomo 和代理规则
```

Apple Internet Sharing 仍拥有 DHCP 和 NAT。`clash-forward` 只改变从 `bridge100` 进入、来源属于 `192.168.2.0/24`、目标不在 bypass 表中的 IPv4 TCP 和 UDP 新连接。

## 控制路径

```text
launchd
  └── clash-forward-watch
        ├── 检查主 anchor
        ├── 检查 bridge100 和 192.168.2.1
        ├── 检查 verge-mihomo
        ├── 用 198.18.0.1 定位唯一 utunX
        ├── 检查当前规则是否对应这个 utunX
        ├── 条件满足且规则缺失或过期 -> refresh --apply
        └── 条件不满足且规则存在 -> refresh --disable
```

## 五个冻结文件的职责

| 文件 | 职责 | 设计边界 |
| --- | --- | --- |
| `/etc/pf.conf` | 建立主规则调用点 | 只增加 `anchor "clash-forward"`，不静态加载子规则 |
| `/etc/pf.anchors/clash-forward` | 保存 PF 规则模板 | `tun_if` 必须通过 `-D` 注入 |
| `/usr/local/sbin/clash-forward-refresh` | 检查、加载、撤销规则和清理 state | 单次事务，带锁和 fail-open |
| `/usr/local/sbin/clash-forward-watch` | 对齐实际拓扑与 PF 状态 | 稳定状态不重复 apply |
| `/Library/LaunchDaemons/local.clash-forward-watch.plist` | 让 watcher 在 system domain 常驻 | `KeepAlive`，日志写入 `/var/log` |

## bypass 表

`clash_bypass` 排除以下目标：

- `0.0.0.0/8` 和 `127.0.0.0/8`
- RFC1918 私网 `10/8`、`172.16/12`、`192.168/16`
- 运营商级 NAT `100.64/10`
- 链路本地 `169.254/16`
- 组播 `224/4`
- 保留地址 `240/4`

DHCP 使用广播和本地链路语义，mDNS 使用组播；这些目标不会命中公网转发规则。

## fail-open 状态机

| 条件 | watcher 行为 | 手机结果 |
| --- | --- | --- |
| 热点、Mihomo、唯一 TUN 和正确规则都存在 | `ACTIVE`，不修改 state | 国内外按 Mihomo 规则访问 |
| 依赖存在但规则缺失或 TUN 已变化 | 预检查后 `--apply` | 短暂重建连接后恢复代理 |
| Clash、TUN 或热点消失 | 有规则时 `--disable`，无规则时 `WAIT` | 退回普通 Internet Sharing |
| 同一拓扑的 apply 已失败 | `ERROR-HOLD` | 避免每 10 秒反复清 state |

## state 清理原则

加载或撤销 `route-to` 后执行：

```bash
sudo pfctl -k 192.168.2.0/24
```

它只清理以热点网段为来源的 PF state，使客户端新连接重新匹配当前规则。现有连接会中断并重建，这是预期副作用。稳定状态绝不执行这条命令。
