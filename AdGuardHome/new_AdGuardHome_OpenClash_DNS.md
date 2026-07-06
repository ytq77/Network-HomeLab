# AdGuard Home 配置：前端过滤并将 OpenClash 作为上游 DNS

> 本文记录在 ImmortalWrt 上将 AdGuard Home 调整为局域网主 DNS 的配置方法。  
> AdGuard Home 直接接收客户端查询，普通公网域名转发到 OpenClash，`.lan` 和私网 PTR 转发到 dnsmasq。
>
> OpenClash 和 dnsmasq 的配置请参考同目录之外的配套文档：  
> [OpenClash + dnsmasq 配置：AdGuard Home 作为前端，OpenClash 作为上游](../OpenClash/OpenClash_AGH_DNS.md)

---

## 更新日志

### 2026-06-16

- AdGuard Home 从旧的 `38617` 端口调整到标准 DNS 端口 `53`。
- 普通上游调整为 OpenClash `127.0.0.1:7874`。
- `.lan` 专用上游调整为 dnsmasq `127.0.0.1:54`。
- 私人反向 DNS 调整为 dnsmasq `127.0.0.1:54`。
- 清空 Bootstrap DNS 和后备 DNS，避免绕过 OpenClash。
- 验证 AGH 可以显示真实客户端 IP 和设备名称。
- 增加 DHCP Option 6 前置要求，确保 LAN 客户端默认使用 `192.168.7.1`。
- 增加“指定 AGH 查询正常，但系统默认 DNS 失败”的排障方法。
- 增加路由器重启后客户端名称刷新说明。

---

## 1. 最终拓扑

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
    │   OpenClash 127.0.0.1:7874
    │       ▼
    │   AliDNS DoH / DoT
    │
    ├── *.lan
    │       ▼
    │   dnsmasq 127.0.0.1:54
    │
    └── 私网 PTR
            ▼
        dnsmasq 127.0.0.1:54
```

---

## 2. 为什么让 AdGuard Home 位于最前端

旧方案：

```text
客户端 → OpenClash/dnsmasq → AGH → 公共 DNS
```

旧方案中，AGH 容易只看到：

```text
127.0.0.1
路由器地址
```

新方案：

```text
客户端 → AGH → OpenClash → 公共 DNS
```

新方案可以：

- 显示真实客户端 IP。
- 根据 PTR 显示设备名称。
- 按客户端查看日志和统计。
- 按客户端应用过滤规则。
- 在进入 OpenClash 前阻止广告和追踪域名。
- 将本地域名和公网域名分开处理。

---

## 3. 修改前备份

```bash
TS="$(date +%Y%m%d-%H%M%S)"
mkdir -p /root/dns-config-backups

cp -a /etc/AdGuardHome.yaml \
  "/root/dns-config-backups/AdGuardHome.yaml.$TS.bak"

ls -lh /root/dns-config-backups/
```

---

## 4. 前置条件

配置 AdGuard Home 前必须确认：

1. dnsmasq 已从 `53` 移到 `54`。
2. LAN DHCP 已通过 Option 6 下发 `192.168.7.1` 作为 DNS。
3. OpenClash 正在监听 TCP/UDP `7874`。
4. OpenClash 已删除所有 `127.0.0.1:38617` 上游。
5. OpenClash 本地 DNS 劫持已关闭。
6. OpenClash 已配置可用的外部 DNS。

检查服务端口：

```bash
netstat -lnptu 2>/dev/null | \
grep -Ei ':54|:7874|dnsmasq|clash'
```

检查 DHCP Option 6：

```bash
uci show dhcp.lan | grep dhcp_option
```

应包含：

```text
dhcp.lan.dhcp_option='6,192.168.7.1'
```

原因：dnsmasq 从标准 DNS 端口 `53` 移到 `54` 后，OpenWrt 不会再自动把路由器地址作为 DNS 服务器通过 DHCP 发布给客户端。必须显式使用 DHCP Option 6。

具体命令见配套 OpenClash/dnsmasq 文档。

## 5. AdGuard Home 监听设置

AdGuard Home DNS 监听：

```text
127.0.0.1:53
192.168.7.1:53
::1:53
```

管理界面：

```text
3000
```

`3000` 是 Web 管理端口，不是 DNS 端口。

推荐的 `AdGuardHome.yaml` 片段：

```yaml
dns:
  bind_hosts:
    - 127.0.0.1
    - ::1
    - 192.168.7.1

  port: 53
```

不建议为了方便直接监听所有 WAN 接口。  
本文只绑定回环地址和 LAN 地址。

---

## 6. 上游 DNS 设置

进入：

```text
AdGuard Home → 设置 → DNS 设置
```

### 6.1 普通上游 DNS

填写：

```text
127.0.0.1:7874
```

所有普通公网域名将进入 OpenClash。

### 6.2 `.lan` 专用上游

继续添加：

```text
[/lan/]127.0.0.1:54
```

作用：

```text
ytq77.lan
MiCO.lan
haier-water-heater.lan
```

等本地域名直接交给 dnsmasq。

最终上游列表：

```text
127.0.0.1:7874
[/lan/]127.0.0.1:54
```

对应 YAML：

```yaml
upstream_dns:
  - 127.0.0.1:7874
  - '[/lan/]127.0.0.1:54'
```

---

## 7. Bootstrap DNS

本文的 AGH 普通上游是：

```text
127.0.0.1:7874
```

它是 IP 地址，不需要 AGH 先解析上游域名，因此 Bootstrap DNS 可以清空：

```yaml
bootstrap_dns: []
```

AliDNS 的 DoH/DoT 域名解析由 OpenClash 的 `default-nameserver` 负责。

---

## 8. 后备 DNS

建议清空：

```yaml
fallback_dns: []
```

不要在 AGH 后备 DNS 中保留运营商 DNS，例如：

```text
202.102.x.x
```

否则 OpenClash 超时或停止时，AGH 可能直接向运营商 DNS 查询，绕过 OpenClash。

如果明确希望 OpenClash 故障时仍能解析，可以自行保留后备 DNS，但需要接受 DNS 绕过。

---

## 9. 私人反向 DNS

在：

```text
AdGuard Home → 设置 → DNS 设置 → 私人反向 DNS 服务器
```

填写：

```text
127.0.0.1:54
```

开启：

```text
使用私人反向 DNS 解析器
启用客户端 IP 地址的反向解析
```

对应 YAML：

```yaml
use_private_ptr_resolvers: true

local_ptr_upstreams:
  - 127.0.0.1:54
```

作用：

```text
192.168.7.181 → MiCO.lan
192.168.7.157 → ytq-18-IPad11.lan
192.168.7.191 → haier-water-heater.lan
```

---

## 10. 上游模式

当前普通上游只有：

```text
127.0.0.1:7874
```

推荐使用：

```text
负载均衡
```

单上游时模式差异不大。

不建议为了“更快”无条件使用并行模式；以后增加多个上游后，并行模式会同时向所有上游发送查询。

---

## 11. DNS 缓存

建议：

- 关闭 AGH DNS 缓存。
- 关闭乐观缓存。
- dnsmasq 缓存保持为 `0`。
- 不要让 AGH 和 dnsmasq 重复承担公网 DNS 缓存。

---

## 12. 不启用 AGH 自带 DNS 重定向

本文由 AdGuard Home 直接监听 LAN 的 `53`，AGH LuCI 页面中的重定向模式保持：

```text
无
```

OpenClash 本地 DNS 劫持也保持关闭。

但“AGH 监听 53”并不等于“客户端会自动使用 AGH”。  
必须在 LAN DHCP 中显式加入：

```uci
list dhcp_option '6,192.168.7.1'
```

并让客户端重新续租。

Windows：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

检查：

```cmd
ipconfig /all | findstr /i /c:"DNS Servers" /c:"DNS 服务器"
```

必须看到：

```text
192.168.7.1
```

若以后需要强制拦截客户端手动填写的普通 DNS，可单独建立 LAN TCP/UDP 53 重定向规则。  
普通 53 端口重定向不能拦截浏览器 DoH、Android 私人 DNS或应用内置加密 DNS。

## 13. 完整 YAML 关键片段

```yaml
dns:
  bind_hosts:
    - 127.0.0.1
    - ::1
    - 192.168.7.1

  port: 53

  upstream_dns:
    - 127.0.0.1:7874
    - '[/lan/]127.0.0.1:54'

  bootstrap_dns: []
  fallback_dns: []

  use_private_ptr_resolvers: true

  local_ptr_upstreams:
    - 127.0.0.1:54
```

其他 AGH 过滤器、日志、缓存和客户端规则按个人需求保留。

> 修改 YAML 前先停止 AGH，并且只修改已有的 `dns:` 段，不要创建第二个 `dns:` 段。

服务控制：

```bash
/etc/init.d/AdGuardHome status
/etc/init.d/AdGuardHome stop
/etc/init.d/AdGuardHome start
/etc/init.d/AdGuardHome restart
```

配置检查命令是否可用取决于当前 AGH 核心版本。  
本文实机主要通过服务启动日志和端口监听验证。

---

## 14. 端口验证

```bash
netstat -lnptu 2>/dev/null | \
grep -Ei ':53|:54|:7874|dnsmasq|AdGuardHome|clash'
```

正确状态：

| 端口 | 进程 |
|---:|---|
| `53` | AdGuardHome |
| `54` | dnsmasq |
| `7874` | clash |
| `3000` | AdGuardHome Web |

不应出现：

```text
dnsmasq :53
AdGuardHome :38617
```

---

## 15. DNS 链路验证

### 15.1 先验证客户端默认 DNS

Windows：

```cmd
ipconfig /all | findstr /i /c:"DNS Servers" /c:"DNS 服务器"
```

正确结果必须包含：

```text
DNS 服务器：192.168.7.1
```

若只显示：

```text
fec0:0:0:ffff::1
```

且没有 `192.168.7.1`，说明系统默认 DNS 没有指向 AGH。

此时可能出现：

```cmd
nslookup openwrt.org 192.168.7.1
```

成功，但：

```cmd
nslookup openwrt.org
```

失败。

这说明 AGH 和 OpenClash 服务端正常，问题在 DHCP DNS 下发，应检查 DHCP Option 6。

### 15.2 公网域名

先让客户端续租：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

测试默认 DNS：

```cmd
nslookup openwrt.org
```

Fake-IP 模式下预期：

```text
服务器:  ytq77.lan
Address:  192.168.7.1

openwrt.org → 198.18.x.x
```

这说明：

```text
客户端默认 DNS → AGH → OpenClash Fake-IP
```

也可以显式测试 AGH：

```cmd
nslookup openwrt.org 192.168.7.1
```

但显式测试成功并不能替代默认 DNS 检查。

### 15.3 `.lan` 正向解析

```cmd
nslookup ytq77.lan 192.168.7.1
```

预期：

```text
ytq77.lan → 192.168.7.1
```

### 15.4 PTR 反向解析

```cmd
nslookup 192.168.7.1 192.168.7.1
```

预期：

```text
192.168.7.1 → ytq77.lan
```

### 15.5 客户端名称

在 AGH 查询日志中，客户端应显示为：

```text
192.168.7.157  ytq-18-IPad11.lan
192.168.7.181  MiCO.lan
192.168.7.191  haier-water-heater.lan
```

而不是全部显示：

```text
127.0.0.1
192.168.7.1
```

路由器本机发起的查询显示为 `127.0.0.1` 属于正常情况。

## 16. 重启后只显示 IP 的处理

路由器重启后，AGH 可能先记录纯 IP。  
此时 `/tmp/dhcp.leases` 中已经有主机名，PTR 查询也能成功，但 AGH 仪表盘尚未刷新运行时客户端名称。

检查租约：

```bash
cat /tmp/dhcp.leases
```

检查 PTR：

```bash
nslookup 192.168.7.181 192.168.7.1
nslookup 192.168.7.157 192.168.7.1
nslookup 192.168.7.191 192.168.7.1
```

如果 PTR 已经返回正确名称，只需：

```bash
/etc/init.d/AdGuardHome restart
```

刷新网页后名称即可恢复。

当前实机没有添加 AGH 自动刷新脚本。  
客户端名称暂时只显示 IP 不影响 DNS 主链路和网页访问。

## 17. DHCP 设备名称说明

普通设备可以只在 `/etc/config/dhcp` 中设置：

```uci
config host
        option name 'MiCO'
        list mac '04:78:63:37:03:04'
```

这表示：

- 主机名按 MAC 标记。
- IP 仍为动态分配。
- 设备取得租约后，dnsmasq 提供当前 IP 的 A/PTR 记录。

无需为了在 AGH 中显示名称而给所有设备固定 IP。

确实有端口转发、远程串流或游戏规则需求的设备，再额外设置固定 IP。

---

## 18. Tailscale

当前 AGH 监听：

```text
127.0.0.1:53
192.168.7.1:53
::1:53
```

没有直接监听 Tailscale `100.x` 地址，这不会影响通过 Tailscale 子网路由访问家庭局域网。

远程游戏数据路径是：

```text
远程设备 → Tailscale → 家庭子网路由 → 192.168.7.x
```

不是通过 AGH 或 OpenClash DNS 转发。

远程设备需要解析 `.lan` 时，可以通过子网路由查询：

```text
192.168.7.1:53
```

直接使用局域网 IP 玩游戏时，不受本次 DNS 改造影响。

---

## 19. 故障排查

### 默认查询失败，但指定 AGH 查询成功

症状：

```cmd
nslookup openwrt.org
```

失败，而：

```cmd
nslookup openwrt.org 192.168.7.1
```

返回 `198.18.x.x`。

检查：

```cmd
ipconfig /all | findstr /i /c:"DNS Servers" /c:"DNS 服务器"
```

如果没有 `192.168.7.1`，在路由器上加入 DHCP Option 6：

```bash
uci -q del_list dhcp.lan.dhcp_option='6,192.168.7.1'
uci add_list dhcp.lan.dhcp_option='6,192.168.7.1'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

然后 Windows 续租：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

### AGH 能看到真实 IP，但没有名称

```bash
cat /tmp/dhcp.leases
nslookup 客户端IP 192.168.7.1
/etc/init.d/AdGuardHome restart
```

### 公网域名超时

检查 OpenClash：

```bash
netstat -lnptu 2>/dev/null | grep ':7874'
```

安装 `bind-dig` 后直接查询非标准端口：

```bash
dig @127.0.0.1 -p 7874 openwrt.org A \
  +time=3 +tries=1 +short
```

检查 AGH 上游：

```bash
grep -nE \
'upstream_dns|127\.0\.0\.1:7874' \
/etc/AdGuardHome.yaml
```

### `.lan` 无法解析

检查 dnsmasq：

```bash
netstat -lnptu 2>/dev/null | grep ':54'
```

检查：

```bash
grep -nE \
"127\.0\.0\.1:54|local_ptr_upstreams|\[/lan/\]" \
/etc/AdGuardHome.yaml
```

### 查询循环或 CPU 异常

全局检查旧端口：

```bash
grep -RnsE \
'127\.0\.0\.1:38617|127\.0\.0\.1#38617|tcp://127\.0\.0\.1:38617' \
/etc/openclash \
/etc/config/openclash \
/etc/AdGuardHome.yaml 2>/dev/null
```

当前生效配置中不应包含 `38617`。  
备份文件中包含旧值不影响运行，但建议移到 `/root/dns-config-backups/`。

## 20. 参考资料

- [原 AdGuard Home 配置记录](https://github.com/ytq77/Network-HomeLab/blob/main/AdGuardHome/AdGuardHome_config.md)
- [原 OpenClash 配置记录](https://github.com/ytq77/Network-HomeLab/blob/main/OpenClash/Clash_config.md)
- [AdGuard Home Wiki：Configuration](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration)
- [AdGuard Home Wiki：Getting Started](https://github.com/AdguardTeam/AdGuardHome/wiki/Getting-Started)
- [OpenWrt Wiki：AdGuard Home](https://openwrt.org/docs/guide-user/services/dns/adguard-home)
- [OpenWrt Wiki：DHCP 与 DNS 配置](https://openwrt.org/docs/guide-user/base-system/dhcp)
- [OpenClash Wiki：DNS 设置](https://github.com/vernesong/OpenClash/wiki/DNS%E8%AE%BE%E7%BD%AE)
- [Mihomo Docs：DNS 配置](https://wiki.metacubex.one/config/dns/)
