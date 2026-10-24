# Xboard 数据库备份：MySQL 导出导入实操

面板跑久了，最怕的不是节点掉线，而是数据库出问题：升级翻车、磁盘写满、误操作删表，任何一种都可能让用户数据、订单和订阅一夜之间消失。节点可以重装，数据库丢了却很难找回来。所以 Xboard 上线后第一件该补的运维功课，就是把 MySQL 备份做成固定动作：定期导出、异地存一份、并且真的恢复演练过一次。本篇只讲 MySQL 导出导入的实操。

## 开始之前：先搞清楚数据库在哪

Xboard 的部署方式不同，MySQL 的位置也不同，备份命令写错位置是最常见的翻车点。动手前先确认三件事：数据库跑在宿主机上还是 Docker 容器里；库名和用户名是什么；备份文件存到哪里。

```bash
# 宿主机直装：确认 MySQL 是否在运行
systemctl status mysql

# Docker 部署：列出容器，确认哪个是数据库容器
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"

# 看磁盘剩余空间，备份目标目录至少留出数据库大小 1 倍以上的空闲
df -h /backup
```

> 为什么要先区分宿主机和容器？宿主机上的 `mysqldump` 直接就能连数据库，容器里的数据库却要先 `docker exec` 进容器。命令写错地方，轻则连不上，重则备份了一个空库还以为成功了。

## 导出：mysqldump 的标准写法

`mysqldump` 是 MySQL 自带的逻辑备份工具，它把表结构和数据导成一个 SQL 文本文件，恢复时只要把这个文件重新执行一遍就行。它的好处是跨版本、跨机器都好用，对绝大多数 Xboard 站点来说完全够用。

```bash
# 宿主机直装：导出单个库（占位符换成你自己的库名）
# 备份文件按日期命名，方便以后找
mysqldump -u root -p xboard_db > /backup/xboard_db_$(date +%F).sql

# Docker 部署：进数据库容器导出，文件落在宿主机
# db 是数据库容器的名字，以 docker ps 看到的为准
docker exec db mysqldump -u root -p xboard_db > /backup/xboard_db_$(date +%F).sql
```

> 为什么导出命令里不直接写密码？`mysqldump -pYourPassword` 会把密码明文留在 shell 历史和进程列表里。只写 `-p`，让它交互式问你要密码，才是安全的做法。

面板还在对外服务、不能停机时，导出建议加上下面这组参数：

```bash
# 单事务 + 快速导出 + 触发器和事件一起带上
mysqldump -u root -p \
  --single-transaction \
  --quick \
  --routines \
  --events \
  --triggers \
  xboard_db > /backup/xboard_db_$(date +%F).sql
```

> 为什么 `--single-transaction` 这么重要？它让导出在一个事务快照里完成，导出来的数据是“某一个时刻一致的状态”。没有它，导出时正好有用户下单或节点上报流量，备份里的数据就可能前后对不上，恢复后出现订单和流量对不上的怪账。注意这个参数只对 InnoDB 表有效，Xboard 默认用的就是 InnoDB。

导完第一件事不是收工，而是检查备份文件是不是真的有内容：

```bash
# 文件大小不能是 0，头部应该能看到建表语句
ls -lh /backup/xboard_db_*.sql
head -n 50 /backup/xboard_db_$(date +%F).sql

# 顺手压缩一份，省磁盘也方便异地传输
gzip /backup/xboard_db_$(date +%F).sql
```

## 导入：恢复到新库的完整步骤

备份的价值只有在恢复成功时才兑现。最常见的场景有两个：一是原库坏了要原地恢复，二是迁服务器、要把数据搬到新机器的新库里。两个场景步骤大同小异，区别只在目标库是不是空的。

```bash
# 1. 先建一个空的目标库（原地恢复时可跳过，直接用原库名）
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS xboard_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 2. 把备份文件导入目标库
mysql -u root -p xboard_db < /backup/xboard_db_2026-10-24.sql

# 备份是 gzip 压缩过的，管道一步到位
gzip -dc /backup/xboard_db_2026-10-24.sql.gz | mysql -u root -p xboard_db

# Docker 部署：把文件拷进容器再导入
docker cp /backup/xboard_db_2026-10-24.sql db:/tmp/xboard_db.sql
docker exec -i db mysql -u root -p xboard_db < /backup/xboard_db_2026-10-24.sql
```

> 为什么导入前要确认字符集是 utf8mb4？Xboard 里有用户昵称、公告、工单内容，难免出现 emoji 或生僻字，旧的 utf8 存不下四字节字符，导入时要么报错，要么悄悄变成问号。建库时显式指定 utf8mb4，这类乱码问题一开始就避掉了。

导入时面板最好先停掉，或者至少确认没有用户正在下单和支付。导入是覆盖写，导一半时面板还在写新订单，导完这些订单就可能丢了。原地恢复的标准顺序是：停面板 → 导入 → 启动面板 → 检查日志。

```bash
# 原地恢复时的停服与重启顺序（容器名以实际为准）
docker stop xboard
docker exec -i db mysql -u root -p xboard_db < /backup/xboard_db_2026-10-24.sql
docker start xboard
docker logs xboard --tail 50
```

如果导入报“表已存在”，先别急着删库重来。`mysqldump` 默认导出的文件里不带删表语句，往非空库里导入就会撞上已有的表。确认目标库确实应该被覆盖，再删库重建——但这一步会清空目标库，执行前务必确认备份文件本身是好的。

## 定时备份：写成脚本 + cron，每天自动跑

手动备份的问题不是不会，而是不坚持。一旦靠记性，迟早会在最需要它的那天发现上一份备份是一个月前的。导出命令确定没问题后，立刻把它写成脚本，交给 cron 每天自动跑。

```bash
# 备份脚本：/usr/local/bin/backup-xboard.sh
cat > /usr/local/bin/backup-xboard.sh <<'EOF'
#!/bin/bash
set -e
BACKUP_DIR="/backup/xboard"
mkdir -p "$BACKUP_DIR"
DATE=$(date +%F)
# 宿主机直装写法；Docker 部署把下面一行换成 docker exec 的版本
mysqldump -u backup -p"$MYSQL_BACKUP_PASSWORD" \
  --single-transaction --quick --routines --events --triggers \
  xboard_db > "$BACKUP_DIR/xboard_db_${DATE}.sql"
gzip -f "$BACKUP_DIR/xboard_db_${DATE}.sql"
# 只保留最近 14 天，自动清理旧备份，避免磁盘被塞满
find "$BACKUP_DIR" -name "xboard_db_*.sql.gz" -mtime +14 -delete
EOF

chmod +x /usr/local/bin/backup-xboard.sh

# 给备份建一个专用账号：只有读权限，没有写权限，就算泄露也改不了数据
mysql -u root -p -e "CREATE USER 'backup'@'localhost' IDENTIFIED BY '一个强密码';"
mysql -u root -p -e "GRANT SELECT, SHOW VIEW, TRIGGER, LOCK TABLES, EVENT ON xboard_db.* TO 'backup'@'localhost'; FLUSH PRIVILEGES;"

# 每天凌晨 3 点执行备份，日志重定向到文件，失败了才看得见
crontab -e
# 加入下面这一行
0 3 * * * /usr/local/bin/backup-xboard.sh >> /var/log/xboard-backup.log 2>&1
```

> 为什么选凌晨 3 点？导出大库时会占用 CPU 和 IO，业务高峰期跑备份等于给面板加负载。凌晨用户最少，对在线服务的影响最小。

最后一步，也是最容易被跳过的一步：异地存一份。备份文件只放在同一台机器上，机器磁盘坏了、VPS 被删了，备份和数据库会一起消失，至少做到以下一种：

```bash
# scp 到另一台机器（占位符换成你自己的备份机地址和用户）
scp /backup/xboard_db_2026-10-24.sql.gz user@203.0.113.10:/backup/xboard/

# 或用 rclone 同步到对象存储（需先配好 rclone 的 remote）
rclone copy /backup/xboard/ remote:xboard-backup/ --include "xboard_db_*.sql.gz"
```

## 验证：恢复演练才是备份完成的标志

没恢复过的备份等于没备份。验证不需要动生产库，在测试库里把最近一份备份导进去，查几张关键表确认数据在、行数对得上，就算一次有效的演练。

```bash
# 在测试库里演练恢复（不要在生产库名上直接练）
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS xboard_restore_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
gzip -dc /backup/xboard_db_2026-10-24.sql.gz | mysql -u root -p xboard_restore_test

# 抽查关键表和表清单，表数量要和生产库对得上
mysql -u root -p -e "SELECT COUNT(*) FROM xboard_restore_test.users;"
mysql -u root -p -e "SHOW TABLES FROM xboard_restore_test;"

# 演练完清理测试库，别让它长期占着磁盘
mysql -u root -p -e "DROP DATABASE xboard_restore_test;"
```

> 为什么表数量也要对？导出账号权限不足时，某些表会被悄悄跳过，导出过程还不报错。演练时表对不上，第一时间去查备份账号的权限，而不是怀疑导入命令。建议每月拿最新一份备份演练一次，确认能导进去、表全、行数合理。

## 常见坑汇总

- **备份了空库还以为成功了**：容器和宿主机命令写错地方最容易出这个，导完先看大小、看头部内容，别只看命令退出了没报错。
- **密码写进 shell 历史**：导出只写 `-p` 交互输入，脚本里用专用备份账号，别把 root 密码明文写进命令和脚本。
- **没加 `--single-transaction`**：导出过程中用户还在下单，备份数据前后不一致，恢复后订单和流量对不上账。
- **字符集没统一成 utf8mb4**：导入后 emoji 变问号，建目标库时就显式指定字符集，别等乱码了再返工。
- **往非空库导入撞表**：默认导出不带删表语句，目标库非空会报“表已存在”，先确认该不该覆盖，再决定删库重建。
- **备份只存本机**：磁盘坏了备份跟着没，至少 scp 到另一台机器或同步到对象存储，异地那一份才是真保险。
- **从来没做过恢复演练**：没导进去过的备份不算备份，每月在测试库里演练一次，表全、行数对得上才算数。

如果排查容器问题时卡住，也可以对照本仓库的[Xboard 部署后容器起不来：常见报错排查](xboard-docker-troubleshoot.md)先确认数据库容器本身是正常运行的——备份命令连不上时，先分清是容器没起来，还是备份命令写错了位置。

数据库备份难的从来不是命令本身，而是把它变成每天自动跑、每月演练过的固定动作。一条 `mysqldump` 命令加一个 cron，再加一份异地备份，面板出问题时你才有底气说“数据丢不了，大不了恢复一次”。

## 不想自己搭？

上面这套走下来，从确认数据库位置、写导出命令、搭定时备份和异地同步、再到每月做一次恢复演练，顺利的话也要花上大半天，而且面板一升级、部署方式一变，备份脚本还得跟着改。如果你只是想有个现成可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
