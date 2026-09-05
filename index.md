---
layout: default
title: Network Deployment 技术知识库
---

# Network Deployment 技术知识库

公开、可复现、经过脱敏处理的网络与系统部署文档。

## 已发布项目

### [Mac Internet Sharing + PF + Mihomo 旁路由](mac-clash-gateway/)

保留 Apple Internet Sharing 的 DHCP、热点和 NAT，通过独立 PF anchor 将热点客户端的公网 TCP/UDP 动态导向 Mihomo TUN。包含从零部署、命令说明、调试、P1–P4 验收、fail-open、恢复回滚以及新 Mac 迁移指南。

状态：**Production Baseline v1.0 — PASS**

## 使用说明

每个项目都是独立文档单元。进入项目首页后，再按“概述 → 架构 → 部署 → 验证 → 恢复 → 升级”的顺序阅读。

> [!IMPORTANT]
> 本站只发布脱敏资料。任何涉及个人设备、实际网络参数、访问凭据或完整运行证据的内容均不进入公开仓库。

