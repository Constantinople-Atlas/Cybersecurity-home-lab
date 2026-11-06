# Linux Network Troubleshooting Fundamentals

## Lab overview

**Phase:** 2 — Linux Networking Fundamentals for Cybersecurity  
**Topic:** Linux network troubleshooting  
**Platform:** Ubuntu Server 24.04.5 LTS (`lab-base`) running in VirtualBox  
**Linux user:** `atlas_admin`  
**Network interface:** `enp0s3`  
**VM IPv4 address:** `10.0.2.15/24`  
**VirtualBox NAT gateway:** `10.0.2.2`

## Objectives

By completing this exercise, the learner practised a systematic approach to checking:

1. Network-interface state and addressing.
2. Reachability of the local gateway.
3. External IP connectivity.
4. DNS resolution.
5. TCP service-port availability.
6. Local and interface-address access to the VM's SSH service.

No network configuration was changed during this exercise. The tests were diagnostic only.

## 1. Check the network interface

Command:

```bash
ip -brief address
```

What it does:

- `ip` displays and manages network information.
- `-brief` requests compact output.
- `address` displays addresses assigned to network interfaces.

### Actual observed output

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 metric 100 fd17:625c:f037:2:a00:27ff:fe28:280/64 fe80::a00:27ff:fe28:280/64
```

### Interpretation

- `lo` is the loopback interface. Its `UNKNOWN` state is normal in this context and is not, by itself, evidence of a fault.
- `enp0s3` is the VM's main network interface and was reported `UP`.
- `10.0.2.15/24` is the VM's IPv4 address and prefix.
- The interface also had IPv6 addresses: a unique-local address (`fd17:.../64`) and a link-local address (`fe80:.../64`).

**Finding:** The main interface was up and had IPv4 and IPv6 addresses. This alone does not prove that other devices or the internet are reachable.

## 2. Test the local gateway

Command:

```bash
ping -c 4 10.0.2.2
```

What it does:

- `ping` sends ICMP Echo Requests and reports replies.
- `-c 4` sends four requests and then stops.
- `10.0.2.2` is the VirtualBox NAT gateway shown in the VM's routing information.

### Actual observed output

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

### Interpretation

All four requests received replies. Packet loss was 0%, and average round-trip time was approximately 0.231 ms.

**Finding:** The VM could reach its VirtualBox NAT gateway at the time of the test.

## 3. Test external IP connectivity without DNS

Command:

```bash
ping -c 4 1.1.1.1
```

This tests reachability to an external IP address directly. Because the destination is already an IP address, this test does not require a DNS lookup to identify the destination.

### Actual observed output

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

### Interpretation

All four requests received replies. Packet loss was 0%, and average round-trip time was approximately 177.694 ms.

**Finding:** The VM could reach this external IP address at the time of the test. A successful ping to one external IP does not prove that every internet destination or application service is reachable.

## 4. Test DNS resolution and connectivity together

Command:

```bash
ping -c 4 example.com
```

This combines two checks:

1. The system resolves `example.com` to an IP address.
2. The system sends ICMP requests to the resolved address.

### Actual observed output

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

### Interpretation

The hostname resolved to `172.66.147.243`, and all four ICMP requests received replies. Packet loss was 0%.

**Finding:** DNS resolution and ICMP connectivity to the resolved address both worked during this test. A successful ping does not guarantee that every application protocol is available.

## 5. Inspect resolver configuration

Command:

```bash
resolvectl status
```

### Actual observed output

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

### Interpretation

- The link associated with `enp0s3` had DNS scope.
- The configured DNS servers were `192.168.0.1` and `fd17:625c:f037:2::3`.
- The currently selected DNS server was `fd17:625c:f037:2::3`.
- `resolv.conf mode: stub` indicates that applications normally use the local systemd-resolved stub resolver, commonly `127.0.0.53`.
- DNS-over-TLS was not enabled in the displayed configuration.
- `DNSSEC=no/unsupported` means DNSSEC authentication was not available/enabled as reported by this resolver status. It does not, by itself, establish that a DNS response is malicious.

**Finding:** DNS servers were configured. Configuration alone does not prove that a DNS server is reachable or responding.

## 6. Query DNS using systemd-resolved

Command:

```bash
resolvectl query example.com
```

### Actual observed output

```text
example.com: 172.66.147.243                    -- link: enp0s3
             104.20.23.154                     -- link: enp0s3
             2606:4700:10::6814:179a           -- link: enp0s3
             2606:4700:10::ac42:93f3           -- link: enp0s3

-- Information acquired via protocol DNS in 77.7ms.
-- Data is authenticated: no; Data was acquired via local or encrypted transport: no
-- Data from: network
```

### Interpretation

- The query returned two IPv4 addresses and two IPv6 addresses.
- The resolver reported acquiring the information in 77.7 ms.
- `Data is authenticated: no` means the answer was not authenticated through DNSSEC according to this result; it does not automatically mean the answer was unsafe.
- `Data was acquired via local or encrypted transport: no` indicates that the resolver did not classify the acquisition as local or encrypted.
- `Data from: network` indicates that the answer came from a network DNS query.

**Finding:** The resolver successfully returned DNS records for `example.com` during this test.

## 7. Test an external HTTPS TCP port

Command:

```bash
nc -vz -w 5 example.com 443
```

Option explanation:

- `nc` is Netcat, a network utility.
- `-v` requests verbose output.
- `-z` checks for a listening service without sending application data.
- `-w 5` sets a five-second timeout.
- `443` is the conventional HTTPS TCP port.

### Actual observed output

```text
Connection to example.com (172.66.147.243) 443 port [tcp/https] succeeded!
```

### Interpretation

A TCP connection to port 443 succeeded for the resolved IPv4 address shown.

**Finding:** TCP connectivity to that destination's port 443 was established. This test did not verify a TLS handshake, certificate, or complete HTTPS request.

## 8. Compare with an external SSH port

Command:

```bash
nc -vz -w 5 example.com 22
```

### Actual observed output

```text
nc: connect to example.com (172.66.147.243) port 22 (tcp) timed out: Operation now in progress
nc: connect to example.com (104.20.23.154) port 22 (tcp) timed out: Operation now in progress
nc: connect to example.com (2606:4700:10::ac42:93f3) port 22 (tcp) failed: Connection refused
nc: connect to example.com (2606:4700:10::6814:179a) port 22 (tcp) failed: Connection refused
```

### Interpretation

- The IPv4 connection attempts timed out: no successful TCP connection was established before the timeout.
- The IPv6 connection attempts were refused: the attempts received an explicit rejection.
- These results can occur when a destination does not offer SSH on that port or when network filtering blocks or rejects the traffic.
- The output does not identify the exact cause of the remote server's policy or configuration.

**Finding:** TCP port 22 was not reachable successfully on the tested external destination, even though port 443 was reachable. This is not evidence that the VM's own SSH service is broken.

## 9. Test the VM's SSH port through loopback

Command:

```bash
nc -vz -w 5 127.0.0.1 22
```

### Actual observed output

```text
Connection to 127.0.0.1 22 port [tcp/ssh] succeeded!
```

### Interpretation

The VM established a TCP connection to its own loopback address on port 22.

**Finding:** The SSH TCP port accepted a local connection through loopback. Netcat did not perform SSH authentication.

## 10. Test the VM's SSH port through its network interface

Command:

```bash
nc -vz -w 5 10.0.2.15 22
```

### Actual observed output

```text
Connection to 10.0.2.15 22 port [tcp/ssh] succeeded!
```

### Interpretation

The VM established a TCP connection to port 22 using its `enp0s3` IPv4 address.

**Finding:** The SSH TCP port accepted a connection through the VM's network-interface address. This does not prove that the VM is reachable from every other LAN device or from the public internet. Reachability depends on the VirtualBox network mode, forwarding rules, host/network firewalls, and routing.

## Overall observed results

| Check | Observed result | What it established |
|---|---|---|
| Interface | `enp0s3` was `UP` | Interface was enabled and had addresses |
| NAT gateway | 4/4 replies, 0% loss | Gateway was reachable |
| External IP | 4/4 replies, 0% loss | `1.1.1.1` was reachable by ICMP |
| Domain ping | 4/4 replies, 0% loss | Name resolution and ICMP to resolved address worked |
| Resolver query | DNS records returned | systemd-resolved completed the query |
| External TCP 443 | Succeeded | TCP connection to port 443 was established |
| External TCP 22 | IPv4 timed out; IPv6 refused | TCP connection to port 22 was not established |
| Loopback SSH TCP 22 | Succeeded | Local TCP connection to SSH port worked |
| VM-address SSH TCP 22 | Succeeded | TCP connection through `10.0.2.15` worked |

## Troubleshooting method learned

When diagnosing a network issue, test from the lower layers toward the application/service:

1. **Interface and addressing:** `ip -brief address`
2. **Local gateway:** `ping -c 4 <gateway-ip>`
3. **External IP connectivity:** `ping -c 4 <known-external-ip>`
4. **DNS and hostname resolution:** `resolvectl status`, `resolvectl query <hostname>`
5. **TCP service reachability:** `nc -vz -w 5 <hostname-or-ip> <port>`
6. **Local service reachability:** test loopback and the machine's own interface address separately.

A failure at one step narrows the investigation, but a single test rarely proves the complete root cause. ICMP may be filtered, DNS may use multiple servers, and a TCP port test does not validate the application protocol or authentication.

## Security notes

- Do not assume that a service is publicly exposed merely because it listens on `0.0.0.0` or accepts a local connection. Confirm the network mode, routing, forwarding and firewall policy.
- Do not change firewall rules, SSH settings, or VirtualBox adapter mode as a troubleshooting guess. First identify the problem and understand the effect of a proposed change.
- Bridged networking can make a VM directly addressable on the LAN, depending on local network and firewall policy. It can increase the VM's exposure compared with the current NAT setup.
- A successful Netcat test only demonstrates that a TCP connection was established. It does not authenticate, exchange application data, or validate TLS.
- A timeout and a connection refusal are different observations. Neither one alone proves the exact remote-side cause.

## Expected examples for future troubleshooting (not observed in this lab)

The following are hypothetical examples only. They are not results from this VM:

- If an external IP ping succeeds but a hostname lookup fails, investigate DNS configuration, DNS-server reachability, and resolver logs.
- If the interface is down or has no expected address, investigate the interface and address configuration before testing remote services.
- If a gateway ping fails, inspect the interface, subnet, route and local virtual/network path. Remember that some gateways may filter ICMP, so corroborate with other tests.
- If a TCP port test times out, investigate routing and filtering as well as whether the service is available.
- If a TCP port test is refused, investigate whether a service is listening and whether a firewall actively rejects the connection.

## Cleanup

No temporary files, users, services, firewall rules, or network settings were created or changed during this exercise. No cleanup command is required.

## Reproducing the checks

The commands used in this lab can be rerun safely as diagnostic checks:

```bash
ip -brief address
ping -c 4 10.0.2.2
ping -c 4 1.1.1.1
ping -c 4 example.com
resolvectl status
resolvectl query example.com
nc -vz -w 5 example.com 443
nc -vz -w 5 example.com 22
nc -vz -w 5 127.0.0.1 22
nc -vz -w 5 10.0.2.15 22
```

Results may differ later because DNS answers, routing, remote service policies, and network conditions can change.

## GitHub portfolio instructions

Suggested repository path:

```text
docs/networking/linux-network-troubleshooting.md
```

From the Windows PowerShell terminal, in the local repository:

```powershell
cd C:\Users\Contessina\cybersecurity-home-lab
git status
```

After placing this document at the suggested path, review the file and stage only this document:

```powershell
git add docs/networking/linux-network-troubleshooting.md
git status
git commit -m "Document Linux network troubleshooting lab"
git push origin main
```

Run each Git command separately and inspect its output before proceeding. Do not assume a commit or push succeeded until Git confirms it.

---

**Lab status:** Practical troubleshooting checks completed. Knowledge assessment: 4/4 correct.  
**Evidence standard:** Sections labelled “Actual observed output” reproduce the terminal output supplied during the exercise. Hypothetical outcomes are kept separate under “Expected examples”.
