# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

客厅大屏时钟页面，设计用于平板/壁挂大屏显示。**单文件架构**——整个应用在 `index.html` 中（HTML + CSS + 内联 JS），无框架、无构建工具、无 package.json。

## 开发命令

```bash
# 浏览器直接打开
open index.html

# 本地静态服务
python3 -m http.server 8080
```

无测试、无 lint、无构建步骤。

## 部署

- 对外服务走副本:nginx 站点根为 `/var/www/living-room-dashboard`(admin 可写)。本家目录为 700,nginx 无法直接读仓库目录,**不要让 nginx 直指本仓库目录**
- 改动后同步副本即对外生效,无需 reload nginx:

```bash
rsync -a --delete ~/work/github/living-room-dashboard/ /var/www/living-room-dashboard/
```

### HTTPS

- 访问入口:`https://114.55.208.51/` 与 `https://gucx.top/`(域名已备案,浙ICP备2026030754号-1);80 端口 301 跳转到 HTTPS
- IP 证书:ZeroSSL 免费 IP 证书(90 天有效期),文件在 `/etc/nginx/ssl/dashboard/`(`fullchain.pem` / `privkey.pem`);root crontab 每日 09:17 执行 `/usr/local/bin/renew-zerossl-cert.sh` 自动续期(剩余有效期 >30 天时跳过),日志在 `/var/log/zerossl-renew.log`
- 域名证书:Let's Encrypt(`CN=gucx.top`),文件在 `/etc/letsencrypt/live/gucx.top/`,nginx 有独立 443 块按 SNI 匹配 `gucx.top`(IP 证书块为默认);certbot 装在 `/opt/certbot-venv`,root crontab 每日 03:42 执行 `certbot renew --quiet`,日志在 `/var/log/certbot-renew.log`
- nginx 80 块中的 `/.well-known/pki-validation/`(ZeroSSL 验证)与 `/.well-known/acme-challenge/`(Let's Encrypt 验证)是证书验证路径,**不要删除**,否则续期会失败

## 架构要点

### 逻辑画布

页面使用 **2048×1536 固定逻辑画布**（`.stage`），通过 `updateStageScale()` 以 `Math.min(vw/2048, vh/1536)` CSS scale 缩放适配视口。所有布局基于此画布尺寸。

### JS 结构（IIFE，全部内联）

脚本在 `(function() { "use strict"; ... })()` 内，主要模块：

- **时钟**：`scheduleClock()` 用 `setTimeout` 对齐秒级更新，`updateClock()` 通过 `lastH/lastM/lastS` 去重 DOM 写入，页面隐藏时暂停
- **天气**：`fetchWeather()` 调用 Open-Meteo API（无需 key），每 20 分钟刷新，定位失败回退杭州坐标（30.29N, 120.16E）
- **ASCII 猫**：`initPetCat()` 管理 idle/blink/walk 状态机，用 `setTimeout` 链驱动，walk 通过 CSS `translate3d` 移动，blink 触发问候语波浪动画
- **问候语**：`getGreeting(hour, daySeed)` 按时段从 6 个池中选猫主题中文短句，`renderGreetingText()` 拆成单字 `<span>` 做波浪动画
- **底部语录**：20 条中文励志语录，按日索引轮换

### 关键 DOM ID

`stage`（画布）、`timeH/M/S`（时钟）、`petCatArt`（ASCII 猫）、`greeting`（问候）、`weekdayHanzi/dateYear/Month/Day`（日期）、`weatherIcon/Temp/Desc/Details`（天气）、`forecast`（3日预报）、`dockCenterText`（语录）

## 修改注意事项

- 所有改动都在 `index.html` 单文件中完成
- CSS 使用 `var()` 做深色主题（背景 `#0f1117`）
- 字体为本地 Inter ttf（fonts/ 目录，权重 200/300/400/600/700）
- 天气代码映射使用 WMO 标准（0-99），在 `WEATHER_MAP` 对象中（key 为 WMO code，value 为 `[emoji, 中文描述]`）
- 默认坐标为杭州，修改 `LAT`/`LNG` 常量即可切换城市
- 底部语录在 `DOCK_QUOTES` 数组中，按日索引轮换
