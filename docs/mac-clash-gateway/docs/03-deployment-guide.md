# 从零部署

## 部署前检查

先打开 Internet Sharing 和 Clash Verge Rev TUN，再确认实际环境：

```bash
ifconfig bridge100
route -n get default
ps aux | grep -i '[m]ihomo'
ifconfig | grep -B2 -A3 '198\.18\.0\.1'
```

预期看到 `bridge100` 上的 `192.168.2.1`、当前默认上游、`verge-mihomo` 进程，以及某个 `utunX` 上的 `198.18.0.1`。如果热点网段或 TUN 地址不同，先评估并统一修改仓库中的固定参数，不要直接部署。

> [!CAUTION]
> 新 Mac 必须以该系统自带 `/etc/pf.conf` 为基础，只插入一个 anchor。不要用旧 Mac 的整份文件覆盖新系统文件。

## 第一阶段备份和主 anchor

```bash
sudo cp -p /etc/pf.conf /etc/pf.conf.before-clash-forward
sudo cp -p /etc/pf.anchors/clash-forward /etc/pf.anchors/clash-forward.before-install 2>/dev/null || true
```

编辑 `/etc/pf.conf`，在现有 `anchor "com.apple/*"` 之前增加：

```pf
anchor "clash-forward"

anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

只做语法检查：

```bash
sudo pfctl -n -f /etc/pf.conf
```

此处不要把 `pfctl -f /etc/pf.conf` 当作日常部署命令。主文件会在重启时进入实际验收。

## 第二阶段安装规则和刷新脚本

从仓库根目录执行：

```bash
sudo install -o root -g wheel -m 0644 config/clash-forward /etc/pf.anchors/clash-forward
sudo install -d -o root -g wheel -m 0755 /usr/local/sbin
sudo install -o root -g wheel -m 0755 config/clash-forward-refresh /usr/local/sbin/clash-forward-refresh
```

先动态取出 TUN 接口，仅用于本轮语法检查：

```bash
TUN_IF="$(ifconfig | awk '/^[a-zA-Z0-9_.-]+:/{i=$1;sub(":","",i)} $1=="inet" && $2=="198.18.0.1"{print i; exit}')"
sudo pfctl -n -a clash-forward -D "tun_if=$TUN_IF" -f /etc/pf.anchors/clash-forward
sudo /bin/sh -n /usr/local/sbin/clash-forward-refresh
sudo /usr/local/sbin/clash-forward-refresh --check
```

只有 `--check` 返回 `CHECK OK` 后才执行：

```bash
sudo /usr/local/sbin/clash-forward-refresh --apply
```

让热点客户端访问国内网站、Google 和 YouTube，再检查：

```bash
sudo pfctl -a clash-forward -s rules -v
```

## 第三阶段安装 watcher

```bash
sudo install -o root -g wheel -m 0755 config/clash-forward-watch /usr/local/sbin/clash-forward-watch
sudo /bin/sh -n /usr/local/sbin/clash-forward-watch
sudo /usr/local/sbin/clash-forward-watch --once
```

若当前规则正确，`--once` 应只输出 `ACTIVE: TUN=utunX`，不应出现 `APPLY OK` 或 `killed N states`。

## 第四阶段安装 LaunchDaemon

```bash
sudo install -o root -g wheel -m 0644 config/local.clash-forward-watch.plist /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo /usr/bin/plutil -lint /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl bootout system/local.clash-forward-watch 2>/dev/null || true
sudo launchctl bootstrap system /Library/LaunchDaemons/local.clash-forward-watch.plist
sudo launchctl enable system/local.clash-forward-watch
sudo launchctl kickstart -k system/local.clash-forward-watch
sudo launchctl print system/local.clash-forward-watch
```

预期 `state = running`，程序路径正确，错误日志为空：

```bash
sudo tail -50 /var/log/clash-forward-watch.log
sudo tail -50 /var/log/clash-forward-watch.err
```

## 第五阶段自愈验证

1. 完整退出 Clash。等待 10 至 20 秒，确认 anchor 为空，手机仍可普通访问国内网络。
2. 启动 Clash 和 TUN。不要手动 apply，确认日志从 `WAIT` 到 `APPLYING`、`ACTIVE`，国内外访问恢复。
3. 保持 Clash 运行，关闭 Internet Sharing。确认 watcher 撤销规则。
4. 重新打开 Internet Sharing。确认 watcher 自动恢复并且计数增长。

## 第六阶段整机重启

正常重启 Mac。登录后不要执行任何 PF 修复命令，按验证文档完成 P4。只有 LaunchDaemon、主 anchor、热点、TUN、watcher、手机访问和安全抓包全部通过，部署才算完成。
