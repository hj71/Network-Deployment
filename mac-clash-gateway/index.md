---
layout: default
title: Mac Internet Sharing + PF + Mihomo 旁路由
---

# Mac Internet Sharing + PF + Mihomo 旁路由

Production Baseline v1.0 的公开、脱敏技术文档。

本项目将 Apple Internet Sharing 保留为 DHCP、热点与 NAT 的管理者，只用独立 PF anchor 把热点客户端的公网 TCP/UDP 动态导向 Mihomo TUN。方案包含启动等待、接口动态发现、连接状态清理和 fail-open，避免把某次运行的 `utun` 编号或客户端地址写死。

## 阅读入口

- [项目 README](README.md)
- [项目概述、条件与根因](docs/01-project-overview.md)
- [架构设计](docs/02-architecture-design.md)
- [从零部署](docs/03-deployment-guide.md)
- [命令与参数参考](docs/04-command-reference.md)
- [调试与验证](docs/05-debugging-and-validation.md)
- [验收记录](docs/06-acceptance-record.md)
- [恢复与回滚](docs/07-recovery-and-rollback.md)
- [迁移与升级](docs/08-migration-and-upgrade.md)
- [Production Baseline v1.0](docs/09-production-baseline-v1.0.md)

> [!WARNING]
> 本方案会修改 macOS Packet Filter 并安装系统级 LaunchDaemon。部署前必须备份；远程操作时尤其要准备本地恢复手段。

## 隐私说明

公开版已移除实际上游接口、上游网关、热点名称、客户端 DHCP 地址和一次性 `utun` 实例。示例中的占位符必须按目标 Mac 的实时环境确认，不能机械复制运行时编号。

