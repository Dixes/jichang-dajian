# MySQL 8.0 下 Xboard 报 caching_sha2_password 认证失败：面板连库排查

面板从 MySQL 5.7 升级到 8.0，或者新装了一台默认就是 MySQL 8.0 的面板机之后，最容易撞上的一个报错就是认证失败：面板打不开、后台登录转圈、日志里反复出现 `caching_sha2_password` 字样。账号密码明明没改过，数据库服务也在运行，问题就出在 MySQL 8.0 换了默认认证插件上。本文讲怎么确认是不是这个问题、按部署方式怎么修、修完怎么验证。示例中的域名、IP 和密码均为占位符，请换成你自己的实际值。

## 报错长什么样，先别急着改密码

典型的报错有下面几种，对上任何一种，都先按认证插件的方向查，不要先改密码、也不要先重装面板：

- 面板页面报数据库连接失败，日志里出现 `caching_sha2_password`、`Authentication plugin` 或 `Access denied`；
- 命令行用 root 能登录 MySQL，但面板用的那个账号就是连不上；
- 升级前一切正常，升级到 MySQL 8.0 或者换了一台新面板机之后立刻出问题。

根因一句话：MySQL 8.0 默认给新用户分配的认证插件是 `caching_sha2_password`，而面板那一侧的数据库客户端版本较旧，不支持这种新认证方式，于是握手直接失败。这不是密码错了，而是双方在认证方式上没谈拢。先确认数据库和面板的部署位置，再动手：数据库跑在宿主机上还是 Docker 容器里，决定了下面命令该在哪里执行。

## 第一步：确认是不是认证插件的问题

先看面板日志，确认报错确实和认证有关，而不是数据库没启动、端口没通：

```bash
# Docker 部署：看面板容器的最近日志
docker logs --tail 100 xboard 2>&1 | grep -i -E "caching_sha2|Access denied|SQLSTATE"

# 宿主机部署：看面板的运行日志，路径以你自己的部署为准
tail -n 100 /var/www/xboard/storage/logs/laravel.log 2>/dev/null | grep -i -E "caching_sha2|Access denied"
```

如果日志里没有认证字样，先别往下走：先确认 MySQL 是否在运行、面板配置里的地址和端口对不对。确认是认证问题之后，进 MySQL 看一眼面板账号当前用的到底是哪个插件：

```bash
# 以 root 登录 MySQL，查看用户、来源主机和认证插件
mysql -u root -p -e "SELECT user, host, plugin FROM mysql.user WHERE user IN ('root', 'xboard');"
```

如果面板账号那一行 `plugin` 一栏显示的是 `caching_sha2_password`，基本就可以断定是它。再顺手看一眼面板配置里写的是哪个账号、连的是哪个主机，避免修错账号：

```bash
# 面板配置里的数据库账号、地址一般长这样，值以你自己的为准
grep -E "DB_HOST|DB_USERNAME|DB_DATABASE" /var/www/xboard/.env
```

这里要特别注意 `host` 这一栏：`xboard@localhost` 和 `xboard@%` 是两个不同的账号。面板如果通过容器网络或者内网 IP 连库，修了 `localhost` 那个账号是没用的，必须修配置里实际用的那个来源主机对应的账号。

## 第二步：三种修法，按部署方式选一种

动手之前先备份数据库，改账号出问题还能恢复。备份的具体做法可以对照本仓库的[Xboard 数据库备份：MySQL 导出导入实操](xboard-mysql-backup.md)，不要在没备份的情况下直接改生产库账号。

### 修法一：把面板账号改回 mysql_native_password（最常用）

这是最直接的修法：不改数据库版本、不改面板代码，只把面板账号的认证插件换成旧客户端普遍支持的 `mysql_native_password`。换插件要连同密码一起重设，因为改插件时 MySQL 需要按新插件重新生成密码散列，只改插件名不给密码是不会生效的：

```sql
-- host 写成面板实际连库用的来源主机，不要照抄 localhost
ALTER USER 'xboard'@'%' IDENTIFIED WITH mysql_native_password BY '换成你自己的强密码';
FLUSH PRIVILEGES;

-- 改完立刻复查，确认 plugin 一栏已经变了
SELECT user, host, plugin FROM mysql.user WHERE user = 'xboard';
```

为什么是 `FLUSH PRIVILEGES`？这条命令是让 MySQL 立即重新加载权限表，否则刚改的账号信息可能要等到服务重启才生效，面板那边还是会继续报认证失败。改完之后不要只改 root 账号：面板配置里写的是哪个账号就改哪个账号，root 只用来执行这条修改命令，日常连库不要用 root。

### 修法二：新装 MySQL 8.0 时直接指定默认插件

如果你还没装数据库、或者可以接受重装一个空库，就可以在启动时直接把默认插件定成 `mysql_native_password`，之后新建的用户默认就是旧插件，不用逐个再改。Docker 部署的写法是在启动命令里加一个参数：

```bash
# 只适用于新建空库：已有数据的容器加这个参数不会把旧用户改掉
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD='换成你自己的强密码' \
  --command --default-authentication-plugin=mysql_native_password \
  mysql:8.0
```

这条路只对新库有效，已经建好的用户不会因为改了默认插件就自动跟着变。已有数据的面板机不要图省事重装数据库，直接用修法一改账号，否则数据就没了。

### 修法三：给面板单独建一个新账号，不动旧账号

如果你不想动现有账号，或者不确定旧账号还被哪些服务共用，就新建一个面板专用账号，插件直接指定为 `mysql_native_password`，再把面板配置切到新账号。这条路最稳，出问题回滚也只要把配置改回去：

```sql
-- 新建面板专用账号，只给它面板库的权限，不要给全局权限
CREATE USER 'xboard_panel'@'%' IDENTIFIED WITH mysql_native_password BY '换成你自己的强密码';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP ON xboard.* TO 'xboard_panel'@'%';
FLUSH PRIVILEGES;
```

为什么权限要收着给？面板账号只需要操作自己的库，给全局权限一旦面板被攻破，攻击者就能动整台数据库。新建账号之后，把面板配置里的 `DB_USERNAME` 和 `DB_PASSWORD` 改成新账号，旧账号留着不动，等面板跑稳了再决定要不要清理。

## 第三步：改完配置，重启面板再验证

只改数据库不改面板配置，或者只改配置不重启面板，都会让问题看起来没修好。按顺序做完这三件事：确认面板配置里的账号、密码和数据库地址都和刚才改的账号一致；重启面板容器或者面板服务，让它重新建立数据库连接；再看一次面板日志，确认认证报错已经消失：

```bash
# Docker 部署：重启面板容器
docker restart xboard

# 重启后立刻看日志，确认不再出现认证报错
docker logs --tail 50 xboard 2>&1 | grep -i -E "caching_sha2|Access denied" || echo "no auth error in recent logs"
```

如果重启后面板能打开、后台能登录、订阅页面能正常生成，就说明认证问题已经解决。顺手再用面板账号从面板机上直连一次数据库，做一次最直接的验证：

```bash
# 用面板账号直连，能登录并且能切进面板库，才算真正通了
mysql -u xboard -p -h 127.0.0.1 -e "USE xboard; SHOW TABLES;" | head -20
```

这里特意用 `-h 127.0.0.1` 而不是默认的 socket 连接，是因为面板很多时候走的是 TCP 连接，而 socket 连接和 TCP 连接命中的可能是不同 host 的账号。命令行用 socket 能登，不代表面板走 TCP 就能登，验证一定要按面板实际的连接方式来。

## 常见坑汇总

- **只改了 localhost 账号**：面板走容器网络或内网 IP 连库时，命中的是 `xboard@%` 或者某个具体 IP 的账号，改 `localhost` 没用，改之前先看配置里的 `DB_HOST`。
- **改完没执行 FLUSH PRIVILEGES，也没重启面板**：数据库和面板两边各缓存了一份旧状态，任何一边没刷新都会继续报错。
- **密码里有特殊字符，写进配置文件没加引号**：`#`、`$` 这类字符在配置文件里容易被当成注释或者变量展开，密码尽量用字母数字组合，或者按配置格式正确加引号。
- **重装容器以为能解决问题**：数据库数据存在 volume 里，重装容器不会重置用户插件，旧账号的插件还是 `caching_sha2_password`，必须按修法一改账号。
- **升级后只查 root**：从 5.7 升级到 8.0 时，root 的插件可能变了，但面板账号未必跟着变，反过来也一样。查的时候要把面板实际用的那个账号列出来，不要只看 root。
- **图省事把密码改成空密码或者弱密码**：认证插件修好了，面板却裸奔了。数据库端口不要对外暴露，面板账号的密码照样要用强密码。

认证插件问题看着吓人，其实定位就两步：日志里确认是认证报错，数据库里确认插件是 `caching_sha2_password`。修法也只有一条主线：让面板账号用旧客户端支持的插件，或者给面板建一个新账号。按部署方式选一种，改完重启、按面板的连接方式验证一次，这类升级后连不上库的问题基本就到此为止了。

## 不想自己搭？

上面这套走下来，从定位报错、确认认证插件、改账号、重启面板再到验证，顺利的话也要花上一两个小时，中间任何一步走错——比如改错 host 的账号、漏了刷新权限——都得从头再查一遍。如果你只是想有个现成可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
