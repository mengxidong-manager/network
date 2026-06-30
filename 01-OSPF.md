# OSPF 协议详解与配置实例

OSPF（Open Shortest Path First，开放最短路径优先）是目前企业网和运营商网络里使用最广泛的内部网关协议（IGP）之一。

## 一、OSPF 的基本定位

OSPF 是一种**链路状态（Link-State）**路由协议，运行在自治系统（AS）内部，用来在路由器之间交换路由信息、计算到各网段的最优路径。

它和 RIP 这类**距离矢量**协议最大的区别在于：距离矢量协议是"道听途说"——每台路由器只知道邻居告诉它的路由，并不了解整个网络的拓扑；而链路状态协议是"人手一张地图"——每台路由器都会收集全网的链路状态信息，在本地构建出一张完整的拓扑图，然后用 **Dijkstra（SPF）算法**独立算出到每个目标的最短路径。

这带来几个直接好处：收敛快、无路由环路、支持大规模分层网络、用带宽而非跳数作为度量。

OSPF 常见有两个版本：OSPFv2（用于 IPv4，RFC 2328）和 OSPFv3（用于 IPv6，RFC 5340）。本文以最常用的 OSPFv2 为主。

## 二、核心概念

### Router ID（路由器标识）

每台运行 OSPF 的路由器有唯一的 32 位 Router ID，格式像 IP 地址。选举顺序是：手工配置的 router-id > 最大的 Loopback 接口 IP > 最大的物理接口 IP。生产环境强烈建议手工指定或配置 Loopback，否则接口一变动 Router ID 就可能跟着变，导致邻居重建。

### 区域（Area）与骨干区域

OSPF 通过分层来控制规模。所有区域必须直接或间接连接到**骨干区域 Area 0**，区域间的流量都要经过 Area 0 中转。分区域的目的是把一个区域内的拓扑变动"隔离"在区域内部，不让 LSA 泛洪到整个网络，从而减小路由表、降低 SPF 计算开销。

### 路由器类型

- **内部路由器**：所有接口在同一区域。
- **骨干路由器**：至少有一个接口在 Area 0。
- **区域边界路由器 ABR**：连接多个区域，做区域间路由汇总。
- **自治系统边界路由器 ASBR**：把外部路由（静态、BGP 等）引入 OSPF。

### LSA（链路状态通告）类型

| 类型 | 名称 | 产生者 | 作用 |
|---|---|---|---|
| Type 1 | Router LSA | 每台路由器 | 描述自己的接口和链路，本区域内泛洪 |
| Type 2 | Network LSA | DR | 描述多路访问网络上的路由器，区域内泛洪 |
| Type 3 | Network Summary LSA | ABR | 把一个区域的网段通告到其他区域 |
| Type 4 | ASBR Summary LSA | ABR | 告诉其他区域如何到达 ASBR |
| Type 5 | AS External LSA | ASBR | 描述引入的外部路由，全 AS 泛洪 |
| Type 7 | NSSA External LSA | NSSA 中的 ASBR | 在 NSSA 区域替代 Type 5 |

### 邻居关系的建立（状态机）

```
Down → Init → Two-Way → ExStart → Exchange → Loading → Full
```

收到对方 Hello 进入 Init；在对方 Hello 里看到自己进入 Two-Way（邻居关系建立）；之后交换 DBD 描述数据库、请求缺失 LSA；数据库完全同步后到达 **Full**（邻接关系建立完成）。只有到 Full 状态才算真正可以交换完整路由。

### DR / BDR 选举

在以太网这种多路访问网络里，如果每台路由器都和其他所有路由器建立 Full 邻接，邻接数会爆炸式增长。所以 OSPF 选出一个 DR（指定路由器）和 BDR（备份指定路由器），其余路由器只和 DR/BDR 建立 Full 邻接。选举依据是接口的 **OSPF 优先级**（默认 1，0 表示永不参选），优先级相同则比 Router ID 大的胜出。DR 是**接口级别**而非路由器级别的概念，且选举是"非抢占"的。

### 度量值 Cost（开销）

OSPF 用 Cost 作为度量，默认 `Cost = 参考带宽 / 接口带宽`，思科默认参考带宽是 100 Mbps。所以 100M 接口 cost=1，10M 接口 cost=10。千兆、万兆接口算出来都小于 1 会被取整成 1，无法区分，因此大网络通常用 `auto-cost reference-bandwidth` 调高参考带宽。

## 三、工作流程总结

四步：路由器之间用 **Hello 包**发现邻居并维持关系 → 邻居间**同步链路状态数据库（LSDB）** → 每台路由器基于相同的 LSDB 用 **SPF 算法**独立计算最短路径树 → 把结果装入路由表。之后只在拓扑变化时增量泛洪 LSA 并重算。

## 四、配置实例（Cisco IOS）

拓扑：R1 在 Area 1，R2、R3 是骨干区域 Area 0 的路由器（同时分别作为 Area 1 和 Area 2 的 ABR），R4 在 Area 2。

```
   [Area 1]               [Area 0 骨干]              [Area 2]

    R1 ------------- R2 =============== R3 ------------- R4
  (内部)           (ABR)             (ABR)            (内部)
  1.1.1.1          2.2.2.2           3.3.3.3          4.4.4.4
       10.1.12.0/24     10.1.23.0/24      10.1.34.0/24
```

`network` 命令后跟的是**反掩码（通配符掩码）**——子网掩码按位取反，比如 255.255.255.0 对应 0.0.0.255。它把本机上落在该地址范围内的接口宣告进指定的 area。

**R1（Area 1 内部路由器）**

```
hostname R1
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.12.1 255.255.255.0
!
router ospf 1
 router-id 1.1.1.1
 network 10.1.12.0 0.0.0.255 area 1
 network 1.1.1.1 0.0.0.0 area 1
```

**R2（ABR，一只脚在 Area 1，一只脚在 Area 0）**

```
hostname R2
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.12.2 255.255.255.0
interface GigabitEthernet0/1
 ip address 10.1.23.2 255.255.255.0
!
router ospf 1
 router-id 2.2.2.2
 network 10.1.12.0 0.0.0.255 area 1
 network 10.1.23.0 0.0.0.255 area 0
 network 2.2.2.2 0.0.0.0 area 0
```

**R3（ABR，连接 Area 0 与 Area 2）**

```
hostname R3
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.23.3 255.255.255.0
interface GigabitEthernet0/1
 ip address 10.1.34.3 255.255.255.0
!
router ospf 1
 router-id 3.3.3.3
 network 10.1.23.0 0.0.0.255 area 0
 network 10.1.34.0 0.0.0.255 area 2
 network 3.3.3.3 0.0.0.0 area 0
```

**R4（Area 2 内部路由器）**

```
hostname R4
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.34.4 255.255.255.0
!
router ospf 1
 router-id 4.4.4.4
 network 10.1.34.0 0.0.0.255 area 2
 network 4.4.4.4 0.0.0.0 area 2
```

### 更现代的写法（接口级配置）

不用 `network` 命令，直接在接口下指定区域，逻辑更直观、不易因反掩码算错而误宣告：

```
interface GigabitEthernet0/0
 ip ospf 1 area 1
```

这种方式下 `router ospf 1` 进程里就不需要 network 语句了。两种写法可混用。

## 五、常用验证与排错命令

```
show ip ospf neighbor          # 看邻居状态，正常应是 FULL
show ip ospf interface brief   # 看接口区域、DR/BDR
show ip route ospf             # O=区域内, O IA=区域间, O E2=外部
show ip ospf database          # 看 LSDB
show ip protocols              # 看进程、router-id、宣告网段
```

邻居建立不起来常见原因（两端必须一致）：区域号不同、Hello/Dead 定时器不一致、接口 MTU 不匹配、认证不一致、网络类型不匹配、一端被设成 passive-interface。

## 六、生产环境常用优化

调整参考带宽（全网统一）：

```
router ospf 1
 auto-cost reference-bandwidth 10000   # 单位 Mbps
```

被动接口（只通告网段不发 Hello）：

```
router ospf 1
 passive-interface GigabitEthernet0/2
```

下发默认路由：

```
router ospf 1
 default-information originate
```

---

[返回目录](README.md) · [下一篇：IS-IS 协议 →](02-ISIS.md)
