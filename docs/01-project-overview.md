# 项目概述 环境和根因

## 项目概述

本项目把一台 Mac 作为 Wi-Fi 热点和旁路由。Apple Internet Sharing 继续负责热点、DHCP 和 NAT；PF 在热点入口识别公网 TCP 和 UDP 流量，并通过 `route-to` 把新连接送入 Clash Verge Rev 的 Mihomo TUN。该方案已经冻结为 Production Baseline v1.0。

## 技术目标

- iPhone 等热点客户端关闭蜂窝网络后，可访问国内网站、Google 和 YouTube。
- 不改变 Apple Internet Sharing 的 DHCP、NAT 和热点生命周期。
- 不把 DHCP、mDNS、本地、私网、组播或保留地址送入 TUN。
- 不写死 `utunX`、客户端 DHCP 地址或 Mihomo PID。
- Clash、TUN 或热点消失时自动撤销旁路由规则，普通 Internet Sharing 仍可使用。
- Clash、TUN 和热点恢复后自动重装规则，只在拓扑变化时清理 state。
- Mac 重启后不执行人工修复命令即可恢复。

## 已验收环境

| 项目 | 冻结值或条件 | 说明 |
| --- | --- | --- |
| Mac 下游热点 | Apple Internet Sharing | 热点名称由部署者自行设置，名称不是脚本条件 |
| 热点接口 | `bridge100` | 脚本固定检查 |
| Mac 热点地址 | `192.168.2.1/24` | 脚本固定检查 |
| 热点客户端网段 | `192.168.2.0/24` | 规则和 state 清理范围 |
| 上游接口 | `<上游接口>` | 当前验收环境；Apple 管理 NAT |
| 上游网关 | `<上游网关>` | 当前验收环境 |
| 代理内核 | `verge-mihomo` | 由 Clash Verge Rev 启动 |
| Mihomo TUN 地址 | `198.18.0.1` | 用于动态定位接口 |
| TUN 接口名 | `utunX` | 每次运行可能变化 |
| PF anchor | `clash-forward` | 只管理本项目规则 |

## 硬件和软件条件

- Mac 支持 Internet Sharing，且上游与下游能够同时工作。
- 上游网络可通过 `<上游接口>` 接入；新 Mac 若接口变化，应先验证，不要把 `<上游接口>` 加进旁路由脚本。
- Clash Verge Rev 已安装，Mihomo TUN 模式能建立 `198.18.0.1`。
- 管理员账户可以使用 `sudo` 修改 `/etc`、`/usr/local/sbin` 和 `/Library/LaunchDaemons`。
- macOS 自带 `/bin/sh`、`pfctl`、`ifconfig`、`launchctl`、`plutil`、`tcpdump`、`awk`、`grep` 和 `pgrep`。

## 约束

1. Apple Internet Sharing 会动态管理自己的 PF anchors。日常操作不得无条件重载整个 `/etc/pf.conf`。
2. PF rule 与 PF state 是两个层面。更换规则不会自动使旧连接重新匹配。
3. `route-to` 只处理入口新流量；本方案依赖 `quick` 阻止后续 Apple filter 规则改变已作出的决定。
4. `198.18.0.1` 是定位 Mihomo TUN 的稳定标记，接口名不是。
5. 客户端地址由 DHCP 分配，不能假定一直是 `.2` 或 `.8`。
6. watcher 不负责启动 GUI 应用。Clash 未启动时，它只保持 fail-open 和等待。

## 原方案故障概述

早期规则近似为：

```pf
pass in on bridge100 route-to (<运行时 utunX> 198.18.0.1)
```

它存在四类问题：匹配范围覆盖全部 IPv4 流量；缺少 `quick`；写死一次运行的 TUN 接口；恢复时只清理某个旧客户端地址的 state。结果包括 DHCP 或 mDNS 误入 TUN、手机无法续租地址、Clash 重启后新规则不接管旧连接，以及 Internet Sharing 或 Mac 重启后需要人工恢复。

## 根因

### 匹配范围过宽

未限制协议和目标地址的 `route-to` 会把 DHCP、mDNS、私网、链路本地、组播等不应代理的流量送入 TUN。TUN 不承担 Apple 热点的 DHCP 服务，基础网络因此可能失效。

### 缺少 quick

Apple Internet Sharing 后续规则可能继续评估同一数据包。`quick` 使命中的 TCP 或 UDP 公网流量在本规则处结束过滤决策，避免后续规则改变路径。

### 规则正确但旧 state 仍存在

Clash 重启或规则重载后，已有连接仍沿旧 PF state 工作。曾经清理 `<旧客户端 DHCP 地址>` 时得到 `killed 0 states`，后来确认手机地址已变为 `<客户端 DHCP 地址>`。清理实际客户端 state 后，TCP 和 UDP 计数立即增长，国内外访问恢复。正式方案因此对整个 `192.168.2.0/24` 执行 state 清理。

### 主 anchor 未持久化

只把规则加载进 named anchor 还不够；主 PF ruleset 必须有 `anchor "clash-forward"` 调用点。正式方案将该挂载点写入 `/etc/pf.conf`，但不从主文件静态加载子规则，使启动阶段天然 fail-open。

### 无条件周期刷新会制造新故障

`--apply` 会清理热点网段 state。定时无条件运行会周期性中断连接。watcher 每 10 秒观察状态，只在 TUN 出现或变化、热点重建、规则丢失或依赖消失时修改 PF。
