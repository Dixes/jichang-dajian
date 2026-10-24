# 机场搭建教程合集

从零搭建一个自己的机场：面板部署、节点对接、协议配置、中转落地、运营维护，一篇一篇写清楚。每篇都是可直接照做的实操步骤，命令和配置都给出完整示例。

## 内容目录

- [机场搭建总览：自建一个机场需要准备什么](guides/jichang-dajian-zonglan.md)
- [Xboard 面板 Docker 部署教程](guides/xboard-docker-deploy.md)
- [V2bX 节点端安装与对接 Xboard](guides/v2bx-install-connect.md)
- [VLESS + Reality 节点搭建](guides/vless-reality-setup.md)
- [Hysteria2 节点搭建与调优](guides/hysteria2-server-setup.md)
- [域名与 Cloudflare 配置要点](guides/domain-cloudflare-setup.md)
- [反代与证书：Caddy/Nginx + 自动续期](guides/caddy-nginx-tls.md)
- [中转机端口转发：realm / iptables 实操](guides/relay-forward-setup.md)
- [落地机怎么选：线路、带宽与住宅 IP](guides/landing-server-choice.md)
- [面板节点分组与倍率设置](guides/node-groups-multiplier.md)
- [订阅安全：防泄露与链接重置](guides/subscription-security.md)
- [面板对接支付：易支付与套餐定价](guides/payment-gateway-setup.md)
- [自建订阅转换（subconverter）](guides/subconverter-setup.md)
- [VLESS + WS + TLS + CDN 传统方案](guides/vless-ws-tls-cdn.md)
- [sing-box 做服务端：手动配置入门](guides/sing-box-server.md)
- [Trojan 与 Shadowsocks 节点速通](guides/trojan-ss-setup.md)
- [机场运维：日志查看与状态监控](guides/ops-monitor-logging.md)
- [节点被封排查与换 IP 流程](guides/node-blocked-troubleshoot.md)
- [面板数据备份与服务器迁移](guides/backup-migration.md)
- [机场成本核算：一台机场每月花多少钱](guides/airport-cost.md)
- [Xboard 部署后容器起不来：常见报错排查](guides/xboard-docker-troubleshoot.md)
- [V2bX 对接失败：面板显示节点离线怎么办](guides/v2bx-offline-troubleshoot.md)

## 适合谁

- 想自己搭机场自用或小圈子分享的
- 已经在运营机场、想补齐中转 / 落地 / 运维短板的
- 对 Xboard、V2bX、VLESS Reality、Hysteria2 这些组件只知其名、不知怎么串起来的人

## 说明

教程里的命令以 Debian / Ubuntu + Docker 环境为主。搭机场需要一台境外 VPS 和一个域名，文中的示例域名请替换成你自己的。

如果只是想科学上网、不想折腾服务器，也可以直接用现成的订阅服务（比如 [漫画机场](https://vps.manhuajichang.fun)，¥10/月起），省去运维时间——本仓库的教程更适合愿意自己动手的人。
