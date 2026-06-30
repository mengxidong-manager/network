# IS-IS 协议详解与配置实例

IS-IS（Intermediate System to Intermediate System，中间系统到中间系统）和 OSPF 一样是**链路状态**型 IGP，工作原理高度相似——都用 Hello 发现邻居、同步链路状态数据库、用 SPF 算最短路径。但它出身于 OSI 协议栈，地址和分层方式跟 OSPF 差别很大。运营商骨干网尤其偏爱它。

## 一、IS-IS 的基本定位

IS-IS 最初是 ISO 为 OSI 网络（CLNP）设计的，后来扩展出**集成 IS-IS（Integrated IS-IS）**，可同时承载 IP 和 CLNP 路由，现在我们说的 IS-IS 基本都是这个版本。

它和 OSPF 一个根本区别是：**IS-IS 直接运行在数据链路层之上，不依赖 IP**。OSPF 报文封装在 IP 协议号 89 里，而 IS-IS 报文直接用二层帧承载。这带来两个实际影响：一是更难被远程攻击（没有 IP 头，无法跨网段直接打它）；二是它天生协议无关，给 IPv6 加支持只是多定义几个 TLV，不像 OSPF 那样要单独搞一个 OSPFv3。

## 二、核心概念

### 术语：IS 和 ES

在 IS-IS 的世界里，路由器叫 **IS（中间系统）**，主机叫 **ES（终端系统）**。所以协议名"IS to IS"的意思就是"路由器到路由器"的路由协议。

### NET 地址（IS-IS 最有特色的地方）

IS-IS 不靠 IP 来标识路由器，而是用一个 OSI 的 NSAP 地址，其中标识设备本身的那部分叫 **NET（Network Entity Title）**。一个典型 NET：

```
49.0001.0000.0000.0001.00
└─┬─┘ └─┬─┘ └──────┬─────┘ └┬┘
 AFI  Area ID   System ID   NSEL
```

- **Area ID（区域 ID）**：可变长（1~13 字节），上例 `49.0001` 即区域号。`49` 是私有 AFI，后面的 `0001` 是自定义区域编号。同一区域内所有路由器 Area ID 必须相同。
- **System ID（系统 ID）**：固定 **6 字节**，全网唯一，相当于 OSPF 的 Router ID。工程上常用固定算法把 Loopback 的 IP 填进去（如 1.1.1.1 → `0010.0100.1001`），方便记忆，但只要唯一即可。
- **NSEL（NSAP Selector）**：固定 1 字节，对路由器而言**必须是 `00`**，表示"这是设备本身"。

### Level-1、Level-2 与分层

- **Level-1（L1）**：负责**区域内部**路由。L1 路由器只了解自己区域的拓扑，去往区域外的流量交给最近的 L1/L2 路由器（靠默认路由）。
- **Level-2（L2）**：负责**区域之间**的路由，所有 L2 路由器及其链路构成**骨干**，骨干必须连续。
- **Level-1-2（L1/L2）**：同时参与两个级别，相当于 OSPF 的 ABR，是区域与骨干的连接点。Cisco 设备默认就是 L1/L2。

### 和 OSPF 区域最关键的区别——边界在哪里

OSPF 里区域边界落在**路由器上**：一台 ABR 的不同接口可属于不同区域。而 IS-IS 里，**一台路由器整体只属于一个区域**，区域边界落在**链路上**——区域分界发生在两台不同区域路由器之间的那条链路上，通过它们之间建立 L2 邻接来跨区域。记住"OSPF 边界在路由器、IS-IS 边界在链路"，两者拓扑差异就清楚了。

### 邻接关系

IS-IS 用 **IIH（IS-IS Hello）**报文发现和维持邻居：同区域路由器可建立 **L1 邻接**；不同区域（或都是 L2）的路由器之间建立 **L2 邻接**；L1/L2 路由器在一条链路上可能同时建立 L1 和 L2 两种邻接。建立邻接要求两端级别、区域（对 L1）、Hello/Hold 定时器、接口 MTU、认证等匹配。

### DIS（指定中间系统）

在广播型网络上 IS-IS 也选一个代表叫 **DIS**，作用类似 OSPF 的 DR，但有几个重要不同：

- **没有备份 DIS**。DIS 挂了直接重选，因为 IS-IS 收敛快，不需要备份。
- **DIS 是抢占式的**。优先级更高的路由器接入后会立刻顶替现任 DIS。
- 选举依据是**接口优先级**（默认 64，越大越优），相同则比 **SNPA/MAC 地址**大的胜出（OSPF 比 Router ID）。

### PDU 类型

**IIH**（Hello）、**LSP**（链路状态报文，相当于 OSPF 的 LSA）、**CSNP**（完整序列号 PDU，类似数据库摘要，DIS 周期性发送）、**PSNP**（部分序列号 PDU，请求和确认 LSP）。

### 度量值（Metric）

IS-IS 默认用**窄度量（narrow metric）**，每条链路 cost 最大 63，整条路径最大 1023——大网络不够用。现代部署一律改用**宽度量（wide metric）**，链路 cost 可达 2^24-1。IS-IS 接口默认 cost 是 **10**（不随带宽自动变化），通常需手工规划。

## 三、工作流程

和 OSPF 同一套思路：IIH 发现并维持邻接 → LSP 泛洪交换链路状态、各自构建相同 LSDB → 基于 LSDB 用 SPF 算法独立算出最短路径 → 装入路由表。L1 和 L2 各自维护独立的 LSDB 并独立做 SPF。

## 四、配置实例（Cisco IOS）

拓扑：R1、R2 在区域 `49.0001`，R3、R4 在区域 `49.0002`，R2 与 R3 之间通过 **L2 邻接**构成骨干。

```
   [Area 49.0001]                            [Area 49.0002]

    R1 ------------- R2 =============== R3 ------------- R4
 (Level-1)       (Level-1-2)        (Level-1-2)      (Level-1)
       10.1.12.0/24  L2 骨干 10.1.23.0/24  10.1.34.0/24
        (L1 邻接)      (L2 邻接)         (L1 邻接)
```

配置要点：`router isis` 进程里写 NET 和级别，每个要跑 IS-IS 的接口（含 Loopback）都要加 `ip router isis`，再用 `isis circuit-type` 控制这条链路建立哪种邻接。`metric-style wide` 强烈建议全网开启。

**R1（区域 49.0001，仅 L1）**

```
hostname R1
!
router isis
 net 49.0001.0000.0000.0001.00
 is-type level-1
 metric-style wide
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 ip address 10.1.12.1 255.255.255.0
 ip router isis
 isis circuit-type level-1
```

**R2（区域 49.0001，L1/L2；朝 R1 走 L1，朝 R3 的骨干口走 L2）**

```
hostname R2
!
router isis
 net 49.0001.0000.0000.0002.00
 is-type level-1-2
 metric-style wide
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0          ! 朝向 R1
 ip address 10.1.12.2 255.255.255.0
 ip router isis
 isis circuit-type level-1
!
interface GigabitEthernet0/1          ! 朝向 R3，骨干链路
 ip address 10.1.23.2 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
```

**R3（区域 49.0002，L1/L2；朝 R2 的骨干口走 L2，朝 R4 走 L1）**

```
hostname R3
!
router isis
 net 49.0002.0000.0000.0003.00
 is-type level-1-2
 metric-style wide
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0          ! 朝向 R2，骨干链路
 ip address 10.1.23.3 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
!
interface GigabitEthernet0/1          ! 朝向 R4
 ip address 10.1.34.3 255.255.255.0
 ip router isis
 isis circuit-type level-1
```

**R4（区域 49.0002，仅 L1）**

```
hostname R4
!
router isis
 net 49.0002.0000.0000.0004.00
 is-type level-1
 metric-style wide
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 ip address 10.1.34.4 255.255.255.0
 ip router isis
 isis circuit-type level-1
```

把 R2–R3 链路设成 `level-2-only` 是关键：它俩在不同区域，本就不可能建立 L1 邻接，显式只跑 L2 既能正确形成骨干，又避免白白发 L1 的 Hello。点到点路由口上可加 `isis network point-to-point`，跳过 DIS 选举、收敛更快。

## 五、常用验证命令

```
show isis neighbors        # 看邻居及级别（L1/L2），状态应为 Up
show isis database         # 看 LSDB
show clns interface        # 看接口 IS-IS 状态、circuit-type、DIS
show ip route isis         # i L1=区域内, i L2=区域间
show clns is-neighbors     # 更详细的邻居信息
```

邻接建立不起来重点查：System ID 是否全网唯一、L1 邻接两端 Area ID 是否相同、接口 MTU、circuit-type/级别是否匹配、认证、是否误设 passive。

## 六、IS-IS 与 OSPF 速查对比

| 维度 | OSPF | IS-IS |
|---|---|---|
| 承载层 | IP（协议号 89） | 直接跑在二层之上 |
| 设备标识 | Router ID（IP 格式） | NET / System ID（OSI 地址） |
| 分层方式 | Area 0 + 普通区域 | Level-1 / Level-2 |
| 区域边界 | 在路由器上 | 在链路上 |
| 代表选举 | DR + BDR，非抢占 | DIS，无备份、抢占式 |
| 度量 | 基于带宽自动算 | 接口默认 10，需手工/宽度量规划 |
| IPv6 支持 | 需独立的 OSPFv3 | 同协议加 TLV 即可 |
| 典型场景 | 企业网为主 | 运营商骨干网偏爱 |

---

[← 上一篇：OSPF 协议](01-OSPF.md) · [返回目录](README.md) · [下一篇：BGP 协议 →](03-BGP.md)
