# IGP 与 BGP 结合实例

这是真实网络里最经典的部署形态——**IGP 管内部、BGP 管外部，两者各司其职又相互配合**。本文用一个自治系统（AS 65001）作为"中转网络"的例子把它们串起来：AS 65001 内部跑 OSPF，三台路由器之间跑 iBGP，两台边界路由器分别用 eBGP 连到外部的 AS 65002 和 AS 65003。

## 拓扑

```
 [AS 65002]      [-------------- AS 65001（内部 OSPF + iBGP）--------------]      [AS 65003]

    R4 ------------- R1 --------------- R2 --------------- R3 ------------- R5
 (eBGP对端)       (边界)             (内部)             (边界)          (eBGP对端)
              Lo0 1.1.1.1        Lo0 2.2.2.2        Lo0 3.3.3.3
   192.0.2.0/30   10.0.12.0/24       10.0.23.0/24       198.51.100.0/30
     eBGP            OSPF               OSPF               eBGP

 宣告172.16.4.0/24   （AS65001 宣告 203.0.113.0/24）         宣告172.16.5.0/24
```

注意 **iBGP 不是物理链路**——它是 R1、R2、R3 之间基于各自 Loopback（1.1.1.1 / 2.2.2.2 / 3.3.3.3）建立的逻辑全互联会话，叠加在 OSPF 提供的内部可达性之上。外部 R4 宣告 172.16.4.0/24，R5 宣告 172.16.5.0/24，AS 65001 自己对外宣告 203.0.113.0/24。

## 一、完整配置

### R1（边界路由器：OSPF + iBGP + eBGP）

```
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
interface Loopback1
 ip address 203.0.113.1 255.255.255.0        ! 本 AS 对外宣告的业务网段
interface GigabitEthernet0/0
 ip address 192.0.2.1 255.255.255.252         ! 朝 R4，eBGP 链路
interface GigabitEthernet0/1
 ip address 10.0.12.1 255.255.255.0           ! 朝 R2，内部链路
!
router ospf 1
 router-id 1.1.1.1
 network 10.0.12.0 0.0.0.255 area 0
 network 1.1.1.1 0.0.0.0 area 0               ! 把 Loopback 宣告进 OSPF（iBGP 要用）
!
router bgp 65001
 bgp router-id 1.1.1.1
 ! ---- iBGP：基于 Loopback 与 R2、R3 全互联 ----
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 next-hop-self
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 3.3.3.3 next-hop-self
 ! ---- eBGP：连 AS 65002 ----
 neighbor 192.0.2.2 remote-as 65002
 ! ---- 对外宣告本地网段 ----
 network 203.0.113.0 mask 255.255.255.0
```

### R2（内部路由器：只跑 OSPF + iBGP，不接外部）

```
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
interface GigabitEthernet0/0
 ip address 10.0.12.2 255.255.255.0
interface GigabitEthernet0/1
 ip address 10.0.23.2 255.255.255.0
!
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.255 area 0
 network 10.0.23.0 0.0.0.255 area 0
 network 2.2.2.2 0.0.0.0 area 0
!
router bgp 65001
 bgp router-id 2.2.2.2
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
```

### R3（边界路由器：OSPF + iBGP + eBGP，与 R1 对称）

```
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
interface GigabitEthernet0/0
 ip address 10.0.23.3 255.255.255.0           ! 朝 R2，内部链路
interface GigabitEthernet0/1
 ip address 198.51.100.1 255.255.255.252       ! 朝 R5，eBGP 链路
!
router ospf 1
 router-id 3.3.3.3
 network 10.0.23.0 0.0.0.255 area 0
 network 3.3.3.3 0.0.0.0 area 0
!
router bgp 65001
 bgp router-id 3.3.3.3
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.1 next-hop-self
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 next-hop-self
 neighbor 198.51.100.2 remote-as 65003
```

### R4（AS 65002）和 R5（AS 65003）只跑 eBGP

```
! R4：Gi0/0 = 192.0.2.2/30，Lo1 = 172.16.4.1/24
router bgp 65002
 bgp router-id 4.4.4.4
 neighbor 192.0.2.1 remote-as 65001
 network 172.16.4.0 mask 255.255.255.0
!
! R5：Gi0/0 = 198.51.100.2/30，Lo1 = 172.16.5.1/24
router bgp 65003
 bgp router-id 5.5.5.5
 neighbor 198.51.100.1 remote-as 65001
 network 172.16.5.0 mask 255.255.255.0
```

## 二、关键说明：两者怎么配合

**① 分工明确，互不混淆。** OSPF 只承载 AS **内部**的基础设施路由——内部链路网段和各路由器的 Loopback。BGP 承载 AS **之间**的业务前缀（172.16.4.0、172.16.5.0、203.0.113.0 等）。两张路由表各管各的，靠"递归"衔接，而不是互相重分发。

**② OSPF 是 iBGP 能跑起来的前提。** iBGP 邻居用的是对端的 Loopback 地址（如 R1 去找 3.3.3.3）。要让到 3.3.3.3 的 TCP 会话建得起来，R1 必须先有一条到 3.3.3.3 的路由——这正是 OSPF 提供的。所以配置里特意把每台路由器的 Loopback 宣告进了 OSPF。`update-source Loopback0` 保证 BGP 报文用 Loopback 作源地址。**如果 OSPF 没配好，iBGP 会一直停在 Active/Idle 连不上。**

**③ `next-hop-self` 解决跨 AS 下一跳不可达。** R1 从 R4 学到 172.16.4.0/24 时下一跳是 eBGP 链路地址 192.0.2.2。R1 通过 iBGP 传给 R2、R3 时默认不改下一跳，可它们路由表里没有 192.0.2.0/30，这条路由就废了。`next-hop-self` 让 R1 把下一跳改成自己的 1.1.1.1——而 1.1.1.1 是 OSPF 宣告过、人人可达的，路由就生效了。

**④ 中转路径上每台路由器都得"认识"这条路由，否则黑洞。** 一个从 AS 65002 去 AS 65003 的包走 R4 → R1 → R2 → R3 → R5。R2 夹在中间，没有任何 eBGP 会话，但它**必须**通过 iBGP 学到 172.16.5.0/24，否则包到了 R2 就被丢弃，形成黑洞。这就是为什么内部路由器 R2 也要加入 iBGP 全互联。（大型网络里为免去这个负担，会改用 MPLS 让中间设备无需感知 BGP 路由。）

**⑤ 千万不要把 BGP 全表重分发进 OSPF。** 互联网有约一百万条路由，灌进 OSPF 会直接把 IGP 压垮。正确做法就是本例这样：BGP 归 BGP、OSPF 归 OSPF，靠递归查找衔接。

## 三、核心机制：递归查找

以 R2 转发一个去往 172.16.5.0/24 的包为例，它在内部经历了两层查表：

```
① BGP 表：按目的前缀查路由
   172.16.5.0/24 → 下一跳 3.3.3.3（经 iBGP 学到）
            │
            │  下一跳 3.3.3.3 非直连 → 递归查 IGP
            ▼
② OSPF 表：把下一跳解析成物理出口
   3.3.3.3/32 → 出接口 Gi0/1，下一跳 10.0.23.3
            │
            │  拿到真实出接口与下一跳
            ▼
③ 转发数据包
   从 Gi0/1 送出，沿 R2 → R3 → R5 到达目的
```

BGP 只告诉你"去这个前缀要把包交给 3.3.3.3"，但 3.3.3.3 是个 Loopback、不是直连接口；于是路由器拿这个下一跳再去 OSPF 那张表里查一次，得到真正的物理出口。**BGP 提供"前缀 → 下一跳"，IGP 提供"下一跳 → 出接口"**，两层拼起来才能转发，这就是 IGP 和 BGP 协作的本质。

## 四、验证排错思路

按依赖顺序自下而上检查，因为上层 BGP 依赖下层 OSPF：

```
show ip ospf neighbor          # 第一步：确认 OSPF 邻居 FULL，Loopback 互相可达
show ip route 3.3.3.3          # 确认能通过 OSPF 学到对端 Loopback（iBGP 的前提）
show ip bgp summary            # 第二步：确认 iBGP / eBGP 邻居都到 Established
show ip bgp 172.16.5.0         # 看这条前缀的下一跳是不是 3.3.3.3（验证 next-hop-self）
show ip route 172.16.5.0       # 看它最终是否进了路由表、递归到了正确出口
```

在 R4 上对 172.16.5.1 做一次 `traceroute`，路径应当是 R4 → R1 → R2 → R3 → R5，能完整跑通就说明 IGP 与 BGP 配合无误。

**排错口诀**：iBGP 连不上先查 OSPF 和 Loopback 可达性；BGP 路由学到了却不可用，多半是 `next-hop-self` 漏配导致下一跳不可达。
