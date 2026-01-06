---
layout: post
title:  "Network Monitoring with Tailscale"
date:   2024-11-10 05:30:52 +0200
categories: update monitoring network
---
Today I'm going to talk about a relatively straight-forward [network monitoring solution][ping-tool-url] made trivial using [Tailscale][tailscale-url]. While this solution could be built using Linux iptables rules with no other dependencies, I already made use of Tailscale's [subnet router][tailscale-sr-url] functionality across remote subnets, so this was just an extension of an existing ecosystem I used.

### Problem

I wanted to install a backup Internet connection in my household to have ad-hoc access to the Internet during periods of service disruption on my primary, fibre connection. While my router supports a failover circuit, it would move an uncontrolled amount of traffic to a more constrained connection which also carries a consumption cap. Given that most modern clients understand the definition of a metered network connection, I decided to use a second WiFi access point (AP) which selected clients could use as a metered, backup connection. This is a simple way to control both the number of clients that use this backup circuit and the amount of data consumption. Metered connection settings are beneficial because they temporarily disable large backup or update jobs, depending on the environment and operating system.

#### Constraints

Due to the data cap, nothing normally uses this backup circuit. This presents the obvious problem of regular testing of backups and the perils of [bimodal operating modes][statstab-url]. Plan A was to configure a machine on my existing network that hosts an instance of [Uptime Kuma][uptime-kuma-url] to also act as WiFi client, and maintain a constant connection to the Mobile Backup AP. I decided against this because I have a number of Internet-facing probes on the fibre network that needed to remain unchanged, including routes that [Tailscale][tailscale-url] manages for me. I needed to avoid unwanted traffic traversing the backup circuit. To manage complexity, I wanted a solution that made no changes to my existing monitoring infrastructure, other than of course configuring new probes to cover the this use-case. Instead, I wondered whether it would be possible to fashion some kind of [Layer 7 Health Check][l7-url] against the backup circuit because Uptime Kuma supports a variety of probe types, including an HTTP client test, complete with custom header support.

In summary, my requirements were:

* No changes to the machine that hosts my existing monitoring functions, other than new probes in Uptime Kuma.
* No changes to core network routing configuration, apart from existing behaviour already provided by Tailscale clients.
* No changes to *any* configuration of the Backup AP device other than to set the WiFi configuration. All interactions *must* be plug-and-play. For this reason, I picked an AP device that has both WiFi and Ethernet ports and hosts a DHCP Server that runs on both interface types. This is actually very useful for my design.

### Solution

I deliberately use IPv4 addresses here for illustration purposes of the network boundaries.

I used a 14-year old Raspberry Pi Model B+ SBC and a spare USB Ethernet dongle to create a dual-NIC device that would physically connect to both the AP device Ethernet port and a spare port on my existing home network. DHCP servers on both sides would give each interface an address on each network. By adding the SBC to my existing Tailscale "tailnet" as a [subnet router][tailscale-sr-url], I can advertise the subnet that belongs to the Backup AP to other nodes, such as the machine on my home network that runs Uptime Kuma. This allows me to build a TCP and HTTP probe that validates the liveness and health of the Backup AP itself but not the actual connection to the Internet. For obvious reasons, I *cannot* configure the SBC as an [exit node][tailscale-sr-url] because that would route all traffic through the SBC and backup Internet circuit, defeating the purpose of the boundary. This is where I built Plan B: host some kind of "deep ping" on the SBC, triggered by an HTTP probe on the interface facing the home network. Here's the final layout:

![classes](/assets/ping-tool/ping-tool.drawio.png)

The magic that Tailscale provides is illustrated by the coloured, dashed flows depicted above. Probes to `192.168.0.10` are routed as normal to the interface connected to the home network. Probes to the backup AP circuit `192.168.2.1` are transparently routed through `192.168.0.10` and `192.168.2.10` so that both parts of the network are accessible, but *not* the second route to the Internet.

The machine `192.168.0.5` on the home network `192.168.0.0/24` issues an HTTP probe from Uptime Kuma to the interface on the SBC at address `192.168.0.10`. This `HTTP GET` returns success based on the criteria specified by the probe configuration (more on that in the next section). What is actually happening is that `ping-tool` is listening on port `80` (or `8080`) on `192.168.0.10`. Upon receipt of the `HTTP GET`, header configuration is read by the tool, either an ICMP *ping* or *traceroute* is issued to the destination configured in the header, using the second interface on `192.168.2.10` as a source address. Based on the out-the-box DHCP client and routing information provided by the Backup AP, packets then reach the destination on that route, delivered back to the application. After some latency and route inspection, the test result is returned to the caller still waiting on the `HTTP GET` to `192.168.0.10`.

ICMP functions are provided by the Python library [imcplib][imcplib-url], though I needed to create a [fork][icmplib-fork-url] to get the network socket to bind to the correct interface using the option `SO_BINDTODEVICE`. I tried to make the change backwards compatible and also added the useful feature of supporting an address or interface name interchangeably. I did not look deeply into why the interface was not correctly implied by the socket address because I actually wanted to use the interface name as supported natively by the Linux `ping` tool.

#### Probe Configuration

`ping` configuration by issuing an `HTTP GET` request `http://192.168.0.10:8080/ping` with these headers:

{% highlight python %}
{
    "Host": "arbitrary.internet.host.com",
    "Source": "eth1",
    "MinLatencyMs": 10
}
{% endhighlight %}

The above configuration will trigger 3 ICMP packets to be sent to the destination host, and succeed if the minimum packet latency is 10ms as specified by the header `MinLatencyMs`. The choice of host is such that if the packets were sent over the fibre inadvertently, the latency would be much less (1-3ms). In my case, the backup circuit tends to have a latency of 15ms or more, depending on network conditions. This is a crude but seemingly robust way to infer the correct circuit.

`traceroute` configuration by issuing an `HTTP GET` request `http://192.168.0.10:8080/traceroute` with these headers:

{% highlight json %}
{
    "Host": "arbitrary.internet.host.com",
    "Source": "eth1",
    "MinLatencyMs": 10,
    "RouteIncludeCsv": "192.168.2.1",
    "RouteExcludeCsv": "192.168.0.1",
    "HopsMustIncludeOrg": "AS12345",
    "HopsMustExcludeOrg": "AS54321"
}
{% endhighlight %}

This configuration will trigger a `traceroute` to the destination, and for the destination hop enforce a minimum expected latency as with the ping test above using the header `MinLatencyMs`. `traceroute` offers additional information to validate the route. `RouteIncludeCsv` ensures that the route always contains the correct default gateway, and `RouteExcludeCsv` ensures that the fibre network gateway is not present in the route. `HopsMustIncludeOrg` ensures that the route must include the organization that owns the network (in this case the Mobile Network ISP) as specified by [IPinfo][ipinfo-url]. `HopsMustExcludeOrg` is the negative test, ensuring that the fibre network provider is not on the route, which obviously also influences the choice of the destination host. Luckily there's plenty out there.

[icmplib-fork-url]: https://github.com/tglucas/icmplib/commit/83a7151bd910485fb8d73511ab69ed3576ab4d21#diff-7e8617a303da9de7a49dbbcb38738ea1b5ad2442d97f516e3d67da8325f603ffR98
[imcplib-url]: https://github.com/ValentinBELYN/icmplib
[ipinfo-url]: https://ipinfo.io/
[l7-url]: https://kemptechnologies.com/load-balancer/layer-7-load-balancing
[ping-tool-url]:       https://github.com/tailucas/ping-tool
[statstab-url]: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_static_stability.html
[tailscale-en-url]: https://tailscale.com/kb/1103/exit-nodes
[tailscale-sr-url]: https://tailscale.com/kb/1019/subnets
[tailscale-url]:       https://tailscale.com/
[tailscale-url]: https://tailscale.com/
[tailucas-url]: https://github.com/tailucas
[uptime-kuma-url]: https://uptime.kuma.pet/