# OpenClash + dnsmasq 配置：AdGuard Home 作为前端，OpenClash 作为上游

> 本文记录在 ImmortalWrt 上将 DNS 链路调整为 **客户端 → AdGuard Home → OpenClash → 公共 DNS** 的配置过程。  
> dnsmasq 不再承担公网 DNS 转发，只负责 DHCP、`.lan` 本地域名和 PTR 反向解析。
>
> 本文为当前实机配置记录，请根据自己的 LAN 地址、配置文件名称和服务名称调整。

---

## 更新日志

### 2026-06-16

- 将 AdGuard Home 调整为局域网主 DNS，监听 `53` 端口。
- 将 dnsmasq 调整到 `54` 端口，仅负责 DHCP、本地域名和 PTR。
- 将 OpenClash 调整为 AdGuard Home 的唯一公网 DNS 上游，监听 `7874`。
- 关闭 OpenClash 本地 DNS 劫持。
- 移除旧链路中的 `127.0.0.1:38617`，避免 DNS 回环。
- 增加 `Default-NameServer`，用于解析 DoH/DoT 上游自身域名。
- 增加 DHCP Option 6，显式向 LAN 客户端下发 `192.168.7.1` 作为 DNS。
- 修复客户端默认 DNS 仅显示 `fec0:0:0:ffff::1`、指定 AGH 查询正常但网页无法解析的问题。
- 验证 Fake-IP、`.lan` 正向解析、PTR 反向解析、默认 DNS 下发和真实客户端识别均正常。

---

## 1. 适用环境

本文实机环境：

| 项目 | 配置 |
|---|---|
| 路由器 | GL.iNet GL-MT6000 |
| 系统 | ImmortalWrt 24.10.4 |
| LAN 地址 | `192.168.7.1/24` |
| OpenClash | `0.47.099` |
| dnsmasq | `dnsmasq-full 2.90-r5` |
| 防火墙 | firewall4 / nftables |
| OpenClash 模式 | Fake-IP 增强模式 |
| IPv6 | 不使用 |
| OpenClash DNS 端口 | `7874` |
| dnsmasq 端口 | `54` |
| AdGuard Home DNS 端口 | `53` |

---

## 2. 最终 DNS 拓扑

```text
局域网客户端
    │
    │ TCP/UDP 53
    ▼
AdGuard Home
192.168.7.1:53
    │
    ├── 普通公网域名
    │       ▼
    │   OpenClash
    │   127.0.0.1:7874
    │       ▼
    │   AliDNS DoH / DoT
    │
    ├── *.lan
    │       ▼
    │   dnsmasq
    │   127.0.0.1:54
    │
    └── 私网 PTR
            ▼
        dnsmasq
        127.0.0.1:54
```

端口职责：

| 服务 | 端口 | 职责 |
|---|---:|---|
| AdGuard Home | `53` | 局域网 DNS 入口、过滤、日志、客户端识别 |
| dnsmasq | `54` | DHCP、`.lan`、hosts、PTR/rDNS |
| OpenClash | `7874` | Fake-IP、域名分流、代理/直连 DNS |
| AdGuard Home Web | `3000` | 管理界面 |

---

## 3. 与旧方案的区别

旧链路：

```text
客户端 → dnsmasq/OpenClash → AdGuard Home:38617 → AliDNS
```

新链路：

```text
客户端 → AdGuard Home:53 → OpenClash:7874 → AliDNS
```

新方案的主要优点：

1. AdGuard Home 可以看到真实客户端 IP，而不是全部显示为路由器或 `127.0.0.1`。
2. 可以按设备查看查询日志、统计和应用客户端规则。
3. `.lan` 和私网 PTR 单独交给 dnsmasq，不进入公网 DNS。
4. OpenClash 专注于 Fake-IP、规则分流和上游解析。
5. 不再依赖 OpenClash 的 DNS 劫持规则。
6. DNS 关系保持单向，避免 `AGH → OpenClash → AGH` 回环。

---

## 4. 修改前备份

```bash
TS="$(date +%Y%m%d-%H%M%S)"
mkdir -p /root/dns-config-backups

tar -czf "/root/dns-config-backups/dns-stack-$TS.tar.gz" \
  /etc/config/dhcp \
  /etc/config/openclash \
  /etc/AdGuardHome.yaml \
  /etc/openclash/riolu.yaml

ls -lh "/root/dns-config-backups/dns-stack-$TS.tar.gz"
```

> 不要将包含旧上游的 `.bak` 文件长期放在 `/etc/openclash/` 中，以免检索时误判。  
> 建议统一移动到 `/root/dns-config-backups/`。

---

## 5. 重要：修改顺序

旧配置中 OpenClash 的上游为 AdGuard Home：

```text
OpenClash → 127.0.0.1:38617
```

新配置中 AdGuard Home 的上游为 OpenClash：

```text
AdGuard Home → 127.0.0.1:7874
```

因此必须先删除 OpenClash 中所有 `38617` 上游，再将 AdGuard Home 指向 `7874`。

否则会形成：

```text
AdGuard Home → OpenClash → AdGuard Home → 无限循环
```

推荐顺序：

1. 备份。
2. OpenClash 删除 `127.0.0.1:38617`。
3. OpenClash 配置 AliDNS DoH/DoT。
4. 关闭 OpenClash 本地 DNS 劫持。
5. dnsmasq 从 `53` 改到 `54`。
6. 使用 DHCP Option 6 向 LAN 客户端下发 `192.168.7.1` 作为 DNS。
7. AdGuard Home 改为监听 `53`，上游填写 `127.0.0.1:7874`。
8. 客户端续租 DHCP，确认默认 DNS 已变为 `192.168.7.1`。
9. 验证端口、配置和解析结果。

---

## 6. OpenClash 配置

### 6.1 模式设置

进入：

```text
OpenClash → 插件设置 → 模式设置
```

建议：

- 运行模式：Fake-IP 增强模式。
- 开启 UDP 流量转发。
- 开启路由本机代理。
- IPv6 DNS：关闭。
- Fake-IP 持久化：开启。
- `respect-rules`：关闭。

本文实机配置：

```yaml
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  listen: 0.0.0.0:7874
  respect-rules: false
  fake-ip-filter-mode: blacklist
```

> `respect-rules: true` 时还需要正确配置 `proxy-server-nameserver`。  
> 本文没有该需求，因此保持 `false`。

---

### 6.2 关闭本地 DNS 劫持

进入：

```text
OpenClash → 插件设置 → DNS 设置
```

关闭：

```text
本地 DNS 劫持
```

保存并重启 OpenClash。

检查 nftables：

```bash
nft list ruleset 2>/dev/null | \
grep -Ei -C 3 'OpenClash DNS Hijack|dport 53'
```

正确状态：无输出。

如果仍出现：

```text
comment "OpenClash DNS Hijack"
```

说明劫持规则仍存在，需要重新检查 OpenClash DNS 劫持开关并重启 OpenClash、防火墙。

---

### 6.3 配置自定义上游 DNS

进入：

```text
OpenClash → 覆写设置 → DNS 设置
```

开启：

```text
自定义上游 DNS
```

#### Default-NameServer

添加两条：

| 项目 | 第一条 | 第二条 |
|---|---|---|
| 服务器分组 | Default-NameServer | Default-NameServer |
| 服务器地址 | `223.5.5.5` | `223.6.6.6` |
| 服务器端口 | 留空 | 留空 |
| 服务器类型 | UDP | UDP |
| 指定接口 | 禁用 | 禁用 |
| 直连域名解析 | 不勾选 | 不勾选 |
| 节点域名解析 | 不勾选 | 不勾选 |
| 禁用 IPv4 | 不勾选 | 不勾选 |
| 禁用 IPv6 | 不勾选 | 不勾选 |

对应 YAML：

```yaml
default-nameserver:
  - 223.5.5.5
  - 223.6.6.6
```

`default-nameserver` 使用纯 IP DNS，用于解析 `dns.alidns.com` 等 DoH/DoT 上游自身域名。

#### NameServer

添加：

```yaml
nameserver:
  - https://dns.alidns.com/dns-query#disable-ipv6=true
  - tls://dns.alidns.com:853#disable-ipv6=true
```

#### Direct-NameServer

添加：

```yaml
direct-nameserver:
  - https://dns.alidns.com/dns-query#disable-ipv6=true
  - tls://dns.alidns.com:853#disable-ipv6=true
```

> 不要再填写：
>
> ```text
> 127.0.0.1:38617
> tcp://127.0.0.1:38617
> ```
>
> 否则会重新建立 `OpenClash → AGH` 旧链路。

---

### 6.4 Fake-IP 过滤

保留 OpenClash 默认和现有 Fake-IP 过滤规则，至少应包括本地域名和 Tailscale 域名：

```yaml
fake-ip-filter:
  - "*.lan"
  - "*.local"
  - "*.home.arpa"
  - "+.tailscale.com"
  - "+.ts.net"
```

不要仅为了精简而删除 OpenClash 内置的 NTP、STUN、游戏服务和局域网相关过滤项。

---

### 6.5 当前生效 DNS 配置示例

```yaml
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  listen: 0.0.0.0:7874
  respect-rules: false
  fake-ip-filter-mode: blacklist

  default-nameserver:
    - 223.5.5.5
    - 223.6.6.6

  nameserver:
    - https://dns.alidns.com/dns-query#disable-ipv6=true
    - tls://dns.alidns.com:853#disable-ipv6=true

  direct-nameserver:
    - https://dns.alidns.com/dns-query#disable-ipv6=true
    - tls://dns.alidns.com:853#disable-ipv6=true

  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "*.home.arpa"
    - "+.tailscale.com"
    - "+.ts.net"
```

> 应优先在 OpenClash 覆写界面中修改，不建议只手工修改订阅生成的 YAML。  
> 订阅更新或覆写时，手改内容可能被重新生成。

---

## 7. dnsmasq 与 LAN DHCP 配置

dnsmasq 不再监听 `53`，改为监听 `54`，只负责：

- DHCP。
- `.lan` 本地域名。
- `/etc/hosts`。
- DHCP 主机名。
- PTR/rDNS。
- 动态或静态主机名映射。

### 7.1 将 dnsmasq 移到 54

执行：

```bash
uci set dhcp.@dnsmasq[0].port='54'
uci set dhcp.@dnsmasq[0].domain='lan'
uci set dhcp.@dnsmasq[0].local='/lan/'
uci set dhcp.@dnsmasq[0].expandhosts='1'
uci set dhcp.@dnsmasq[0].cachesize='0'
uci set dhcp.@dnsmasq[0].noresolv='1'

uci -q delete dhcp.@dnsmasq[0].server
```

关键配置：

```uci
config dnsmasq
        option local '/lan/'
        option domain 'lan'
        option expandhosts '1'
        option cachesize '0'
        option port '54'
        option noresolv '1'
```

说明：

- `port 54`：释放 `53` 给 AdGuard Home。
- `local /lan/`：由 dnsmasq 处理 `.lan`。
- `domain lan`：本地域后缀。
- `expandhosts 1`：将本地主机扩展为 `hostname.lan`。
- `cachesize 0`：dnsmasq 不再承担公网 DNS 缓存。
- `noresolv 1`：不读取运营商 DNS，避免绕过 OpenClash。
- 删除 `server`：dnsmasq 不再向 OpenClash 或其他公网 DNS 转发普通查询。

### 7.2 必须使用 DHCP Option 6 下发 AGH 地址

这是本方案不可省略的一步。

当 dnsmasq 从标准 DNS 端口 `53` 移到 `54` 后，OpenWrt 不再自动把路由器地址作为 DNS 服务器发布给 DHCP 客户端。必须显式通过 DHCP Option 6 下发：

```text
192.168.7.1
```

先检查现有 LAN DHCP 选项：

```bash
uci show dhcp.lan | grep dhcp_option
```

在保留其他 DHCP 选项的前提下，加入：

```bash
uci -q del_list dhcp.lan.dhcp_option='6,192.168.7.1'
uci add_list dhcp.lan.dhcp_option='6,192.168.7.1'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

最终 LAN 配置应包含：

```uci
config dhcp 'lan'
        option interface 'lan'
        option start '100'
        option limit '150'
        option leasetime '12h'
        option dhcpv4 'server'
        list dhcp_option '6,192.168.7.1'
```

> 不建议使用 `uci -q del dhcp.lan.dhcp_option`，除非确认没有其他自定义 DHCP 选项。  
> 本文仅删除完全相同的旧值后重新添加，避免重复，同时保留其他 DHCP 选项。

OpenWrt 官方 AdGuard Home 指南明确指出：dnsmasq 更改默认监听端口后，路由器地址不会再自动作为 DNS 服务器通过 DHCP 下发，需要使用 DHCP Option 6 指定 DNS 地址。

### 7.3 让客户端重新获取 DNS 配置

Windows：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

Wi-Fi 设备也可以断开后重新连接。

验证：

```cmd
ipconfig /all | findstr /i /c:"DNS Servers" /c:"DNS 服务器"
```

正确结果必须包含：

```text
DNS 服务器：192.168.7.1
```

若默认 DNS 仅显示：

```text
fec0:0:0:ffff::1
```

而没有 `192.168.7.1`，客户端没有使用 AGH，网页会出现“找不到服务器的 IP 地址”，即使以下命令仍可能成功：

```cmd
nslookup openwrt.org 192.168.7.1
```

因为该命令是手动指定 AGH，并不能证明系统默认 DNS 已正确下发。

## 8. DHCP 主机名标记

### 8.1 只标记名称，不固定 IP

普通设备只需用 MAC 绑定一个易识别的名称：

```uci
config host
        option name 'ytq-***'
        list mac '******'
```

这种配置表示：

```text
MAC 固定对应主机名
IP 仍由 DHCP 动态分配
```

设备取得租约后，dnsmasq 会建立当前 IP 与主机名的映射。

适用于：

- 手机。
- 平板。
- 电视。
- 空调。
- 智能家居。
- 不依赖固定地址的普通设备。

不需要为了显示名称而给所有设备固定 IP。

### 8.2 有特殊需求时固定 IP

端口转发、远程串流、游戏主机、服务器等设备可同时配置固定地址：

```uci
config host
        option name '***-proxy'
        option ip '192.168.7.***'
        list mac '***28:40:DD:3B:15:E4***'
```

只对确实依赖固定地址的设备配置 `option ip`。

---

## 9. 重启后的客户端名称问题

### 9.1 现象

路由器重启后，AdGuard Home 仍能看到真实客户端 IP，但可能暂时只显示：

```text
192.168.7.xxx
```

手动重启或等待一段时间后 AdGuard Home 后恢复：

```text
设备名.lan
```

### 9.2 判断方法

检查 DHCP 租约：

```bash
cat /tmp/dhcp.leases
```

检查 PTR：

```bash
nslookup 192.168.7.181 192.168.7.1
```

如果可以返回：

```text
181.7.168.192.in-addr.arpa name = MiCO.lan
```

说明 dnsmasq 和 PTR 配置正常，仅是 AdGuard Home 的运行时客户端名称尚未刷新。

手动刷新：

```bash
/etc/init.d/AdGuardHome restart
```

当前实机没有配置自动重启 AGH 的脚本。  
客户端名称暂时只显示 IP 不影响 DNS 主链路和网页访问，不建议为了显示名称增加不必要的开机自动重启逻辑。

## 10. 验证 OpenClash、dnsmasq 与 DHCP 下发

### 10.1 检查端口

```bash
netstat -lnptu 2>/dev/null | \
grep -Ei ':53|:54|:7874|dnsmasq|AdGuardHome|clash'
```

预期：

```text
53   → AdGuardHome
54   → dnsmasq
7874 → clash
```

### 10.2 检查 OpenClash 不再指向旧 AGH 端口

```bash
grep -nE \
'127\.0\.0\.1:38617|127\.0\.0\.1#38617|tcp://127\.0\.0\.1:38617' \
/etc/openclash/riolu.yaml \
/etc/config/openclash \
/etc/AdGuardHome.yaml 2>/dev/null
```

正确状态：无输出。

备份文件中出现旧值不影响运行，但建议将备份移到：

```text
/root/dns-config-backups/
```

### 10.3 查看生效的 OpenClash DNS 段

```bash
sed -n '/^dns:/,/^[^[:space:]]/p' \
/etc/openclash/riolu.yaml
```

至少应确认：

```yaml
listen: 0.0.0.0:7874

default-nameserver:
  - 223.5.5.5

nameserver:
  - https://dns.alidns.com/dns-query#disable-ipv6=true
  - tls://dns.alidns.com:853#disable-ipv6=true
```

### 10.4 检查 DHCP Option 6

```bash
uci show dhcp.lan | grep dhcp_option
```

预期包含：

```text
dhcp.lan.dhcp_option='6,192.168.7.1'
```

### 10.5 检查客户端默认 DNS

Windows：

```cmd
ipconfig /all | findstr /i /c:"DNS Servers" /c:"DNS 服务器"
```

正确结果必须包含：

```text
192.168.7.1
```

然后测试默认解析：

```cmd
nslookup openwrt.org
```

预期：

```text
服务器:  ytq77.lan
Address:  192.168.7.1

名称:    openwrt.org
Address: 198.18.x.x
```

如果：

```cmd
nslookup openwrt.org 192.168.7.1
```

成功，但：

```cmd
nslookup openwrt.org
```

失败，优先检查 DHCP Option 6，而不是继续修改 OpenClash 或 AGH 上游。

### 10.6 直接检查 OpenClash 7874

BusyBox `nslookup` 不能用 `127.0.0.1#7874` 指定非标准端口。  
安装 `bind-dig` 后使用：

```bash
opkg update
opkg install bind-dig
```

测试：

```bash
dig @127.0.0.1 -p 7874 openwrt.org A \
  +time=3 +tries=1 +short
```

Fake-IP 模式下预期：

```text
198.18.x.x
```

### 10.7 检查 AGH 公网解析

```bash
dig @192.168.7.1 -p 53 openwrt.org A \
  +time=3 +tries=1 +short
```

预期同样返回：

```text
198.18.x.x
```

### 10.8 检查 `.lan`

```bash
nslookup ytq77.lan 192.168.7.1
```

预期：

```text
ytq77.lan → 192.168.7.1
```

### 10.9 检查 PTR

```bash
nslookup 192.168.7.1 192.168.7.1
```

预期：

```text
192.168.7.1 → ytq77.lan
```

### 10.10 检查网页访问

Windows：

```cmd
curl.exe -4 -I --connect-timeout 8 https://www.baidu.com
curl.exe -4 -I --connect-timeout 8 https://openwrt.org
```

若提示：

```text
Could not resolve host
```

先查看系统默认 DNS 是否为 `192.168.7.1`。

## 11. Tailscale 说明

该 DNS 改造不会改变 Tailscale 隧道和游戏数据流。

远程访问家庭局域网仍是：

```text
远程设备
    ↓ Tailscale
家庭路由器 tailscale0
    ↓ 子网路由
192.168.7.0/24
```

直接使用局域网 IP 玩游戏时，DNS 基本不参与。

若远程设备需要解析：

```text
ytq77.lan
gamepc.lan
```

可以通过 Tailscale 子网路由访问：

```text
192.168.7.1:53
```

不需要让 AdGuard Home 额外监听路由器的 Tailscale `100.x` 地址。

游戏无法自动发现局域网房间时，应优先考虑广播/组播限制，并尝试手动输入局域网 IP，而不是修改 DNS。

---

## 12. 回滚

从备份恢复：

```bash
tar -xzf /root/dns-config-backups/dns-stack-时间戳.tar.gz -C /
```

然后按顺序重启：

```bash
/etc/init.d/dnsmasq restart
/etc/init.d/AdGuardHome restart
/etc/init.d/openclash restart 2>/dev/null
```

OpenClash 实际服务名称可能随安装包不同，以本机 `/etc/init.d/` 中存在的脚本为准。

---

## 13. 参考资料

- [原 OpenClash 配置记录](https://github.com/ytq77/Network-HomeLab/blob/main/OpenClash/Clash_config.md)
- [原 AdGuard Home 配置记录](https://github.com/ytq77/Network-HomeLab/blob/main/AdGuardHome/AdGuardHome_config.md)
- [OpenClash Wiki：DNS 设置](https://github.com/vernesong/OpenClash/wiki/DNS%E8%AE%BE%E7%BD%AE)
- [OpenClash Wiki：配置文件](https://github.com/vernesong/OpenClash/wiki/%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
- [Mihomo Docs：DNS 配置](https://wiki.metacubex.one/config/dns/)
- [OpenWrt Wiki：AdGuard Home](https://openwrt.org/docs/guide-user/services/dns/adguard-home)
- [OpenWrt Wiki：DHCP 与 DNS 配置](https://openwrt.org/docs/guide-user/base-system/dhcp)
- [AdGuard Home Wiki：Configuration](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration)
