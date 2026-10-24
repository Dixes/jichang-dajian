# 面板数据备份与服务器迁移

面板里是全部用户数据和订单，节点上有手写的配置，丢了任何一个都是灾难。这篇给出备份清单、自动备份脚本和整机迁移流程，照着做一次，以后换机器、换商家都不慌。

## 准备工作

先理清要备份什么：

```bash
# Xboard 数据（Docker volume）
docker volume ls | grep xboard
# 数据库位置（按实际部署确认）
ls /var/lib/docker/volumes/ | grep -i mysql
# 节点配置
ls /etc/sing-box/config.json /opt/V2bX/ 2>/dev/null
# 证书
ls /etc/letsencrypt/live/
```

核心是三样：面板数据库（用户、订单、套餐）、节点配置文件、证书。其他都可以重装，唯独数据库丢了找不回来。

## 自动备份脚本

```bash
#!/bin/bash
# /root/backup.sh
DATE=$(date +%Y%m%d)
BACKUP_DIR=/root/backups
mkdir -p $BACKUP_DIR
# 1. 备份数据库（Xboard 的 MySQL 容器为例）
docker exec xboard-db mysqldump -u root -p'数据库密码' xboard > $BACKUP_DIR/xboard-$DATE.sql
# 2. 打包配置文件和证书
tar -czf $BACKUP_DIR/configs-$DATE.tar.gz /etc/sing-box /etc/letsencrypt /opt/V2bX 2>/dev/null
# 3. 只保留最近 7 天
find $BACKUP_DIR -mtime +7 -delete
echo "备份完成：$DATE"
```

`mysqldump` 导出的是完整 SQL，恢复时直接导入；`find -mtime +7 -delete` 自动删旧备份，防止磁盘被备份撑爆。加到 crontab 每天凌晨跑：

```bash
crontab -e
# 每天 3 点备份
0 3 * * * /root/backup.sh >> /var/log/backup.log 2>&1
```

备份文件还要异地存一份：用 `scp` 或 `rclone` 同步到另一台机器或对象存储。只存本机的备份，机器挂了备份一起没。

## 服务器迁移流程

```bash
# 1. 新机器装好系统和 Docker
# 2. 传备份过去
scp /root/backups/xboard-最新日期.sql root@新IP:/root/
scp /root/backups/configs-最新日期.tar.gz root@新IP:/root/
# 3. 新机器恢复数据库
docker exec -i xboard-db mysql -u root -p'数据库密码' xboard < /root/xboard-最新日期.sql
# 4. 恢复配置和证书
tar -xzf /root/configs-最新日期.tar.gz -C /
# 5. 重启服务
systemctl restart sing-box
```

关键顺序：先在新机器把服务都装好，恢复数据，测试通过，最后再切 DNS。DNS TTL 提前改小，切换后观察 10 分钟确认无异常，再下线旧机器。

## 验证

```bash
# 数据库记录数对比
# 旧机器：
docker exec xboard-db mysql -u root -p'密码' -e "SELECT COUNT(*) FROM users;" xboard
# 新机器执行同样命令，数字一致
# 证书有效性
openssl x509 -in /etc/letsencrypt/live/域名/cert.pem -noout -dates
# 节点实际连接测试
```

用户数、订单数对上，证书有效，测试账号能连，迁移才算完成。

## 常见坑

一是从没备份，等机器挂了才想起来。二是备份了但没验证过，恢复时发现 SQL 文件是空的或损坏的——每季度做一次恢复演练。三是数据库密码含特殊字符，mysqldump 命令里没加引号导致备份失败。四是迁移时先切了 DNS 才发现新机器有问题，回滚手忙脚乱。五是旧机器销毁太快，新机器跑了三天才发现少了个配置文件。

## 不想自己搭？

备份和迁移是典型的「平时用不上、出事要人命」的工作：脚本要写、定时任务要盯、恢复要演练，全是持续投入。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
