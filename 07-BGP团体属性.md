# BGP 团体属性（Community）

## 一、团体属性是什么

[03-BGP.md](03-BGP.md) 里列过 BGP 的路径属性，**团体（Community）**是其中做策略最灵活的一个。它本质上是给一条路由打的**标签**——一个 32 位的数值，习惯写成 `AA:NN` 两段（前段通常用 AS 号，后段自定义，如 `65001:100`）。

它的价值在于"成组操作"：给一批路由打上同一个团体值，下游路由器不用去逐条匹配前缀，只要匹配这个团体值，就能对一整组路由统一应用策略（改本地优先级、不再外发、过滤等）。这在大型网络里极大简化了策略管理。

团体属性是**可选、可传递**的——默认能跨 AS 传递，但思科设备**默认不发送团体属性**，必须显式开启 `send-community`。

## 二、知名团体（Well-Known Communities）

有几个预定义的团体值，所有厂商通用：

| 团体 | 含义 |
|---|---|
| `NO_EXPORT` | 收到的路由不再通告给其他 AS（可在 AS 内或联盟内传递） |
| `NO_ADVERTISE` | 收到的路由不再通告给**任何**邻居 |
| `LOCAL_AS`（NO_EXPORT_SUBCONFED） | 不通告出本子 AS（用于联盟场景） |
| `INTERNET` | 通告给所有人（无限制） |

## 三、配置实例

### 例 1：给外发路由打团体标签

在 [03-BGP.md](03-BGP.md) 的 AS 65001（R1）上，把通告出去的 192.168.1.0/24 打上团体 `65001:100`，并记得开启 `send-community`：

```
router bgp 65001
 neighbor 10.1.12.2 send-community             ! 关键：默认不发，必须开
 neighbor 10.1.12.2 route-map SET-COMM out
!
ip prefix-list MYNET permit 192.168.1.0/24
!
route-map SET-COMM permit 10
 match ip address prefix-list MYNET
 set community 65001:100
route-map SET-COMM permit 20                   ! 放行其余路由，否则会被隐式拒绝
```

### 例 2：根据收到的团体值应用策略

下游路由器收到带 `65001:100` 标签的路由后，把它的本地优先级抬高到 200（优先选用）：

```
ip community-list standard CUST permit 65001:100
!
route-map FROM-PEER permit 10
 match community CUST
 set local-preference 200
route-map FROM-PEER permit 20                   ! 放行其余路由
!
router bgp 65002
 neighbor 10.1.23.2 route-map FROM-PEER in
```

### 例 3：用知名团体 NO_EXPORT 限制路由扩散

希望某条路由只在直连邻居 AS 内部使用、不被它再传给别人，就打上 `no-export`：

```
route-map TAG-NOEXPORT permit 10
 set community no-export
!
router bgp 65001
 neighbor 10.1.12.2 send-community
 neighbor 10.1.12.2 route-map TAG-NOEXPORT out
```

### 追加 vs 覆盖

`set community` 默认会**覆盖**已有的团体值。如果想在保留原有团体的基础上再加一个，用 `additive`：

```
route-map ADD-COMM permit 10
 set community 65001:200 additive
```

## 四、扩展团体与大团体（了解）

- **扩展团体（Extended Community）**：64 位，结构化，主要用于 MPLS L3VPN（如 Route-Target 控制 VPN 路由的导入导出）。
- **大团体（Large Community，RFC 8092）**：96 位、分三段 `AA:BB:CC`，为 32 位 ASN 时代设计——传统 `AA:NN` 的 AS 段放不下 32 位 ASN，大团体解决了这个问题。

## 五、验证命令

```
show ip bgp 192.168.1.0                 # 看某前缀携带的团体值
show ip bgp community 65001:100         # 按团体值筛选 BGP 表
show ip bgp community no-export         # 看带知名团体的路由
show route-map                          # 检查 route-map 逻辑
```

排错要点：策略没生效，**第一个要查的就是 `send-community` 是否漏配**（思科默认不发团体）；其次检查 route-map 末尾是否漏了"放行其余"的条目，导致未匹配路由被隐式丢弃。

---

[← 上一篇：BGP 路由反射器](06-路由反射器.md) · [返回目录](README.md)
