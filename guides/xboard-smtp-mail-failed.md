# Xboard 注册验证邮件收不到：SMTP 配置与队列失败排查

新用户注册后迟迟收不到验证邮件，是 Xboard 机场上线初期最常见的“假故障”之一：面板看着正常、注册接口也没报错，但邮箱里就是没有邮件。这个问题一般不出在注册流程本身，而是出在邮件发送链路的某一环——配置写错、队列没跑、邮件进了垃圾箱，或者发件域名没有通过对方的反垃圾校验。

排查的思路是先分段：先确认面板到底有没有尝试发信，再确认发信有没有成功，最后再看收信端为什么没收到。不要一上来就换邮箱服务商，先把现有链路跑通。

## 先定位：是没发出去，还是发了没收到

打开面板服务器，先看 Laravel 日志里有没有和邮件相关的报错：

```bash
cd /opt/xboard
tail -n 200 storage/logs/laravel.log | grep -i -E "mail|smtp|swift|failed"
```

如果日志里出现 `Connection could not be established`、`Authentication failed`、`Connection timed out`，说明发信这一步就失败了，重点查第二步的 SMTP 配置和网络。

如果日志里没有任何邮件报错，但用户还是收不到，就要怀疑两个方向：一是邮件被队列积压，还没真正发出去；二是邮件已经发出，但被收件方判定为垃圾邮件或直接拒收。后两种情况分别对应下面的第三步和常见坑。

顺带提醒：邮件发送失败和支付回调失败经常是同一个根因——队列没跑起来。如果你之前排过支付回调问题，可以对照本仓库的[支付回调排查](xboard-pay-callback-queue.md)一起看队列部分。

## 第一步：确认面板里的邮件配置

Xboard 的邮件配置在 `.env` 文件里，关键的几项缺一不可：

```bash
# 先查看当前配置（注意不要把真实密码贴到公开场合）
grep -E "^MAIL_" /opt/xboard/.env
```

一个能正常工作的 SMTP 配置大致是这样（以示例占位符为准，换成你自己的发件邮箱）：

```ini
MAIL_MAILER=smtp
MAIL_HOST=smtp.example.com
MAIL_PORT=465
MAIL_USERNAME=noreply@example.com
MAIL_PASSWORD=xxxx-xxxx-xxxx-xxxx
MAIL_ENCRYPTION=ssl
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="Your Panel"
```

这几个参数容易写错，逐项核对：

- `MAIL_PORT` 和 `MAIL_ENCRYPTION` 必须成对：465 对应 `ssl`，587 对应 `tls`。端口和加密方式错配是最常见的报错来源，连接会直接超时或握手失败。
- `MAIL_PASSWORD` 填的不是邮箱登录密码。QQ 邮箱、163 邮箱这类服务商要求使用“授权码”，在邮箱设置里单独开启 SMTP 并生成，登录密码在这里是无效的。
- `MAIL_FROM_ADDRESS` 最好和 `MAIL_USERNAME` 保持一致。发件地址与认证账号不一致时，不少服务商会直接拒信，报错往往是 `Sender address rejected`。
- 改完 `.env` 之后必须清缓存，否则面板读到的还是旧配置：

```bash
cd /opt/xboard
php artisan config:clear
php artisan cache:clear
```

为什么要清缓存？因为 Laravel 会把配置缓存起来，直接改文件不生效，很多人就是卡在这一步反复换配置、却始终测的是旧值。

## 第二步：绕过面板，直接测 SMTP 通不通

不要只靠“注册一个新账号”来测试，那样变量太多。用一条命令直接发一封测试邮件，把问题锁定在 SMTP 本身：

```bash
cd /opt/xboard
php artisan tinker
```

在 tinker 里执行：

```php
Mail::raw('这是一封 Xboard 邮件连通性测试', function ($message) {
    $message->to('your-test@example.com')->subject('SMTP 测试');
});
```

如果这封测试邮件能收到，说明 SMTP 配置和网络都没问题，问题在面板的注册邮件逻辑或队列环节；如果连这封都发不出去，终端里会直接抛出异常，顺着异常信息回去改配置，比猜要快得多。

同时确认服务器的出站端口没有被云厂商封掉。很多云厂商默认封禁 25 端口，所以不要用 25 端口发信，优先用 465 或 587，并确认安全组放行了出站方向的对应端口：

```bash
# 测试到 SMTP 服务器的端口是否可达
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/smtp.example.com/465' && echo "465 通" || echo "465 不通"
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/smtp.example.com/587' && echo "587 通" || echo "587 不通"
```

## 第三步：查队列，邮件可能根本还没发出去

Xboard 默认把邮件投递到队列里异步发送。如果队列消费者没有运行，邮件会一直积压在数据库或 Redis 里，用户自然收不到，而日志里还不一定有报错。

先看队列里有没有积压任务：

```bash
cd /opt/xboard
php artisan queue:failed
# 查看失败任务表里是否有邮件任务
mysql -u xboard -p -e "SELECT id, queue, failed_at FROM xboard.failed_jobs ORDER BY failed_at DESC LIMIT 20;"
```

如果 `failed_jobs` 里躺着大量邮件任务，先看失败原因，再决定是重试还是修配置后重发：

```bash
# 只重试某一条失败任务
php artisan queue:retry <任务 ID>
# 确认配置无误后，全部重试
php artisan queue:retry all
```

再确认队列消费者是不是真的在跑，并且是开机自启的：

```bash
# 查看队列进程是否在运行
ps aux | grep -v grep | grep "queue:work"
# 如果没有输出，说明队列根本没跑起来，手动启动一个
cd /opt/xboard && nohup php artisan queue:work --sleep=3 --tries=3 > storage/logs/queue.log 2>&1 &
```

更稳妥的做法是用 Supervisor 托管队列进程，让它崩了能自动拉起来、服务器重启后也能自动恢复。只靠 `nohup` 临时启动，重启一次就又会回到“邮件收不到”的老问题。

## 验证：按这个顺序确认修好了

修完之后不要只测一次就收工，按下面顺序验证一遍：

1. 先用第二步的 tinker 命令发一封测试邮件，确认 SMTP 本身通了。
2. 再用一个真实邮箱注册一个新账号，确认验证邮件能在几分钟内到达收件箱。
3. 注册测试时同时盯着队列日志，确认任务被消费、没有新增失败任务：

```bash
cd /opt/xboard
tail -f storage/logs/queue.log
php artisan queue:failed
```

4. 如果测试邮件能收到、注册邮件收不到，重点查注册邮件的模板和触发逻辑，而不是继续换 SMTP 配置——这两类问题修法完全不同。

## 常见坑

- **用了登录密码而不是授权码**：QQ、163 等邮箱必须用 SMTP 授权码，登录密码在这里永远通不过认证。
- **465 配了 tls、587 配了 ssl**：端口和加密方式错配，连接会直接失败，先按“465 配 ssl、587 配 tls”对齐。
- **改了 `.env` 没清缓存**：面板读的还是旧配置，换多少次服务商都没用，改完务必执行 `config:clear`。
- **队列没跑、邮件积压**：面板不报错、用户收不到信，`queue:failed` 和进程检查是定位这个问题的最快办法。
- **邮件进了垃圾箱**：发件域名没有配 SPF、DKIM、DMARC 记录时，Gmail、QQ 等收件方很容易把验证邮件判为垃圾邮件。先让用户检查垃圾箱，再去域名解析商处补齐这三条记录——这是长期可信度的基础配置，不是可选项。
- **用 25 端口发信**：云厂商普遍封禁 25 端口出站，改走 465 或 587 才是正解。
- **发件地址与认证账号不一致**：`MAIL_FROM_ADDRESS` 与 `MAIL_USERNAME` 不一致时容易被拒信，先把两者对齐。

注册验证邮件看似是小功能，但它直接决定新用户能不能完成注册、能不能开始使用。按“日志定位 → 配置核对 → 直连测试 → 队列检查 → 真实注册验证”这个顺序走一遍，大多数收不到邮件的问题都能在半小时内定位。

## 不想自己搭？

从零搭一套带面板、节点、支付和邮件配置的机场，前前后后要踩的坑远不止邮件这一处：面板部署、节点对接、证书续期、队列和定时任务，每一步都得自己排障，时间成本并不低。如果你只是想先稳定用起来，可以直接用漫画机场的现成订阅：标准套餐 ¥10/月起、家宽套餐 ¥30/月起，100GB/月流量、支持 3 台设备，订阅开通即用，不用自己维护面板和节点。官网：https://vps.manhuajichang.fun
