---
layout: default
title: 用 Tailscale 连接家庭与办公室：双 NAS 网关与 Cloudflare 独立管控平面
permalink: /dual-site-tailscale-cloudflare-control-plane/
---

# 用 Tailscale 连接家庭与办公室：双 NAS 网关与 Cloudflare 独立管控平面

*一个双站点私有网络的建设、诊断与受控恢复实践*

办公室与家庭各有一台 Synology NAS。项目用 Tailscale 把两个局域网连接起来，让经过授权的客户端访问对侧网络；Cloudflare 则提供一条与 Tailscale 数据面分离的状态观察和协调通道。本文记录项目如何从“让两边能连通”逐步发展到“发现异常、协调诊断，并在严格条件下发起受控操作”，以及已有材料支持哪些结论。

文中设备名和网段均为虚构示例。NAS A 与 NAS B 对应两个站点网关；实际地址、设备标识、用户名和日志没有放入公开附件。

## 先把两个站点连起来

Tailscale 为经过授权的设备提供私有网络。两台 NAS 分别宣告自己的局域网路由，并可按需充当 Exit Node。验证 Subnet Router 时，不能只访问 NAS 自己的地址；还要从对侧客户端访问 NAS 后面的另一台 LAN 主机，才能确认流量确实经过路由节点。Exit Node 则要比较切换前后的出口，并在重启后再验一次。

真实环境使用两台不同 DSM 世代的 Synology。一次实际测试中，平台行为没有完全照搬通用 Linux 对 forwarding 和 TUN 的预期。这只能说明所记录设备和版本经过端到端验证，不能简化成适用于所有 Linux 或未来 DSM 版本的统一配置。升级后仍应复测真实转发路径。

## 连通不等于可诊断

后来遇到的问题不总是“整个 Tailnet 都离线”。有时对端仍能通过 DERP 交换控制或发现信息，具体数据连接却不稳定。只看设备在线状态或一次健康检查，无法判断对端路径和另一站点 LAN 是否可用。短暂 WAN、DNS 或 peer 路径异常还可能在采样后自行恢复。

因此架构分成两面：Tailscale 负责站点间私有数据流量；Cloudflare Worker 与 D1 接收两端快照和事件，保留时间线，并协调双方对同一诊断轮次返回结果。Cloudflare 是独立的协调位置，但仍依赖站点自己的 Internet 路径和服务可用性，不能修复完全断网。

## 从观察到受控操作

流程从有限证据开始：Agent 上传带时间戳的状态，Worker 检查新鲜度并计算派生状态；出现需要调查的信号时，协调两端采集匹配的诊断结果。只有满足项目定义的前置条件，才会将短期控制请求放入队列。NAS 端 Parser 校验请求，Executor 使用 journal 处理重复投递、冷却和结果记录；ACK 返回后还要观察后续状态，才能区分“命令已处理”和“网络确实恢复”。

代码也对应这条链路：Worker 负责状态、诊断轮次和仲裁；NAS Agent 采集现场并交付结果；Parser 验证命令；Executor 做有限系统操作并留下 journal；本地巡检工具则用有界读取生成报告。公开仓库中的代码是历史候选的脱敏参考版本，缺少完整迁移和部署包装，不能直接作为生产安装包。

## 目前有哪些验收证据

保存的历史记录包括两侧 Subnet Router 与 Exit Node 端到端测试、两次维护模式服务重启的 ACK、三轮双端诊断关联，以及正常逐台整机重启后任务和 Tailscale 服务恢复的检查。这些分别验证网络功能、维护控制流程和计划内重启持久性。

2026 年 9 月 28 日的一次巡检在当时采样中看到两台 NAS 与 D1 状态健康；检查窗口内也记录到短时基础网络和 peer 路径异常，稍后恢复。报告标记为 WARN / PARTIAL，没有确认根因，也没有发现自动 Tailscale 重启请求。**真实故障触发后的检测、双端诊断、仲裁、重启、ACK 与恢复后验证还没有完整验收。**断电恢复也未验证。

## 复用时的做法

先替换自己的示例 LAN 与节点角色，逐站验证路由和 Exit Node，再接入状态上报与只读诊断。之后在隔离环境审阅控制权限、到期时间、幂等处理、cooldown、回退和后检条件。将计划维护测试和真实故障恢复分开记录；每条结论都保存日期、版本、证据来源及数据覆盖限制。不要在版本未知或权限未经审阅时自动重启远程 NAS。

完整架构、历史限制和脱敏参考代码见 [公开技术仓库](https://github.com/hj71/dual-site-tailscale-cloudflare-control-plane)。可下载随该版本发布的 [脱敏代码附件](https://github.com/hj71/dual-site-tailscale-cloudflare-control-plane/releases/download/v0.1.0/dual-site-tailscale-cloudflare-control-plane-v0.1.zip)。包内地址均为示例；运行前请阅读 README、限制说明和源码 provenance。
