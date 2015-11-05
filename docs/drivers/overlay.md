<!--[metadata]>
+++
title = "The Overlay driver"
description = "Understanding and using the overlay driver"
keywords = ["docker, network, plugin, driver, overlay"]
[menu.main]
parent = "smn_networking_drivers"
+++
<![end-metadata]-->

# The overlay driver

## Design

The overlay driver has been designed to allow the creation of 
[Overlay Networks](https://en.wikipedia.org/wiki/Overlay_network)
between Docker Engines

This allows containers attached to the same network to communicate
regardless of where they are physically created

### VXLAN

The initial implementation of the overlay driver uses 
[VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN).
It is possible to provide additional implementation in future, for example
[GENEVE](https://tools.ietf.org/html/draft-ietf-nvo3-geneve-00) or 
[NVGRE](https://en.wikipedia.org/wiki/Network_Virtualization_using_Generic_Routing_Encapsulation)

VXLAN has been available in the Linux Kernel since 3.7

In order to provide separation between networks, each network receives
a unique VXLAN Identifer

This limits the total number of networks to 2^16 or 65,536

### Linux Bridging and Network Namespaces

In order to provide isolation between networks, we create one network
namespace (`netns`) per network. 

Within this network we create a new Linux Bridge, `br0` and the 
necessary VXLAN ports to communicate with our peers. The list of peers 
is obtained from Docker discovery.

> Note: The above only happens when the first container is added to 
a network

For each container added to the network, we create an additional
network namespace. We then create a Virtual Ethernet Pair (`veth`),
one end residing in the networks `netns` and the other
inside the containers `netns`

To inspect this setup within a Docker Engine, you can perform the
following steps

```
$ docker network create -d overlay foo
f494e05e19896dfae9e990c17d38ca37eb752fccbcf5c1dd392fe965618081a8

$ docker run -d --net=foo nginx
2d44a6e4598bc81859a5de2bcd46012b726b883b6b049a548a4e7bfaf8982d7d

$ ls /var/run/docker/netns
1-f494e05e19  6323ed5eebb2

$ mkdir -p /var/run/netns

$ ln -s /var/run/docker/netns/1-f494e05e19 /var/run/netns/1-f494e05e19
$ ln -s /var/run/docker/netns/6323ed5eebb2  /var/run/netns/6323ed5eebb2

$ ip netns exec 1-f494e05e19 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 1a:a6:da:19:8b:58 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::7c96:cdff:fe83:bf4/64 scope link
       valid_lft forever preferred_lft forever
6: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN group default
    link/ether 1a:a6:da:19:8b:58 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::18a6:daff:fe19:8b58/64 scope link
       valid_lft forever preferred_lft forever
8: veth2@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
    link/ether 52:2d:39:ad:22:a3 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::502d:39ff:fead:22a3/64 scope link
       valid_lft forever preferred_lft forever

$ ip netns exec 6323ed5eebb2 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe00:2/64 scope link
       valid_lft forever preferred_lft forever
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:2/64 scope link
       valid_lft forever preferred_lft forever

```

You can take a closer look at the VXLAN interface configuration

```
$ ip netns exec 1-f494e05e19 ip -d link show vxlan1
6: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default
    link/ether 1a:a6:da:19:8b:58 brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 256 srcport 0 0 dstport 4789 proxy l2miss l3miss ageing 300
    bridge_slave
```

You can also discover any MAC Addresses that have been learned:

```
$ ip netns exec 1-f494e05e19 bridge fdb show dev vxlan1
1a:a6:da:19:8b:58 permanent
02:42:0a:00:00:03 dst 192.168.99.136 self permanent
```

## Using the overlay driver

### Pre-requisites

As this plugin is designed to permit networks that span multiple docker
hosts it is required that your Engine has been configured with the
`--cluster-store` and `--cluster-advertise` daemon flags
For more information, see [Docker discovery]()

### Usage

The following example assumes you have two hosts called `d1` and `d2`, 
that were created using Docker Machine. For an example of how to set up
such an environment, see the [Getting started with overlay]() guide.

Create a new network:

```
$ docker $(docker-machine config d1) network create -d overlay mynet
```

Create two containers, a web server on `d1` and a client on `d2`.
The client container will use `wget` to access the page hosted on
the server

```
$ docker $(docker-machine config d1) run -d --net=mynet --name web nginx
$ docker $(docker-machine config d2) run -it --rm --net=mynet busybox wget -qO- http://web
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

