# Containerized Zero-Trust VPN Gateway & Traffic Inspection Lab

Two isolated Docker sites (`site_a_net`, `site_b_net`) joined only by an encrypted WireGuard tunnel.
The gateway enforces stateful iptables rules and drops cleartext Telnet through the tunnel.

![Topology](images/01-topology.png)

## Stack
Docker Compose, WireGuard (linuxserver/wireguard), iptables, tcpdump, Kali Linux

## Network design
| Network | Subnet | Members |
|---|---|---|
| site_a_net | 172.28.1.0/24 | client-a, vpn-gateway-a |
| site_b_net | 172.28.2.0/24 | client-b, vpn-peer-b |
| transit_net | 172.28.3.0/24 | vpn-gateway-a, vpn-peer-b (WireGuard UDP/51820 only) |

`transit_net` keeps the two sites from having any direct Docker link, so no traffic can bypass the tunnel.

## Proof

### 1. Lab deployed
![compose up](images/02-compose-up.png)

### 2. Gateway wg0 up
![gateway](images/03-gateway-wg0-up.png)

### 3. Peer B config (keys redacted)
![peer config](images/04-peer-config-redacted.png)

### 4. Tunnel established (latest handshake)
![handshake](images/05-wg-handshake.png)

### 5. Cross-site connectivity (0% loss)
![ping](images/06-cross-site-ping.png)

### 6. Encryption proof
Outside the tunnel (eth1): only UDP/51820, unreadable payload.

![encrypted](images/07-encrypted-udp-51820.png)

Inside the tunnel (wg0): plaintext ICMP.

![plaintext](images/08-wg0-plaintext-icmp.png)

### 7. Zero-trust iptables rules
![rules](images/09-iptables-rules.png)

### 8. Telnet blocked (DROP counter 0 to 3 packets)
![telnet](images/10-telnet-blocked.png)

## Results
| Test | Result |
|---|---|
| WireGuard handshake on both ends | PASS |
| Tunnel ping 10.13.13.1 (0% loss) | PASS |
| client-a and client-b connectivity | PASS |
| Only UDP/51820 visible on the wire | PASS |
| Stateful ESTABLISHED rule matched | PASS |
| Telnet (tcp/23) dropped, port 8080 allowed | PASS |

## Limitations
- Reverse-direction Telnet rule (`-o wg0`) was added but not traffic-tested.
- iptables rules are runtime only; persist them with `PostUp` in `wg0.conf`.
- Lab keys were rotated and deleted after testing. Never commit `wg_config_*` folders.

## .gitignore
```
wg_config_a/
wg_config_b/
```
