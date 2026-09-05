# Mac Internet Sharing 加 PF 加 Clash 旁路由

Production Baseline v1.0 是一套已经完成 P1 至 P4 验收的 macOS 旁路由方案。它保留 Apple Internet Sharing 的 DHCP 和 NAT，只把热点客户端访问公网的 TCP 和 UDP 流量送入 Clash Verge Rev 使用的 Mihomo TUN。Clash、TUN 或热点不可用时，系统自动清空自有 PF anchor，退回普通 Internet Sharing。

> [!WARNING]
> 本仓库修改 macOS Packet Filter 和系统级 LaunchDaemon。先完整备份，再逐阶段部署。不要把一次运行中观察到的 `<运行时 utunX>`、手机地址 `<客户端 DHCP 地址>` 或 Mihomo PID 写入配置。

## 已冻结结论

- 热点接口和地址：`bridge100 = 192.168.2.1/24`
- 热点客户端网段：`192.168.2.0/24`
- 已验收上游：`<上游接口>`，网关 `<上游网关>`
- Mihomo TUN 地址：`198.18.0.1`
- TUN 接口：运行时动态发现 `utunX`
- 主 PF 只保留 `anchor "clash-forward"` 挂载点
- 子 anchor 仅转发非本地目标的 TCP 和 UDP
- 规则或拓扑变化时清理整个热点网段的旧 PF state
- Clash 或热点不可用时 fail-open

## 阅读路线

1. [项目概述、环境与根因](docs/01-project-overview.md)
2. [架构设计](docs/02-architecture-design.md)
3. [从零部署](docs/03-deployment-guide.md)
4. [命令参考](docs/04-command-reference.md)
5. [调试与验证](docs/05-debugging-and-validation.md)
6. [P1 至 P4 验收记录](docs/06-acceptance-record.md)
7. [回滚与故障恢复](docs/07-recovery-and-rollback.md)
8. [新 Mac 迁移与升级](docs/08-migration-and-upgrade.md)
9. [冻结文件清单](docs/09-production-baseline-v1.0.md)

## 仓库结构

```text
Mac-Clash-Gateway/
├── README.md
├── CHANGELOG.md
├── config/
│   ├── pf.conf.snippet
│   ├── clash-forward
│   ├── clash-forward-refresh
│   ├── clash-forward-watch
│   └── local.clash-forward-watch.plist
└── docs/
    ├── 01-project-overview.md
    ├── 02-architecture-design.md
    ├── 03-deployment-guide.md
    ├── 04-command-reference.md
    ├── 05-debugging-and-validation.md
    ├── 06-acceptance-record.md
    ├── 07-recovery-and-rollback.md
    ├── 08-migration-and-upgrade.md
    └── 09-production-baseline-v1.0.md
```

## 快速使用

新机器不要直接复制整份 `/etc/pf.conf`。先阅读部署文档，在新系统自带文件中，把 `anchor "clash-forward"` 插入 Apple filter anchor 之前，再安装其他四个文件。逐项完成语法检查、人工 `--check`、人工 `--apply`、LaunchDaemon 自愈测试和整机重启验收。

## 支持边界

本基线冻结的是已验收环境和技术路线，不保证未来 macOS、Clash Verge Rev 或 Mihomo 升级后仍保持内部行为不变。任何系统大版本升级都应按 [迁移与升级](docs/08-migration-and-upgrade.md) 重新执行差异检查和 P1 至 P4 回归测试。
