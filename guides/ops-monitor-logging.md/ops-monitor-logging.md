# 机场运维：日志查看与状态监控

机场上线只是开始，长期稳定靠运维：日志告诉你哪里出错，监控在出问题时第一时间通知你。这篇把常用的日志命令和一套轻量监控方案讲清楚，花半小时搭好，以后排错效率翻倍。

## 准备工作

所有服务器都可以用 journald 统一管理日志。先确认关键服务都在运行：

```bash
# 各服务状态速查
systemctl status sing-box v2bx xboard 2>/dev/null | grep -E 'Active|●'
# 端口监听速查
ss -tlnp | grep -E '443|8388|10000'
```

`systemctl status` 一眼看出服务是否存活；`ss -tlnp` 确认端口确实在监听，服务活着但端口没监听往往是配置没生效。

## 日志查看实操

```bash
# 实时看 sing-box 日志
journalctl -u sing-box -f
# 只看最近 100 行
journalctl -u sing-box -n 100
# 看今天的错误
journalctl -u sing-box --since today -p err
# V2bX 日志
journalctl -u v2bx -f
# Nginx 访问与错误日志
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

关键参数：`-f` 实时跟踪，`-n` 限制行数，`--since today -p err` 过滤今天的错误级别，排错时先缩小范围再细看。sing-box 的 JSON 日志可以配 `log.level: warn` 减少噪音，生产环境别开 debug，磁盘会被打爆。

Docker 部署的 Xboard 看日志：

```bash
docker logs -f --tail 100 xboard
```

## 轻量监控方案

装一个 uptime-kuma（Docker 一行命令），监控所有节点的端口和面板：

```bash
docker run -d --name uptime-kuma -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

在 uptime-kuma 里添加监控项：每个节点的 443 端口（TCP 监控）、面板域名（HTTP 监控）、中转机端口。监控间隔设 60 秒，失败 3 次后通过 Telegram/邮件通知。`--restart=always` 别忘了加，监控自己挂了就没意义了。

服务器资源监控用 netdata 或直接看：

```bash
# 内存、CPU、磁盘一眼看
htop
df -h
# 流量统计
vnstat -d
```

`vnstat` 按天统计流量，方便核对带宽账单和发现异常跑量。

## 验证

```bash
# 模拟服务挂掉，确认监控告警
systemctl stop sing-box
# 等待约 3 分钟，检查 uptime-kuma 是否发出通知
# 恢复服务
systemctl start sing-box
```

告警通道一定要实测一次：没测试过的告警等于没有告警。

## 常见坑

一是日志级别开太高，debug 日志几天就把磁盘写满，定期 `journalctl --vacuum-size=500M` 清理。二是监控只监控了面板没监控节点，节点挂了用户先发现。三是告警发到没人看的邮箱，通知渠道要选自己每天都看的。四是 uptime-kuma 和被监控服务装在同一台机器上，机器挂了监控一起挂，重要节点分开部署。

## 不想自己搭？

一套完整的运维体系搭起来不难，但要长期有人盯：日志定期看、告警及时处理、磁盘定期清，这些都是持续的时间投入。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
