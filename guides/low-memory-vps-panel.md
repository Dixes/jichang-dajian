# 1G 小内存 VPS 跑面板的优化清单

1G 内存的 VPS 便宜，但跑机场面板（比如 Xboard）很容易踩到内存红线：面板本身不算大，可 MySQL、PHP-FPM、Redis、队列进程、Nginx 全加起来，1G 内存眨眼就满，一满就 OOM 杀进程，表现为面板时不时 502、后台打不开、定时任务悄悄停摆。下面是一份按实操顺序来的优化清单，目标是让 1G 机器把面板稳稳跑起来，而不是勉强"能打开"。

> 前置：本文假设面板已装好、能打开，只是经常卡顿或 OOM。如果是容器根本起不来，先看本仓库的 [Xboard 部署后容器起不来：常见报错排查](xboard-docker-troubleshoot.md)，先把部署问题排掉再谈优化。

## 第一步：先看清内存花在哪

优化之前先量现状，否则全凭感觉调参数。连上 VPS 跑三条命令：

```bash
# 总览：用了多少、还剩多少、Swap 用了多少
free -h

# 按内存占用排进程，看谁是大头（RES 列是真实占用的物理内存）
ps aux --sort=-%mem | head -15

# 最近有没有进程被 OOM 杀掉（有输出就说明已经爆过内存了）
dmesg | grep -i "out of memory" | tail -5
```

1G 机器的典型内存账本是这样的：MySQL 约 300-400MB、PHP-FPM 每个子进程约 30-50MB、Redis 约 50-100MB、队列 Worker 每个约 50MB、Nginx 约 20-30MB。加起来还没算系统本身，账已经快算不过来了。所以优化思路就一条：**每个组件都按 1G 的量给配额，不让任何一个组件按默认配置敞开吃。**

## 第二步：加 Swap，兜住突发峰值

Swap 不是用来长期跑业务的，它是防止突发峰值直接 OOM 的安全垫。1G 内存的机器建议加 1G-2G Swap：

```bash
# 创建 2G 的 Swap 文件并启用（1G 内存机器 2G Swap 是常用配比）
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# 写入 fstab，重启后仍然生效
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 确认生效
free -h
```

> 为什么 Swap 要设 600 权限？Swap 文件里可能换出进程的内存数据（包括敏感信息），权限放开等于把内存内容摆在所有用户面前。600 是 Swap 文件的标准做法。

另外把 `swappiness` 调低一点，让系统尽量用物理内存、只在真不够时才换出：

```bash
# 临时生效
sysctl vm.swappiness=10
# 永久生效
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

## 第三步：MySQL 是最大头，先砍它

默认安装的 MySQL 是按几 G 内存的服务器调的，1G 机器上必须手动收紧。编辑 MySQL 配置（一般在 `/etc/mysql/my.cnf` 或 `/etc/my.cnf`，找不到就用 `mysqld --verbose --help | grep -A1 "Default options"` 查）：

```ini
[mysqld]
# InnoDB 缓冲池是 MySQL 内存大头，1G 机器给 128M 就够了
innodb_buffer_pool_size = 128M
# 每个连接的排序/Join 缓冲，默认太大，改小
sort_buffer_size = 256K
join_buffer_size = 256K
# 限制最大连接数，面板场景 30 个连接绰绰有余
max_connections = 30
# 关掉 Performance Schema，省约几十 MB
performance_schema = OFF
```

改完重启 MySQL 再看 `free -h`，一般能立刻省出 200MB 左右。

> 为什么 `innodb_buffer_pool_size` 敢砍到 128M？因为机场面板的数据库很小（用户、订单、节点配置加起来通常不到几百 MB），缓冲池不需要多大；面板场景读多写少，命中率差一点不影响使用。给面板数据库配 1G 缓冲池是纯浪费。

## 第四步：PHP-FPM 子进程数按内存算

PHP-FPM 默认的 `pm.max_children` 在很多发行版里是 50，1G 机器上 50 个子进程全起来能吃掉 2G 内存——这就是面板一有并发就 OOM 的常见原因。按公式算：`(可用内存 - 其他组件预留) / 单个进程内存`。1G 机器留给 PHP 约 300MB、单个进程 40MB，算下来 `max_children` 设 5-8 就够了：

```ini
; PHP-FPM 池配置（一般在 /etc/php/*/fpm/pool.d/www.conf）
pm = dynamic
pm.max_children = 8
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 4
; 每个子进程处理 500 个请求后回收，防止内存缓慢泄漏越长越大
pm.max_requests = 500
```

改完重启 PHP-FPM：

```bash
systemctl restart php8.2-fpm   # 版本号按你实际安装的改
```

> 为什么用 dynamic 而不用 static？static 会一启动就拉满全部子进程，1G 机器上等于开局就把内存占满；dynamic 按需拉起，闲时只留 2 个，面板这种低频访问场景更省。

## 第五步：Redis 和队列 Worker 也要设上限

Redis 默认没有内存上限，数据一多就一直涨。给 Redis 设上限并配淘汰策略：

```bash
# redis-cli 里执行，或写进 redis.conf
CONFIG SET maxmemory 64mb
CONFIG SET maxmemory-policy allkeys-lru
```

> 为什么淘汰策略选 allkeys-lru？面板用 Redis 主要放缓存和队列，数据丢了可以重建；内存满时按"最近最少使用"淘汰旧缓存，比直接报错、比写不进队列安全得多。

队列 Worker（Xboard 里跑订阅同步、邮件发送这些后台任务）也别多开：1G 机器开 1 个就够了，命令示例：

```bash
# 只开 1 个 Worker，并用 supervisor 或 systemd 守护，挂了自动拉起
php artisan queue:work --sleep=3 --tries=3 --max-time=3600
```

> 为什么 Worker 只能开 1 个？每个 Worker 常驻约 50MB，面板的后台任务量本来就不大，1 个 Worker 排队处理完全够用；开 3 个 Worker 只是让三个进程抢同一份内存。

## 第六步：Nginx 和日志也别浪费

Nginx 本身很轻，但两个地方容易被忽略：

```nginx
# 面板站的 Nginx 配置片段
# 1. 关掉 access_log 的缓冲浪费？不，恰恰相反：小机器上日志要限量
# 日志按天切割、只保留 7 天，避免日志文件把磁盘和内存拖垮
access_log /var/log/nginx/panel.access.log;
error_log /var/log/nginx/panel.error.log warn;
```

```bash
# 用 logrotate 限制日志（一般系统自带，确认保留期）
# /etc/logrotate.d/nginx 里把 rotate 改成 7
# 另外：面板的 storage/logs/laravel.log 也要定期清，几个月不清理能涨到几百 MB
find /path/to/panel/storage/logs -name "*.log" -mtime +7 -delete   # 路径按实际安装目录改
```

## 第七步：1G 机器上不该干的事

清单做到这里，还要守住几条"不做"：

1. **面板机和节点端别挤同一台 1G 机器**：面板（MySQL+PHP+Redis）加节点端（Xray/V2bX）全塞一台，内存铁定不够用。面板机和落地机分离是正道，分离部署的注意事项见池子里的后续文章（面板机与落地机分离部署）。
2. **别在 1G 机器上编译东西**：Node.js 构建、Composer 全量安装、Docker 镜像构建都是内存大户，构建时 OOM 是常态。构建放本地或 CI 机器做，1G 机器只负责运行。
3. **别装用不上的组件**：面板用不到的 Elasticsearch、额外的监控 Agent、图形界面，能不装就不装，每个常驻进程都是内存。
4. **Docker 用户注意**：Docker 本身约占 100MB 以上，1G 机器上跑 Docker 版面板要把上面所有配额再压一档；如果内存实在吃紧，直接部署（不用 Docker）是更省的选择。
5. **别把 Swap 当内存用**：`free -h` 里 Swap 长期占用超过一半，说明物理内存真的不够，优化到头了就该升级配置——Swap 跑业务会让面板慢到没法用。

## 验证：优化完怎么确认有效

按清单改完，跑一轮验证再收工：

```bash
# 1. 内存余量：available 一栏最好长期保持 150MB 以上
free -h

# 2. 压一下面板：连续请求 20 次，看有没有 502、内存有没有暴涨
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code}\n" https://panel.example.com; done

# 3. 观察 24 小时：第二天再看 free -h 和 dmesg，确认没有新的 OOM 记录
dmesg | grep -i "out of memory"
```

三项都通过，这台 1G 机器才算真正"跑稳了"：面板日常打开流畅、突发流量不 OOM、后台任务不丢。

## 常见坑

1. **只加 Swap 不调参数**：Swap 兜住了 OOM，但 MySQL 和 PHP-FPM 还是按默认配置吃内存，面板照样卡。Swap 是安全垫，不是优化本身。
2. **max_children 设太高**：以为子进程越多并发越好，结果 1G 机器上并发一上来就 OOM。面板场景用户量不大，8 个子进程够了。
3. **改了配置没重启**：MySQL、PHP-FPM、Redis 的配置改完都要重启对应服务才生效，改完不重启等于没改。
4. **日志爆磁盘**：内存优化完了，磁盘被日志塞满，MySQL 写不进数据照样挂。日志切割和清理要一起做。
5. **面板和节点同机还开 Docker**：1G 机器 + Docker + 面板全家桶 + 节点端，四重负担叠加，再怎么调参数也救不回来，先分离再优化。

记住 1G 机器的优化口诀：**先量再调、每个组件都设上限、Swap 只兜底、不该装的不装、扛不住就升级。**

## 不想自己搭？

按这份清单从量内存、调 MySQL、压 PHP-FPM 子进程、设 Redis 上限一路做到验证，顺利的话也要花大半天，还不算之后每次面板卡顿都要回来翻配置、查 OOM 日志的持续运维。如果你只是想有个稳定可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
