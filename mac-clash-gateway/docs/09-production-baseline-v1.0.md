# Production Baseline v1 0 冻结清单

## 冻结决定

截至 2026-09-05，本方案 P1、P2、P3 和 P4 全部通过，版本冻结为 Production Baseline v1.0。没有明确故障或受控升级计划时，不修改五个冻结文件。

## 五个冻结文件

| 正式路径 | 仓库来源 | 权限 | 内容要求 |
| --- | --- | --- | --- |
| `/etc/pf.conf` | `config/pf.conf.snippet` 仅供插入 | 系统原权限 | Apple filter anchor 前增加项目挂载点 |
| `/etc/pf.anchors/clash-forward` | `config/clash-forward` | `root:wheel 0644` | bypass 表和两条动态 TCP UDP 规则 |
| `/usr/local/sbin/clash-forward-refresh` | `config/clash-forward-refresh` | `root:wheel 0755` | check apply disable fail-open 和 state 清理 |
| `/usr/local/sbin/clash-forward-watch` | `config/clash-forward-watch` | `root:wheel 0755` | 10 秒协调 幂等和错误保持 |
| `/Library/LaunchDaemons/local.clash-forward-watch.plist` | `config/local.clash-forward-watch.plist` | `root:wheel 0644` | system LaunchDaemon 和日志路径 |

## 固定设计参数

```text
HOTSPOT_IF  bridge100
HOTSPOT_IP  192.168.2.1
HOTSPOT_NET 192.168.2.0/24
TUN_IP      198.18.0.1
ANCHOR      clash-forward
PROCESS     verge-mihomo
```

这些值属于 v1.0 已验收环境。新机器若不满足，不能静默修改后宣称仍是同一冻结基线；应建立候选版本并重新验收。

## 环境记录但非脚本依赖

```text
上游接口 <上游接口>
上游网关 <上游网关>
热点名称 <自定义名称>
```

## 严禁写死的运行时变量

```text
<运行时 utunX>       某次验收实例
<客户端 DHCP 地址>    某次客户端 DHCP 地址
Mihomo PID     每次进程启动可能变化
PF 插入 PID    每次加载可能变化
```

## 冻结规则全文

```pf
table <clash_bypass> const { 0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 127.0.0.0/8, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16, 224.0.0.0/4, 240.0.0.0/4 }

pass in quick on bridge100 route-to ($tun_if 198.18.0.1) inet proto tcp from 192.168.2.0/24 to ! <clash_bypass> flags S/SA keep state
pass in quick on bridge100 route-to ($tun_if 198.18.0.1) inet proto udp from 192.168.2.0/24 to ! <clash_bypass> keep state
```

## 版本校验

从仓库根目录安装前，可记录仓库副本校验和：

```bash
shasum -a 256 config/clash-forward config/clash-forward-refresh config/clash-forward-watch config/local.clash-forward-watch.plist
```

部署后对正式路径再次计算并比较。`/etc/pf.conf` 不能与 snippet 直接比较，必须用 diff 人工确认只增加了挂载点。

## 发布前隐私检查

- 删除终端用户名、真实 MAC 地址、家庭公网或内网设备标识。
- 抓包只保留说明结论所需的最少行。
- 不提交 Clash 订阅、代理节点、密钥、完整配置或个人路径。
- 运行时实例可保留在验收记录中，但必须标注为证据而非部署值。
