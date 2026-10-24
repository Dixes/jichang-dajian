# 晚高峰丢包：落地机开启 BBR 与 TCP 调优

晚高峰最常见的投诉不是“连不上”，而是“能连但卡”：白天测速正常，一到晚上八九点就丢包、延迟飙升、视频转圈。很多运维第一反应是换线路、加带宽，其实问题常常出在落地机自己的拥塞控制上——默认的 Cubic 算法在晚高峰这种丢包率上升的链路上会过度减速，越丢越慢。把拥塞控制换成 BBR，再把 TCP 缓冲和队列参数调到合理范围，往往不用换机器就能把晚高峰体验拉回来。本篇只讲落地机一侧的调优，不动面板和客户端。

## 先确认：到底是不是拥塞控制的问题

动手调参之前，先分清丢包发生在链路还是本机。链路本身被限速或被墙，调 BBR 也救不回来；本机参数不合理，才值得调。

```bash
# 连续 ping 一段时间，看晚高峰时段的丢包和延迟抖动
ping -c 100 8.8.8.8

# 看当前拥塞控制算法和可用算法列表
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control

# 看当前 TCP 缓冲区设置
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.core.rmem_max net.core.wmem_max
```

> 为什么先看 `tcp_congestion_control`？如果当前已经是 BBR，晚高峰还卡，问题大概率不在拥塞控制，而在带宽、队列或链路本身，继续调缓冲只是白忙。先确认现状，再决定调什么，这是排障最省时间的一步。

如果结果显示当前是 `cubic`，且丢包集中在晚高峰、白天基本正常，就可以按下面步骤开 BBR。如果丢包全天都高、或者某个目的地址全天不通，先按本仓库的[realm 转发 UDP 的配置要点](realm-udp-forward.md)里的逐跳排查思路定位链路问题，不要把链路问题当成调参问题。

## 开启 BBR：两行配置就够

BBR 是 Google 的拥塞控制算法，它根据实际带宽和往返时延来算发送速率，而不是像 Cubic 那样“丢包就减半”。在晚高峰这种轻微丢包、带宽其实还在的场景下，BBR 不会因为几个丢包就把速度砍下来，这是它对机场落地机最有用的地方。

```bash
# 检查内核是否支持 BBR（Linux 4.9 及以上一般都内置）
# 输出里有 bbr 才继续
sysctl net.ipv4.tcp_available_congestion_control

# 写入配置：/etc/sysctl.d/99-bbr.conf
cat > /etc/sysctl.d/99-bbr.conf <<'EOF'
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF

# 生效并验证
sysctl --system
sysctl net.ipv4.tcp_congestion_control
```

> 为什么要同时写 `default_qdisc = fq`？BBR 需要配合 fq（Fair Queue）队列调度才能准确地按流 pacing，少了这一行，BBR 的效果会打折扣。这两个参数是一对，缺一不可，不要只写拥塞控制那一行。

注意：这一步不需要重启机器，`sysctl --system` 生效后新建立的 TCP 连接就会走 BBR，老连接要等重连后才切换。验证时不要只看 sysctl 输出，还要实际跑一次晚高峰时段的测速对比。

## TCP 缓冲与队列：配合 BBR 一起调

只开 BBR、不调缓冲，效果也可能不完整。落地机同时服务多个用户，接收和发送缓冲区太小，突发流量一来就溢出丢包；太大则会引入缓冲膨胀（bufferbloat），延迟反而上升。下面的值是落地机常用的起点，不是万能值，调完要实测。

```bash
# 追加到同一个配置文件：/etc/sysctl.d/99-bbr.conf
cat >> /etc/sysctl.d/99-bbr.conf <<'EOF'
# TCP 读写缓冲区：最小值 默认值 最大值，单位字节
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
# 套接字缓冲上限
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
# 增大 backlog，避免突发连接被内核直接丢弃
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 4096
EOF

sysctl --system
```

> 为什么最大值给到 16MB 而不是越大越好？高带宽、高延迟的链路需要大缓冲才能跑满速度，但缓冲无限放大只会让排队更长、延迟更高。16MB 对绝大多数落地机已经够用，调完后如果晚高峰延迟反而上升，第一件事就是把最大值往回调，而不是继续往上加。

另外一个容易漏的点：`tcp_slow_start_after_idle`。默认开启时，空闲一阵的连接恢复传输要重新慢启动，对间歇性流量的用户体验不友好，落地机上通常建议关掉：

```bash
# 追加到 /etc/sysctl.d/99-bbr.conf
echo "net.ipv4.tcp_slow_start_after_idle = 0" >> /etc/sysctl.d/99-bbr.conf
sysctl --system
```

## 验证：不要只看 sysctl 输出

参数写进去不等于生效，验证要分两层：内核参数层和真实传输层。

```bash
# 1. 参数层：确认 BBR 和 fq 都已生效
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
cat /etc/sysctl.d/99-bbr.conf

# 2. 连接层：看一条真实 TCP 连接实际用的拥塞算法
# 先产生一条连接（例如 curl 一次），再查它的拥塞算法
ss -tin | grep -A1 bbr

# 3. 传输层：晚高峰时段再跑一次测速和 ping，对比调参前的数据
# 重点看丢包率和延迟抖动有没有下降，而不是只看峰值速度
ping -c 100 8.8.8.8
```

> 为什么 `ss -tin` 这一步不能省？sysctl 只证明全局默认值改了，`ss` 才能证明某条连接真的在用 BBR。调参后如果 `ss` 里看不到 bbr，说明配置没生效或连接是调参前建立的老连接，重启服务或等连接重建后再查。

验证的时间点也很重要：白天测没意义，一定要复测晚高峰时段。白天链路空闲，Cubic 和 BBR 测出来可能差不多，只有晚高峰丢包上来时，BBR 的优势才会显现。建议调参前后各保留一份晚高峰的 ping 和测速记录，否则“好像变好了”只是感觉，没有证据。

## 常见坑汇总

- **只开 BBR、没配 fq**：BBR 配 fq 才完整，缺了 fq 效果打折，先查 `default_qdisc` 再怀疑别的问题。
- **调完不验证连接层**：sysctl 输出是 bbr 不代表连接在用 bbr，用 `ss -tin` 确认过才算数。
- **缓冲一味往大调**：缓冲区越大延迟越高，16MB 是起点，延迟上升就往回调，不要无上限加。
- **白天验证、晚上翻车**：晚高峰问题必须晚高峰复测，白天测速正常证明不了任何事。
- **把链路问题当调参问题**：全天丢包、单点不通、疑似被墙的，先排查链路，BBR 救不了带宽和封锁。
- **多台落地机只调一台**：用户被分流到不同落地机，只调一台等于只修了一部分，面板下的落地机要逐台确认。
- **改完不留档**：参数写在 `/etc/sysctl.d/99-bbr.conf` 里就是留档，不要只用 `sysctl -w` 临时改，机器一重启就丢，晚高峰一来又原地复发。

BBR 加 TCP 调优不是万能药，它修的是“带宽还在、丢包拖慢速度”这一类问题，修不了带宽不足和链路封锁。但对晚高峰丢包的落地机来说，这是成本最低的一步：两行配置开 BBR，一组参数调缓冲，晚高峰复测一次，效果立见分晓。

## 不想自己搭？

按上面这套走下来，从确认丢包时段、开 BBR、调缓冲、再到晚高峰时段复测对比，顺利的话也要花上一两个小时，而且多台落地机还得逐台确认。如果你只是想有个现成可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
