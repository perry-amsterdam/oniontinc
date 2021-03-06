# oniontinc

## Rationale

[tinc](https://www.tinc-vpn.org/) is a VPN daemon that creates securely encrypted full-mesh private networks among Internet hosts. In other words, tinc is a P2P mesh VPN. However, all hosts see the IP addresses of all peers. And there are no built-in mechanisms for path obfuscation, or traffic padding.

[Tor](https://www.torproject.org/) is an onion-routing anonymity network, which helps defend users and onion servers against tracking and surveillance. Given that UDP traffic is arguably harder to anonymize, Tor routes only TCP traffic. Also, circuit bandwidths are often low, and there are no built-in mechanisms for multipathing and link aggregation.

Basically, [oniontinc](http://mtnj6jcrlimpqh5q76e7xdxfdoadtd67asyojknhvpic5dmluokdbiad.onion) routes tinc via Tor, and uses [MPTCP](http://multipath-tcp.org/pmwiki.php/Main/HomePage) for multipathing and link aggregation. tinc hosts listen for connections to Tor v3 onion services, and they connect to peers using Tor SocksPorts. MPTCP aggregates full-mesh connections between hosts, with a preference for those that have greater bandwidth and lower latency. Given the [combinatorial](./tor-tinc-mptcp-diagram.pdf) nature of full-mesh networks with multipathing, six tinc interfaces per host is apparently the practical maximum. 

For Internet hosts with well-peered gigabit uplinks, this permits [throughput](./tor-tinc-mptcp-throughput.png) among peers at 30-50 Mbps for multiple streams. Throughput depends on underlying bandwidth and peering, and also on transport specifics. Iperf3 in TCP mode saturates six tinc networks with multiple streams. So does [parsyncfp](http://moo.nac.uci.edu/~hjm/parsync/), with multiple rsync processes specified. And even scp does, with scripted creation of multiple processes. tinc also routes UDP, but there is no multipathing.

Increasing bandwidth via Tor does reduce anonymity, of course. And full-mesh connectivity does require many circuits. However, there are no Tor exits involved, for which system bandwidth is most limited. Also, I gather that the v3 onion specification is more efficient and more secure, and uses less resources overall.

## Usage

There are bash scripts for almost everything involved in creating full-mesh oniontinc networks, using Debian hosts. The scripts draw on a simple configuration table with required information about all hosts in the network. That is: 

* machine hostnames
* tor instance names, torrc files, and SocksPorts
* tinc network and host names, and listening ports
* third and fourth octets of tinc IPv4 addresses
* Tor v3 onion addresses for tinc instances

The machine hostname is the index. So after installing requisite software and the MPTCP kernel, you just run the scripts in each host. That creates everything needed for a full-mesh oniontinc network (tor instances, tinc configuration files, tinc keys, tinc host files, tinc-up and tinc-down scripts, etc). Iptables rules for IPv4 allow ssh login via all interfaces, but allow icmp, iperf3, and http/https traffic only via tinc interfaces. They allow traffic via eth0 (or equivalent) only for tor processes. That is, tinc can only connect via tor. They allow all outgoing traffic via tinc interfaces, but no other unrelated/unestablished incoming traffic. You may need to change "eth0" to whatever your system uses. Iptables rules for IPv6 drop everything.

Once you have exchanged host files among all peers, you start the tinc instances, and all links should be up.

## tinc network subnets

The scripts specify subnets 10.101.1N.0/24 with N=0-5, which are part of the subnet 10.100.0.0/14 that [ChaosVPN](https://wiki.hamburg.ccc.de/ChaosVPN:IPRanges) has allocated for American and other hackerspaces. However, I do not currently plan to use more than the first 30 IPs of each subnet. So you are welcome to use other IPs in those /24 subnets, and link via Tor with my host "one". If you do that, please update the ChaosVPN wiki promptly, to avoid (sorry) chaos. I'll also need your host files archive, and you can email it to <annobrown@protonmail.com>. If you want to use your own subnets, you'll need to edit the scripts "tweak-full-mesh-tinc-hosts.sh" and "create-full-mesh-tinc-up.sh" accordingly. 

## Peering from clearnet tinc subnets

If you'd rather not use Tor, you can peer with host gateway6, at the public IPv4 194.36.190.113, using a standard tinc setup, as covered in the [tinc Manual](https://www.tinc-vpn.org/documentation/). The tinc subnet is 10.101.20.0/24, the tinc port is 6560, and the host file is [here](./gateway6-host.txt). It's a SNAT gateway to the oniontinc network (10.101.1N.0/24, with N=0-5). Alternatively, you could peer from an existing tinc subnet. In either case, you would need to provide your hosts file to <annobrown@protonmail.com>. You would also need to [redirect](https://www.tinc-vpn.org/examples/redirect-gateway/) the peer's gateway for 0.0.0.0/1 and 128.0.0.0/1 to 10.101.20.6/32, with route exceptions for the tinc daemon, SSH access, etc.
