# Redis 在 Xboard 里起什么作用、挂了什么表现

不少人装好 Xboard 之后，第一次听说 Redis 还是在报错里：面板明明能打开，登录却转圈，或者订阅链接突然拉不出来，日志里飘着一行 `RedisException`。Redis 在 Xboard 里不是可有可无的点缀，它是缓存和队列的底座——挂了不一定让整个面板立刻宕机，但会让一堆功能以「时好时坏」的方式出问题。这篇把 Redis 在 Xboard 里的三个角色讲清楚，再给出挂了之后的表现、排查顺序和恢复办法。

## Redis 在 Xboard 里干了什么

一句话概括：Redis 负责「快取」和「排队」，MySQL 负责「存档」。用户数据、订单、套餐这些必须长期保存的东西在 MySQL 里；那些读得多、变不了那么快，或者不需要立刻落盘的东西，Xboard 交给 Redis。

### 1. 缓存：让面板少查几次数据库

Xboard 的不少查询是高频的：订阅链接生成、节点列表、套餐信息、用户剩余流量展示。每次请求都去 MySQL 查一遍，面板一忙就慢。Redis 把这些结果缓存一段时间，下一次请求直接从内存里取，响应快得多。

典型表现是：同一个订阅链接，第一次打开稍慢，之后明显变快；面板后台的节点和套餐列表刷新也更顺。这就是缓存命中在起作用。

### 2. 队列：把耗时任务排队慢慢做

有些事不需要、也不应该在用户点击的瞬间做完，比如：

- 发送注册验证邮件、到期提醒邮件
- 订单支付成功后的后续处理
- 定时结算、流量统计汇总
- 节点测速、订阅内容更新

Xboard 把这些任务推入队列，由后台的队列 worker 一个个消费。如果没有队列，这些任务要么拖慢当前请求，要么在请求超时时直接丢掉。

### 3. 会话与临时数据

登录态、验证码、临时令牌这类「用完就扔」的数据也常放 Redis。它的好处是读写快、过期自动清理，不用专门写定时任务去删过期记录。

所以记住这个对应关系：**面板能打开但功能异常 → 先看 Redis；数据丢了、订单对不上 → 先看 MySQL**。两者分工不同，排查方向也不一样。

## Redis 挂了会有什么表现

Redis 出问题很少是「整个站 502」这种干净利落的死法，更多是下面这些症状，对上号就可以优先查它：

| 现象 | 背后原因 |
| --- | --- |
| 登录、注册转圈或直接报错 | 会话、验证码读写失败 |
| 订阅链接打不开或内容为空 | 订阅缓存、队列任务未执行 |
| 支付成功了但套餐没到账 | 支付后续处理卡在队列里 |
| 面板整体变慢、时不时超时 | 每次请求都回源查库，缓存全 miss |
| 后台报错 `RedisException` / `Connection refused` | 应用连不上 Redis 服务 |
| 定时任务（邮件提醒、流量结算）停了 | 队列 worker 消费不到任务 |

一个关键判断：如果你改了 Redis 配置或重启过服务器，之后出现上面任何一种症状，先不要急着重装面板，大概率是 Redis 没起来或者配置对不上。

## 第一步：确认 Redis 服务本身在不在

登录面板所在的服务器，先看 Redis 进程和端口：

```bash
# 看服务状态
systemctl status redis-server
# 或者（部分系统服务名是 redis）
systemctl status redis

# 看端口是否在监听，默认 6379
ss -tlnp | grep 6379

# 直接连一下
redis-cli ping
```

正常应该返回 `PONG`。如果 `redis-cli ping` 报 `Connection refused`，说明 Redis 根本没起来；如果返回 `PONG`，说明服务活着，问题可能在 Xboard 的连接配置上。

Redis 没起来时，先看它为什么起不来：

```bash
# 看最近的报错
journalctl -u redis-server -n 50 --no-pager

# 常见原因：内存不足被 OOM kill、配置文件语法错误、端口被占用
free -h
```

小内存机器上 Redis 被系统杀掉并不罕见，Xboard、MySQL、PHP-FPM、队列 worker 全挤在一台 1G/2G 的机器上，内存一紧张就先牺牲 Redis。如果你之前按[低内存优化清单](low-memory-vps-panel.md)做过部署，更要留意这一点——先保证 Redis 有口饭吃，而不是一味加缓存。

## 第二步：核对 Xboard 的 Redis 配置

Xboard 通过 `.env` 连接 Redis，关键的几项是：

```bash
# 面板目录下的 .env（示例值，按自己实际填写）
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=你的Redis密码
REDIS_PORT=6379
REDIS_DB=0
```

逐项核对：

1. **地址**：Redis 和面板在同一台机器上就写 `127.0.0.1`，不要写成公网 IP——既慢又暴露风险。
2. **密码**：如果 Redis 配置了 `requirepass`，`.env` 里的 `REDIS_PASSWORD` 必须一致；一边改了密码另一边没改，是最常见的「昨天还好好的」。
3. **端口**：改过 Redis 端口的话，两边同步改。
4. **数据库编号**：一般用默认的 `0` 即可，除非你在一台 Redis 上跑了多个应用，特意做了隔离。

改完 `.env` 之后，一定要清配置缓存再重启队列，否则面板读的还是旧配置：

```bash
cd /www/wwwroot/panel.example.com   # 换成你自己的面板目录
php artisan config:clear
php artisan config:cache
php artisan queue:restart
```

> 为什么要 `queue:restart`？队列 worker 是常驻进程，启动时把配置读进内存，之后 `.env` 再怎么改它都不知道。只重启 Web 服务不重启 worker，队列任务照样连旧地址。

## 第三步：检查队列 worker 是不是在跑

Redis 活着、配置也对，但邮件不发、订单处理卡住——这时问题往往在队列 worker 身上。Xboard 的队列 worker 通常由 Supervisor 托管：

```bash
# 看 Supervisor 管理的进程
supervisorctl status

# 正常应该看到类似这样的 RUNNING 状态
# panel-queue   RUNNING   pid 1234, uptime 2 days
```

如果 worker 是 `STOPPED` 或 `FATAL`，先看它的日志：

```bash
supervisorctl tail -f panel-queue stderr
```

常见原因有三个：

- **worker 进程被杀**：内存不足时 Supervisor 的子进程也会被 OOM，重启即可；
- **代码或配置更新后没重启 worker**：更新面板、改 `.env` 后都要 `supervisorctl restart panel-queue`；
- **队列里卡了一条坏任务**：某条任务反复失败重试，把队列堵住。看日志里重复出现的那条任务，先处理它，再清掉积压：

```bash
# 看队列里积压了多少任务
redis-cli LLEN queues:default

# 紧急情况下清空队列（会丢掉未处理任务，谨慎使用）
redis-cli DEL queues:default
```

> `DEL` 是核选项：积压的邮件、订单处理任务会被一并丢掉。不到万不得已不要用，先确认积压的任务是不是可以重发的类型。

## 第四步：区分「Redis 挂了」和「Redis 满了」

还有一种容易误判的情况：Redis 活着、能连上，但内存满了，新的缓存写不进去。看一眼内存使用：

```bash
redis-cli INFO memory | grep -E 'used_memory_human|maxmemory_human|evicted_keys'
```

如果 `used_memory_human` 接近 `maxmemory_human`，且 `evicted_keys` 在持续增长，说明缓存一直在被逐出——面板表现为时快时慢、订阅偶尔为空。

处理办法：

```bash
# 看当前淘汰策略
redis-cli CONFIG GET maxmemory-policy

# 临时把淘汰策略改成 allkeys-lru（优先淘汰最久没用的 key）
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

要根治的话，要么给 Redis 多分点内存（改 `maxmemory`），要么减少缓存的数据量和过期时间。不要把 `maxmemory` 设成 0（不限制）——小内存机器上 Redis 会一直吃内存，直到被系统 OOM 杀掉，回到第一步的死循环。

## 预防：让 Redis 别再悄悄挂掉

排障做完，建议顺手把这几件事做到位：

1. **开机自启**：`systemctl enable redis-server`，服务器重启后 Redis 自己起来，不用每次手动救火。
2. **设密码、只监听内网**：Redis 配置里 `bind 127.0.0.1`，并设 `requirepass`。Redis 默认没有鉴权，暴露到公网等于把缓存大门敞开。
3. **留好监控口子**：定期跑一次 `redis-cli ping` 和 `redis-cli INFO memory`，写进巡检脚本或监控告警里，Redis 一挂立刻知道，而不是等用户来报「订阅打不开了」。
4. **定期重启队列 worker**：`supervisorctl restart panel-queue` 可以作为每周巡检的一部分，worker 跑久了内存泄漏、连接断开都能顺手清掉。
5. **备份只备 MySQL 就够了吗？**：Redis 里是缓存和临时数据，丢了可以重建，不用像 MySQL 那样天天备份。但如果你把队列当作唯一通道、且任务丢了无法重发，那就得把队列持久化（AOF）和任务幂等一起考虑——这是进阶话题，小机场先保证服务活着更重要。

## 排查清单（照着走一遍）

1. `redis-cli ping` 通不通？不通 → 看 `systemctl status` 和 `journalctl`，先把服务拉起来。
2. `.env` 里的 `REDIS_HOST` / `REDIS_PASSWORD` / `REDIS_PORT` 和 Redis 实际配置一致吗？不一致 → 改完清缓存、重启队列。
3. `supervisorctl status` 里队列 worker 在跑吗？没跑 → 看日志、重启 worker。
4. 队列积压了吗？`LLEN` 数字持续变大 → 看失败任务日志，处理坏任务。
5. 内存满了吗？`INFO memory` 接近上限 → 调淘汰策略或加内存。

按这个顺序走一遍，九成 Redis 相关的问题都能定位到。定位不到时再去翻 Xboard 的应用日志 `storage/logs/laravel.log`，把报错时间点和上面几项的状态对上看——很多时候日志里的第一行报错就是答案。

## 不想自己搭？

从装面板、配 Redis、调队列、做备份，到上线后半夜起来处理「订阅又打不开了」，自己搭一个机场的真实成本不是一台 VPS 的月租，而是持续投入的排查时间。如果你只是想有个稳定可用的订阅，不妨直接用现成的：漫画机场提供开箱即用的订阅服务，标准套餐 ¥10/月起、家宽套餐 ¥30/月起，每月 100GB 流量、支持 3 台设备，详情见官网 https://vps.manhuajichang.fun 。
