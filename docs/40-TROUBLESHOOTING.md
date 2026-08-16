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
