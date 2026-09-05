# 命令参考

## 阅读说明

每条命令按用途、参数、执行时机和副作用说明。涉及 PF 或 system launchd 的命令需要管理员权限。

## PF 主规则语法检查

```bash
sudo pfctl -n -f /etc/pf.conf
```

- 用途：解析主 PF 配置，不加载。
- 参数：`-n` 表示只检查；`-f` 指定规则文件。
- 时机：每次编辑 `/etc/pf.conf` 后、覆盖或重启前。
- 风险：通常只读，但 `pfctl` 仍显示 `-f` 的通用刷新警告；只要保留 `-n` 就不会实际加载。

## 子规则模板语法检查

```bash
sudo pfctl -n -a clash-forward -D "tun_if=$TUN_IF" -f /etc/pf.anchors/clash-forward
```

- 用途：验证 anchor 文件及运行时宏。
- 参数：`-a clash-forward` 限定 named anchor；`-D` 定义 `tun_if`；`-f` 读取模板；`-n` 禁止加载。
- 时机：规则模板或 TUN 识别逻辑变化后。
- 风险：不要删除 `-n` 做试探；没有 `-D` 时模板应因宏未定义而拒绝加载。

## 查看主过滤规则

```bash
sudo pfctl -sr -vv
```

- 用途：确认主 ruleset 中有 `anchor "clash-forward"`，并观察 Apple Internet Sharing anchors。
- 参数：`-s rules` 可缩写为 `-sr`；`-vv` 显示计数和插入信息。
- 时机：P1、P4 和怀疑主挂载点丢失时。
- 风险：只读。`No ALTQ support` 在本项目中不是故障。

## 查看项目规则及计数

```bash
sudo pfctl -a clash-forward -s rules -v
```

- 用途：确认两条 TCP 和 UDP 规则、当前动态 TUN、Packets、Bytes 和 States。
- 参数：`-a` 限定 anchor；`-s rules` 显示规则；`-v` 显示统计。
- 时机：每次 apply、自愈、重启和手机访问测试后。
- 风险：只读。规则刚加载时计数为 0 正常；产生手机流量后仍为 0 才表示数据未进入 anchor。

## 查看 PF states

```bash
sudo pfctl -ss -vv | grep -B2 -A6 '192\.168\.2\.'
```

- 用途：判断热点连接是否仍受旧 state 支配。
- 参数：`-ss` 显示 states；`-vv` 详细输出；`grep` 保留匹配前后上下文。
- 时机：规则存在但计数不增长，或 Clash 重启后国外访问失败时。
- 风险：只读；输出可能很长。

## 清理热点网段 state

```bash
sudo pfctl -k 192.168.2.0/24
```

- 用途：删除来源属于整个热点网段的 state，使新连接重新匹配当前规则。
- 参数：`-k` 按 source host 或 network 删除；`/24` 覆盖所有动态客户端地址。
- 时机：只在规则加载、撤销或 TUN 变化时。正式脚本已自动执行。
- 副作用：热点客户端现有 TCP 和 UDP 会话会中断并重建。
- 禁止：不要定时执行，不要以某个历史手机 IP 替代整个网段。

## 清空项目 anchor

```bash
sudo pfctl -a clash-forward -F all
```

- 用途：移除项目 anchor 内的规则和私有表。
- 参数：`-a` 把操作限制在项目 anchor；`-F all` 清空该 anchor 内容。
- 时机：紧急 fail-open 或人工回滚。
- 副作用：国外代理访问停止；Apple 主 ruleset 和 NAT 不应被清空。
- 后续：再清理 `192.168.2.0/24` state。

## 刷新脚本检查

```bash
sudo /usr/local/sbin/clash-forward-refresh --check
```

- 用途：检查主 anchor、热点地址、Mihomo、唯一 TUN 和 PF 语法。
- 参数：`--check` 不加载规则，不清 state。
- 时机：部署、升级或故障定位时优先执行。
- 风险：只读检查；退出码 10 至 15 对应不同前置条件失败。

## 刷新脚本应用

```bash
sudo /usr/local/sbin/clash-forward-refresh --apply
```

- 用途：预检查、动态注入 TUN、加载规则并清理热点 state。
- 参数：`--apply` 执行完整事务。
- 时机：首次 P2 人工验收；正常运行后由 watcher 触发。
- 副作用：会中断热点现有连接。预检查或加载失败时脚本转为 fail-open。

## 刷新脚本停用

```bash
sudo /usr/local/sbin/clash-forward-refresh --disable
```

- 用途：清空项目 anchor、清理热点 state，恢复普通 Internet Sharing。
- 时机：回滚、隔离 PF 问题或停止代理。
- 副作用：代理目标不可访问，现有热点连接重建。

## watcher 单次协调

```bash
sudo /usr/local/sbin/clash-forward-watch --once
```

- 用途：执行一次状态协调，便于安装前验证。
- 参数：`--once` 不进入 10 秒循环。
- 时机：安装或修改 watcher 后。
- 风险：若实际规则与拓扑不一致，它可能 apply 或 disable，并清 state。

## Shell 语法检查

```bash
sudo /bin/sh -n /usr/local/sbin/clash-forward-refresh
sudo /bin/sh -n /usr/local/sbin/clash-forward-watch
```

- 用途：发现 shell 语法错误。
- 参数：`-n` 只解析，不执行。
- 时机：每次安装或修改脚本后。
- 风险：只读；正常无输出。

## LaunchDaemon 检查和控制

```bash
sudo /usr/bin/plutil -lint /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl enable system/local.clash-forward-watch
sudo launchctl kickstart -k system/local.clash-forward-watch
sudo launchctl print system/local.clash-forward-watch
sudo launchctl bootout system/local.clash-forward-watch
```

- `plutil -lint`：检查 plist；正常为 `OK`。
- `bootstrap system`：注册 system-domain LaunchDaemon。
- `enable`：清除可能的禁用状态。
- `kickstart -k`：立即启动；`-k` 会终止已有实例再启动，因此可能触发一次协调。
- `print`：人工查看状态、程序、PID 和退出码；不要让脚本依赖其文本格式。
- `bootout`：停止并注销 daemon；执行后 watcher 不再自动维护 PF。

## 动态定位 TUN

```bash
ifconfig | awk '/^[a-zA-Z0-9_.-]+:/{i=$1;sub(":","",i)} $1=="inet" && $2=="198.18.0.1"{print i}'
```

- 用途：根据稳定 IP 找到当前 `utunX`。
- 解释：遇到接口头时保存接口名；遇到 `inet 198.18.0.1` 时输出该接口。
- 时机：诊断和抓包，不用于写死配置。
- 异常：0 个结果表示 TUN 未就绪；多个结果表示拓扑不唯一，正式脚本会拒绝 apply。

## TUN 安全抓包

```bash
TUN_IF="$(ifconfig | awk '/^[a-zA-Z0-9_.-]+:/{i=$1;sub(":","",i)} $1=="inet" && $2=="198.18.0.1"{print i; exit}')"
sudo tcpdump -ni "$TUN_IF" -vvv 'udp port 67 or udp port 68 or udp port 5353'
```

- 用途：确认 DHCP 67/68 和 mDNS 5353 没有误入 TUN。
- 参数：`-n` 不做名称解析；`-i` 指定接口；`-vvv` 详细显示；最后是 BPF 过滤条件。
- 预期：正常使用 10 秒后 `0 packets captured`。
- 风险：只读抓包；用 `Ctrl+C` 停止。`packets received by filter` 不是捕获数量，以 `packets captured` 为准。

## TUN 数据面抓包

```bash
sudo tcpdump -ni "$TUN_IF" -c 10 -vvv 'net 192.168.2.0/24 and (tcp or udp)'
```

- 用途：证明热点客户端双向流量进入动态 TUN。
- 参数：`-c 10` 自动在十个包后结束；`net` 匹配整个热点网段。
- 预期：看到 `192.168.2.x` 与目标地址之间的双向包。
- 风险：抓包可能显示访问元数据，公开日志前应脱敏。

## 不应作为日常操作的命令

```bash
sudo pfctl -f /etc/pf.conf
```

该命令会重载主 ruleset，可能干扰 Apple 动态 Internet Sharing 规则。只在明确的恢复方案、停机窗口和完整备份下考虑；本基线的日常维护只操作 `clash-forward` anchor。
