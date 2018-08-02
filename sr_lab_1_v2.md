---
typora-copy-images-to: ./image-20180802084820672.png
typora-root-url: ../git
---

# 在单一路由协议环境下的segment-routing mapping server配置

## 网络拓扑

1. SP设备运行一个单一的ISIS进程
2. cr和pe模拟ran核心和汇聚设备，ldp
3. ar和pe模拟ran接入环，segment-routing
4. 华为给方案是单独在接入环跑ospf进程，我计划使用cisco实现华为的方案，先用保守的单进程igp试一下

![image-20180802084820672](/image-20180802084820672.png)

## SR/LDP边界PE配置

1. PE1同时也是mapping server ，ar1侧运行segment-routing，cr1侧运行ldp。
2. 在PE1上，从SR->LDP的标签转换是自动的，因为LDP会自动为RIB中的条目产生local label，同时通过IGP又会学习到下游通过segment-routing扩展学习到的sid，自动将两个label接续起来，传递给上游路由器。
3. 在PE1上，从LDP->SR的标签转换需要手工配置mapping server，因为LDP域的IGP中，源头宣告没有sid扩展，因此进入segment-routing域以后，也不会自动产生sid。需要手工配置为特定的宣告产生sid，然后通过IGP进行泛洪。
4. 注意配置中出现了group，这是为了简化配置，可以把预定的大量重复的配置内容，例如路由协议下的端口配置事先建立好，然后在路由协议下apply。注意这样的配置直接show running是看不到具体继承内容的，需要show running inheritance才可以看到。

```
`!! IOS XR Configuration 6.1.2`
`!! Last configuration change at Wed Aug  1 04:57:07 2018 by cisco`
`!`
`hostname pe1`
`group g-isis-gage`
 router isis '.*'
  interface 'GigabitEthernet.*'
   point-to-point
   address-family ipv4 unicast
   !
   address-family ipv6 unicast
   !
  !
 !
`end-group`
`ipv4 prefix-list isis-2-ospf`
 10 permit 10.0.0.1/32 eq 32
`!`
`interface Loopback0`
 ipv4 address 10.0.0.3 255.255.255.255
 ipv6 address 2048:8000::3/128
`!`
`interface MgmtEth0/0/CPU0/0`
 ipv4 address 10.255.36.3 255.255.255.0
 ipv6 address 2048:8000:36::3/64
`!`
`interface GigabitEthernet0/0/0/0`
 ipv4 address 10.255.13.3 255.255.255.0
 ipv6 address 2048:8000:13::3/64
`!`
`interface GigabitEthernet0/0/0/1`
 ipv4 address 10.255.23.3 255.255.255.0
 ipv6 address 2048:8000:23::3/64
`!`
`interface GigabitEthernet0/0/0/2`
 ipv4 address 10.255.35.3 255.255.255.0
 ipv6 address 2048:8000:35::3/64
`!`
`prefix-set isis-2-ospf`
  10.0.0.1/32,
  10.0.0.2/32,
  10.0.0.4/32
`end-set`
`!`
`route-policy isis-2-ospf`
  if destination in isis-2-ospf then
    pass
  endif
`end-policy`
`!`
`router isis ldp-domain`
 apply-group g-isis-gage
 is-type level-2-only
 net 49.0001.0100.0000.0003.00
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls sr-prefer
  segment-routing prefix-sid-map advertise-local
 !
 address-family ipv6 unicast
  metric-style wide
  segment-routing mpls
  segment-routing prefix-sid-map advertise-local
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 3
  !
  address-family ipv6 unicast
   prefix-sid index 1003
  !
 !
 interface MgmtEth0/0/CPU0/0
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
 !
 interface GigabitEthernet0/0/0/1
 !
 interface GigabitEthernet0/0/0/2
 !
`!`
`mpls oam`
`!`
`mpls ldp`
 router-id 10.0.0.3
 address-family ipv4
 !
 address-family ipv6
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4
  !
  address-family ipv6
  !
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4
  !
  address-family ipv6
  !
 !
`!`
`segment-routing`
 mapping-server
  prefix-sid-map
   address-family ipv4
    10.0.0.1/32 1
    10.0.0.2/32 2
    10.0.0.4/32 4
   !
  !
 !
`!`
``end
```

## SR边界AR配置

1. 注意区分IP PING/TRACEROUTE和MPLS PING/TRACEROUTE，IP的不能确保一定会走MPLS，因为只要有路由，哪怕中间LSP断了，只要能走下去，还是会走下去；MPLS的是严格查找FEC，按照MPLS LSP走，而且具备多路径发现能力。
2. 启用MPLS PING/TRACEROUTE需要在所有路由器上启用mpls oam
3. SR环境下启用MPLS PING/TRACEROUTE必须要增加fec generic参数，否则找不到FEC，很诡异的现象。



```
!! IOS XR Configuration 6.1.2
!! Last configuration change at Wed Aug  1 03:07:45 2018 by cisco
!
hostname ar1
interface Loopback0
 ipv4 address 10.0.0.5 255.255.255.255
 ipv6 address 2048:8000::5/128
!
interface MgmtEth0/0/CPU0/0
 shutdown
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.255.56.5 255.255.255.0
 ipv6 address 2048:8000:56::5/64
!
interface GigabitEthernet0/0/0/1
 shutdown
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.255.35.5 255.255.255.0
 ipv6 address 2048:8000:35::5/64
!
router isis ldp-domain
 is-type level-2-only
 net 49.0001.0100.0000.0005.00
 log adjacency changes
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
 !
 address-family ipv6 unicast
  metric-style wide
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 5
  !
  address-family ipv6 unicast
   prefix-sid index 1005
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
!
mpls oam
!
end
```

```
!! IOS XR Configuration 6.1.2
!! Last configuration change at Tue Jul 31 08:33:34 2018 by cisco
!
hostname cr1
interface Loopback0
 ipv4 address 10.0.0.1 255.255.255.255
 ipv6 address 2048:8000::1/128
!
interface MgmtEth0/0/CPU0/0
 shutdown
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.255.13.1 255.255.255.0
 ipv6 address 2048:8000:13::1/64
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.255.14.1 255.255.255.0
 ipv6 address 2048:8000:14::1/64
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.255.12.1 255.255.255.0
 ipv6 address 2048:8000:12::1/64
!
router isis ldp-domain
 is-type level-2-only
 net 49.0001.0100.0000.0001.00
 log adjacency changes
 address-family ipv4 unicast
  metric-style wide
 !
 address-family ipv6 unicast
  metric-style wide
 !
 interface Loopback0
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
!
mpls oam
!
mpls ldp
 router-id 10.0.0.1
 address-family ipv4
 !
 address-family ipv6
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4
  !
  address-family ipv6
  !
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4
  !
  address-family ipv6
  !
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv4
  !
  address-family ipv6
  !
 !
!
end
```

## 检查从LDP->SR域的标签分发情况，以PE2的loopback为例：

- 最下游的源路由器PE2上，loopback地址分配了local label并分发，注意本地路由器并不需要产生lib表象，fib表项为直接转发给接口，向上游路由器分发的标签是implict null (3)

```
RP/0/0/CPU0:pe2#sh mpls ldp bindings 10.0.0.4/32
Thu Aug  2 00:59:41.260 UTC
10.0.0.4/32, rev 16
        Local binding: label: ImpNull
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          24004   
            10.0.0.2:0          24005 
```

```
RP/0/0/CPU0:pe2#sh mpls ldp ipv6 bindings 2048:8000::4/128
Thu Aug  2 01:00:40.306 UTC
2048:8000::4/128, rev 16
        Local binding: label: ImpNull
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          24016   
            10.0.0.2:0          24016   
```

```
RP/0/0/CPU0:pe2#show mpls forwarding prefix 10.0.0.4/32
Thu Aug  2 01:01:31.852 UTC
```

```
RP/0/0/CPU0:pe2#show cef 10.0.0.4/32 detail 
Thu Aug  2 01:02:03.200 UTC
10.0.0.4/32, version 2, attached, receive
  Updated Jul 30 06:32:09.779
  Prefix Len 32
  internal 0x3006041 (ptr 0xa1455674) [3], 0x0 (0xa143b2d8), 0x0 (0x0)
```

- 在CR/P路由器上，同时具备MPLS和IP的转发能力，如果从上游发来的是MPLS报文，根据lib转发（因为swapped label为implict null ，所以实际上是pop label恢复IP报文）；如果从上游发来的是IP报文，将按照CEF表转发（因为push implict null label，所以也是IP报文转发）。注意CR是到PE2的倒数第二跳，这里是PHP过程。

```
RP/0/0/CPU0:cr1#sh mpls ldp bind 10.0.0.4/32
Thu Aug  2 01:10:30.412 UTC
10.0.0.4/32, rev 24
        Local binding: label: 24004
        Remote bindings: (3 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.2:0          24005   
            10.0.0.3:0          24007   
            10.0.0.4:0          ImpNull 
```

```
RP/0/0/CPU0:cr1#show mpls for labels 24004
Thu Aug  2 01:10:54.971 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

24004  Pop         10.0.0.4/32        Gi0/0/0/1    10.255.14.4     3428      
```

```
RP/0/0/CPU0:cr1#show route 10.0.0.4/32
Thu Aug  2 01:11:03.810 UTC

Routing entry for 10.0.0.4/32
  Known via "isis ldp-domain", distance 115, metric 20, type level-2
  Installed Jul 31 06:03:54.694 for 1d19h
  Routing Descriptor Blocks
    10.255.14.4, from 10.0.0.4, via GigabitEthernet0/0/0/1
      Route metric is 20
  No advertising protos. 
```

```
RP/0/0/CPU0:cr1#show cef 10.0.0.4/32       
Thu Aug  2 01:14:32.276 UTC
10.0.0.4/32, version 56, internal 0x1000001 0x0 (ptr 0xa1456174) [1], 0x0 (0xa143b5f0), 0xa20 (0xa159d348)
 Updated Jul 31 08:31:24.158 
 local adjacency 10.255.14.4
 Prefix Len 32, traffic index 0, precedence n/a, priority 3
   via 10.255.14.4/32, GigabitEthernet0/0/0/1, 7 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0xa10f32a4 0xa10f33a0]
    next hop 10.255.14.4/32
    local adjacency
     local label 24004      labels imposed {ImplNull}
```

- 在PE1路由器上，需要完成LDP->SR的标签转换，这个过程不是自动完成的，需要设置mapping server
- 首先，我们检查LDP绑定关系，确认PE1获得了PE2的LDP标签

```
RP/0/0/CPU0:pe1#show mpls ldp bind 10.0.0.4/32
Thu Aug  2 01:20:22.658 UTC
10.0.0.4/32, rev 28
        Local binding: label: 24007
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          24004   
            10.0.0.2:0          24005   
RP/0/0/CPU0:pe1#show mpls ldp ipv6 bind 2048:8000::4/128
Thu Aug  2 01:20:56.586 UTC
2048:8000::4/128, rev 26
        Local binding: label: 24015
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          24016   
            10.0.0.2:0          24016   
```

- 由于ldp域通过ldp传递标签信息，sr域通过igp传递标签信息，而pe2在ldp域内，所以pe2上isis进程的lsp里是不会发起sid的；即使pe2和pe1、ar1运行了同一个isis进程，pe1上也不会有pe2 loopback的任何sid。必须启用mapping server，手工为LDP域内的地址宣告指定sid，然后通过SR域的igp泛洪，才可以把LDP和SR域用标签接续起来。

```
RP/0/0/CPU0:pe1#sh segment-routing mapping-server  prefix-sid-map ipv4
Thu Aug  2 01:29:39.500 UTC
Prefix               SID Index    Range        Flags
10.0.0.1/32          1            1            
10.0.0.2/32          2            1            
10.0.0.4/32          4            1            

Number of mapping entries: 3
RP/0/0/CPU0:pe1#sh segment-routing mapping-server  prefix-sid-map ipv6
Thu Aug  2 01:29:41.970 UTC
Prefix                                        SID Index    Range        Flags
2048:8000::1/128                              1001         1            
2048:8000::2/128                              1002         1            
2048:8000::4/128                              1004         1            

Number of mapping entries: 3
```

- 检查mapping server agent（pe1同时是server和agent）将sid通过ISIS LSP宣告出去了

```
RP/0/0/CPU0:pe1#show isis database pe1.00-00 verbose  | utility egrep ' SID'
Thu Aug  2 01:41:53.850 UTC
  SID Binding:    10.0.0.1/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    10.0.0.2/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:2, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    10.0.0.4/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:4, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::1/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1001, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::2/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1002, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::4/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1004, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
```

- 检查pe1把SR标签和LDP标签接续起来了。这里需要注意，默认情况下是优先使用LDP为pe2 loopback分配的标签作为local标签的（例如写入CEF表，其实作为LDP最上游，pe1给下游路由器分配的local label是没啥用的，因为不会再有上游路由器用这个标签给pe1发mpls报文了）,但是配置了sr-prefer 的话，可以优先使用SR 标签作为local标签。

```
RP/0/0/CPU0:pe1#show mpls forwarding prefix 10.0.0.4/32
Thu Aug  2 01:49:54.227 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

16004  24004       SR Pfx (idx 4)     Gi0/0/0/0    10.255.13.1     1664        
       24005       SR Pfx (idx 4)     Gi0/0/0/1    10.255.23.2     248         
```

注意16004是segment-routing 根据SRGB+index计算出来的，如果网络规划得当，所有路由器使用同样的SRGB，分布计算得到的prefix-sid应该是一致的。

- 检查CEF表，注意其中三点：
  - 分配的local标签是sr的，因为设置了sr-prefer。
  - cef表项标志位，体现标签进行了接续（LDP/SR merge）,转发表的优先写入者是SR( RIB pref over LSD,SR Prefix)。
  - 到pe2有两条路径，cr1和cr2，所以cef有token bucket，用于负载均担。

```
RP/0/0/CPU0:pe1#show cef 10.0.0.4/32 flags | i flags
Thu Aug  2 02:14:02.338 UTC
 leaf flags: owner locked, inserted
 leaf flags2: LDP/SR merge requested,RIB pref over LSD,LDP/SR merge active,SR Prefix,
 leaf ext flags: PriChange,EXTERNAL_REACH_LC,illegal-0x00000200,illegal-0x00000800,
   via 10.255.13.1/32, GigabitEthernet0/0/0/0, 15 dependencies, weight 0, class 0 [flags 0x0]
   via 10.255.23.2/32, GigabitEthernet0/0/0/1, 15 dependencies, weight 0, class 0 [flags 0x0]
```

```
RP/0/0/CPU0:pe1#show cef 10.0.0.4/32 det
Thu Aug  2 02:12:04.076 UTC
10.0.0.4/32, version 65, internal 0x1000001 0x87 (ptr 0xa141a274) [1], 0x0 (0xa13e5d1c), 0xa28 (0xa182f368)
 Updated Jun  2 02:14:31.575 
 local adjacency 10.255.13.1
 Prefix Len 32, traffic index 0, precedence n/a, priority 15
  gateway array (0xa12ae80c) reference count 3, flags 0x68, source rib (7), 2 backups
                [2 type 5 flags 0x8401 (0xa159da64) ext 0x0 (0x0)]
  LW-LDI[type=5, refc=3, ptr=0xa13e5d1c, sh-ldi=0xa159da64]
  gateway array update type-time 1 Jun  2 02:14:31.575
 LDI Update time Aug  1 04:57:07.647
 LW-LDI-TS Aug  1 04:57:07.647
   via 10.255.13.1/32, GigabitEthernet0/0/0/0, 15 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0xa10f33a0 0x0]
    next hop 10.255.13.1/32
    local adjacency
     local label 16004      labels imposed {24004}
   via 10.255.23.2/32, GigabitEthernet0/0/0/1, 15 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 1 NHID 0x0 [0xa10f3448 0x0]
    next hop 10.255.23.2/32
    local adjacency
     local label 16004      labels imposed {24005}

Load distribution: 0 1 (refcount 2)

Hash  OK  Interface                 Address
0     Y   GigabitEthernet0/0/0/0    10.255.13.1    
1     Y   GigabitEthernet0/0/0/1    10.255.23.2    
```

使用了哈希的方式，在CEF表对两个出接口做均担，根据五元组或者七元组的计算结果来决定向转发方向。可以通过cef匹配结果来检查具体数据包的落点。

```
RP/0/0/CPU0:pe1#show cef exact 10.0.0.5 10.0.0.4 pro icmp in g0/0/0/2 de
Thu Aug  2 02:21:23.967 UTC
10.0.0.4/32, version 65, internal 0x1000001 0x87 (ptr 0xa141a274) [1], 0x0 (0xa13e5d1c), 0xa28 (0xa182f368)
 Updated Jun  2 02:14:31.574 
 local adjacency 10.255.13.1
 Prefix Len 32, traffic index 0, precedence n/a, priority 15
   via GigabitEthernet0/0/0/0
   via 10.255.13.1/32, GigabitEthernet0/0/0/0, 15 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0xa10f33a0 0x0]
    next hop 10.255.13.1/32
    local adjacency
     local label 16004      labels imposed {24004}
RP/0/0/CPU0:pe1#show cef exact 10.0.0.6 10.0.0.4 pro icmp in mgmtEth 0/0/CPU0/0 de
Thu Aug  2 02:23:33.719 UTC
10.0.0.4/32, version 65, internal 0x1000001 0x87 (ptr 0xa141a274) [1], 0x0 (0xa13e5d1c), 0xa28 (0xa182f368)
 Updated Jun  2 02:14:31.575 
 local adjacency 10.255.23.2
 Prefix Len 32, traffic index 0, precedence n/a, priority 15
   via GigabitEthernet0/0/0/1
   via 10.255.23.2/32, GigabitEthernet0/0/0/1, 15 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 1 NHID 0x0 [0xa10f3448 0x0]
    next hop 10.255.23.2/32
    local adjacency
     local label 16004      labels imposed {24005}
```

- 在ar1上，通过IGP LSP学习到了sid，并且根据sid计算出了local label和remote label，由于SRGB是一致的，所以lib表的出入标签是一个数值。

```
RP/0/0/CPU0:ar1#show isis da pe1.00-00 v | uti egrep ' SID'
Thu Aug  2 02:32:22.661 UTC
  SID Binding:    10.0.0.1/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    10.0.0.2/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:2, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    10.0.0.4/32 F:0 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:4, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::1/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1001, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::2/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1002, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
  SID Binding:    2048:8000::4/128 F:1 M:0 S:0 D:0 A:0 Weight:0 Range:1
    SID: Start:1004, Algorithm:0, R:0 N:0 P:0 E:0 V:0 L:0
RP/0/0/CPU0:ar1#show mpls for pre 10.0.0.4/32
Thu Aug  2 02:32:46.739 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

16004  16004       SR Pfx (idx 4)     Gi0/0/0/2    10.255.35.3     576    
```

## **注意：在这里试验中发现了一点问题**

sr往lib里写的标签，是没有严格的fec概念的，不体现为prefix而是体现为sid，如下：

```
RP/0/0/CPU0:ar1#sh mpls for
Thu Aug  2 02:56:52.740 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

16001  16001       SR Pfx (idx 1)     Gi0/0/0/2    10.255.35.3     192         
16002  16002       SR Pfx (idx 2)     Gi0/0/0/2    10.255.35.3     0           
16003  Pop         SR Pfx (idx 3)     Gi0/0/0/2    10.255.35.3     96          
16004  16004       SR Pfx (idx 4)     Gi0/0/0/2    10.255.35.3     56448       
17001  17001       SR Pfx (idx 1001)  Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
17002  17002       SR Pfx (idx 1002)  Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
17003  Pop         SR Pfx (idx 1003)  Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
17004  17004       SR Pfx (idx 1004)  Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
24000  Pop         SR Adj (idx 1)     Gi0/0/0/2    10.255.35.3     0           
24001  Pop         SR Adj (idx 3)     Gi0/0/0/2    10.255.35.3     0           
24002  Pop         SR Adj (idx 1)     Gi0/0/0/0    10.255.56.6     0           
24003  Pop         SR Adj (idx 3)     Gi0/0/0/0    10.255.56.6     0           
24004  Pop         SR Adj (idx 1)     Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
24005  Pop         SR Adj (idx 3)     Gi0/0/0/2    fe80::5200:ff:fe03:3   \
                                                                   0           
24006  Pop         SR Adj (idx 1)     Gi0/0/0/0    fe80::5200:ff:fe04:1   \
                                                                   0           
24007  Pop         SR Adj (idx 3)     Gi0/0/0/0    fe80::5200:ff:fe04:1   \
```


作为对比，我们看一下pe1上的lib，由于同时运行着SR和LDP，因此通过SR写入的标签体现为sid，通过LDP写入的标签体现为prefix。

1. 对于同一个地址10.0.0.5：通过SR传递来的sid是5（16005），通过LDP为RIB表分配的本地标签是24000，而且因为pe1不是mpls交换的尽头，因此出操作还是mpls pop（PHP implict null）
2. 对于同一个地址10.0.0.6:   因为此时我还没在ar2上启用sr，所以只有LDP分配的本地标签24001，而且pe1是MPLS交换的尽头，因此要把所有标签都去除掉，变成IP包转发，出操作是unlabelld
3. 对于IPv6地址 2408:8000::5:  只有一个LDP分配的local标签，没有SR分配的标签，且执行出操作unlabelled，这说明发生了标签接续（merge）但中止了MPLS交换。！！**<u>这个问题我没找到原因和解决办法，怀疑是cisco segment-routing mapping-server对IPv6的支持存在问题。</u>**

```
RP/0/0/CPU0:pe1#show mpls for
Thu Aug  2 03:19:58.697 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

16001  Pop         SR Pfx (idx 1)     Gi0/0/0/0    10.255.13.1     588         
16002  Pop         SR Pfx (idx 2)     Gi0/0/0/1    10.255.23.2     0           
16004  24004       SR Pfx (idx 4)     Gi0/0/0/0    10.255.13.1     1664        
       24005       SR Pfx (idx 4)     Gi0/0/0/1    10.255.23.2     248         
16005  Pop         SR Pfx (idx 5)     Gi0/0/0/2    10.255.35.5     104         
24000  Pop         10.0.0.5/32        Gi0/0/0/2    10.255.35.5     2576        
24001  Unlabelled  10.0.0.6/32        Mg0/0/CPU0/0 10.255.36.6     0           
24002  Unlabelled  10.255.56.0/24     Mg0/0/CPU0/0 10.255.36.6     0           
       Unlabelled  10.255.56.0/24     Gi0/0/0/2    10.255.35.5     0           
24003  Pop         10.0.0.1/32        Gi0/0/0/0    10.255.13.1     0           
24004  Pop         10.255.24.0/24     Gi0/0/0/1    10.255.23.2     1232        
24005  Pop         10.255.14.0/24     Gi0/0/0/0    10.255.13.1     0           
24006  Pop         10.255.12.0/24     Gi0/0/0/0    10.255.13.1     0           
       Pop         10.255.12.0/24     Gi0/0/0/1    10.255.23.2     0           
24007  24004       10.0.0.4/32        Gi0/0/0/0    10.255.13.1     0           
       24005       10.0.0.4/32        Gi0/0/0/1    10.255.23.2     0           
24008  24005       10.255.47.0/24     Gi0/0/0/0    10.255.13.1     0           
       24006       10.255.47.0/24     Gi0/0/0/1    10.255.23.2     0           
24009  Pop         10.0.0.2/32        Gi0/0/0/1    10.255.23.2     0           
24010  Pop         2048:8000::2/128   Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   398988      
24011  Pop         2048:8000::1/128   Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   398802      
24012  Pop         2048:8000:24::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24013  Pop         2048:8000:14::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24014  Pop         2048:8000:12::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       Pop         2048:8000:12::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24015  24016       2048:8000::4/128   Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       24016       2048:8000::4/128   Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24016  24017       2048:8000:47::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       24017       2048:8000:47::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24017  Unlabelled  2048:8000::5/128   Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24018  Unlabelled  2048:8000::6/128   Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24019  Unlabelled  2048:8000:56::/64  Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
       Unlabelled  2048:8000:56::/64  Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24024  Pop         SR Adj (idx 1)     Gi0/0/0/0    10.255.13.1     0           
24025  Pop         SR Adj (idx 3)     Gi0/0/0/0    10.255.13.1     0           
24026  Pop         SR Adj (idx 1)     Gi0/0/0/1    10.255.23.2     0           
24027  Pop         SR Adj (idx 3)     Gi0/0/0/1    10.255.23.2     0           
24028  Pop         SR Adj (idx 1)     Mg0/0/CPU0/0 10.255.36.6     0           
24029  Pop         SR Adj (idx 3)     Mg0/0/CPU0/0 10.255.36.6     0           
24030  Pop         SR Adj (idx 1)     Gi0/0/0/2    10.255.35.5     0           
24031  Pop         SR Adj (idx 3)     Gi0/0/0/2    10.255.35.5     0           
24032  Pop         SR Adj (idx 1)     Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24033  Pop         SR Adj (idx 3)     Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24034  Pop         SR Adj (idx 1)     Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24035  Pop         SR Adj (idx 3)     Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24036  Pop         SR Adj (idx 1)     Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24037  Pop         SR Adj (idx 3)     Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24038  Pop         SR Adj (idx 1)     Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24039  Pop         SR Adj (idx 3)     Gi0/0/0/2    fe80::5200:ff:fe07:3   \
```

在上面的lib中，我们发现ipv4 是sr-prefer，ipv6是ldp优先，如果全改为sr-prefer，会发现cisco IOS XR对v6和v4的lib/fib处理机制存在差异：

- 对于v4，可以同时出现SR和LDP的local label，并且sr label被自动交换为ldp label。
- 对于v6，只能出现SR或者LDP其中一个local label，并且可以merge，但是中间的MPLS交换莫名其妙的断掉了！断掉了！（不能自动为SR label和 LDP label之间建立交换关系)
- show mpls forwarding 后面只能直接查ipv4 fec（prefix参数），是不能直接显示IPv6的 fec的。如果是LDP写入LIB的，可以用show mpls ldp ipv6 forwarding来看；如果是SR写入的，可以直接通过IGP来查。不是很方便。 

```
RP/0/0/CPU0:pe1#sh mpls for
Thu Aug  2 05:17:38.903 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    

------

16001  Pop         SR Pfx (idx 1)     Gi0/0/0/0    10.255.13.1     0           
16002  Pop         SR Pfx (idx 2)     Gi0/0/0/1    10.255.23.2     0           
16004  24004       SR Pfx (idx 4)     Gi0/0/0/0    10.255.13.1     0           
       24005       SR Pfx (idx 4)     Gi0/0/0/1    10.255.23.2     0           
16005  Pop         SR Pfx (idx 5)     Gi0/0/0/2    10.255.35.5     0           
17001  Unlabelled  SR Pfx (idx 1001)  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   12834       
17002  Unlabelled  SR Pfx (idx 1002)  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   12774       
17004  Unlabelled  SR Pfx (idx 1004)  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       Unlabelled  SR Pfx (idx 1004)  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
17005  Pop         SR Pfx (idx 1005)  Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24000  Pop         10.0.0.5/32        Gi0/0/0/2    10.255.35.5     0           
24001  Unlabelled  10.0.0.6/32        Mg0/0/CPU0/0 10.255.36.6     0           
24002  Unlabelled  10.255.56.0/24     Mg0/0/CPU0/0 10.255.36.6     0           
       Unlabelled  10.255.56.0/24     Gi0/0/0/2    10.255.35.5     0           
24003  Pop         10.0.0.1/32        Gi0/0/0/0    10.255.13.1     0           
24004  Pop         10.255.24.0/24     Gi0/0/0/1    10.255.23.2     0           
24005  Pop         10.255.14.0/24     Gi0/0/0/0    10.255.13.1     0           
24006  Pop         10.255.12.0/24     Gi0/0/0/0    10.255.13.1     0           
       Pop         10.255.12.0/24     Gi0/0/0/1    10.255.23.2     0           
24007  24004       10.0.0.4/32        Gi0/0/0/0    10.255.13.1     0           
       24005       10.0.0.4/32        Gi0/0/0/1    10.255.23.2     0           
24008  24005       10.255.47.0/24     Gi0/0/0/0    10.255.13.1     0           
       24006       10.255.47.0/24     Gi0/0/0/1    10.255.23.2     0           
24009  Pop         10.0.0.2/32        Gi0/0/0/1    10.255.23.2     0           
24012  Pop         2048:8000:24::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24013  Pop         2048:8000:14::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24014  Pop         2048:8000:12::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       Pop         2048:8000:12::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24016  24017       2048:8000:47::/64  Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
       24017       2048:8000:47::/64  Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24018  Unlabelled  2048:8000::6/128   Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24019  Unlabelled  2048:8000:56::/64  Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
       Unlabelled  2048:8000:56::/64  Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24024  Pop         SR Adj (idx 1)     Gi0/0/0/0    10.255.13.1     0           
24025  Pop         SR Adj (idx 3)     Gi0/0/0/0    10.255.13.1     0           
24026  Pop         SR Adj (idx 1)     Gi0/0/0/1    10.255.23.2     0           
24027  Pop         SR Adj (idx 3)     Gi0/0/0/1    10.255.23.2     0           
24028  Pop         SR Adj (idx 1)     Mg0/0/CPU0/0 10.255.36.6     0           
24029  Pop         SR Adj (idx 3)     Mg0/0/CPU0/0 10.255.36.6     0           
24030  Pop         SR Adj (idx 1)     Gi0/0/0/2    10.255.35.5     0           
24031  Pop         SR Adj (idx 3)     Gi0/0/0/2    10.255.35.5     0           
24032  Pop         SR Adj (idx 1)     Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24033  Pop         SR Adj (idx 3)     Gi0/0/0/0    fe80::5200:ff:fe01:1   \
                                                                   0           
24034  Pop         SR Adj (idx 1)     Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24035  Pop         SR Adj (idx 3)     Gi0/0/0/1    fe80::5200:ff:fe02:2   \
                                                                   0           
24036  Pop         SR Adj (idx 1)     Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24037  Pop         SR Adj (idx 3)     Mg0/0/CPU0/0 fe80::5200:ff:fe04:2   \
                                                                   0           
24038  Pop         SR Adj (idx 1)     Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0           
24039  Pop         SR Adj (idx 3)     Gi0/0/0/2    fe80::5200:ff:fe07:3   \
                                                                   0        
```

 