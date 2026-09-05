# 新 Mac 迁移和升级

## 新 Mac 快速复现清单

1. 记录旧 Mac 的五个冻结文件、文件权限和校验和。
2. 在新 Mac 安装并验证 Clash Verge Rev，不复制历史 `utunX`。
3. 建立 Internet Sharing，确认新机是否仍使用 `bridge100 = 192.168.2.1/24`。
4. 确认上游默认路由。`<上游接口>/<上游网关>` 是旧环境验收值，不是脚本依赖。
5. 确认 Mihomo TUN 仍使用 `198.18.0.1`，且仅对应一个 `utunX`。
6. 备份新系统自带 `/etc/pf.conf`，只插入项目 anchor。
7. 安装规则、refresh、watcher 和 plist，核对 owner 与 mode。
8. 依次完成 P1、P2、P3 和 P4，不跳过整机重启。

## 迁移前采集

```bash
sudo shasum -a 256 /etc/pf.conf /etc/pf.anchors/clash-forward /usr/local/sbin/clash-forward-refresh /usr/local/sbin/clash-forward-watch /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo ls -l /etc/pf.conf /etc/pf.anchors/clash-forward /usr/local/sbin/clash-forward-refresh /usr/local/sbin/clash-forward-watch /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl print system/local.clash-forward-watch
sudo pfctl -sr -vv
```

公开仓库时，删除用户名、MAC 地址、PID、访问目标和不必要的抓包内容。

## 新机必须重新确认的条件

| 条件 | 可否沿用 | 处理 |
| --- | --- | --- |
| `bridge100` | 需验证 | 不同则修改脚本和规则后重新全验收 |
| `192.168.2.1/24` | 需验证 | 不同则同步修改热点 IP 和网段 |
| `198.18.0.1` | 需验证 | 不同则修改 TUN 标记和 route-to 网关 |
| `utunX` | 不可沿用 | 始终动态发现 |
| 客户端 `192.168.2.x` | 不可沿用 | 规则使用网段 |
| 上游 `<上游接口>` | 只作记录 | Apple NAT 管理，本脚本不引用 |
| 上游网关 `<上游网关>` | 只作记录 | 新网络通常不同 |

## macOS 升级注意事项

PF 和 Internet Sharing 属于系统实现边界。升级前保留备份和仓库副本；升级后先检查 Apple `/etc/pf.conf` 的默认结构是否变化，再重新插入或核对 anchor 位置。不要用旧文件覆盖新版本系统文件。

升级后的回归顺序：

1. `pfctl -n` 检查主文件和子模板。
2. 检查 Internet Sharing、DHCP 和普通 NAT。
3. 检查 Mihomo 进程名和 `198.18.0.1`。
4. 执行 refresh `--check`，再做一次人工 `--apply` 验收。
5. 验证 Clash 退出和恢复。
6. 验证热点关闭和重开。
7. 完成整机重启、安全抓包和数据面抓包。

## Clash Verge Rev 或 Mihomo 升级

重点检查进程名、TUN 地址、Fake-IP 行为和接口输出格式。如果进程名不再是 `verge-mihomo`，refresh 和 watcher 都会正确保持 fail-open，但需要受控修改 `pgrep` 条件并重新执行 P2 至 P4。

## 变更纪律

- 先复制仓库并建立新版本分支或标签，再修改冻结文件。
- 一次只改变一个设计变量，记录原因和回归证据。
- 不把观察到的运行时实例写进模板。
- 修改 watcher 的规则匹配字符串时，要同时检查 `pfctl -a clash-forward -sr` 的实际格式。
- 修改 state 策略时，要评估热点所有客户端的连接中断。
- 只有 P1 至 P4 全部重新通过，才能冻结新版本。
