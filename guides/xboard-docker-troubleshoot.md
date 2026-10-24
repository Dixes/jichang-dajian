# Xboard 部署后容器起不来：常见报错排查

按《Xboard 面板 Docker 部署教程》把 `docker compose up -d` 跑完，打开面板域名却是一片空白或 502——先别急着重装。Xboard 是多容器应用（PHP 应用 + MySQL + Redis），九成"起不来"都是单个容器的配置或资源问题。用 10 分钟顺着日志查一遍，大多数情况都能原地修好，不用推倒重来。

> 环境说明：本文命令基于 Docker Compose v2（即 `docker compose`，中间有空格）。如果你机器上还是老版的 `docker-compose`（带横杠），把命令里的 `docker compose` 整体替换成 `docker-compose` 即可，排查思路完全一样。

## 第一步：先看日志，别急着重启

容器的状态和日志才是真相。先跑这几条命令，把故障定到具体容器：

```bash
# 看所有容器状态：重点看 STATUS 列是 Up 还是 Restarting / Exited
docker compose ps

# 看最近 100 行日志，换成报错的容器名
docker compose logs --tail=100 app
docker compose logs --tail=100 mysql
docker compose logs --tail=100 redis
```

`Restarting (1)` 表示容器一直在崩溃重启，这时候 `docker compose restart` 没有任何意义——先查日志，再动手。下面的报错按出现频率排序，对照你的日志处理。

## 报错 1：端口被占用，app 容器直接退出

**日志特征：**

```
Error: bind: address already in use
```

**原因：** 宿主机上已经有进程占用了 80 或 443 端口（常见的是宿主机装过的 nginx、Caddy，或之前失败的 Xboard 残留容器）。

**解决步骤：**

```bash
# 查占用 80 端口的进程
ss -tlnp | grep ':80'

# 如果是之前失败的残留容器，彻底清理后重来
docker compose down
docker compose up -d

# 如果是宿主机上的 nginx/Caddy 占用了端口
# 方案 A：停掉它（确认面板走自己的反代后）
systemctl stop nginx

# 方案 B：改 docker-compose.yml 里 app 的映射端口
ports:
  - "8080:80"    # 宿主机 8080 -> 容器 80
  - "8443:443"
```

改完端口映射后，记得反代（Caddy/Nginx）里把上游改成对应端口，否则反代会 502。

## 报错 2：MySQL 容器反复重启（Exit 1）

这是最常见的一个，日志里通常能看到以下两种之一。

**情况 A：密码不匹配**

```
Access denied for user 'xboard'@'%' (using password: YES)
```

原因是 `docker-compose.yml` 里 `MYSQL_PASSWORD` 和 app 容器环境变量 `DB_PASSWORD` 写的不一致。两个文件里的密码必须完全相同，检查时注意有没有多敲空格或换行：

```bash
# 查看两个容器实际拿到的密码
docker compose exec mysql env | grep MYSQL_PASSWORD
docker compose exec app env | grep DB_PASSWORD
```

对齐后重建容器：`docker compose up -d --force-recreate mysql app`。

**情况 B：内存不足被 OOM Killer 干掉**

1G 小内存 VPS 的重灾区。`dmesg` 里能看到 `Out of memory: Killed process xxx (mysqld)`。MySQL 8 默认配置在小内存机器上很容易被杀。应急办法是在 compose 里给 MySQL 加限制参数：

```yaml
mysql:
  command: --innodb-buffer-pool-size=128M --performance-schema=0
```

长期方案参考仓库里的《1G 小内存 VPS 跑面板的优化清单》那篇，里面有完整的内存优化清单。

## 报错 3：Redis 连不上，面板报 500

**日志特征（app 容器日志）：**

```
Connection refused [tcp://redis:6379]
```

或页面直接 500。原因通常有三个，按顺序查：

1. **Redis 容器本身没起来**：`docker compose ps` 看 redis 的状态，如果也是 Exited，先单独看它的日志。
2. **app 里的 Redis 地址配错了**：Xboard 的 `.env` 里应该是 `REDIS_HOST=redis`（容器名），不是 `127.0.0.1`——app 容器里的 127.0.0.1 指的是它自己，根本连不到 Redis 容器。
3. **Redis 设了密码但 app 没配**：如果 compose 里给 redis 加了 `--requirepass`，app 的 `REDIS_PASSWORD` 必须同步填上。

```bash
# 进 app 容器手动测连通性
docker compose exec app php -r '$r=new Redis(); $r->connect("redis",6379); echo $r->ping();'
# 输出 +PONG 说明通了
```

## 报错 4：APP_KEY 没配，页面全站 500

如果 MySQL 和 Redis 都正常，页面还是 500，大概率是 Laravel 的应用密钥问题。app 容器日志里会有：

```
No application encryption key has been specified.
```

Xboard 部署时需要生成 `APP_KEY` 并写入 `.env`：

```bash
# 进 app 容器生成并写入
docker compose exec app php artisan key:generate

# 然后清缓存（这一步很多人漏掉）
docker compose exec app php artisan config:clear
docker compose exec app php artisan cache:clear
```

改过 `.env` 之后不清缓存，Laravel 会继续用旧配置，症状和没改一模一样。

## 报错 5：磁盘满了，所有容器起不来

`docker compose up -d` 卡住或容器秒退，但日志里看不到明显报错时，查一下磁盘：

```bash
df -h /
docker system df
```

Docker 的构建缓存和旧镜像很容易把小硬盘吃光。清理命令：

```bash
# 删除没用的镜像、容器、网络和构建缓存（谨慎：会删掉未使用的卷之外的所有悬空资源）
docker system prune -a --volumes
```

**注意：** `--volumes` 会删掉未被任何容器使用的匿名卷。如果 MySQL 的数据是用匿名卷存的，删之前先确认 `docker compose ps` 里 mysql 还在运行、卷还被挂着。更安全的做法是先给数据库做个导出备份（见《Xboard 数据库备份：MySQL 导出导入实操》）。

## 修复后的验证清单

逐项改完后，按这个顺序验证，别跳步：

```bash
# 1. 所有容器状态都是 Up
docker compose ps

# 2. 数据库迁移状态正常
docker compose exec app php artisan migrate:status

# 3. 面板首页能打开（把 panel.example.com 换成你的域名）
curl -s -o /dev/null -w "%{http_code}" https://panel.example.com

# 4. 后台能登录、节点列表页不报错
```

`curl` 返回 200 但浏览器打不开，多半是反代或证书的问题，参考《反代与证书：Caddy/Nginx + 自动续期》那篇继续查。验证时建议用无痕窗口或清掉浏览器缓存，避免缓存的旧页面误导判断。

## 常见坑：修完又犯的三种情况

1. **重启机器后面板没了**：compose 文件里没加 `restart: always`，VPS 重启后容器不会自动拉起。三个服务都加上，然后跑一次 `docker compose up -d` 让策略生效。
2. **改了 `.env` 没重建容器**：环境变量是在容器创建时注入的，改 compose 文件后要用 `docker compose up -d`（会自动重建变化的容器），只 `restart` 是不生效的。
3. **日志里全是问号乱码**：终端编码问题，不是故障。用 `docker compose logs --tail=100 app | cat` 看原始输出，或换个支持 UTF-8 的终端。

## 报错速查表

| 现象 | 大概率原因 | 先看哪个日志 |
| --- | --- | --- |
| 域名打不开 / 502 | 端口被占用、反代没配对 | `docker compose logs app` |
| 容器反复 Restarting | MySQL 密码错、内存不足 | `docker compose logs mysql` + `dmesg` |
| 页面 500 | Redis 连不上、APP_KEY 没生成 | `docker compose logs app` |
| up -d 卡住或秒退 | 磁盘满了 | `df -h`、`docker system df` |
| 重启 VPS 后面板没了 | 没加 restart 策略 | `docker compose ps` |

排查的思路其实就一条：先 `docker compose ps` 定容器，再 `logs` 定报错，按报错类型对号入座。把每次修好的步骤记下来，下次换机器部署能省一半时间。

## 不想自己搭？

说实话，上面这一套排查走下来，第一次部署 Xboard 的人平均要花一下午甚至一整天——端口冲突、密码对不上、内存不够、证书续期，每个坑单独看都不大，凑在一起就是时间黑洞。如果你只是想有个稳定可用的订阅、不想把周末花在 `docker compose logs` 上，可以直接用现成的：[漫画机场](https://vps.manhuajichang.fun) 标准套餐 ¥10/月起、家宽套餐 ¥30/月起，每月 100GB 流量、支持 3 台设备，注册就有订阅链接可用，把折腾服务器的时间省下来。
