# BGP 的 BFD 快速故障检测

## 一、要解决的问题：BGP 自身收敛太慢

BGP 默认靠 Keepalive/Hold 定时器感知邻居是否还活着：默认每 60 秒发一个 Keepalive，Hold 时间 180 秒。也就是说，如果链路中断但物理接口没 down（比如中间隔了交换机、或光路单通），BGP 最坏要等 **180 秒**才发现邻居挂了——这期间流量一直往黑洞里送。

把定时器调小（比如 hold 3 秒）能改善，但定时器是软件层周期任务，调到很低会消耗 CPU、在大量邻居时不可持续，而且秒级仍不够快。

## 二、BFD 是什么

**BFD（Bidirectional Forwarding Detection，双向转发检测）**是一个独立的、极轻量的"链路探活"协议。它在两端之间以**毫秒级**间隔互发探测包，一旦在约定时间内收不到，就判定路径故障，并**立即通知上层协议**（BGP、OSPF、IS-IS、静态路由都能用）去做收敛。

它的价值在于把"故障检测"从路由协议里解耦出来，交给一个专门为快而生、可以跑在转发硬件上的小协议。BGP 自己不用再纠结定时器，只要"订阅"BFD 的故障通知即可——检测速度从几十秒降到亚秒甚至几十毫秒。

![BGP + BFD 示意](images/11-bfd.svg)

## 三、配置实例（Cisco IOS）

两步：先在接口上启用 BFD 并设定探测参数，再让 BGP 邻居"挂靠"到 BFD。

```
! 1) 接口上启用 BFD，设定收发间隔与倍数
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3
!
! 2) 让 BGP 邻居使用 BFD 做故障联动
router bgp 65001
 neighbor 10.1.12.2 fall-over bfd
```

参数含义：本端每 300ms 发一个 BFD 包、期望对端最少每 300ms 发一个、连续丢 3 个（`multiplier 3`）即判定故障——检测时间约 `300ms × 3 = 900ms`。硬件支持的话间隔可压到 50ms，实现几十毫秒级收敛。`fall-over bfd` 是关键：它把这个 BGP 邻居和 BFD 绑定，BFD 一报故障，BGP 立刻拆会话、撤路由、重新选路。

**两端都要配。** BFD 是双向协议，对端也必须启用 BFD 并对该邻居配 `fall-over bfd`，会话才能建立。

## 四、用于多跳（Loopback 邻居）

iBGP 常基于 Loopback 建邻居，跨多跳，这时要用 **BFD 多跳**，并指明源/目的：

```
bfd map ipv4 2.2.2.2/32 1.1.1.1/32 MULTIHOP-TMPL
```

不同平台语法略有差异，核心是告诉 BFD"这对地址之间走多跳模板"，否则默认的单跳 BFD 起不来。

## 五、验证命令

```
show bfd neighbors             # 看 BFD 会话，状态应为 Up
show bfd neighbors details     # 看协商出的实际收发间隔、检测时间
show ip bgp neighbors 10.1.12.2 | include BFD   # 确认该 BGP 邻居已联动 BFD
```

排错要点：BFD 会话起不来，先确认**两端都配了** BFD 且参数兼容、接口确实转发 BFD 包；BFD Up 了但 BGP 不联动，多半是漏了 `neighbor ... fall-over bfd`。

## 六、何时用、注意什么

BFD 适合对收敛速度敏感的场景（金融、骨干、双归出口配合[选路策略](09-BGP选路实验.md)做快速切换）。但它也会带来持续的探测开销，间隔越小 CPU/硬件压力越大，需在"切换速度"和"资源消耗"之间权衡；并且要避免把间隔设得过激导致链路轻微抖动就误判故障、引发路由震荡。生产中常配合**路由抑制/收敛优化**一起用。

---

[← 上一篇：MPLS L3VPN 进阶](10-MPLS-HubSpoke-Extranet.md) · [返回目录](README.md) · [下一篇：prefix-list 与 route-map 实战 →](12-prefix-list与route-map实战.md)
