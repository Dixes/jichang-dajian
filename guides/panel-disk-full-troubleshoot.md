# 面板机磁盘爆满导致订阅生成失败：日志定位与清理实操

面板跑着跑着，某天用户突然反馈订阅链接打不开、后台保存配置报错，或者面板容器反复重启，登录服务器一看，磁盘使用率已经 100%。磁盘满不是小毛病：MySQL 写不进数据、日志写不进文件、证书续期也可能跟着失败，症状五花八门，根因却常常只有一个——磁盘满了。

本文以常见的 Xboard 面板机为例，讲一套从定位到清理、再到预防的完整流程。示例中的域名和 IP 均为占位符，请换成你自己的实际值。

## 问题长什么样

磁盘爆满的典型表现有几种，对上任何一种都可以先查磁盘：

- 面板后台报错，提示写入失败或数据库错误；
- 订阅链接返回 500，或者间歇性打不开；
- 容器状态反复在重启和运行之间横跳；
- 定时任务、备份任务突然不再执行，也没有任何新日志产生——因为日志本身已经写不进去了。

先不要急着重启面板。重启解决不了磁盘满，容器起来后写不进数据，还是会立刻再出问题。先确认磁盘状态：

```bash
df -h
df -i
```

第一条看空间，第二条看 inode。空间还剩很多但依然写不进文件，多半是 inode 耗尽——通常是海量小文件（比如失控的会话文件、碎片化的缓存）造成的，这时删大文件没用，要删数量。

## 第一步：找出是谁占了空间

用 `du` 从根目录一级一级往下钻，不要一上来就 `rm`：

```bash
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/lib/docker/* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* 2>/dev/null | sort -rh | head -20
```

机场面板机上最常见的几个大户，按出现频率排序：

1. **Docker 悬空镜像和构建缓存**：面板升级、重建过几次之后最容易堆起来。
2. **容器日志**：某个节点端或面板容器一直在报错刷屏，日志文件几个月就能涨到几 GB。
3. **MySQL 的 binlog**：开了 binlog 又没设过期时间，备份没做、日志一直在攒。
4. **面板的备份文件和上传文件**：定时备份只生成不清理，旧备份越攒越多。
5. **系统 journal**：`journald` 默认不设上限，长期运行的机器上也可能吃掉几个 GB。

定位到具体目录后，再看里面最大的文件，不要整目录端掉：

```bash
du -ah /var/log 2>/dev/null | sort -rh | head -20
ls -lh /var/lib/docker/containers/*/*-json.log 2>/dev/null | head
```

## 第二步：安全清理，分清能删和不能删

### 能放心清理的

Docker 的悬空资源可以先做一次无害清理。`docker system prune` 只清悬空镜像、停止的容器和无用网络，不动正在运行的容器；但带 `-a` 会删掉所有未被使用的镜像，执行前先确认没有要保留的旧版本镜像：

```bash
docker ps
docker system prune
docker system df
```

容器日志如果已经确认是刷屏的旧日志，可以截断而不是删文件——正在运行的容器持有文件句柄，直接删文件空间未必释放，截断才立竿见影：

```bash
# 先确认是哪个容器的日志在涨
ls -lh /var/lib/docker/containers/*/*-json.log
# 截断（把 <容器长ID> 换成实际值）
: > /var/lib/docker/containers/<容器长ID>/<容器长ID>-json.log
```

系统 journal 也可以按时间和大小双重清理，`--vacuum-size` 是硬上限，比只按时间清理更可靠：

```bash
journalctl --disk-usage
journalctl --vacuum-size=500M
```

### 要先备份再动的

MySQL 的 binlog 和面板备份目录属于“删之前先想想”的类型。清理 binlog 之前，先确认最近一次数据库备份是可用的——备份文件本身如果也存在磁盘满的机器上，等于没备份。相关流程可以参考本仓库的另一篇教程：《Xboard 数据库备份：MySQL 导出导入实操》（https://github.com/Dixes/jichang-dajian/blob/main/guides/xboard-mysql-backup.md）。

```bash
# 只清理 7 天前的 binlog，保留近期可用于恢复的部分
mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);"
# 查看 binlog 占用
mysql -u root -p -e "SHOW BINARY LOGS;"
```

面板的旧备份文件，保留最新两份、删掉更早的即可，不要全清：

```bash
ls -lht /opt/xboard/backup/ 2>/dev/null | head -20
# 保留最新 2 份，删掉其余（目录换成你自己的备份路径）
ls -t /opt/xboard/backup/* | tail -n +3 | xargs -r rm --
```

清理完立刻再跑一次 `df -h`，确认使用率降下来了，再重启面板容器：

```bash
df -h
docker restart xboard
docker ps
```

这里的容器名 `xboard` 只是示例，换成你 `docker ps` 里看到的实际名字。

## 第三步：验证恢复

不要只看面板首页能打开就收工。按这个顺序验一遍，全部通过才算恢复：

1. **面板后台**：能登录、能保存一次配置、改动能生效；
2. **订阅链接**：用一个测试订阅链接请求一次，返回正常、内容不是空的；
3. **数据库写入**：新建一个测试用户或改一次套餐，确认写得进去；
4. **节点同步**：看节点端日志，确认面板到节点的数据同步恢复了；
5. **定时任务**：手动触发一次备份或结算任务，确认能跑完。

```bash
# 订阅链接自测（把地址换成你自己的测试订阅）
curl -s -o /dev/null -w "%{http_code}\n" "https://panel.example.com/api/v1/client/subscribe?token=test"
# 看面板容器最近的日志，确认没有新的写入报错
docker logs --tail 50 xboard
```

如果订阅返回正常、日志里没有新的报错，才算真正恢复。否则回到第二步，继续查还有没有第二个大户——磁盘满常常是两个问题叠加，比如日志和 binlog 同时爆。

## 第四步：防止下次再满

清理是一次性的，不做预防，几个月后还会重演。建议把下面四项一次性配好，花的时间比下次半夜救火少得多：

**1. 给容器日志设上限。** 在 `docker-compose.yml` 里给面板服务加上日志轮转配置，`max-size` 是单个日志文件的上限，`max-file` 是保留几个轮转文件，两个参数配合起来，日志总量就被锁死了：

```yaml
services:
  xboard:
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"
```

改完之后要 `docker compose up -d` 重建容器配置才生效，只改文件不重建等于没改。

**2. 给 journald 设上限。** 在 `/etc/systemd/journald.conf` 里加上：

```ini
SystemMaxUse=500M
```

然后重启服务：`systemctl restart systemd-journald`。

**3. 给 binlog 设过期时间。** 在 MySQL 配置里加上 `binlog_expire_logs_seconds = 604800`（7 天），重启 MySQL 后生效，老日志会自动过期，不用再手动 purge。

**4. 加一个磁盘告警。** 不需要上重型监控，一个定时任务就够了：

```bash
# 每天检查一次，使用率超过 80% 就写日志提醒
cat > /usr/local/bin/disk-check.sh <<'EOF'
#!/bin/bash
USE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$USE" -ge 80 ]; then
  echo "$(date): 磁盘使用率 ${USE}%，请及时清理" >> /var/log/disk-alert.log
fi
EOF
chmod +x /usr/local/bin/disk-check.sh
echo "0 9 * * * root /usr/local/bin/disk-check.sh" >> /etc/crontab
```

80% 这个阈值是留给人工处理的时间：从 80% 涨到 100% 通常还有几天，足够从容清理；等到 95% 才发现，往往已经伴随业务报错了。

## 常见坑

- **删了文件空间没释放**：文件被运行中的进程占用着句柄，删文件只删了目录项，空间要等进程退出或文件被截断才释放。容器日志优先用截断，不用删除。
- **清理时误删了正在用的数据库文件**：只删日志、备份和缓存，绝不要直接动 MySQL 的数据目录。拿不准的文件先 `mv` 到别处、观察一天再删。
- **`docker system prune -a` 删了要回滚的旧镜像**：升级面板之前，先确认新版本跑稳了再清旧镜像；刚升级完就清，等于放弃了最快的回滚路径。
- **只清空间不查 inode**：`df -h` 看着还有空间但写不进文件时，一定要跑 `df -i`。inode 满了要删的是小文件数量，不是大文件体积。
- **清理完不验证节点同步**：面板能打开不等于业务恢复了，节点同步断过一次之后，要专门确认节点端拿到了最新配置，否则用户侧还是旧的、甚至失效的订阅。

## 不想自己搭？

从定位大户、清理日志、配轮转上限到加磁盘告警，上面这一套做下来，一台面板机从部署到长期运维要持续投入的时间并不少，尤其磁盘、备份、证书这类脏活是按月复发的。如果你只是想先用起来、不想自己守一台面板机，漫画机场的现成订阅是个替代选项：标准套餐 ¥10/月起，家宽套餐 ¥30/月起，100GB/月流量，支持 3 台设备，官网见 https://vps.manhuajichang.fun 。
