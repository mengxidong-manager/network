# 网络学习

网络协议学习笔记，整理自一次关于路由协议的系统对话。涵盖两大类内部网关协议（IGP）、边界网关协议（BGP），以及它们在真实网络中协同工作的综合实例与进阶专题。所有配置示例均以 Cisco IOS 语法给出，每篇均配有 SVG 拓扑图。

## 目录

### 基础篇

| 序号 | 主题 | 文件 |
|---|---|---|
| 1 | OSPF 协议详解与配置实例 | [01-OSPF.md](01-OSPF.md) |
| 2 | IS-IS 协议详解与配置实例 | [02-ISIS.md](02-ISIS.md) |
| 3 | BGP 协议详解与配置实例 | [03-BGP.md](03-BGP.md) |
| 4 | IGP 与 BGP 结合实例 | [04-IGP-BGP-结合.md](04-IGP-BGP-结合.md) |

### 进阶篇

| 序号 | 主题 | 文件 |
|---|---|---|
| 5 | OSPFv3：用 OSPF 跑 IPv6 | [05-OSPFv3.md](05-OSPFv3.md) |
| 6 | BGP 路由反射器 | [06-路由反射器.md](06-路由反射器.md) |
| 7 | BGP 团体属性 | [07-BGP团体属性.md](07-BGP团体属性.md) |
| 8 | MPLS L3VPN | [08-MPLS-L3VPN.md](08-MPLS-L3VPN.md) |
| 9 | BGP 选路实验：主备出口 | [09-BGP选路实验.md](09-BGP选路实验.md) |
| 10 | MPLS L3VPN 进阶：Hub-Spoke 与 Extranet | [10-MPLS-HubSpoke-Extranet.md](10-MPLS-HubSpoke-Extranet.md) |
| 11 | BGP 的 BFD 快速故障检测 | [11-BGP-BFD.md](11-BGP-BFD.md) |
| 12 | prefix-list 与 route-map 综合实战 | [12-prefix-list与route-map实战.md](12-prefix-list与route-map实战.md) |

### 实战篇

| 序号 | 主题 | 文件 |
|---|---|---|
| 13 | Cloudflare + Grafana 安全暴露配置 | [13-Cloudflare-Grafana安全暴露配置.md](13-Cloudflare-Grafana安全暴露配置.md) |

## 协议速查对比

| 维度 | OSPF | IS-IS | BGP |
|---|---|---|---|
| 类型 | 链路状态 IGP | 链路状态 IGP | 路径矢量 EGP |
| 作用范围 | 单个 AS 内部 | 单个 AS 内部 | AS 之间 |
| 承载层 | IP（协议号 89） | 直接跑在二层之上 | TCP 179 |
| 设备标识 | Router ID | NET / System ID | Router ID + AS 号 |
| 分层方式 | Area 0 + 普通区域 | Level-1 / Level-2 | eBGP / iBGP |
| 区域边界 | 在路由器上 | 在链路上 | AS 边界 |
| 选路依据 | 最短路径（cost） | 最短路径（metric） | 策略 + 路径属性 |
| 典型场景 | 企业网 | 运营商骨干 | 互联网 AS 互联 |

## 学习路线建议

先掌握一种 IGP（OSPF 最常见），理解"链路状态 + SPF + 区域分层"这套思路；再看 IS-IS，重点对比它与 OSPF 的差异（地址、分层、边界位置）；然后学 BGP，理解它"策略选路、AS 之间互联"的本质；第 4 篇把 IGP 与 BGP 串起来，理解真实网络中"IGP 管内部可达、BGP 管外部策略、靠递归查找衔接"的部署形态。

打好基础后，进阶篇分别延伸到：IPv6（OSPFv3）、大规模 iBGP 的简化（路由反射器）、精细化策略控制（团体属性）、运营商多客户隔离方案（MPLS L3VPN）、把选路属性落到实操的动手实验（BGP 主备出口）；再进一步是 MPLS L3VPN 的非全互联拓扑设计（Hub-Spoke 与 Extranet 的 RT 设计）、亚秒级收敛（BGP 的 BFD 快速故障检测），以及路由策略的两大基础工具综合实战（prefix-list 与 route-map）。

实战篇收录实际运维中的网络配置案例，如 Cloudflare CDN/WAF 结合业务服务的安全暴露方案。

## 许可

本仓库内容采用 [MIT License](LICENSE) 授权。
