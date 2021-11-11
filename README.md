# Azure Virtual WAN BGP Peering (_preview_)- Anycast multi-region load balancing

- [Azure Virtual WAN BGP Peering (preview)- Anycast multi-region load balancing](#azure-virtual-wan-bgp-peering-preview--anycast-multi-region-load-balancing)
    - [Introduction](#introduction)
    - [Overview](#overview)
        - [Configuration](#configuration)
        - [North Europe - primary region](#north-europe---primary-region)
            - [Pre-req (not covered)](#pre-req-not-covered)
            - [Network Virtual Appliance](#network-virtual-appliance)
            - [Verify](#verify)
        - [West Europe - secondary region](#west-europe---secondary-region)
            - [Pre-req (not covered)](#pre-req-not-covered)
            - [Network Virtual Appliance](#network-virtual-appliance)
            - [Verify](#verify)
    - [Resilience demonstration](#resilience-demonstration)
        - [Baseline](#baseline)
        - [Primary region failure](#primary-region-failure)
        - [Primary region recovery](#primary-region-recovery)
    - [Caveat Empor](#caveat-empor)
    - [Future enhancement / work](#future-enhancement--work)

## Introduction

![](images/2021-11-11-15-14-33.png)

> :bulb: Looking for an Anycast design based on traditional Hub/Spoke Virtual Networks (and **Azure Route Server**) instead? Click [here](https://github.com/adstuart/azure-routeserver-anycast).

The subject of load balancing across Azure regions is a common topic within application design in the Cloud. Often in the context of providing HA or DR for an application hosted in Azure, but also for purposes of load sharing and/or blue-green deployments.

If the source, remote endpoints (clients, APIs, etc), are accessing the applications from a public source (public IP, over the Internet) then we have well known patterns that many customers already use, comprised of products like Azure Front Door and Azure Traffic Manager – these products sit at the edge of the Microsoft network, and can therefore naturally intercept requests from clients on the Internet, before the traffic is passed to an Azure Region ([example](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/app-service-web-app/multi-region)).

However, exploring the ability to provide a network load balancing strategy across regions for endpoints accessing an application from a **private network**, is also something that gets raised from time-to-time.  I summarise the available solutions options in this wider document [here](https://github.com/adstuart/azure-crossregion-private-lb)

**This article is focused on a new pattern that is possible using [_BGP peering with Virtual WAN hub_ feature](https://docs.microsoft.com/en-us/azure/virtual-wan/create-bgp-peering-hub-portal) (currently public preview) and BGP integration with reverse proxy NVAs. In the same way that Azure Front door uses Anycast for resilience over the public Internet, we are able to utilise _BGP peering with Azure Virtual WAN hub_ to build our own custom Anycast solution with reachability across our private network**

*This approach offers advantages over traditional DNS based GSLB in respect to performance (no DNS TTL), complexity (no need to make NVA authoritative for DNS ) and scalability (add additional Azure Regions and utilise standard BGP route manipulation to steer traffic to nearest origin)*

## Overview

### Configuration

![](images/2021-11-11-15-20-20.png)

### North Europe - primary region

#### Pre-req (not covered)
- Build Linux VM *2 within Spoke Virtual Network
- Deploy apache and example website on backend VM
- Enable IP forwarding on NVA NIC
- Deploy Virtual WAN infrastructure:
  - Deploy Virtual WAN Hub (standard, branch-to-branch enabled)
  - Deploy ExpressRoute Gateway in Hub and connect to ExpressRoute Circuit
  - [Sign-up](https://docs.microsoft.com/en-us/azure/virtual-wan/create-bgp-peering-hub-portal#prerequisites) to BGP Endpoint/peering feature
  - Within VWAN Hub define [BGP Peering](https://docs.microsoft.com/en-us/azure/virtual-wan/create-bgp-peering-hub-portal#configure-a-bgp-peer) settings define NVA peer

#### Network Virtual Appliance 
To demonstrate the functionality we will keep the config as simple and lightweight as possible;
- [HAproxy](http://www.haproxy.org/) as reverse proxy function
- [ExaBGP](https://github.com/Exa-Networks/exabgp) as BGP speaker

```
# SSH to NVA
sudo apt update
sudo apt install exabgp
sudo apt install haproxy
# Loopback IF
sudo ifconfig lo:9 9.9.9.9 netmask 255.255.255.255 up
# ExaBGP config
vi conf.ini
neighbor 10.16.96.4 {
        router-id 172.16.156.70;
        local-address 172.16.156.70;
        local-as 65010;
        peer-as 65515;
        static {
        route 9.9.9.9/32 next-hop 172.16.156.70;

}
}

neighbor 10.16.96.5 {
        router-id 172.16.156.70;
        local-address 172.16.156.70;
        local-as 65010;
        peer-as 65515;
        static {
        route 9.9.9.9/32 next-hop 172.16.156.70;
}
}
## HAProxy config
 vi /etc/haproxy/haproxy.cfg
frontend http_front
   bind *:80
   stats uri /haproxy?stats
   default_backend http_back
backend http_back
   balance roundrobin
   server backend01 172.16.156.69:80 check
   
   sudo systemctl restart haproxy

## Start ExaBGP
exabgp ./conf.ini
```

#### Verify

Confirm 9.9.9.9/32 is learnt by VWAN Hub, note BGP peer 172.16.156.70 with expected AS-path (no prepend)

![](images/2021-11-11-15-23-43.png)

### West Europe - secondary region

#### Pre-req (not covered)
- Build Linux VM *2 within Spoke Virtual Network
- Deploy apache and example website on backend VM
- Enable IP forwarding on NVA NIC
- Deploy Virtual WAN infrastructure:
  - Deploy Virtual WAN Hub (standard, branch-to-branch enabled)
  - Deploy ExpressRoute Gateway in Hub and connect to ExpressRoute Circuit
  - [Sign-up](https://docs.microsoft.com/en-us/azure/virtual-wan/create-bgp-peering-hub-portal#prerequisites) to BGP Endpoint/peering feature
  - Within VWAN Hub define [BGP Peering](https://docs.microsoft.com/en-us/azure/virtual-wan/create-bgp-peering-hub-portal#configure-a-bgp-peer) settings define NVA peer

#### Network Virtual Appliance 
To demonstrate the functionality we will keep the config as simple and lightweight as possible;
- [HAproxy](http://www.haproxy.org/) as reverse proxy function
- [ExaBGP](https://github.com/Exa-Networks/exabgp) as BGP speaker _(note longer as-path length [prepend] within static route config, within this secondary region)_

```
# SSH to NVA
sudo apt update
sudo apt install exabgp
sudo apt install haproxy
# Loopback IF
sudo ifconfig lo:9 9.9.9.9 netmask 255.255.255.255 up
# ExaBGP config
neighbor 10.15.96.4 {
        router-id 172.16.136.70;
        local-address 172.16.136.70;
        local-as 65010;
        peer-as 65515;
        static {
        route 9.9.9.9/32 next-hop 172.16.136.70 as-path [ 65010 65010 65010 ];
        }
}

neighbor 10.15.96.5 {
        router-id 172.16.136.70;
        local-address 172.16.136.70;
        local-as 65010;
        peer-as 65515;
        static {
        route 9.9.9.9/32 next-hop 172.16.136.70 as-path [ 65010 65010 65010 ];
        }
}
## HAProxy config
 vi /etc/haproxy/haproxy.cfg
frontend http_front
   bind *:80
   stats uri /haproxy?stats
   default_backend http_back
backend http_back
   balance roundrobin
   server backend01 172.16.136.69:80 check
   
   sudo systemctl restart haproxy

 ## Start ExaBGP
exabgp ./conf.ini
```

#### Verify

Confirm 9.9.9.9/32 is learnt by VWAN Hub, note BGP peer 172.16.136.70 with expected AS-path prepend.

![](images/2021-11-11-14-03-29.png)

## Resilience demonstration

### Baseline

Check ExpressRoute Circuit (MSEE) receiving Anycast IP from both regions.

```
adam@Azure:~$ az network express-route list-route-tables -g gbb-er-lab-ne -n Intercloud-London --path primary --peering-name AzurePrivatePeering --query value -o table | grep 9.9.9.9

9.9.9.9/32         10.16.0.15                  0         65515 65010
9.9.9.9/32         10.16.0.14*                 0         65515 65010
9.9.9.9/32         10.15.0.12                  0         65515 65010 65010 65010
9.9.9.9/32         10.15.0.13                  0         65515 65010 65010 65010
```

From On-Premises client verify reachability. 

```
PS C:\Users\Administrator> Invoke-RestMethod  9.9.9.9
<html>
<head>
  <title> Example WWW </title>
</head>
<body>
  <p> I'm running this website in North Europe
</body>
</html>
```

Http://9.9.9.9 is served from North Europe as expected from our configuration. as as-path is shorter to this region, and therefore the preferred route chosen by the MSEE.

### Primary region failure

Simulate failure by shutting down ExaBGP on NVA in primary region. NVA stops advertising routes to VWAN Hub ER-GW, which retracts from MSEE. Leaving behind only the West Europe originated routes with the longer as-path length.

```
adam@Azure:~$ az network express-route list-route-tables -g gbb-er-lab-ne -n Intercloud-London --path primary --peering-name AzurePrivatePeering --query value -o table | grep 9.9.9.9

9.9.9.9/32         10.15.0.12                  0         65515 65010 65010 65010
9.9.9.9/32         10.15.0.13                  0         65515 65010 65010 65010
```

We are now in a failed-over state and Http://9.9.9.9 is served from West Europe as expected.

```
PS C:\Users\Administrator> Invoke-RestMethod  9.9.9.9
<html>
<head>
  <title>Example WWW</title>
</head>
<body>
  <p> I'm running this website in West Europe!
</body>
</html>
```

No pings lost, almost instant failover.

![](images/2021-11-11-15-28-58.png)

### Primary region recovery

Simulate recovery of primary region by restarting ExaBGP on NVA in primary region. NVA starts advertising routes to VWAN Hub ER-GW, which advertises to MSEE, making North Europe the preferred region again due to shorter as-path.

```
adam@Azure:~$ az network express-route list-route-tables -g gbb-er-lab-ne -n Intercloud-London --path primary --peering-name AzurePrivatePeering --query value -o table | grep 9.9.9.9

9.9.9.9/32         10.16.0.14                  0         65515 65010
9.9.9.9/32         10.16.0.15*                 0         65515 65010
9.9.9.9/32         10.15.0.12                  0         65515 65010 65010 65010
9.9.9.9/32         10.15.0.13                  0         65515 65010 65010 65010
```

Http://9.9.9.9 is again being served from primary region.

```
PS C:\Users\Administrator> Invoke-RestMethod  9.9.9.9
<html>
<head>
  <title> Example WWW </title>
</head>
<body>
  <p> I'm running this website in North Europe
</body>
</html>
```

Total fail-back time <1s.

![](images/2021-11-11-15-30-12.png)

## Caveat Empor

The above is proof of value/function only, a production grade implementation would add additional resilience, security and automation, both on the frontend network infra, but also system state sync on the backend.

## Future enhancement / work

The concept of using VWAN Hub BGP Peering (and/or [Azure Route Server](https://github.com/adstuart/azure-routeserver-anycast)) to originate Anycast addresses within Azure is a powerful new tool in the Azure Networking toolkit, one which in the future will hopefully be leveraged by 1st-party and 3rd-party offerings alike. Examples could include;

- Optimized open sourced based designs (iterating on the design presented in the document) to add a level of automation. For example linking state of ExaBGP announcement to reachability of backend server within HAProxy. (Similar: https://blog.plessis.info/blog/2020/02/11/haproxy-exabgp.html)
- Enterprise NVA vendors of reverse proxy solutions should look to offer reference architecture that integrate with Azure Route Server for Anycast based designs
- Anycast based DNS designs to simplify hybrid Enterprise DNS designs


