# Network-HomeLab

家庭网络实验室配置集合，包含路由器代理、DNS 广告拦截、订阅转换等网络工具的配置文档和规则文件。

## 📁 项目结构

```
Network-HomeLab/
├── AdGuardHome/          # AdGuardHome DNS 广告拦截配置
├── MihomoParty/          # Windows 平台 Mihomo/Sparkle 客户端配置
├── OpenClash/            # OpenWrt/ImmortalWrt 平台 OpenClash 插件配置
├── RefuseADS/            # 广告拦截规则列表
└── subconverter/         # 订阅转换工具 Docker 部署教程
```

## 🛠️ 组件介绍

### 1. AdGuardHome

DNS 级别的广告拦截和隐私保护工具，适用于路由器环境。

**主要功能：**
- DNS 广告拦截
- 隐私保护
- 查询日志统计
- 自定义过滤规则

**相关文档：**
- [AdGuardHome 配置教程](./AdGuardHome/AdGuardHome_config.md)

**关键配置：**
- 工作端口：8008（Web 界面）、5335（DNS 服务）
- 与 OpenClash 配合使用，通过 dnsmasq 转发

### 2. OpenClash

基于 Clash 内核的 OpenWrt/ImmortalWrt 代理插件，提供完整的透明代理功能。

**主要功能：**
- 透明代理（Fake-IP 增强模式）
- 规则分流（直连/代理/拒绝）
- 订阅管理和自动更新
- 流媒体解锁支持

**相关文档：**
- [OpenClash 配置教程](./OpenClash/Clash_config.md)

**关键配置：**
- 使用 Meta 内核，Fake-IP 增强模式
- 与 AdGuardHome 配合，DNS 转发到 `127.0.0.1:5335`
- 支持 UDP 流量转发
- 绕过中国大陆 IP，使用延迟低的 DNS

### 3. MihomoParty (Sparkle)

Windows 平台的 Clash 客户端，提供图形化界面和丰富的配置选项。

**主要功能：**
- 系统代理和 TUN 模式
- 订阅管理和自动更新
- 规则自定义和覆写
- 连接监控和日志查看

**相关文档：**
- [Windows Mihomo 配置教程](./MihomoParty/Win-Mihomo.md)

**关键配置：**
- 支持 Clash 和通用订阅格式
- 支持模板自定义订阅转换
- DNS 配置使用运营商延迟低的 DNS
- 支持 Steam 下载直连规则

### 4. subconverter

订阅转换工具，支持多种订阅格式之间的转换。

**主要功能：**
- 订阅格式转换（Clash、V2Ray、Surge 等）
- 配置模板自定义
- Docker 容器化部署

**相关文档：**
- [subconverter Docker 部署教程](./subconverter/dockerAnalysis.md)

**部署方式：**
- Docker Desktop（Windows）
- Docker（Linux）
- 默认端口：25500

### 5. RefuseADS

广告拦截规则集合，适用于 AdGuardHome 和浏览器扩展。

**相关文档：**
- [广告拦截规则列表](./RefuseADS/list.md)

**推荐规则：**
- [217heidai/adblockfilters](https://github.com/217heidai/adblockfilters) - AdGuardHome 整合版规则
- [8680/GOODBYEADS](https://github.com/8680/GOODBYEADS) - 浏览器扩展整合版规则

## 🔧 快速开始

### 路由器环境（OpenWrt/ImmortalWrt）

1. **安装 OpenClash**
   - 参考 [OpenClash 配置教程](./OpenClash/Clash_config.md)
   - 配置订阅和规则

2. **安装 AdGuardHome**
   - 参考 [AdGuardHome 配置教程](./AdGuardHome/AdGuardHome_config.md)
   - 配置 DNS 转发和过滤规则

3. **配置联动**
   - OpenClash DNS 设置转发到 AdGuardHome
   - DHCP/DNS 重定向到 AdGuardHome

### Windows 环境

1. **部署 subconverter（可选）**
   - 参考 [subconverter Docker 部署教程](./subconverter/dockerAnalysis.md)
   - 用于订阅转换和模板自定义

2. **配置 MihomoParty**
   - 参考 [Windows Mihomo 配置教程](./MihomoParty/Win-Mihomo.md)
   - 导入订阅和配置规则

## 📝 配置要点

### DNS 配置

- **AdGuardHome**：使用运营商延迟低的 DNS（如 `219.147.1.66`）
- **OpenClash**：转发到 AdGuardHome（`127.0.0.1:5335`）
- **MihomoParty**：使用阿里云 DNS（`https://dns.alidns.com/dns-query`）

### 规则配置

- **直连规则**：中国大陆 IP、内网 IP、常用服务域名
- **代理规则**：国外服务、流媒体、特定域名
- **拒绝规则**：广告域名、恶意网站

### 订阅转换

支持通过 subconverter 实现：
- 配置模板自定义
- 后端订阅转换
- 一个链接同时实现模板和订阅转换

## ⚠️ 注意事项

1. **IPv6 配置**：建议在路由器环境关闭 IPv6，避免 DNS 泄露
2. **端口冲突**：AdGuardHome 默认 53 端口与 dnsmasq 冲突，建议改为 5335
3. **日志清理**：定期清理 AdGuardHome 日志文件，避免占用空间
4. **订阅更新**：建议开启自动更新，保持规则和节点最新
5. **DNS 泄露测试**：配置完成后使用 [browserleaks.com/dns](https://browserleaks.com/dns) 测试

## 📚 相关资源

- [ImmortalWrt 官方文档](https://immortalwrt.org/)
- [Clash 官方文档](https://clash.gitbook.io/doc/)
- [AdGuardHome 官方文档](https://github.com/AdguardTeam/AdGuardHome/wiki)
- [subconverter 官方仓库](https://github.com/tindy2013/subconverter)

## 📄 许可证

本项目仅用于学习和研究目的，请遵守相关法律法规。

## 🤝 贡献

欢迎提交 Issue 和 Pull Request 来改进本项目。

---

**最后更新**：2025-08-27
