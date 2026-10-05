# OpenClash 排查记录

> 本文只记**排查方法与已定结论**。配置本体在路由器上，
> `main_router/openclash` · `second_router/openclash` 是 uci 备份（2026-08-10 上传）。

---

## 🔑 通用规律：国内外共用同一域名 + GeoDNS 分区解析 ⇒ 分流规则永远不会被执行

**触发条件**（两条同时满足才会出现，缺一不会）：

1. 域名在 geosite 里被归类为**大陆域名**（归属链最终落到 `geolocation-cn`）
2. 你**需要它走境外**

**失效链条**：

```
国内 DNS 解析出国内 IP
  → 「区域绕过 = 大陆」在防火墙层（chnroute）直接放行
  → 流量根本不进入 mihomo 内核
  → 任何分流规则都不可能匹配（这也是 yacd Conns 里搜不到记录的原因）
```

**关键认知**：分流规则解决的是「流量往哪走」，但本问题出在上一步 ——
「域名解析成什么 IP」。**DNS 没纠正之前，规则永远没有执行机会。**

**排查顺序**（任一步失败即可定位问题所在层）：

| 顺序 | 看什么 | 说明什么 |
|---|---|---|
| 1 | `nslookup <域名>` | 解析出国内 IP ⇒ 问题在 DNS 层，不用往下看 |
| 2 | yacd → Conns 搜该域名 | **无任何记录 ⇒ 流量没进内核**，通常就是 chnroute 绕过 |
| 3 | 规则写法 / 顺序 | 只有前两步都正常才轮到怀疑这里 |

### 解决模板：Nameserver-Policy 定向到境外 DoH

覆写设置 → DNS 设置 → Nameserver-Policy：

```yaml
"+.example.com": "https://dns.google/dns-query#<策略组名>"
```

三个必须注意的点（每一条都踩过）：

1. **整个值必须用双引号包住** —— 否则 YAML 会把 `#` 之后当注释吃掉，策略静默失效
2. `#` 后接**策略组名**是 mihomo 语法，作用是让 **DNS 查询本身走代理**。
   不加则默认直连查询，国内访问 `dns.google` 必然失败，策略形同虚设
3. 策略组名需与 Proxies 页面显示的名称**一字不差**

保存后依次点击「清理 DNS 缓存」→「关闭链接」。

---

## 案例 1 · DeepSeek 强制 +86 手机号登录（2026-08-16）

### 现象

- 分流规则已写 `DOMAIN-SUFFIX,deepseek.com,<境外节点>`，应用后网页仍强制 +86 手机号 / 微信登录
- yacd 连接列表搜 `deepseek` **一条记录都没有**
- V2rayN 全局模式 + Chrome system proxy 则可正常用 Gmail 登录
- 同配置下 YouTube / Claude / ChatGPT / Qwen / Kimi 全部正常，**唯独 DeepSeek 异常**

### 根因

DeepSeek 国内版与国际版**共用同一域名** `chat.deepseek.com`，按查询来源 GeoDNS 分区解析：

| 解析来源 | 结果 IP | 归属 | 响应头特征 | 登录方式 |
|---|---|---|---|---|
| 国内 DNS | 116.205.40.114 | 华为云 HWCSNET (CN) | `server: CW`、`HWWAFSESID` | 仅手机号 / 微信 |
| 境外 DNS | 3.173.21.63 | AWS CloudFront | `server: elb`、`x-amz-cf-pop: NRT57` | 支持 Google / 邮箱 |

`deepseek.com` 在 v2fly geosite 中的归属链：
`data/deepseek` → `category-ai-cn` → `geolocation-cn` ⇒ 命中「区域绕过 = 大陆」。

### 解决

```yaml
"+.deepseek.com": "https://dns.google/dns-query#自建日本1-故转"
```

### 验证（三步，按顺序）

1. `nslookup chat.deepseek.com`
   → 返回 `3.x.x.x`（CloudFront）或 fake-ip 段地址 = 成功；
   返回 `116.205.x.x` = DNS 策略未生效
   ⚠️ **fake-ip 段以本机 `fakeip_range` 实际值为准**，见下方「关联 ③」
2. F12 → Network → `sign_in` 请求响应头：
   `server: elb` + `x-amz-cf-pop` = 国际版；`server: CW` = 仍是国内版
3. yacd → Conns 搜 `deepseek`，应能看到条目，且 RULE / CHAINS 列显示预期的规则与策略组

### 本次被排除的错误方向（别再走一遍）

| 方向 | 为什么排除 |
|---|---|
| ~~补充 fengkongcloud / portal101 / volces 等第三方风控域名~~ | 抓包证实不存在此类隐藏请求 |
| ~~Fake-IP-Filter 命中导致不分配假 IP~~ | 实际列表中 geosite 条目均为注释状态 |
| ~~规则顺序被 GEOSITE / GEOIP 兜底规则抢先~~ | 自定义规则位置本身正确 |
| ~~Cookie / UA / Accept-Language 导致的区域判定~~ | 两次请求 Cookie 完全一致但登录方式不同，已反证 |

### 为什么 Qwen / Kimi 不受影响

它们的国际版使用**独立域名**，固定解析到境外 IP，不会被 chnroute 拦截 ——
域名与地区一一对应，规则匹配后 IP 本身就在境外段，不触发区域绕过。

| 站点 | 国际版域名 → IP | 国内版域名 → IP |
|---|---|---|
| Qwen | `chat.qwen.ai` → 47.77.4.100（Alibaba Cloud LLC, US） | `tongyi.aliyun.com` → 39.97.x（北京阿里云） |
| Kimi | `www.kimi.com` → 104.18.20.246（Cloudflare） | `kimi.moonshot.cn` → 103.143.17.156（CN） |

---

## 案例 2 · LinkedIn 被跳到 `linkedin.cn/incareer`，HTTP 451（2026-10-05）

### 现象

- 打开 www.linkedin.com，浏览器落在 `https://www.linkedin.cn/incareer/home`，页面报 **HTTP ERROR 451**
- 自定义规则已写 `DOMAIN-KEYWORD,linkedin.com,<美国节点>`，`Global.list` 里也有 linkedin.com（排查期间曾在 `Proxy.list` 里也加过），**换成哪个节点都不行**
- V2rayN 全局模式（系统代理）下正常

### 根因：又是 GeoDNS，但这次 DNS 是在**客户端本地**被解析成国内的

LinkedIn 的 www 现在托管在 Cloudflare 上，但国内 DNS 给的是另一条链：

| 解析来源 | 结果 | 访问后 |
|---|---|---|
| 国内 DNS（运营商 / 手机热点） | `www.linkedin.com` → CNAME `www.linkedin.cn` → `pop-az-sh…` → **Azure 中国（世纪互联，AS58593）** | 302 → `www.linkedin.cn/incareer/home` → 451（领英职场已停止服务） |
| 境外 DNS（`dns.google`） | `www.linkedin.com.cdn.cloudflare.net` → Cloudflare | 302 → `/uas/login`，正常 |

🔴 **关键认知：出口在哪不重要，目标 IP 才重要。** 客户端拿国内解析结果去连的是 LinkedIn 中国那台服务器，
流量就算真从美国出去，到的还是那台服务器，照样被跳到 linkedin.cn。何况那是大陆 IP，
「区域绕过 = 大陆」直接放行，流量根本不进内核，规则写成什么都没有执行机会（与通用规律同一条链）。

**与案例 1 的区别**：DeepSeek 是网关自己的 DNS 解析成了国内；这次网关 DNS 是对的
（局域网内查 `www.linkedin.com` 拿到 fake-ip，没有 AAAA），错在**客户端根本没用网关的 DNS**。
本次实例：笔记本在手机热点下，经 Tailscale 把家里网关设为出口节点，但 Tailscale 的
「使用 Tailscale DNS」是关着的（`accept-dns=false`），DNS 走了热点下发的蜂窝 DNS；
路由器 conntrack 里能看到这条连接从 tailnet 来、直接 NAT 出了 WAN。

V2rayN 全局之所以好：系统代理把**域名**交给代理端去解析，本地 DNS 不参与。

### 判据（先查 DNS，再查规则）

| 顺序 | 看什么 | 说明什么 |
|---|---|---|
| 1 | Windows：`Resolve-DnsName www.linkedin.com` | 出现 `www.linkedin.cn` 的 CNAME = 本地 DNS 是国内的，问题在这一层 |
| 2 | `curl -sI --resolve www.linkedin.com:443:<Cloudflare IP> https://www.linkedin.com/feed/` | 302 到 `/uas/login` = 链路和出口都没问题，只差 DNS |
| 3 | 网关 conntrack 搜 Azure 中国那个 IP | 有记录且回程是 WAN 地址 = 流量绕过了内核 |

### 处置

- **出口节点用法**：客户端必须同时用 tailnet 的 DNS：`tailscale set --accept-dns=true`
  （Windows 托盘：Use Tailscale DNS settings）。只设出口节点不设 DNS = DNS 泄漏回本地网络。
- **局域网内设备**：用网关下发的 DNS 即可；不要在浏览器里开「安全 DNS」并指定国内 DoH 服务商。
- 可选加固：Nameserver-Policy 加 `"+.linkedin.com": "https://dns.google/dns-query#<策略组名>"`。
  网关内核自己解析 `www.linkedin.com` 也会拿到 Azure 中国的 IP（连接记录里的目标 IP 就是它）；
  域名规则先命中、代理端按域名连接时不受影响，这条只是防哪天有规则按 IP 判。

---

## 案例 3 · SEEK：直连阵发性被劫持，走机房节点又过不了 Cloudflare 挑战（2026-10-05）

### 现象

- 原规则 `DOMAIN-KEYWORD,seek.com,DIRECT`（配套 `challenges.cloudflare.com` 也 DIRECT，两者必须同出口）
- 当天 19:00 前后起直连 `nz.seek.com` 拿到**伪造证书**：连的是 Cloudflare 的真实 IP，
  证书却是自签名 `O = redirect-cnzz`（或一张只写了别处 IP 的短期证书），HTTP 302 到 `m.baidu.com`
- 同一时刻走代理的 `www.seek.co.nz` 正常

### 实测

| 时间（UTC+8） | 路径 | 结果 |
|---|---|---|
| 19:13~19:35 | 苏州家宽直连 | 持续被劫持（伪造证书 + 302 百度） |
| 22:12 | 苏州家宽直连（绑 WAN 口、指定真实 IP） | 证书校验通过，403 = Cloudflare 挑战页正常响应；**8 次里 3 次超时** |
| 22:12 | 老家家宽直连 | 正常 |
| 22:10 | 美国机房节点 | TLS 正常，但 Cloudflare 给**交互式 Turnstile 复选框**，真浏览器等 38 秒不放行 |

⇒ 劫持是**阵发的**，不是永久封锁。两条路各有一半问题：

| 路径 | 网络 | Cloudflare 挑战 |
|---|---|---|
| 家宽直连 | 阵发劫持、偶发超时 | 住宅 IP，**十秒内自动通过**（2026-08-22 实测） |
| 机房节点 | 干净 | 机房 IP，**要人手勾选**，自动化做不了 |

### 处置方案（待定，未改）

用 fallback 组让两条路互为备份，健康检查打在 SEEK 自己的域名上：

```yaml
- name: SEEK
  type: fallback
  proxies: [DIRECT, <美国节点>]
  url: https://nz.seek.com/cdn-cgi/trace
  interval: 300
```

规则 `seek.com` 与 `challenges.cloudflare.com` **都指向这个组**（同出口的约束由组来保证）。
劫持时健康检查的 TLS 会失败，自动切到美国节点；劫持结束后切回直连，挑战恢复自动通过。
⚠️ 上线前要先校准：劫持期间健康检查确实判失败（否则这个组永远停在 DIRECT）。

---

## 🔴 与本仓库的关联（2026-08-16 核对，三条都未处理）

### ① `AI.list` 里**没有任何 deepseek 条目**

本次的分流规则只写在路由器的自定义规则里，**没有回流到共享规则集**。
⇒ 其它订阅这份 `AI.list` 的设备完全没有这条规则。

### ② 两份 uci 备份里，这次的修复**一条都没有**

`main_router/openclash` 与 `second_router/openclash` 中：

```
option custom_name_policy '0'      ← 自定义 Nameserver-Policy 开关是关的
（且全文没有 config nameserver_policy 段，即没有任何策略条目）
option china_ip_route   '1'        ← 「区域绕过 = 大陆」，正是本问题的触发开关
```

⇒ **刷机或恢复这份备份 = 修复丢失，问题原样复现。**
备份需要重新导出一份，或至少把这两处补上。

### ③ 验证判据里的 fake-ip 段写错了

原始记录写「返回 `198.18.x.x`(fake-ip) 为成功」，但备份里实际是：

```
option fakeip_range '198.19.0.1/16'
```

⇒ 本机 fake-ip 会落在 **`198.19.x.x`**。按 `198.18.x.x` 判会**把成功误判成失败**。
（`198.18.0.0/15` 同时覆盖 198.18 与 198.19，OpenClash 默认是 198.18.0.1/16，这台改过。）
用判据前先 `uci get openclash.config.fakeip_range` 核一下。
