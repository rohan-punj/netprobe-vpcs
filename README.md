# netprobe

A small virtual PC that speaks raw Ethernet over a Linux tap device — its own
ARP / IPv4 / ICMP / DHCP-client / DNS-client / minimal-TCP stack, written in
C, with no dependency on the host kernel's IP stack for its own traffic.

Its main use case is as a **binary-swap, drop-in replacement for `vpcs`
(Virtual PC Simulator) in [PNetLab](https://pnetlab.com/)**. It's a
general-purpose tool otherwise too — anywhere you want a scriptable
line-CLI "PC" attached to a tap device.

Static binary, no runtime dependencies (musl-linked), x86-64 Linux.

## What it does

- Ethernet/ARP, IPv4 (including TX fragmentation and RX reassembly), ICMP
  (ping, with traceroute-style TTL probing)
- DHCP client (lease acquire/renew/release)
- DNS client (A/AAAA/CNAME/PTR lookups against a configured or DHCP-supplied
  server)
- Minimal TCP: handshake, in-order established-state transfer, one bounded
  retransmission queue, clean teardown — a lab-network reachability tool,
  not a general-purpose TCP stack (no congestion control, no out-of-order
  reassembly, no window scaling)
- UDP send/listen, simple traffic generator (`flow`)
- A telnet-style line console with command history, and a `startup.vpc`
  config file compatible with `save`/`reset`

## Usage

```
Usage: netprobe -m <session> -p <port> [-N <pcname> ... -i <anything>] [-e -d <tap>]...
       netprobe --tap <tap> -m <session> -p <port>

Options:
  -m <int>       PNetLab session id
  -N <tokens...> PC name, through the next literal -i
  -p <int>       required console TCP port
  -e             enable Ethernet/TAP mode
  -d <name>      TAP device name (repeatable)
  -i <anything>  accepted and ignored
  --tap <name>   standalone alias for -e -d <name>
  --help         show this help
  --version      show the netprobe version
```

Once running, connect to the console TCP port (e.g. `telnet localhost
<port>`) and use the in-console command set:

```
Addressing:
  ip <addr>/<prefix> [<gateway>] | ip dhcp [-r]
  ip dns <server>
  mac eth0 <addr>
  show ip
Diagnostics:
  ping <host> [-c <n>] [-i <ms>] [-s <bytes>] [-t <ttl>] [-D]
  trace <host> [-m <maxttl>] [-q <probes>]
  arp show | arp clear
Services:
  dns <name> [A|AAAA|CNAME|PTR] [@<server>]
  tcp connect <host> <port> [-m <msg>] | tcp listen <port> [-e] | tcp close <port>
  udp send <host> <port> <msg> | udp listen <port> [-e] | udp close <port>
  flow start <dst> <pps> <bytes> [-p udp|tcp] [-d <port>] | flow stop [<id>] | flow show
Config:
  save
  reset
```

### PNetLab drop-in

Rename/copy the released binary to wherever your PNetLab install expects
`vpcs` (typically `/opt/vpcsu/bin/vpcs`) and restart the node. netprobe
accepts the same launch arguments PNetLab's engine already passes to vpcs.

## License

MIT — see [LICENSE](LICENSE).
