# Linux Networking Fundamentals

## Cybersecurity Home Lab — Phase 2

**Environment:** Ubuntu Server 24.04.5 LTS (`lab-base`) running in VirtualBox  
**Linux account:** `atlas_admin`  
**Network mode:** VirtualBox NAT  
**Primary interface:** `enp0s3`  
**Guest IPv4 address observed:** `10.0.2.15/24`

---

## 1. Purpose and learning objectives

This document consolidates the completed Phase 2 lessons on Linux networking fundamentals. It records the concepts, commands, and terminal output observed in the home lab. It is intended as a learning and portfolio record, not as a claim that every possible networking scenario has been tested.

By completing these lessons, the learner practised how to:

- identify network interfaces and assigned addresses;
- interpret MAC addresses, IPv4, IPv6, and subnet prefixes;
- read a Linux routing table and identify the default gateway;
- inspect DNS resolver configuration and test name resolution;
- distinguish TCP from UDP and inspect listening sockets;
- identify local and peer endpoints in active connections;
- understand TCP connection states, connection establishment, and connection closure;
- associate sockets with processes;
- troubleshoot connectivity in layers, from the local interface to a remote service.

## 2. Lab topology and known settings

The Ubuntu guest uses VirtualBox NAT networking. The guest's observed IPv4 address is `10.0.2.15/24`. The VirtualBox NAT gateway observed in the guest's routing table is `10.0.2.2`.

The VM's SSH service listens on TCP port `22`. VirtualBox has a host-side port-forwarding rule from host port `2222` to guest port `22`. This allows the Windows host to connect to the guest through `127.0.0.1:2222`.

**Important:** A NAT guest address and a peer address seen inside the guest are not necessarily the same as the Windows host's physical LAN address. NAT and port forwarding can translate addresses. A bridged VM would normally have its own LAN address; a direct connection from another device on that LAN would typically show that device's LAN address as the peer. The actual peer address depends on the network path and any address translation.

No adapter-mode change was made during these lessons.

## 3. Network interfaces and addresses

### 3.1 List interfaces and addresses briefly

Command:

```bash
ip -brief address
```

What it does:

- `ip` displays and manages network information.
- `-brief` requests a compact summary.
- `address` displays addresses assigned to network interfaces.

**Observed output:**

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 metric 100 fd17:625c:f037:2:a00:27ff:fe28:280/64 fe80::a00:27ff:fe28:280/64
```

Interpretation:

- `lo` is the loopback interface. It allows the machine to communicate with itself. `127.0.0.1` and `::1` are the IPv4 and IPv6 loopback addresses.
- `UNKNOWN` on loopback is not, by itself, evidence of a fault.
- `enp0s3` is the VM's primary network interface and was reported `UP`.
- `10.0.2.15/24` is the guest's IPv4 address and prefix length.
- `fd17:.../64` is an IPv6 unique-local address; `fe80:.../64` is an IPv6 link-local address.
- `metric 100` is a route/interface preference value used when choosing among routes; lower metrics are generally preferred when otherwise comparable.

### 3.2 Inspect one interface in detail

Command:

```bash
ip address show enp0s3
```

**Observed output:**

```text
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:28:02:80 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 71642sec preferred_lft 71642sec
    inet6 fd17:625c:f037:2:a00:27ff:fe28:280/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86190sec preferred_lft 14190sec
    inet6 fe80::a00:27ff:fe28:280/64 scope link
       valid_lft forever preferred_lft forever
```

Key fields:

- `BROADCAST` and `MULTICAST` describe supported link-layer capabilities.
- `UP` means the interface is administratively enabled; `LOWER_UP` indicates the underlying link is detected as up.
- `mtu 1500` is the maximum IP packet size, in bytes, for this interface before fragmentation or other handling is considered.
- `link/ether 08:00:27:28:02:80` is the interface's MAC address.
- `brd ff:ff:ff:ff:ff:ff` is the Ethernet broadcast MAC address.
- `inet 10.0.2.15/24` is the IPv4 address and prefix. `/24` corresponds to subnet mask `255.255.255.0`.
- `brd 10.0.2.255` is the IPv4 subnet broadcast address.
- `scope global` means the address is not limited to the local link; it does **not** mean the address is publicly reachable on the internet.
- `dynamic` indicates that the address was dynamically assigned.
- `valid_lft` and `preferred_lft` show address lifetime information.
- The IPv6 `fe80::...` address is link-local and is not routed beyond its local link.

## 4. IPv4, IPv6, and subnet prefixes

The guest's observed IPv4 address is `10.0.2.15/24`.

- IPv4 addresses are 32 bits, conventionally written as four decimal octets.
- `/24` means the first 24 bits identify the network prefix. In this case, the subnet is `10.0.2.0/24` and the usual host range is `10.0.2.1` through `10.0.2.254`; the broadcast address is `10.0.2.255`.
- The guest also has IPv6 addresses. `fd17:...` is in the unique-local range, while `fe80:...` is link-local.
- IPv4 and IPv6 can coexist on the same interface. This is called dual-stack operation.

These values describe the VM's VirtualBox NAT network, not the Windows host's physical LAN subnet.

## 5. Routing and the default gateway

Command:

```bash
ip route
```

**Observed output:**

```text
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.0.1 via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
```

Interpretation:

- `default via 10.0.2.2` is the default route. Traffic without a more specific route is sent to the VirtualBox NAT gateway at `10.0.2.2`.
- `10.0.2.0/24 dev enp0s3` is the directly connected guest subnet.
- The route to `10.0.2.2` identifies the gateway as reachable through `enp0s3`.
- The route to `192.168.0.1` goes via `10.0.2.2`; it does not mean the guest is directly attached to the Windows host's `192.168.0.0/24` LAN.
- `src 10.0.2.15` indicates the preferred source address for that route.
- `proto dhcp` indicates routes supplied through DHCP; `proto kernel` identifies a route created by the kernel for the connected subnet.

A routing table tells the system where to send packets. It does not, by itself, prove that the next hop or final destination is reachable.

## 6. DNS and name resolution

### 6.1 Inspect resolver configuration

Command:

```bash
resolvectl status
```

**Observed output:**

```text
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (enp0s3)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: fd17:625c:f037:2::3
       DNS Servers: 192.168.0.1 fd17:625c:f037:2::3
        DNS Domain: tail8dae02.ts.net lan
```

Interpretation:

- `resolv.conf mode: stub` means applications normally send DNS requests to the local systemd-resolved stub, commonly `127.0.0.53`.
- The configured DNS servers are `192.168.0.1` and `fd17:625c:f037:2::3`.
- The currently selected DNS server shown at that time was `fd17:625c:f037:2::3`.
- `tail8dae02.ts.net` and `lan` are configured search domains.
- `-DNSOverTLS` indicates DNS-over-TLS was not enabled in the displayed configuration.
- `DNSSEC=no/unsupported` indicates DNSSEC validation was not enabled or supported in that configuration. This alone does not establish that a DNS answer is malicious.

### 6.2 Query with nslookup

Command:

```bash
nslookup example.com
```

**Observed output:**

```text
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   example.com
Address: 172.66.147.243
Name:   example.com
Address: 104.20.23.154
Name:   example.com
Address: 2606:4700:10::ac42:93f3
Name:   example.com
Address: 2606:4700:10::6814:179a
```

Interpretation:

- `127.0.0.53` is the local DNS stub resolver, not one of the upstream DNS servers.
- `#53` is DNS port 53.
- The IPv4 answers are A records; the IPv6 answers are AAAA records.
- `Non-authoritative answer` means the responding resolver supplied an answer rather than being authoritative for the domain. It is normal for a recursive resolver to return this kind of answer.

### 6.3 Query with resolvectl

Command:

```bash
resolvectl query example.com
```

**Observed output (first recorded query):**

```text
example.com: 172.66.147.243                    -- link: enp0s3
             104.20.23.154                     -- link: enp0s3
             2606:4700:10::ac42:93f3           -- link: enp0s3
             2606:4700:10::6814:179a           -- link: enp0s3

-- Information acquired via protocol DNS in 84.2ms.
-- Data is authenticated: no; Data was acquired via local or encrypted transport: no
-- Data from: network
```

A later query also succeeded in `77.7ms`, returning the same four addresses in a different order:

```text
example.com: 172.66.147.243                    -- link: enp0s3
             104.20.23.154                     -- link: enp0s3
             2606:4700:10::6814:179a           -- link: enp0s3
             2606:4700:10::ac42:93f3           -- link: enp0s3

-- Information acquired via protocol DNS in 77.7ms.
-- Data is authenticated: no; Data was acquired via local or encrypted transport: no
-- Data from: network
```

The reported authentication and transport fields describe the resolver's assessment of that query. `Data is authenticated: no` is not, by itself, proof of tampering. `Data was acquired via local or encrypted transport: no` indicates the resolver did not classify the acquisition as local or encrypted.

## 7. TCP, UDP, ports, and listening sockets

### 7.1 Show listening TCP and UDP sockets

Command:

```bash
ss -tuln
```

Option meanings:

- `ss` displays socket statistics.
- `-t` selects TCP sockets.
- `-u` selects UDP sockets.
- `-l` selects listening sockets.
- `-n` displays numeric addresses and port numbers instead of resolving names.

**Observed output:**

```text
Netid              State               Recv-Q               Send-Q                                Local Address:Port                             Peer Address:Port              Process
udp                UNCONN              0                    0                                        127.0.0.54:53                                    0.0.0.0:*
udp                UNCONN              0                    0                                     127.0.0.53%lo:53                                    0.0.0.0:*
udp                UNCONN              0                    0                                  10.0.2.15%enp0s3:68                                    0.0.0.0:*
tcp                LISTEN              0                    4096                                     127.0.0.54:53                                    0.0.0.0:*
tcp                LISTEN              0                    4096                                  127.0.0.53%lo:53                                    0.0.0.0:*
tcp                LISTEN              0                    4096                                        0.0.0.0:22                                    0.0.0.0:*
tcp                LISTEN              0                    4096                                           [::]:22                                       [::]:*
```

Interpretation:

- DNS resolver sockets are listening on local addresses `127.0.0.53` and `127.0.0.54`, port `53`.
- UDP port `68` is associated with DHCP client activity.
- SSH is listening on TCP port `22` on IPv4 wildcard address `0.0.0.0` and IPv6 wildcard address `[::]`.
- `0.0.0.0` means all local IPv4 interfaces; `[::]` is the IPv6 wildcard. These entries do not alone prove internet exposure. Reachability depends on network mode, routing, and firewall rules.
- UDP `UNCONN` is normal; UDP does not use TCP's connection handshake.

### 7.2 Show TCP sockets, including active connections

Command:

```bash
ss -tan
```

**Observed output:**

```text
State                  Recv-Q                 Send-Q                                 Local Address:Port                                   Peer Address:Port                  Process
LISTEN                 0                      4096                                      127.0.0.54:53                                          0.0.0.0:*
LISTEN                 0                      4096                                   127.0.0.53%lo:53                                          0.0.0.0:*
LISTEN                 0                      4096                                         0.0.0.0:22                                          0.0.0.0:*
ESTAB                  0                      0                                          10.0.2.15:22                                         10.0.2.2:56153
LISTEN                 0                      4096                                            [::]:22                                             [::]:*
```

`-a` includes listening and non-listening sockets. `ESTAB` is an established TCP connection. `LISTEN` is a socket waiting for incoming TCP connections.

### 7.3 Understand local and peer endpoints

The `Local Address:Port` column is the endpoint on the machine where `ss` is being run. `Peer Address:Port` is the other endpoint of the socket.

For the observed SSH connection:

- Local endpoint: `10.0.2.15:22` — the Ubuntu guest's SSH service.
- Peer endpoint: `10.0.2.2:56153` — the peer address and ephemeral source port as seen by the Ubuntu guest through the NAT path.

For the observed DNS socket:

- Local endpoint: `10.0.2.15:53720`.
- Peer endpoint: `192.168.0.1:53` — the DNS server's address and DNS port.

Port `56153` and `53720` are source ports selected for those client-side sockets; they are not the SSH and DNS server ports. SSH's server port is `22`; DNS's server port is `53`.

### 7.4 Show active TCP and UDP sockets

Command:

```bash
ss -tun
```

**Observed output at the time:**

```text
tcp ESTAB 0 0 10.0.2.15:22 10.0.2.2:56153
```

The TCP connection was established. The output at that moment did not show the UDP DNS socket; socket listings are snapshots and can vary depending on activity and timing.

### 7.5 Associate sockets with processes

Command:

```bash
sudo ss -tupn
```

**Observed output (first capture):**

```text
Netid           State           Recv-Q           Send-Q                     Local Address:Port                      Peer Address:Port           Process
udp             ESTAB           0                0                              10.0.2.15:53720                      192.168.0.1:53              users:("systemd-resolve",pid=8732,fd=11)
tcp             ESTAB           0                0                              10.0.2.15:22                            10.0.2.2:56153           users:("sshd",pid=8465,fd=4),("sshd",pid=8340,fd=4)
```

A later capture showed the DNS client using a different ephemeral source port, while the SSH connection remained:

```text
Netid           State           Recv-Q           Send-Q                     Local Address:Port                      Peer Address:Port           Process
udp             ESTAB           0                0                              10.0.2.15:36812                      192.168.0.1:53              users:("systemd-resolve",pid=8732,fd=11)
tcp             ESTAB           0                0                              10.0.2.15:22                            10.0.2.2:56153           users:("sshd",pid=8465,fd=4),("sshd",pid=8340,fd=4)
```

Interpretation:

- `-p` requests process information. `sudo` is used because process details for sockets owned by other users may otherwise be hidden.
- `systemd-resolve` is associated with the DNS UDP socket; `pid=8732` is the process ID and `fd=11` is its file descriptor.
- The SSH socket was reported with `sshd` PIDs `8465` and `8340`, both associated with file descriptor `4` in that capture. Do not assign exact parent/child roles from this listing alone; inspect process details separately if needed.
- `ESTAB` on a UDP socket does not mean a TCP-style handshake occurred. A UDP socket can be connected to a peer for local socket handling without UDP becoming connection-oriented.

## 8. TCP connection concepts and states

### 8.1 TCP versus UDP

TCP is connection-oriented. It establishes a connection before application data is exchanged and provides ordered, reliable delivery. SSH uses TCP; the guest's SSH server listens on port `22`.

UDP is connectionless at the transport-protocol level. It does not perform TCP's three-way handshake and does not provide TCP's built-in reliable, ordered byte stream. DNS commonly uses UDP port `53`, though DNS can also use TCP.

### 8.2 TCP three-way handshake

A typical TCP connection is established in three steps:

1. Client sends `SYN` to request a connection.
2. Server responds with `SYN-ACK`.
3. Client sends `ACK`.

After this exchange, the TCP connection is established. SSH authentication occurs later, at the application layer; the TCP handshake is not SSH login authentication.

### 8.3 Multiple clients and connection identity

A server can handle multiple TCP connections to the same local IP address and port. Each connection is distinguished by a four-part tuple:

- source IP address;
- source port;
- destination IP address;
- destination port.

In the lab, two simultaneous Windows-to-Ubuntu SSH sessions were observed with the same peer IP but different peer source ports:

```text
State Recv-Q Send-Q Local Address:Port Peer Address:Port
ESTAB 0 0 10.0.2.15:22 10.0.2.2:49687
ESTAB 0 0 10.0.2.15:22 10.0.2.2:56153
```

After the second Windows PowerShell SSH session was closed with `exit`, the first remained:

```text
ESTAB 0 52 10.0.2.15:22 10.0.2.2:56153
```

The second connection disappeared from the snapshot. `Send-Q 52` means 52 bytes were queued for sending at that instant; a nonzero queue in a single snapshot does not, by itself, prove a fault.

### 8.4 TCP connection closure

A common orderly TCP close uses four steps:

1. One endpoint sends `FIN` to indicate it has finished sending.
2. The other endpoint acknowledges with `ACK`.
3. The other endpoint sends its own `FIN` when ready to close its sending direction.
4. The first endpoint acknowledges with `ACK`.

Either side can initiate closure. This is a typical sequence, not a guarantee that every connection trace will look identical.

### 8.5 TIME-WAIT and CLOSE-WAIT

Commands used:

```bash
ss -tan state time-wait
ss -tan state close-wait
```

Both commands returned only the column headings at the time of observation; no matching sockets were listed.

- `TIME-WAIT` is a TCP state commonly held by the endpoint that actively closes a connection. It allows delayed packets to expire and helps ensure the final acknowledgement can be handled.
- `CLOSE-WAIT` means the local TCP stack has received a peer's `FIN` and acknowledged it, while the local application has not yet closed its socket. A large number of persistent `CLOSE-WAIT` sockets can indicate an application is failing to close connections.
- Seeing no entries in a snapshot is normal and does not indicate a problem.

## 9. Network troubleshooting practical

This section consolidates the actual checks performed in the troubleshooting lesson. The sequence moves from the local interface toward a remote application service.

### 9.1 Check interface state and addressing

Command:

```bash
ip -brief address
```

**Observed result:** `enp0s3` was `UP` and had `10.0.2.15/24` plus IPv6 addresses. This confirmed interface state and address assignment, but not end-to-end reachability.

### 9.2 Test the VirtualBox NAT gateway

Command:

```bash
ping -c 4 10.0.2.2
```

**Observed output:**

```text
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=0.271 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=0.206 ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=0.191 ms
64 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=0.257 ms

--- 10.0.2.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3159ms
rtt min/avg/max/mdev = 0.191/0.231/0.271/0.033 ms
```

Result: all four packets received; zero packet loss; average round-trip time `0.231 ms`. The guest could reach its VirtualBox NAT gateway at the time of the test.

### 9.3 Test external IP connectivity without DNS

Command:

```bash
ping -c 4 1.1.1.1
```

**Observed output:**

```text
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=64 time=186 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=64 time=149 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=64 time=197 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=64 time=179 ms

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3027ms
rtt min/avg/max/mdev = 149.404/177.694/196.661/17.537 ms
```

Result: external IPv4 reachability succeeded at the time of testing, with zero packet loss and average round-trip time `177.694 ms`. Because the target was specified as an IP address, DNS was not needed for this test.

### 9.4 Test DNS resolution and connectivity together

Command:

```bash
ping -c 4 example.com
```

**Observed output:**

```text
PING example.com (172.66.147.243) 56(84) bytes of data.
64 bytes from 172.66.147.243: icmp_seq=1 ttl=64 time=161 ms
64 bytes from 172.66.147.243: icmp_seq=2 ttl=64 time=215 ms
64 bytes from 172.66.147.243: icmp_seq=3 ttl=64 time=228 ms
64 bytes from 172.66.147.243: icmp_seq=4 ttl=64 time=251 ms

--- example.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 7259ms
rtt min/avg/max/mdev = 161.022/213.892/250.659/33.021 ms
```

Result: the hostname resolved to `172.66.147.243`, and all four ICMP requests received replies. This demonstrated working name resolution and reachability for this test at that time.

### 9.5 Inspect DNS resolver and query directly

Commands:

```bash
resolvectl status
resolvectl query example.com
```

Both succeeded during the lesson. The resolver configuration and actual query outputs are recorded in Section 6.

### 9.6 Test TCP HTTPS port 443

Command:

```bash
nc -vz -w 5 example.com 443
```

**Observed output:**

```text
Connection to example.com (172.66.147.243) 443 port [tcp/https] succeeded!
```

Result: a TCP connection to port `443` succeeded. This verifies TCP reachability to that port, not the TLS certificate or the full HTTPS application exchange.

### 9.7 Compare with remote SSH port 22

Command:

```bash
nc -vz -w 5 example.com 22
```

**Observed output:**

```text
nc: connect to example.com (172.66.147.243) port 22 (tcp) timed out: Operation now in progress
nc: connect to example.com (104.20.23.154) port 22 (tcp) timed out: Operation now in progress
nc: connect to example.com (2606:4700:10::ac42:93f3) port 22 (tcp) failed: Connection refused
nc: connect to example.com (2606:4700:10::6814:179a) port 22 (tcp) failed: Connection refused
```

Interpretation:

- IPv4 connection attempts timed out. No successful TCP connection was established before the timeout.
- IPv6 attempts were refused, meaning the connection attempt received an explicit rejection.
- These results do not prove that the entire network is broken. HTTPS port `443` worked while SSH port `22` did not.
- The outcome is specific to the destination, address family, network path, and time of testing. Do not generalise it to all servers or all SSH traffic.

### 9.8 Test the guest's own SSH service through loopback

Command:

```bash
nc -vz -w 5 127.0.0.1 22
```

**Observed output:**

```text
Connection to 127.0.0.1 22 port [tcp/ssh] succeeded!
```

Result: the guest could establish a TCP connection to its own SSH listener through loopback. This did not test SSH authentication.

### 9.9 Test the guest's SSH service through its interface address

Command:

```bash
nc -vz -w 5 10.0.2.15 22
```

**Observed output:**

```text
Connection to 10.0.2.15 22 port [tcp/ssh] succeeded!
```

Result: the guest could establish a TCP connection to SSH through its primary network-interface address. This did not test authentication or prove that the service was reachable from the public internet.

## 10. A layered troubleshooting method

When a network problem occurs, test in order and interpret each result carefully:

1. **Interface and address:** `ip -brief address` — is the relevant interface up, and does it have an address?
2. **Route selection:** `ip route` — what route would be used for the destination?
3. **Local gateway:** `ping -c 4 <gateway>` — does the next hop respond to ICMP?
4. **External IP:** `ping -c 4 <known-ip>` — can the host reach an external IP without needing DNS?
5. **DNS:** `resolvectl status` and `resolvectl query <hostname>` — are resolver settings present, and does name resolution work?
6. **TCP service:** `nc -vz -w 5 <hostname-or-ip> <port>` — can a TCP connection be established to the target port?
7. **Local listener and process:** `ss -ltnp` or `sudo ss -tupn` — is a service listening, and which process is associated with a socket?

Not every host responds to ping. ICMP may be filtered while TCP services remain reachable. A failed ping alone is not proof that a host or service is down. Likewise, a successful TCP connection does not prove that application authentication, TLS validation, or the full application protocol works.

## 11. Security implications

- Listening services increase the VM's potential attack surface. Keep services limited to what the lab needs.
- A listener on `0.0.0.0` or `[::]` is bound to wildcard addresses, but actual reachability still depends on network configuration and firewall policy.
- NAT with port forwarding and Bridged networking have different exposure characteristics. Bridged mode places the VM more directly on the LAN and can make its services reachable to other LAN devices, depending on firewall and network policy.
- Do not change the VM from NAT to Bridged merely to perform these exercises. Any network-mode change should be planned, explained, and explicitly approved.
- Use `sudo ss -tupn` to investigate which processes own sockets. Treat unfamiliar listeners and connections as items to investigate, not automatic proof of compromise.
- DNSSEC and DNS-over-TLS were not enabled in the observed resolver configuration. This is a recorded configuration observation, not a conclusion that the network is compromised.
- Do not test remote systems or ports without authorisation. The tests in this lesson were limited to the user's own VM and public test destinations.

## 12. Expected results versus observed results

The observed outputs in this document are copied from the learner's terminal captures during the lessons. They are historical snapshots; interface addresses, ephemeral ports, DNS answers, process IDs, connection states, and response times can change.

Expected general patterns when repeating the exercises:

- `ip -brief address` should show the active interface and its currently assigned addresses.
- `ip route` should show connected routes and a default route if one is configured.
- A gateway ping may return replies if ICMP is permitted and the gateway is reachable.
- An external IP ping may return replies if the route and destination permit ICMP.
- `resolvectl query example.com` should return address records when DNS resolution works.
- `ss -ltn` should show TCP listeners that exist at the time of the command.
- `nc -vz` should report success when a TCP connection is accepted, a timeout when no successful response arrives in time, or refusal when the connection is actively rejected.

These are expected patterns, not guaranteed outcomes. Network policies, firewalls, DNS answers, and service availability can change.

## 13. Cleanup and system changes

The exercises in this document were observational or used short-lived connection tests. No firewall rules, routes, DNS settings, SSH settings, or VirtualBox adapter settings were changed as part of this practical.

Netcat (`nc -z`) performed connection checks and did not create a persistent service. Ping processes ended after four requests. No special cleanup is required.

If repeating the tests, do not terminate unfamiliar processes or change network configuration merely because an output differs. Capture the output first and investigate the difference.

## 14. Knowledge checks completed

The learner completed the following conceptual checks during the lessons:

- Identifying the peer address under a hypothetical direct Bridged connection: correct answer was the Windows host's LAN IP (`192.168.0.126`) in the stated example.
- Interpreting socket process ownership, PID, file descriptor, and why `sudo` was used: 4/4 correct.
- Distinguishing DNS failure from external IP connectivity: correct answer was DNS resolution failure.
- Distinguishing successful HTTPS connectivity from an unestablished SSH connection: correct answer was that HTTPS worked but SSH connectivity to that destination was not established.
- Final troubleshooting assessment: 4/4 correct.

## 15. Suggested GitHub location and commit workflow

Recommended repository path:

```text
docs/networking/linux-networking-fundamentals.md
```

From Windows PowerShell, in the repository:

```powershell
cd C:\Users\Contessina\cybersecurity-home-lab
git status
```

After placing this file at the recommended path, stage it:

```powershell
git add docs/networking/linux-networking-fundamentals.md
```

Review the staged change:

```powershell
git diff --cached --stat
git status
```

Commit it:

```powershell
git commit -m "Add Linux networking fundamentals documentation"
```

Push the commit:

```powershell
git push origin main
```

Run each command separately and review its output. Do not assume the commit or push succeeded until Git confirms it.

---

## 16. Current progress and next topics

Completed in Phase 2:

- interfaces and MAC addresses;
- IPv4/IPv6 addressing and subnet prefixes;
- routing and default gateway;
- DNS and name resolution;
- TCP and UDP;
- ports and listening services;
- active network connections and socket ownership;
- TCP handshake, multiple clients, and connection states;
- layered network troubleshooting.

Potential next topics include more focused Linux network diagnostic tools, packet inspection fundamentals, and introductory network security monitoring. These should be introduced as new lessons rather than repeating the completed exercises.
