# Creating a 4G WAN Backup with Automated Failover

## Why?

With the recent COVID-19 lockdown, working from home has become more important than ever. With two adults working at home, and a toddler with a YouTube habit, I thought it would be best to introduce a backup WAN link sooner rather than later.

I had also inherited a 3 HomeFi 4G router and contract, which it would seem a shame to not use, although the signal wasn't the best in my house.

I have a static public facing IPv4 address, using NAT with private address space on my local network. I also have a static IPv6 /64 block of addresses routed to my internal network. As I have these I was keen to be able to keep them during failover. This allows me to keep using firewall rules that are in place and also (maybe...) keep TCP connections established during failover.

## Requirements recap
* Backup internet connection
* Automated failover
* With (some) alerting
* Maintaining public IPv4 address, so that outgoing NAT requests come from the same source
* Maintaining incoming routes for our IPv6 addresses

## Original Network

![Original Network](images/InitialState.png)

I have a VDSL connection using my ISP supplied Zyxel VMG-3925-B10C router, with WIFI disabled and using separate wireless access points. The router established a PPPoE connection to my ISP and and was then responsible for NAT/routing/firewall. It also ran local network services (DHCP, ND/RA, DNS forwarder)

## Plan

My ISP rather brilliantly provide an L2TP service to allow you to continue to use your allocated address space in case of issues with their VDSL link. They also make it easy for you to use the Zyxel router in bridge mode so that you can assign the WAN addresses to another router.

As such, all I needed was another router that could use the links from the Zyxel and the HomeFi and failover automatically between them.

## Hardware choice

I had come up with requirements for a router:
* L2TP client
* Configurable
* Performant (enough)
* Low power consumption
* Cheap(ish)

I already knew of Ubiquiti and looked into their routers, but they didn't seem to be quite as configurable as I wanted and didn't support an out of the box L2TP client. I started looking into small PCs, but they started to get expensive, or didn't have enough ethernet ports, or had high power consumption. My ISP are heavily involved with [Firebrick](https://www.firebrick.co.uk/), which look great, but too expensive.

In the end my research led me to Mikrotik, a Latvian manufacturer of a very broad range of networking kit. 

They make an x86 image of their RouterOS operating system available, so I tested this out in a VM first to convince myself that it could do everything I wanted it to. It certainly seemed to! RouterOS is super configurable and quite a different experience from the typical consumer router.

I took the plunge and bought a [hEX](https://mikrotik.com/product/RB750Gr3) 5 port router for about £60. It performs well and does hardware switching. I slightly regretted not buying an hEX S because it has a POE port and an SFP port (that you could put an RJ45 module into for an extra port), but it's not made a big difference.

## 4G Reception

My house is in a slight 4G black spot, and I didn't have any choice of network as I was tied into a 3 contract. Near the router I was getting speeds of roughly 1 Mbps in each direction. Luckily in my loft I was getting much better speeds (up to about 20Mbps each way, but typically more like 15Mbps.) I didn't particlarly want to drill more holes (having recently done this to run ethernet down the garden), but had a spare pair of Powerline adapters which worked well enough between the two routers.

## Initial Router Configuration

There's quite a bit to do in getting the MikroTik set up providing normal internet duties, so I've pulled that out into a separate page:

[Initial Config](initial-config.md)

## 4G Failover Configuration

RouterOS provides the facility to execute a script on a PPP connection going either up or down. I decided that for simplicity I would use this to establish or stop the L2TP tunnel. This won't cover all failure scenarios - e.g. failures within my ISP's network, however I believe the most likely failure scenario to be a problem with my VDSL connection, which would cause the PPPoE link to drop.

### Set up an ethernet port to connect to my 4G router

As I mentioed in the introduction, I put the 4G router in the loft so the two routers are actually connected to Powerline adapters, which are paired together and only used for this purpose. Originally I did try using the 4G router in bridge mode, and then using the DHCP client on the MikroTik, so that it was assigned a public IP, but this was unreliable, which I belive to be due to an issue between the RouterOS DHCP client and the Huawei DHCP server. Since changing it back to using NAT, and using a static IP address, these issues have disappeared.

```
/interface ethernet set [ find default-name=ether2 ] name=backup-wan
/ip address add address=192.168.8.50/24 interface=backup-wan network=192.168.8.0
```

### Create a static route so packets to the L2TP gateway are routed over the 4G link
```
/ip route
add comment="Route l2tp over backup-wan" distance=1 dst-address=90.155.53.19/32 gateway=backup-wan
```

### Create the L2TP client

We also need to add IP routes that are set at a higher priority (distance) than the PPPoE routes, so that if both connections are up, the backup is used in preference of the main link.
```
/ppp profile
add change-tcp-mss=yes name=aaisp-l2tp only-one=yes use-mpls=no use-ipv6=yes

/interface l2tp-client
add allow=chap connect-to=90.155.53.19 disabled=yes name=aaisp-l2tp profile=aaisp-l2tp password=******* user=***********

/ip route
add distance=1 gateway=aaisp-l2tp

/ipv6 route
add distance=1 gateway=aaisp-l2tp
```

### Add scripts to enable/disable the L2TP link

and add them to the main PPPoE profile so that if the PPPoE link comes up, bring down the L2TP and vice-versa.
```
/system script
add name=aaisp-l2tp-enable source=\
    "if ([/interface get aaisp-l2tp disabled] = yes) do={ /interface enable aaisp-l2tp }"
add name=aaisp-l2tp-disable source=\
    "/interface l2tp-client disable [find name=aaisp-l2tp]"

/ppp profile set [find name=aaisp] on-up=aaisp-l2tp-disable on-down=aaisp-l2tp-enable
```
