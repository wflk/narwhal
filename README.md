narwhal - A docker network configuration tool
=============================================

## What does `narwhal` do?

`narwhal` is used with the `--net=none` option to `docker run` and creates a
virtual ethernet device pair (with one end in the container and the other end
on the host system). It assigns static IPv4 and IPv6 addresses with the
minimum neccessary routing setup.

## What is this good for?

Docker currently offers 4 different networking modes: `bridge`, `host`, 
`container` and `none`.

Most users use `bridge` which is the default. In bridge mode Docker creates
a seperate networking namespace per container and a virtual ethernet device pair.
This is fairly secure as a container is not capable to interfere with other
containers' or even the host systems' networking stack.

In addition to that the Docker daemon creates a virtual bridge device on
startup which is (more or less) a layer 2 (ethernet) switch. It adds the host
side of your containers' virtual ethernet pair to that bridge. This allows
all containers to talk to each other via ethernet. Read 
[here](https://nyantec.com/en/2015/03/20/docker-networking-considered-harmful/)
why this is extremely dangerous. It completely undermines the carefully crafted
container isolation based on networking namespaces.

The solution proposed by `narwhal` is to just have a virtual ethernet device
and networking namespace per container and to just only IPv4 and IPv6 (layer 3)
routing between containers and the outside world.

It is common knownledge to most operators how to work with 
[iptables](http://www.netfilter.org/projects/iptables/) and 
[ip6tables](http://ipset.netfilter.org/ip6tables.man.html), but how many have
ever heard of [ebtables](http://ebtables.netfilter.org/)? ;-)

__tl;dr:_Docker's default networking mode is vulnerable to ARP spoofing attacks.
A single malicious container corrupts all running containers.__

## How do I use it?

```
narwhal container [--ipv4 addr] [--ipv4-host addr] [--ipv6 addr] [--ipv6-host addr] [--interface name] [--host-interface name]
```

__container__ 
:    The ID or name of a running Docker container. See `docker ps`.

__--ipv4 addr__
:    The IPv4 address assigned to the container.

__--ipv4-host addr__
:    The IPv4 address that the host will be known as to the container. It'll
also be configured as the containers default gateway. Defaults to `169.254.0.1`

__--ipv6 addr__
:    The IPv6 address assigned to the container.

__--ipv6-host addr__
:    The IPv6 address that the host will be known as to the container. It'll
also be configured as the containers' default gateway. Defaults to `fe80::1`

__--interface name__
:    The name of the created interface inside the container. Defaults to `eth0`.

__--host-interface name__
:    The name of the created interface on the host. Defaults to `nw-$CONTAINERID`.


### Example

You need to configure your container with `--net=none`:

```bash
root@host:/# docker run --rm -t -i --net=none ubuntu /bin/bash
root@bb9b0be2a4d3:/# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
```

Now that your container is running configure networking. We assume we own the public subnets
`5.9.235.144/28` and `2a01:4f8:161:310e:4::/80`:

```bash
root@host:/# ip address
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 7c:05:07:0e:01:ea brd ff:ff:ff:ff:ff:ff
    inet 5.9.42.20/27 brd 5.9.42.31 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2a01:4f8:161:310e::1/80 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::7e05:7ff:fe0e:1ea/64 scope link 
       valid_lft forever preferred_lft forever

root@host:/# narwhal bb9b0be2a4d3 --ipv4 5.9.235.146 --ipv4-host 5.9.42.20 --ipv6 2a01:4f8:161:310e:4::1 --ipv6-host 2a01:4f8:161:310e::1

root@host:/# ip address
...
74: nw-bb9b0be2a4d3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether ca:46:43:64:d0:ef brd ff:ff:ff:ff:ff:ff
    inet 5.9.42.20 peer 5.9.235.147/32 scope global nw-bb9b0be2a4d3
       valid_lft forever preferred_lft forever
    inet6 2a01:4f8:161:310e::1 peer 2a01:4f8:161:310e:4::1/128 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::c846:43ff:fe64:d0ef/64 scope link 
       valid_lft forever preferred_lft forever

root@host:/# ip route
default via 5.9.42.1 dev eth0 
5.9.235.147 dev nw-bb9b0be2a4d3  proto kernel  scope link  src 5.9.42.20 

root@host:/# ip -6 route
2a01:4f8:161:310e::1 dev nw-bb9b0be2a4d3  proto kernel  metric 256 
2a01:4f8:161:310e::/80 dev eth0  proto kernel  metric 256 
2a01:4f8:161:310e:4::1 dev nw-bb9b0be2a4d3  proto kernel  metric 256 
fe80::/64 dev eth0  proto kernel  metric 256 
fe80::/64 dev nw-bb9b0be2a4d3  proto kernel  metric 256 
default via fe80::1 dev eth0  metric 1024 
```

`narwhal` created the virtual ethernet pair which is named `nw-bb9b0be2a4d3` on the host side.

This is how it now looks from inside the container:

```bash
root@bb9b0be2a4d3:/# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
73: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 96:f2:bd:b4:64:ee brd ff:ff:ff:ff:ff:ff
    inet 5.9.235.147/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2a01:4f8:161:310e:4::1 peer 2a01:4f8:161:310e::1/128 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::94f2:bdff:feb4:64ee/64 scope link 
       valid_lft forever preferred_lft forever
root@bb9b0be2a4d3:/# ip route
default via 5.9.42.20 dev eth0 
5.9.42.20 dev eth0  scope link 
root@bb9b0be2a4d3:/# ip -6 route
2a01:4f8:161:310e::1 dev eth0  metric 1024 
2a01:4f8:161:310e:4::1 dev eth0  proto kernel  metric 256 
fe80::/64 dev eth0  proto kernel  metric 256 
default via 2a01:4f8:161:310e::1 dev eth0  metric 1024 
```

If `sysctl net/ipv4/conf/all/forwarding` and `sysctl net/ipv6/conf/all/forwarding` 
are enabled and your packet filter is not interfering your should now be able to reach the outside world:

```bash
root@bb9b0be2a4d3:/# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=9.16 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 9.169/9.169/9.169/0.000 ms
root@bb9b0be2a4d3:/# ping6 heise.de 
PING heise.de(redirector.heise.de) 56 data bytes
64 bytes from redirector.heise.de: icmp_seq=1 ttl=55 time=5.47 ms
^C
--- heise.de ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 5.472/5.472/5.472/0.000 m
```

## FAQ

### How do I undo what `narwhal` did?

Stop the container or run

```bash
ip link del nw-$CONTAINERID

```

All routes etc will vanish automatically.

### Does `narwhal` configure `iptables`?

No, but here are your options:

  - Assign public IPv4 or IPv6 addresses, enable IP forwarding and be happy.
    Perform filtering via the `FORWARDING` chain in the filter table as you like.
  - Assign private IPv4 or IPv6 addresses and just use them to connect your
    containers with the host or with each other.
    Additionally, configure source- and/or destination NAT manually.

### What happens if `narwhal` is (accidentially) applied more than once?

Nothing. It will find that an `eth0` device already exists and just fail
after rolling back alrady acquired resources.

### Can `narwhal` be used in combination with the other networking modes?

Absolutely! Your other containers may use other networking modes.
Containers you want to configure with `narwahl` should use `--net=none`.
Trying to apply `narwahl` to otherwise configured containers just fails
if an `eth0` device already exists, but it won't cause any harm.

### What happens on container termination?

The containers' networking namespace and the virtual ethernet pair are destroyed
automatically.

### What happens when I configure networking after my container started?

If the container is started with `--net=none` it only has a
[loop back device](https://en.wikipedia.org/wiki/Loop_device). When your
container has services listening on ports it should __not__ bind to
a specific address (except `127.0.0.1` or `::1`). 

Not binding to an address is usually achieved by supplying `0.0.0.0`, `*`, `:::`
as listen address. Consult your services' documentation for details.

After `narwhal` was run and created the `eth0` device your service will
automatically accept packets via the supplied addresses.

