# netprobe

A small virtual PC that speaks raw Ethernet over a Linux tap device — its own
ARP / IPv4 / ICMP / DHCP-client / DNS-client / minimal-TCP stack, written in
C, with no dependency on the host kernel's IP stack for its own traffic.

Its main use case is as a **binary-swap, drop-in replacement for `vpcs`
(Virtual PC Simulator)** in network lab emulators that use it as their
lightweight end-host node. It's a general-purpose tool otherwise too —
anywhere you want a scriptable line-CLI "PC" attached to a tap device.

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
  -m <int>       session/instance id
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

### Drop-in replacement for vpcs

netprobe accepts the same launch arguments the standard `vpcs` binary does,
so it can be swapped in as a binary replacement anywhere `vpcs` is used —
**GNS3, EVE-NG, PNetLab**, or a manual invocation. Rename/copy the released
binary over the existing `vpcs` binary in your simulator's install
(e.g. `/opt/vpcsu/bin/vpcs` on PNetLab) and restart the node/VM.

## Resource footprint vs. stock vpcs

Measured side by side against stock vpcs 0.8.2 on the same host, same tap
setup, same flood (`ping -f -c 20000`, whole process family measured):

| | stock vpcs 0.8.2 | netprobe | delta |
|---|---|---|---|
| idle CPU (steady state) | 1.900% of a core | 0.033% | ~58× less |
| idle RSS | 4256 kB | 316 kB | ~13× less |
| OS processes per node | 2 (forks a worker) | 1 | half |
| runs as | uid 0 (root) | unprivileged, drop verified irreversible | privilege dropped |
| CPU for 20000 ICMP echoes | 1.89 s | 0.74 s | 2.6× less |
| wall time for that flood | 3.60 s | 1.92 s | 1.9× faster |
| ICMP RTT avg / max | 0.134 / 0.913 ms | 0.050 / 0.464 ms | ~2.7× lower |
| RSS under load | 4748 kB | 324 kB | ~14.6× less |
| on-disk binary | 204280 B (dynamic, PIE) | 153128 B (static) | — |
| shared-lib deps | libpthread, libutil, libc, ld.so | none (static) | — |

Stock vpcs busy-polls, burning ~1.9% of a core per node even with no
traffic and no console attached — on a 20-node lab that's most of a core
reclaimed before a single packet moves. netprobe idles at a 10 Hz timer
instead. Caveat: the per-packet numbers are ICMP-echo only; TCP/UDP/flow
throughput wasn't benchmarked.

## License

MIT — see [LICENSE](LICENSE).
