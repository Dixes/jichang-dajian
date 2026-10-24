# Let's Encrypt 证书自动续期失败排查

证书刚部署好的时候一切正常，面板、订阅地址都能用 HTTPS 打开，于是很多人就把续期这件事忘了。Let's Encrypt 的证书有效期只有 90 天，靠的就是自动续期兜底；一旦自动续期悄悄失败，证书过期那天面板和订阅会同时报不安全，用户连不上、节点订阅拉不到，问题爆发时往往已经没有缓冲时间。本篇按排查顺序讲：先确认证书还剩几天、续期任务到底跑没跑，再按失败现象逐一定位，最后给出防复发的检查清单。

## 先确认现状：还剩几天、续期任务长什么样

动手排障之前，先把三条基本信息拿到手。很多“续期失败”其实是续期根本没装上，或者装上了但从没成功过。

```bash
# 1. 证书还剩几天过期（把域名换成你自己的面板域名）
openssl s_client -connect panel.example.com:443 -servername panel.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# 2. 本机证书列表与过期时间（certbot 管理的证书都在这里）
certbot certificates

# 3. 自动续期任务是否存在
systemctl list-timers | grep -i certbot
crontab -l | grep -i certbot
```

这三条结果要能对上号：`certbot certificates` 里能看到该域名，过期时间在 30 天以内还没续上才算异常；`systemctl list-timers` 或 crontab 里应当有一条 certbot 续期任务。如果定时任务根本不存在，问题不在续期本身，而在当初部署时没把自动续期装好——补一条任务比调证书本身更重要，后面会讲。

> 为什么是“30 天以内”才算异常？certbot 默认只在证书剩余有效期不足 30 天时才真正续期，手动跑 `certbot renew` 时如果证书还很新，它会直接跳过并告诉你“not due for renewal”。拿这条输出当失败，是最常见的误判。

顺带确认证书是由哪种方式签发的：HTTP-01 验证、DNS-01 验证，还是 Caddy/Nginx 插件自动处理的。不同方式的失败点完全不同，下一节按方式分开讲。签发方式可以从续期配置文件里看到：

```bash
# 看该域名的续期配置，重点看 authenticator 和 installer 两行
cat /etc/letsencrypt/renewal/panel.example.com.conf
```

反代与证书的完整部署流程，见本仓库的[反代与证书：Caddy/Nginx + 自动续期](caddy-nginx-tls.md)。

## 第一步：手动跑一次续期，看真实报错

自动任务的日志往往只有一行“失败”，信息不够。排障的第一步永远是手动跑一次，把完整报错拿到手：

```bash
# 干跑：只验证，不真的换证书，适合先定位问题
certbot renew --dry-run

# 对单个域名干跑，输出更聚焦
certbot renew --dry-run --cert-name panel.example.com
```

> 为什么先干跑而不是直接真续？干跑不会动现有证书，失败了面板照常运行；真续一旦中途出错，还可能把旧证书的续期状态搅乱。先干跑拿到报错，修好再真续，顺序不能反。

把报错信息按关键词归类，下面是最常见的四类，每类对应不同的修法，不要上来就删证书重签——删了重签会撞上 Let's Encrypt 的签发频率限制，越急越慢。

## 第二类问题：80 端口不通，HTTP-01 验证失败

这是最常见的一类，报错里通常有 `Connection refused`、`Timeout` 或 `Invalid response`。HTTP-01 的原理是：Let's Encrypt 的验证服务器访问你域名的 80 端口，取一个临时文件，取到了才发证。所以 80 端口必须从公网可达，而且必须由 certbot（或反代）正确响应。

按这个顺序查：

```bash
# 1. 80 端口是否在监听、防火墙和安全组是否放行
ss -tlnp | grep ':80'
ufw status | grep 80

# 2. 从公网视角测 80 端口通不通（在另一台机器或用在线端口检测做）
curl -v http://panel.example.com/.well-known/acme-challenge/test --max-time 10

# 3. 域名解析是否指向了这台机器
dig +short panel.example.com
```

常见坑位逐个对：

- **Nginx/Caddy 做了 80 到 443 的强制跳转，但跳转目标本身证书已过期**：验证请求被跳到 443 后因证书问题失败。排障期间可以先临时放行 `/.well-known/acme-challenge/` 不跳转，续期成功后再恢复。
- **Cloudflare 开了橙云（代理）**：验证请求走到 Cloudflare 而不是源站，源站的验证文件自然取不到。续期期间把该域名切回灰云（仅 DNS），或者改用 DNS-01 验证。
- **安全组只开了 443 没开 80**：机内防火墙放行了不等于入口放行了，两层都要查，和节点端 API 防护那篇讲的两层思路一样。
- **反代配置里把验证路径代理到了别处**：检查 Nginx/Caddy 配置，确认 `/.well-known/` 没有被反代到面板程序或返回 404。

修好后重新干跑，验证通过再真续：

```bash
# 干跑通过后，真正续期一次
certbot renew --cert-name panel.example.com

# 续期成功后重载反代，让新证书生效（不要用 restart，reload 不断连接）
nginx -t && systemctl reload nginx
# 如果是 Caddy
systemctl reload caddy
```

> 为什么是 reload 而不是 restart？restart 会断开现有连接，面板和订阅正在用的连接会被切断；reload 只重读配置和证书，已建立的连接不受影响。证书更新这种常规操作，永远优先 reload。

## 第三类问题：DNS-01 验证失败，TXT 记录没生效

如果当初用的是 DNS-01（常见于泛域名证书，或 80 端口不便开放的场景），失败点通常在 DNS 一侧。报错里常出现 `DNS problem` 或 `No TXT record found`。

```bash
# 查该域名当前的 TXT 记录，看验证记录是否真的存在
dig +short TXT _acme-challenge.panel.example.com

# 如果用了 Cloudflare 之类带代理的 DNS，注意查的是权威结果还是缓存结果
dig +short TXT _acme-challenge.panel.example.com @1.1.1.1
```

常见原因：

1. **手动加 TXT 记录时加错了主机名**：应当是 `_acme-challenge.panel` 这样的相对名，而不是把完整域名再贴一遍；不同 DNS 商的填法不一样，填之前先看它家示例。
2. **DNS 传播还没完成就点了验证**：刚改完记录立刻验证，Let's Encrypt 查到的还是旧结果。等几分钟再跑一次，或者把 TTL 事先调低。
3. **自动 DNS 插件的 API Token 过期或权限不足**：用 certbot 的 DNS 插件自动维护记录时，Token 一旦被轮换、权限被收窄，续期就会静默失败。去 DNS 商后台确认 Token 仍有效、且有改该域名 TXT 记录的权限。
4. **上一轮的旧 TXT 记录没清掉**：多条 `_acme-challenge` 记录并存时，验证服务器可能取到旧的那条。清理旧记录，只留最新的一条。

DNS-01 的排障要多一点耐心：每次改完记录都先用 `dig` 从公共 DNS 确认能查到，再跑干跑，不要凭“后台显示已保存”就认定生效。

## 第四类问题：续期任务本身没跑，或跑了但环境不对

如果手动干跑能通过，自动续期却从不成功，问题往往不在证书，而在任务本身。

```bash
# 看 certbot 定时任务最近一次跑了没有、结果如何
systemctl status certbot.timer
journalctl -u certbot --since "7 days ago" --no-pager | tail -n 50

# crontab 方式的，看系统日志里有没有这条任务的执行记录
grep -i certbot /var/log/syslog | tail -n 20
```

逐项核对：

- **定时任务被系统升级或重装清掉了**：重装 certbot、换反代、迁移服务器后，旧的 timer 或 crontab 可能没跟过来。重新启用一次并确认下次触发时间。
- **续期脚本里写了旧路径**：比如脚本里还指着旧的 Nginx 配置路径或旧的 webroot 目录，服务器迁移后路径变了，脚本就一直在对着空气验证。把脚本里的路径和当前实际路径逐项对一遍。
- **环境变量缺失**：DNS 插件靠环境变量或凭证文件里的 Token 续期，手动在终端跑时环境是全的，cron 里跑时环境是空的，Token 读不到就失败。凭证文件要放在 cron 也能读到的固定路径，不要只放在某个用户的 shell 配置里。
- **续期后没有重载反代**：证书其实续上了，但 Nginx/Caddy 还拿着旧证书在内存里服务，看起来像没续。确认续期配置里有 `--deploy-hook`，或在定时任务后补一句 reload：

```bash
# 给续期加部署钩子：证书一换就重载 Nginx
certbot renew --deploy-hook "systemctl reload nginx"

# 也可以直接写进该域名的续期配置文件，之后每次自动续期都会执行
# /etc/letsencrypt/renewal/panel.example.com.conf 里加入：
# renew_hook = systemctl reload nginx
```

> 为什么续期成功还要专门管 reload？certbot 只负责把新证书写到磁盘，反代进程不会自己察觉文件变了。不 reload，旧证书会一直服务到进程重启，等于续了个寂寞。

## 第五类问题：频率限制与证书文件损坏

这两类不常见，但撞上时最容易让人误判成别的故障。

**频率限制**：短时间内反复失败、反复删证书重签，会撞上 Let's Encrypt 每周每域名 5 张的签发上限，报错里有 `too many certificates already issued`。撞上后不要再试，等窗口过去；同时先把失败原因修好再重试，否则窗口一过又会立刻撞上。生产域名不要用 `--staging` 之外的反复试错来“碰运气”。

**证书文件损坏或权限错乱**：`/etc/letsencrypt/` 下的文件被手动改过权限、被备份还原覆盖、磁盘满了写了一半，都会导致续期读不下去。先查磁盘和权限：

```bash
# 磁盘满了是续期失败里最冤的一种，先排除
df -h /etc/letsencrypt

# 续期配置与证书目录的权限应当是 root 可读写，不要放宽给其他用户
ls -ld /etc/letsencrypt /etc/letsencrypt/live/panel.example.com
ls -l /etc/letsencrypt/live/panel.example.com/
```

磁盘满了先清日志和旧备份，不要动证书目录本身；权限错乱先按默认的 root 所有权恢复，不要图省事改成 777。

## 续期成功后：做一次端到端验证

不要只看 `certbot renew` 输出了成功就收工。证书换了，反代重载了，用户视角是不是真的好了，要从头测一遍：

```bash
# 1. 浏览器视角：证书有效期是不是真的变长了
openssl s_client -connect panel.example.com:443 -servername panel.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# 2. 订阅地址视角：订阅链接能正常拉取，不报证书错误
curl -I https://panel.example.com/api/v1/client/subscribe?token=示例TOKEN --max-time 10

# 3. 自动任务视角：确认下次触发时间在未来，且 deploy-hook 已配好
systemctl list-timers | grep -i certbot
```

三项都通过，这次排障才算闭环。顺手把这次的失败原因和修法记下来：证书出问题很少只出一次，同样的域名、同样的反代配置，下个 90 天大概率还会撞上同一个坑。

## 防复发清单

- 证书到期前 30 天、15 天、7 天各设一次提醒，不要只依赖自动续期本身。
- 每月手动跑一次 `certbot renew --dry-run`，把它当成定期体检，而不是等过期了才想起来。
- 每次改反代配置、换 DNS 商、迁服务器、换 Cloudflare 代理状态后，都顺手跑一次干跑，确认验证路径没被改坏。
- 把验证方式（HTTP-01 还是 DNS-01）、Token 位置、deploy-hook、定时任务类型记在一处，换人维护或半年后自己回看时不用重新猜。
- 面板、订阅、节点端 API 各自的域名和证书分开记，不要用一张表含糊带过，某个域名出问题才不用全站停摆去猜。

证书续期失败从来不是证书本身突然坏了，基本都是验证路径、DNS、定时任务、反代重载这四个环节里有一个悄悄变了。按“先干跑拿报错，再按报错分流，最后端到端验证”的顺序走，大多数问题半小时内都能定位；怕的是证书过期当天才发现，那时修什么都得顶着用户连不上的压力。

## 不想自己搭？

按上面这套走下来，从确认剩余天数、干跑拿报错、逐项查 80 端口或 DNS 记录、再到补 deploy-hook 和端到端验证，顺利的话也要花上一两个小时，中间任何一步域名解析或配置路径填错都得回来重查。如果你只是想有个现成可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
