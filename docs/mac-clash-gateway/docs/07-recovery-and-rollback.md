# 回滚与故障恢复

## 最快恢复普通 Internet Sharing

当手机无法联网、TUN 指向异常或 DHCP 流量疑似误入 TUN 时，先停止自动维护并撤销项目规则：

```bash
sudo launchctl bootout system/local.clash-forward-watch
sudo /usr/local/sbin/clash-forward-refresh --disable
```

结果应是项目 anchor 为空、热点 state 被清理、Apple Internet Sharing 保留。手机现有连接会重建；国外代理访问会停止。

## watcher 故障恢复

```bash
sudo /bin/sh -n /usr/local/sbin/clash-forward-watch
sudo /bin/sh -n /usr/local/sbin/clash-forward-refresh
sudo /usr/bin/plutil -lint /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo tail -100 /var/log/clash-forward-watch.log
sudo tail -100 /var/log/clash-forward-watch.err
```

修复后重新注册：

```bash
sudo launchctl bootout system/local.clash-forward-watch 2>/dev/null || true
sudo launchctl bootstrap system /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl enable system/local.clash-forward-watch
sudo launchctl kickstart -k system/local.clash-forward-watch
```

`kickstart -k` 会重新启动 watcher；若规则需要更新，可能触发一次 state 清理。

## Clash 或 TUN 未恢复

确认 Clash Verge Rev 已启动并启用 TUN：

```bash
pgrep -x verge-mihomo
ifconfig | grep -B2 -A3 '198\.18\.0\.1'
sudo /usr/local/sbin/clash-forward-refresh --check
```

不要手工把观察到的 `utunX` 写回配置。TUN 出现后 watcher 应自动恢复；若停在 `ERROR-HOLD`，先修复 `--check` 的具体错误，再重启 watcher。

## 规则存在但国外网络不通

```bash
sudo pfctl -sr -vv | grep clash-forward
sudo pfctl -a clash-forward -s rules -v
sudo pfctl -ss -vv | grep -B2 -A6 '192\.168\.2\.' | head -150
```

如果主 anchor 和子规则都存在，但手机产生流量后计数不增长，旧 state 是主要怀疑对象。优先让 watcher 通过一次正确拓扑变化自动处理；紧急人工处理可执行：

```bash
sudo pfctl -k 192.168.2.0/24
```

该命令会中断所有热点客户端的现有连接，不要用于定时任务。

## 恢复 `/etc/pf.conf` 备份

只有明确要完全移除本方案，并已停止 watcher 时才恢复备份：

```bash
sudo launchctl bootout system/local.clash-forward-watch 2>/dev/null || true
sudo /usr/local/sbin/clash-forward-refresh --disable
sudo cp -p /etc/pf.conf.before-clash-forward /etc/pf.conf
sudo pfctl -n -f /etc/pf.conf
```

磁盘文件恢复后，安排正常重启让系统按原始配置重建 PF。不要在热点承载重要连接时直接重载主 ruleset。

## 完全卸载

在已备份文件、停止 daemon 和确认普通 Internet Sharing 正常后，可移走以下五个项目文件，并从 `/etc/pf.conf` 删除项目 anchor：

```text
/etc/pf.anchors/clash-forward
/usr/local/sbin/clash-forward-refresh
/usr/local/sbin/clash-forward-watch
/Library/LaunchDaemons/local.clash-forward-watch.plist
/etc/pf.conf 中的 anchor "clash-forward"
```

删除是不可逆操作，实际执行前应逐个确认路径和备份。本仓库不提供宽泛的递归删除命令。

## 恢复后的最低验收

- `bridge100` 和客户端 DHCP 正常。
- anchor 状态符合预期：回滚后为空或不存在。
- 普通国内网络可用。
- 如重新启用旁路由，完成安全抓包、规则计数和数据面抓包。
