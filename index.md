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

### [用 Tailscale 连接家庭与办公室：双 NAS 网关与 Cloudflare 独立管控平面](dual-site-tailscale-cloudflare-control-plane/)

以 Tailscale 连接办公室与家庭局域网，并记录如何加入独立的 Cloudflare 状态观察、双端诊断和受控恢复流程。文章及附件使用统一的虚构名称和地址。

状态：实践记录已发布；真实故障自动恢复完整链路仍待验收。

## 使用说明

每个项目都是独立文档单元。进入项目首页后，再按“概述 → 架构 → 部署 → 验证 → 恢复 → 升级”的顺序阅读。

> [!IMPORTANT]
> 本站只发布脱敏资料。任何涉及个人设备、实际网络参数、访问凭据或完整运行证据的内容均不进入公开仓库。
