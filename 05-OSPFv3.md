# OSPFv3：用 OSPF 跑 IPv6

OSPFv3（RFC 5340）是 OSPF 的 IPv6 版本。它沿用了 OSPFv2 的核心机制——链路状态、SPF 算法、区域分层、DR/BDR、邻居状态机全都一样——所以学过 [01-OSPF.md](01-OSPF.md) 后，这里只需重点掌握**它和 OSPFv2 的差异**。

## 一、和 OSPFv2 相比变了什么

**① 直接为 IPv6 而生，按"链路"而非"网段"运行。** OSPFv2 的运作单位是 IP 子网，OSPFv3 改成按链路（link）运作，一条链路上可以承载多个 IPv6 前缀。

**② 邻居用链路本地地址（Link-Local）通信。** OSPFv3 的 Hello 和大部分协议报文都用接口的 `FE80::/10` 链路本地地址收发，邻居的下一跳也记成链路本地地址。所以即使接口上还没配全局 IPv6 地址，只要有链路本地地址，邻居就能起来。

**③ Router ID / Area ID 仍是 32 位，且必须手工配。** 它们的格式还是点分的 32 位（像 IPv4），**但不再从地址自动派生**——因为设备上可能根本没有 IPv4 地址。所以 OSPFv3 里 `router-id` 通常必须手工指定，否则进程起不来。

**④ 认证交给 IPv6 的 IPsec。** OSPFv2 报文里自带认证字段，OSPFv3 把认证能力移除，改为依赖 IPv6 的 AH/ESP（IPsec）来做，协议本身更简洁。

**⑤ 新增了两类 LSA。** 在原有 LSA 基础上增加了 **Type 8（Link LSA）**：通告本链路的链路本地地址和前缀，仅在链路范围内有效；以及 **Type 9（Intra-Area-Prefix LSA）**：把 IPv6 前缀信息和拓扑信息解耦后单独通告。这样做的好处是拓扑变化和前缀变化互不影响。

**⑥ 配置方式是接口级的。** 不再用 `network` 命令配反掩码宣告网段，而是直接在接口下用 `ipv6 ospf <进程> area <区域>` 把接口加进 OSPFv3。

## 二、配置实例（Cisco IOS 传统写法）

拓扑沿用单区域三台路由器，全部在 Area 0，跑 IPv6：

```
   R1 --------------- R2 --------------- R3
        2001:DB8:12::/64   2001:DB8:23::/64
  Lo0 2001:DB8::1      Lo0 2001:DB8::2    Lo0 2001:DB8::3
   全部 Area 0
```

先要全局打开 IPv6 路由转发，再在每个接口上启用 OSPFv3，并手工配 router-id。

**R1**

```
ipv6 unicast-routing
!
interface Loopback0
 ipv6 address 2001:DB8::1/128
 ipv6 ospf 1 area 0
!
interface GigabitEthernet0/0
 ipv6 address 2001:DB8:12::1/64
 ipv6 ospf 1 area 0
!
ipv6 router ospf 1
 router-id 1.1.1.1            ! 必须手工配置
```

**R2**

```
ipv6 unicast-routing
!
interface Loopback0
 ipv6 address 2001:DB8::2/128
 ipv6 ospf 1 area 0
!
interface GigabitEthernet0/0
 ipv6 address 2001:DB8:12::2/64
 ipv6 ospf 1 area 0
interface GigabitEthernet0/1
 ipv6 address 2001:DB8:23::2/64
 ipv6 ospf 1 area 0
!
ipv6 router ospf 1
 router-id 2.2.2.2
```

**R3**

```
ipv6 unicast-routing
!
interface Loopback0
 ipv6 address 2001:DB8::3/128
 ipv6 ospf 1 area 0
!
interface GigabitEthernet0/0
 ipv6 address 2001:DB8:23::3/64
 ipv6 ospf 1 area 0
!
ipv6 router ospf 1
 router-id 3.3.3.3
```

## 三、地址族（Address-Family）写法

新版 IOS 推荐用 OSPFv3 的**地址族**形式，一个进程里同时承载 IPv4 和 IPv6（这套也叫 OSPFv3 AF）：

```
router ospfv3 1
 router-id 1.1.1.1
 address-family ipv6 unicast
 exit-address-family
!
interface GigabitEthernet0/0
 ospfv3 1 ipv6 area 0
```

## 四、常用验证命令

把 OSPFv2 命令里的 `ip` 换成 `ipv6` 即可：

```
show ipv6 ospf neighbor          # 看邻居，正常应为 FULL
show ipv6 ospf interface brief   # 看接口区域、DR/BDR
show ipv6 route ospf             # 看学到的 OSPFv3 路由（标记 O）
show ipv6 ospf database          # 看 LSDB（可看到新增的 Link/Intra-Area-Prefix LSA）
```

排错思路和 OSPFv2 一致，额外要注意两点：**router-id 是否漏配**（IPv6 环境下不会自动生成），以及**接口链路本地地址是否正常**（邻居靠它通信）。

---

[← 上一篇：IGP 与 BGP 结合实例](04-IGP-BGP-结合.md) · [返回目录](README.md) · [下一篇：BGP 路由反射器 →](06-路由反射器.md)
