# Xboard 后台登录报 419 Page Expired：会话与缓存排查

在 Xboard 后台输入账号密码点登录，页面不提示密码错误，而是直接跳出 `419 Page Expired`，刷新再登录还是同样结果——这是很多站长在换域名、换反代、清缓存之后撞上的典型问题。419 不是账号密码错了，而是 Laravel 的 CSRF 校验没通过：浏览器提交的表单令牌和服务器端会话里存的令牌对不上，服务器直接把这次提交当作过期请求拒绝了。

所以排查 419 不要先去改密码、改管理员账号，先查会话：会话能不能正常建立、能不能在多次请求之间保持住、反代有没有把会话搞丢。会话通了，419 自然就消失了。

## 先定位：419 到底是哪一环断了

419 的完整链路是这样的：打开登录页时，服务器在会话里存一个 CSRF 令牌，同时把令牌写进表单隐藏字段；点登录时，浏览器把表单令牌和会话 Cookie 一起发回来，服务器比对两者是否一致。任何一环断掉都会报 419：会话根本没建立、会话在两次请求之间丢了、Cookie 没发回来、反代改写了请求，都会表现为同一个 419。

先看日志，确认是不是 CSRF 校验在报错：

```bash
cd /opt/xboard
tail -n 200 storage/logs/laravel.log | grep -i -E "419|csrf|tokenmismatch|session"
```

如果日志里出现 `TokenMismatchException`，说明就是 CSRF 校验失败，继续按下面步骤查会话；如果日志里完全没有相关报错，还要怀疑是反代或浏览器缓存把旧页面/旧 Cookie 一直留着，后面第三步会处理。

再做一个最简单的区分：用浏览器无痕模式打开后台登录页再登录一次。如果无痕模式能登录，说明问题出在原浏览器的旧 Cookie 或缓存上，重点看第三步；如果无痕模式也报 419，说明问题在服务器端配置，重点看第一步和第二步。

## 第一步：确认会话驱动与域名配置

Xboard 的会话配置在 `.env` 里，先看当前写的是什么：

```bash
grep -E "^SESSION_|^CACHE_" /opt/xboard/.env
```

一个能正常工作的会话配置大致是这样（以示例占位符为准）：

```ini
SESSION_DRIVER=file
SESSION_LIFETIME=120
SESSION_DOMAIN=panel.example.com
SESSION_SECURE_COOKIE=true
SESSION_SAME_SITE=lax
CACHE_STORE=file
```

逐项核对，错一项就可能报 419：

- `SESSION_DOMAIN` 必须和你实际访问的域名一致。换域名后如果这里还写着旧域名，浏览器会把 Cookie 发给旧域名，登录页拿到的会话和提交时用的会话根本不是同一个，直接报 419。只用单个域名访问时，也可以把这一项留空，让 Laravel 按当前域名自动处理。
- `SESSION_SECURE_COOKIE` 必须和访问协议匹配：全站 HTTPS 时设为 `true`；如果还在用 HTTP 调试却设成了 `true`，浏览器会拒绝在 HTTP 下发送这个 Cookie，会话每次都新建，登录永远过不了校验。
- `SESSION_SAME_SITE` 一般保持 `lax` 就够用。把它改成 `strict` 后，从外部链接点进后台再登录也可能被浏览器拦截 Cookie，表现为偶发 419。
- `SESSION_DRIVER` 如果是 `redis`，要确认 Redis 真的在跑（见下文）；如果是 `file`，要确认 `storage/framework/sessions` 目录可写，权限不对会导致会话写不进去。

改完 `.env` 之后必须清配置缓存，否则面板读到的还是旧值：

```bash
cd /opt/xboard
php artisan config:clear
php artisan cache:clear
```

为什么要清缓存？因为 Laravel 会把配置缓存起来，只改文件不生效，很多人就是卡在这一步：配置明明改对了，测的却一直是旧配置，419 反复出现。

如果会话驱动是 Redis，先确认 Redis 本身没挂：

```bash
redis-cli ping
# 应返回 PONG；如果连不上，先把 Redis 拉起来再测登录
systemctl status redis-server --no-pager | head -n 20
```

Redis 挂了的表现和 419 很像：会话写不进去，每次请求都是新会话，令牌永远对不上。关于 Redis 在 Xboard 里起什么作用，可以对照本仓库的[面板时间同步排查](panel-node-time-sync-ntp.md)一起理解会话与时间的配合关系——时间差过大时会话和令牌同样会提前失效。

## 第二步：检查反代有没有破坏会话

面板前面挂了 Nginx 或 Caddy 反代时，反代配置错了也会稳定复现 419，常见有两种：协议识别错了，或者 Cookie 域名被改写了。

先确认反代把真实协议透传给了面板。Nginx 的关键配置是这两行：

```nginx
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

为什么 `X-Forwarded-Proto` 这么重要？因为面板是根据这行来判断当前请求是 HTTP 还是 HTTPS 的。如果反代没传它，面板会以为用户在用 HTTP 访问，于是按 HTTP 的规则处理 Secure Cookie，浏览器那边却按 HTTPS 的规则发 Cookie，两边一错位，登录提交就被判定为令牌不匹配。

再确认 Laravel 信任了反代的转发头。检查 `app/Http/Middleware/TrustProxies.php`（或新版本的 `bootstrap/app.php` 中的受信任代理配置），确保反代服务器的内网地址在信任列表里：

```php
// 示例：信任本地反代
protected $proxies = [
    '127.0.0.1',
];
```

如果这一步没配，`X-Forwarded-Proto` 传了也白传，面板还是按直连来判断协议，419 照样出现。改完反代配置后记得重载并清缓存：

```bash
nginx -t && systemctl reload nginx
cd /opt/xboard && php artisan config:clear
```

还有一种容易被忽略的情况：反代层开了缓存，把带 CSRF 令牌的登录页缓存起来发给所有人。每个人拿到的都是同一张旧表单，令牌早就过期，登录必报 419。面板后台和登录页绝对不能被反代缓存，Nginx 里要确保没有对 `/admin` 或登录路径做 `proxy_cache`，Caddy 同理不要对动态路径开静态缓存。

## 第三步：清掉坏掉的会话与浏览器缓存

如果服务器配置已经对了、还是报 419，很可能是旧会话或旧 Cookie 在捣乱。先在服务器端清掉会话和缓存：

```bash
cd /opt/xboard
# 文件会话：清掉旧会话文件（会让所有人重新登录，操作前先确认）
rm -f storage/framework/sessions/*
php artisan cache:clear
php artisan config:clear
```

如果是 Redis 会话，不要直接 `FLUSHALL`，那样会把队列、缓存等其他数据一起清掉，只清会话相关的键：

```bash
# 只查看会话键的数量，先确认再动手
redis-cli --scan --pattern "laravel_session*" | wc -l
redis-cli --scan --pattern "laravel_session*" | xargs -r redis-cli DEL
```

服务器端清完后，浏览器端也要清一次：清掉该域名下的 Cookie，然后重新打开登录页再登录。只清服务器端、不清浏览器，浏览器还会带着旧会话 ID 反复提交，419 照样出现；反过来也一样，两边要一起清才彻底。

## 第四步：检查服务器时间与多实例会话不一致

会话和令牌都有有效期，服务器时间差过大时，刚生成的令牌可能在服务器看来已经过期，或者 Cookie 一发下来就被判定过期：

```bash
# 面板机时间是否正常
date
timedatectl status | grep -E "System clock|NTP service"
# 时间差超过几分钟就先校时
timedatectl set-ntp true
```

如果面板跑了多台实例、前面挂了负载均衡，还要确认所有实例共用同一份会话存储（同一个 Redis 或数据库）。每台实例各存各的文件会话时，登录页可能由 A 实例生成、提交却落到 B 实例，B 实例手里根本没有这个会话，必然报 419。这种部署下必须把 `SESSION_DRIVER` 改成 `redis` 或 `database`，文件会话是扛不住多实例的。

## 验证：按这个顺序确认修好了

修完之后不要只登录一次就收工，按下面顺序验证一遍：

1. 用无痕模式打开后台登录页，输入账号密码登录，确认能正常进入后台，不再报 419。
2. 在后台点几个页面、做一次保存操作，确认后续请求不再跳 419，说明会话在多次请求之间保持住了。
3. 换回正常浏览器模式，清掉该域名 Cookie 后再登录一次，确认普通模式也正常。
4. 如果改了反代配置，分别用域名和 IP（如果允许）各测一次，确认不是某个入口碰巧通了。
5. 观察一轮日志，确认没有新增的 `TokenMismatchException`：

```bash
cd /opt/xboard
tail -n 100 storage/logs/laravel.log | grep -i -E "tokenmismatch|419"
```

## 常见坑

- **换域名后没改 `SESSION_DOMAIN`**：Cookie 发给了旧域名，新域名下会话永远建不起来，登录必报 419。
- **HTTPS 下 `SESSION_SECURE_COOKIE` 设错**：全站 HTTPS 却设成 `false` 或反过来，Cookie 的发送规则和实际协议对不上。
- **改了 `.env` 没清缓存**：面板读的还是旧配置，改多少次都像没改，`config:clear` 是必做的一步。
- **反代没传 `X-Forwarded-Proto`**：面板误判协议，Secure Cookie 规则错位，419 稳定复现。
- **登录页被反代缓存**：所有人共用一张过期表单，令牌一提交就过期，后台路径绝不能缓存。
- **Redis 会话挂了**：会话写不进去，每次请求都是新会话，令牌永远对不上，先 `redis-cli ping` 确认。
- **只清服务器端、不清浏览器**：旧会话 ID 还在浏览器里反复提交，两边要一起清。
- **多实例各存文件会话**：登录页和提交落在不同实例上，必须共用 Redis 或数据库会话。
- **服务器时间差过大**：令牌和 Cookie 刚生成就被判定过期，先校时再谈其他排查。

419 看着吓人，其实根因就一句话：表单令牌和服务器会话对不上。按“配置核对 → 反代检查 → 两端清缓存 → 时间与多实例确认”这个顺序走一遍，大多数后台登录 419 都能在半小时内定位。

## 不想自己搭？

从零搭一套带面板、节点、支付和会话配置的机场，前前后后要踩的坑远不止登录这一处：面板部署、节点对接、证书续期、队列和定时任务，每一步都得自己排障，时间成本并不低。如果你只是想先稳定用起来，可以直接用漫画机场的现成订阅：标准套餐 ¥10/月起、家宽套餐 ¥30/月起，100GB/月流量、支持 3 台设备，订阅开通即用，不用自己维护面板和节点。官网：https://vps.manhuajichang.fun
