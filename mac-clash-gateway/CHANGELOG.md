# 变更记录

## 1.0.0 2026-09-05

- 冻结 `Mac Internet Sharing + PF + Clash Verge Rev/Mihomo` 旁路由方案。
- 完成 P1 主 PF anchor 持久化、P2 动态刷新、P3 LaunchDaemon 自愈和 P4 整机重启验收。
- 固定 `bridge100 = 192.168.2.1/24`、热点网段 `192.168.2.0/24` 和 Mihomo TUN 地址 `198.18.0.1`。
- 将 `utunX`、热点客户端地址和进程 PID 定义为运行时变量。
- 增加 fail-open、动态 state 清理、部署、回滚、验证和升级说明。
