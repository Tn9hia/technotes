## What is Iptables?
Iptables are the interface of **netfilter**. Netfilter is a linux kernel module that decides what packets are allowed to come in or to go outside.

The packet filtering mechanism provided by iptables is organized into three different kinds of structures: **tables**, **chains** and **targets**.

When a packet arrives (or leaves, depending on the chain), iptables matches it against rules in these chains one-by-one. When it finds a match, it jumps onto the target and performs the action associated with it. If it doesn’t find a match with any of the rules, it simply does what the **default policy** of the chain tells it to. **The default policy** is also a target. By default, all chains have a default policy of allowing packets.

![[Screenshot 2024-09-23 at 22.39.35.png]]
### Tables

As we’ve mentioned previously, tables allow you to do very specific things with packets. On a modern Linux distributions, there are four tables:

- The _filter_ table: This is the default and perhaps the most widely used table. It is used to make decisions about whether a packet should be allowed to reach its destination.

- The _mangle_ table: This table allows you to alter packet headers in various ways, such as changing TTL values.

- The _nat_ table: This table allows you to route packets to different hosts on NAT (Network Address Translation) networks by changing the source and destination addresses of packets. It is often used to allow access to services that can’t be accessed directly, because they’re on a NAT network.

- The _raw_ table: iptables is a stateful firewall, which means that packets are inspected with respect to their “state”. (For example, a packet could be part of a new connection, or it could be part of an existing connection.) The _raw_ table allows you to work with packets before the kernel starts tracking its state. In addition, you can also exempt certain packets from the state-tracking machinery.

In addition, some kernels also have a _security_ table. It is used by SELinux to implement policies based on [SELinux security contexts](https://selinuxproject.org/page/NB_SC).

### Chains

Now, each of these tables are composed of a few default chains. These chains allow you to filter packets at various points. The list of chains iptables provides are:

- **The PREROUTING chain**: Rules in this chain apply to packets as they just arrive on the network interface. This chain is present in the _nat_, _mangle_ and _raw_ tables.

- **The INPUT chain**: Rules in this chain apply to packets just before they’re given to a local process. This chain is present in the _mangle_ and _filter_ tables.

- **The OUTPUT chain**: The rules here apply to packets just after they’ve been produced by a process. This chain is present in the _raw_, _mangle_, nat and _filter_ tables.

- **The FORWARD chain**: The rules here apply to any packets that are routed through the current host. This chain is only present in the _mangle_ and _filter_ tables.

- **The POSTROUTING chain**: The rules in this chain apply to packets as they just leave the network interface. This chain is present in the _nat_ and _mangle_ tables.

### Target

Some targets are **terminating**, which means that they decide the matched packet’s fate immediately. The packet won’t be matched against any other rules. The most commonly used terminating targets are:
- **ACCEPT**: This causes iptables to accept the packet.

- **DROP**: iptables drops the packet. To anyone trying to connect to your system, it would appear like the system didn’t even exist.

- **REJECT:** iptables “rejects” the packet. It sends a “connection reset” packet in case of TCP, or a “destination host unreachable” packet in case of UDP or ICMP.

On the other hand, there are **non-terminating** targets, which keep matching other rules even if a match was found. An example of this is the built-in LOG target. When a matching packet is received, it logs about it in the kernel logs. However, iptables keeps matching it with rest of the rules too.

When you want to have a set of complext rules, you can create custome chain.


## Some common flag
- -t: Specify Table
- -A: Add rule (+ Chain)
- -D: Delete rule (+ Chain)
- -s: Source
- -j: target policy
- -p: protocol
- --dport: destination port
- -i: interface
## Preserving iptables rules across reboots
For Redhat base
```
sudo yum install iptables-services
```

For Ubuntu base
```
sudo apt install iptables-persistent
```
## Example

### Listing rules:
```
iptables -L
```
### Block IPs

```
iptables -t filter -A INPUT -s 59.45.175.62 -j REJECT
```

### Limit packet

```
iptables -A INPUT -p icmp -m limit --limit 1/sec --limit-burst 1 -j ACCEPT
```
### Configure nat gateway
**eth0**: WAN network
**eth1**: LAN network

- sudo sysctl -w net.ipv4.ip_forward=1
- net.ipv4.ip_forward = 1
- sudo sysctl -p
- sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
- sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
- sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
Save config
- sudo apt install iptables-persistent
- sudo netfilter-persistent save
- sudo netfilter-persistent reload

Reference: [An In-Depth Guide to iptables, the Linux Firewall - Boolean World](https://www.booleanworld.com/depth-guide-iptables-linux-firewall/)

