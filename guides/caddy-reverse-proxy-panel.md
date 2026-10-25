# Caddy 反代面板的完整配置示例

面板装好了，但直接把 Xboard 的 80/3000 端口暴露给用户并不体面：没有 HTTPS、域名难记、面板和节点 API 混在一个端口上。常见的做法是在面板机前面放一层反向代理，用域名 + HTTPS 对外提供面板访问，面板本身只监听本机端口。Caddy 是做这件事最省心的工具：配置只有几行，证书自动申请、自动续期，不用再单独折腾 certbot 和定时任务。这篇给出从安装到上线的完整配置，照着改域名和端口就能用。

## 场景：为什么面板前面要加反代

直接暴露面板的典型问题有三个：

- **没有 HTTPS**：用户在浏览器里登录面板、面板和节点端之间传通讯密钥，走的都是明文 HTTP，中间人能直接看到。
- **端口难记又容易撞**：面板默认监听非标准端口，用户访问要带端口号，而且这个端口还可能和节点协议端口撞上。
- **证书要自己管**：用 Nginx + certbot 的组合，证书申请、续期、重载服务每一步都要自己配，漏了续期面板就掉证书。

Caddy 把这三件事打包了：监听 80/443、按域名自动申请 Let's Encrypt 证书、证书到期前自动续期。面板本身不需要改端口监听方式，只需要监听本机，剩下的交给 Caddy。

## 第 1 步：安装 Caddy

以 Debian/Ubuntu 为例，用官方源安装：

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy
```

装完后确认服务在跑：

```bash
systemctl status caddy
```

> 为什么用官方源而不是发行版自带的包？发行版仓库里的 Caddy 版本往往滞后几个大版本，自动证书和某些指令的行为会有差异。官方源装的是当前稳定版，和本文的配置写法一致。

## 第 2 步：写最基本的 Caddyfile

假设面板域名是 `panel.example.com`，面板本身监听本机 `127.0.0.1:3000`（Xboard 的默认端口按你的实际部署改），Caddyfile 只要写：

```
panel.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

Caddyfile 默认在 `/etc/caddy/Caddyfile`。写好后重载生效：

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
systemctl reload caddy
```

`caddy fmt` 会把配置文件格式化一遍，顺手发现缩进和括号问题，重载前跑一次能省不少排障时间。重载后浏览器访问 `https://panel.example.com`，能打开面板登录页，说明反代已经通了——证书是 Caddy 在首次访问时自动向 Let's Encrypt 申请的，不用你手动跑任何命令。

## 第 3 步：补齐生产环境需要的几项配置

基础版能跑，但生产环境建议把下面几项一次补齐：

```
panel.example.com {
    # 面板本身只接受本机请求时，反代地址写 127.0.0.1，不要写公网 IP
    reverse_proxy 127.0.0.1:3000 {
        # 面板和节点端的长连接、心跳不要被反代提前掐断
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }

    # 安全响应头：面板后台是登录态页面，基础防护值得加上
    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        Referrer-Policy "strict-origin-when-cross-origin"
        -Server
    }

    # 面板日志单独记一份，出问题先看这里而不是翻系统日志
    log {
        output file /var/log/caddy/panel.log
        format json
    }
}
```

逐项解释为什么要加：

- `header_up X-Forwarded-*`：反代之后，面板看到的来源 IP 会变成 `127.0.0.1`，后台登录记录、风控和节点 API 的来源判断都会失真。把真实 IP 透传过去，面板才能正确记录是谁在访问。
- `X-Frame-Options "SAMEORIGIN"`：面板登录页不允许被别的网站嵌进 iframe，能挡掉一类点击劫持的利用方式。
- 单独的访问日志：面板出问题时，先看 `/var/log/caddy/panel.log` 里有没有对应的请求，比在系统日志里翻要快得多。日志目录要提前建好并给 caddy 用户写权限，否则 Caddy 启动会报错。

改完同样 `caddy fmt` + `systemctl reload caddy` 生效。

## 第 4 步：面板侧的配合设置

反代只是把流量接进来，面板本身也要知道自己在反代后面，否则会出现两种典型症状：登录后反复跳回登录页、或者面板生成的订阅链接、回调地址还是 `http://` 开头。

- **站点 URL 填带 HTTPS 的域名**：Xboard 后台的站点地址、订阅链接域名，一律填 `https://panel.example.com`，不要填 `http://`，也不要填带端口的内网地址。面板按这个地址生成订阅和回调链接，填错了用户拿到的订阅链接就是错的。
- **信任反代传来的协议头**：面板判断当前请求是不是 HTTPS，依赖的是反代传来的 `X-Forwarded-Proto`。第 3 步已经把这个头透传了，如果面板仍然认为请求是 HTTP，先检查 Caddyfile 里 `header_up` 有没有写对，而不是去改面板的强制 HTTPS 开关。
- **节点 API 走同一个域名**：V2bX 配置里的 `ApiHost` 填 `https://panel.example.com`，和用户访问面板用同一个域名。这样节点端拉配置、报流量走的也是 HTTPS，不用再给 API 单独开端口、单独配证书。节点端 API 的来源 IP 建议再单独收紧，做法见[节点端 API 防护：只允许面板 IP 访问](node-api-firewall.md)。

## 第 5 步：一个域名不够用时的写法

### 面板和官网分域名

面板机上往往不止面板一个服务，例如官网、状态页也在同一台机器上。Caddy 的写法是每个域名一个块，互不干扰：

```
panel.example.com {
    reverse_proxy 127.0.0.1:3000
}

www.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

每个域名独立申请证书，互不影响。一个域名证书出问题，不会拖垮另一个。

### 同一域名下用路径区分

如果只有一个域名，又想把面板挂在子路径下，例如 `https://example.com/panel`，Caddy 可以这样写：

```
example.com {
    handle_path /panel/* {
        reverse_proxy 127.0.0.1:3000
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}
```

> 子路径方案有个坑：面板前端很多资源路径、跳转地址是按根路径写的，挂在子路径下容易出现静态资源 404、登录后跳转回根路径的问题。除非域名实在不够，否则优先用子域名方案，不要用子路径硬套面板。

## 证书与续期：Caddy 替你做了什么

Caddy 的证书流程是全自动的，不需要你再跑 certbot：

1. 首次有请求访问 `panel.example.com` 时，Caddy 向 Let's Encrypt 申请证书，域名解析必须已经指向这台机器的公网 IP，否则验证失败。
2. 证书到期前约 30 天，Caddy 自动续期，续期成功后自动重载，不用重启服务。
3. 续期失败时，Caddy 会在日志里报错并按退避策略重试，不会直接让旧证书过期——但你要定期看日志，不能假设「自动」等于「永远不用看」。

对应的排障思路见[Let's Encrypt 证书自动续期失败排查](cert-renew-failed.md)，那里按「解析 → 端口 → 限速」列了检查顺序。Caddy 场景下最常见的失败原因只有一个：80 端口被别的程序占了，或者防火墙没放行 80 端口，Let's Encrypt 的 HTTP 验证走不通。排查时先跑 `ss -tlnp | grep ':80'`，确认监听 80 的是 Caddy。

## 验证：上线前按这个顺序查一遍

1. **HTTPS 能打开**：浏览器访问 `https://panel.example.com`，地址栏有锁形标志，证书颁发者是 Let's Encrypt，且域名和你填的一致。
2. **HTTP 会跳转**：访问 `http://panel.example.com`，应该自动跳转到 HTTPS，而不是直接显示面板内容。Caddy 默认会做这个跳转，如果没有跳转，先检查是不是有别的程序占了 80 端口。
3. **登录态正常**：登录面板后台，刷新几次页面、切换几个菜单，没有被反复踢回登录页。被踢回登录页多半是 `X-Forwarded-Proto` 没透传，面板以为请求是 HTTP，Cookie 的 Secure 属性对不上。
4. **节点端能拉配置**：在落地机上重启 V2bX，看面板里节点状态变在线、流量开始上报。节点端走的 `ApiHost` 也是这个域名，这一步通了才算反代真正接管了面板的全部流量。
5. **订阅链接能用**：用测试账号拉一次订阅，确认订阅里的节点地址、协议信息正常，订阅链接本身也是 `https://` 开头。

## 常见坑

- **域名解析还没生效就配 Caddy**：Caddy 首次申请证书时域名必须已经解析到本机 IP。刚改完解析就重载 Caddy，证书申请失败，面板会以 HTTP 方式裸奔或者干脆打不开。先 `dig panel.example.com` 确认解析结果，再重载。
- **面板还监听在公网地址上**：反代配好了，但面板本身还在监听 `0.0.0.0:3000`，用户绕过反代直接访问 `http://IP:3000` 就能打开面板，等于反代白配了。面板改成只监听 `127.0.0.1`，再用防火墙把 3000 端口的入方向关掉。
- **80 端口被 Nginx 或其他服务占着**：一台机器上同时跑 Nginx 和 Caddy，两个都想监听 80/443，先启动的那个会占住端口，后启动的起不来。面板机上只留一个反代入口，另一个要么停掉、要么改成由 Caddy 统一反代。
- **Caddyfile 改完只 reload 没检查**：`systemctl reload caddy` 在配置有语法错误时不一定报错，旧配置会继续跑，你以为生效了其实没有。每次改完先跑 `caddy validate --config /etc/caddy/Caddyfile`，通过了再 reload。
- **日志目录权限没给**：`log` 块里写了文件路径，但 `/var/log/caddy` 不存在或者 caddy 用户写不进去，Caddy 启动失败。建目录并改属主：`mkdir -p /var/log/caddy && chown caddy:caddy /var/log/caddy`。
- **证书续期失败没人看**：自动续期不等于不用管，每月抽空看一次 Caddy 日志里有没有证书相关的报错。续期失败拖到证书过期，面板、节点 API、订阅链接会同时掉 HTTPS。

## 小结

面板前面加 Caddy 反代，核心就三句话：面板只监听本机端口，Caddy 对外提供域名和 HTTPS，证书申请和续期全部交给 Caddy 自动处理。配好之后，面板访问、节点 API、订阅链接走的都是同一个 HTTPS 域名，证书不用你再操心，排障时先看 Caddy 日志、再看面板日志，顺序不要反。这层反代搭好后，面板机的安全基线才算完整，剩下的就是面板本身的加固和节点端的日常运维。

## 不想自己搭？

从买机器、装面板、配反代和证书，到上线后盯着证书续期和节点状态，自己搭一套机场的真实成本不是一台 VPS 的月租，而是持续投入的配置和排障时间。如果你只是想有个开箱即用的订阅，不妨直接用现成的：漫画机场提供现成的订阅服务，标准套餐 ¥10/月起、家宽套餐 ¥30/月起，每月 100GB 流量、支持 3 台设备，详情见官网 https://vps.manhuajichang.fun 。
