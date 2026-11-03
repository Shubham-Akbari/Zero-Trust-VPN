# 🔐 Zero Trust Site-to-Site VPN using WireGuard & Docker

<p align="center">
<img src="https://img.shields.io/badge/Platform-Kali%20Linux-blue?logo=kalilinux" />
<img src="https://img.shields.io/badge/VPN-WireGuard-green?logo=wireguard" />
<img src="https://img.shields.io/badge/Containers-Docker-blue?logo=docker" />
<img src="https://img.shields.io/badge/Status-Lab%20Project-success" />
</p>

## 📌 Project Overview

This proof of concept (PoC) builds a **Site-to-Site VPN** with **WireGuard** and **Docker**.

Two isolated private networks (**Site A** and **Site B**) are joined by an encrypted WireGuard tunnel between two gateway containers. Devices in both networks can talk to each other without exposing their internal services directly to the internet. A **default-deny firewall** on each gateway applies the Zero Trust idea: nothing is forwarded unless a rule explicitly allows it.

## 🎯 Project Objectives

- 🔐 Establish a secure WireGuard VPN tunnel.
- 🐳 Deploy the VPN gateways as Docker containers.
- 🌐 Connect two isolated Docker networks.
- 🔄 Enable communication between Site A and Site B.
- 🛡️ Configure IP forwarding and default-deny firewall rules.
- 📡 Test private network connectivity and inspect the encrypted traffic.

## 🏗️ Network Architecture

```text
                    SITE-TO-SITE VPN

        SITE A                              SITE B
    172.28.1.0/24                       172.28.2.0/24
          |                                   |
          |                                   |
   VPN Gateway A                       VPN Gateway B
   LAN  172.28.1.254                   LAN  172.28.2.254
   VPN  10.13.13.1                     VPN  10.14.14.1
   Link 172.18.0.2                     Link 172.18.0.3
          |                                   |
          +-------- WireGuard tunnel ---------+
                 UDP 51820 over the
              transport network 172.18.0.0/16
          |                                   |
   Client A (optional)                  Client B (Alpine)
   172.28.1.10                          172.28.2.10
```

> Client A is optional. The lab traffic is verified mainly with Client B at Site B, and a Site A host at `172.28.1.10` is used for the reverse test.

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| Kali Linux | Host operating system (VirtualBox VM) |
| Docker | Containerization |
| Docker Compose | Container orchestration |
| WireGuard | VPN tunnel |
| iptables | Firewall, forwarding rules and NAT |
| Alpine Linux | Test client |
| tcpdump | Packet capture |

## 📋 IP Address Configuration

| Component | IP Address |
|---|---|
| Site A network | `172.28.1.0/24` |
| Site B network | `172.28.2.0/24` |
| VPN transport network | `172.18.0.0/16` |
| Gateway A (LAN) | `172.28.1.254` |
| Gateway B (LAN) | `172.28.2.254` |
| Gateway A (transport) | `172.18.0.2` |
| Gateway B (transport) | `172.18.0.3` |
| Gateway A VPN IP | `10.13.13.1` |
| Gateway B VPN IP | `10.14.14.1` |
| Site B client | `172.28.2.10` |

---

## 📦 Step 1: Install Required Packages

```bash
sudo apt update
sudo apt install docker.io docker-compose tcpdump -y
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Check the versions:

```bash
sudo docker --version
sudo docker-compose --version
```

*Docker, Docker Compose and tcpdump were already at the newest version, and the Docker service is enabled at boot.*

## 📁 Step 2: Create the Project Directory

```bash
mkdir -p ~/zero-trust-vpn
cd ~/zero-trust-vpn
mkdir -p wg_config_a wg_config_b
```

Project structure:

```text
zero-trust-vpn/
│
├── docker-compose.yml
│
├── wg_config_a/
│   └── wg_confs/
│       └── wg0.conf        (generated on first start)
│
├── wg_config_b/
│   └── wg_confs/
│       └── wg0.conf        (generated on first start)
│
```

## ⚙️ Step 3: Configure Docker Compose

```bash
nano docker-compose.yml
```

```yaml
services:

  vpn_gateway_a:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: vpn-gateway-a

    cap_add:
      - NET_ADMIN

    sysctls:
      net.ipv4.ip_forward: "1"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0

    volumes:
      - ./wg_config_a:/config

    networks:
      site_a_net:
        ipv4_address: 172.28.1.254
      vpn_transport_net:
        ipv4_address: 172.18.0.2

    restart: unless-stopped

  vpn_gateway_b:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: vpn-gateway-b

    cap_add:
      - NET_ADMIN

    sysctls:
      net.ipv4.ip_forward: "1"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.14.14.0

    volumes:
      - ./wg_config_b:/config

    networks:
      site_b_net:
        ipv4_address: 172.28.2.254
      vpn_transport_net:
        ipv4_address: 172.18.0.3

    restart: unless-stopped

  site_b_client:
    image: alpine
    container_name: site-b-client
    command: sleep infinity

    networks:
      site_b_net:
        ipv4_address: 172.28.2.10

networks:

  site_a_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.1.0/24

  site_b_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.2.0/24

  vpn_transport_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16
```

*Each gateway has the `NET_ADMIN` capability and sits on its own site network plus the shared transport network, each with a fixed IP.*

## 🚀 Step 4: Start the Containers

```bash
sudo docker-compose up -d
```

The first start makes the LinuxServer WireGuard image generate its keys and `wg0.conf` for each gateway.

Check the logs:

```bash
sudo docker logs vpn-gateway-a
sudo docker logs vpn-gateway-b
```

*All 6 resources started: both gateways, the Site B client and the three networks.*

*Gateway B logs show the LinuxServer.io WireGuard container initialising.*

> **Important:** Do not overwrite the generated private keys. Only add the `[Peer]` block described below.

## 🔐 Step 5: Configure Site-to-Site Peers

Read each gateway's **public** key:

```bash
sudo cat wg_config_a/server/publickey-server
sudo cat wg_config_b/server/publickey-server
```

Edit Gateway A:

```bash
sudo nano wg_config_a/wg_confs/wg0.conf
```

Add (replace the placeholder with Gateway B's real public key):

```ini
[Peer]
PublicKey = GATEWAY_B_PUBLIC_KEY
AllowedIPs = 10.14.14.1/32, 172.28.2.0/24
Endpoint = 172.18.0.3:51820
PersistentKeepalive = 25
```

Edit Gateway B:

```bash
sudo nano wg_config_b/wg_confs/wg0.conf
```

Add (replace the placeholder with Gateway A's real public key):

```ini
[Peer]
PublicKey = GATEWAY_A_PUBLIC_KEY
AllowedIPs = 10.13.13.1/32, 172.28.1.0/24
Endpoint = 172.18.0.2:51820
PersistentKeepalive = 25
```

Restart both gateways:

```bash
sudo docker-compose restart vpn_gateway_a vpn_gateway_b
```

*Public keys are read, `wg0.conf` is edited and the gateways are restarted.*

### 🧩 Troubleshooting seen in this lab

| Message | Cause | Fix |
|---|---|---|
| `Key is not the correct length or format: GATEWAY_B_PUBLIC_KEY` | The placeholder was left in `wg0.conf` | Replace it with the real public key |
| `wg-quick: 'wg0' is not a WireGuard interface` (on `wg-quick down`) | The interface was not up yet | Safe to ignore, then run `wg-quick up wg0` |

After fixing the keys, bring the tunnel up again if needed:

```bash
sudo docker exec vpn-gateway-a wg-quick down wg0
sudo docker exec vpn-gateway-a wg-quick up wg0
sudo docker exec vpn-gateway-b wg-quick down wg0
sudo docker exec vpn-gateway-b wg-quick up wg0
```

## 🛡️ Step 6: Default-Deny Firewall (Zero Trust Policy)

Block all forwarding by default, then allow only what is needed.

**Gateway A**

```bash
sudo docker exec vpn-gateway-a iptables -P FORWARD DROP
sudo docker exec vpn-gateway-a iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo docker exec vpn-gateway-a iptables -A FORWARD -i wg0 -o eth+ -s 10.14.14.0/24 -d 172.28.1.0/24 -j ACCEPT
```

**Gateway B**

```bash
sudo docker exec vpn-gateway-b iptables -P FORWARD DROP
sudo docker exec vpn-gateway-b iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo docker exec vpn-gateway-b iptables -A FORWARD -i wg0 -o eth+ -s 10.13.13.0/24 -d 172.28.2.0/24 -j ACCEPT
```

*`FORWARD` defaults to `DROP`. Only established traffic and the peer's VPN subnet reaching the local site subnet are accepted.*

> ⚠️ The `PostUp` lines that the LinuxServer image can generate may contain `iptables -A FORWARD -i %i -j ACCEPT`. That accepts all forwarded traffic and would weaken this policy. Review `PostUp` if you want strict default-deny.

### 💾 Persistence

Rules added with `docker exec` live only inside the running container and are lost when the container is recreated. The lab also installs `iptables-persistent` on the Kali host:

```bash
sudo apt install iptables-persistent -y
```

To make the gateway rules permanent, move them into the `PostUp` line of each `wg0.conf` (not tested in this lab).

## 🖥️ Step 7: Prepare the Site B Test Client

```bash
sudo docker exec site-b-client apk update
sudo docker exec site-b-client apk add iputils curl
```

*`iputils` and `curl` are added so the Alpine client can run ping and HTTP tests.*

## 📡 Step 8: Test VPN Connectivity

From Gateway A:

```bash
sudo docker exec vpn-gateway-a ping -c 4 172.28.2.10
sudo docker exec vpn-gateway-a ping -c 4 10.14.14.1
```

From Gateway B:

```bash
sudo docker exec vpn-gateway-b ping -c 4 10.13.13.1
sudo docker exec vpn-gateway-b ping -c 4 172.28.1.10
```

*Gateway A reaches the Site B client and Gateway B's VPN IP with 0% packet loss. TTL 63 shows the client sits one routed hop behind Gateway B.*

*Gateway B also reaches the Site A host `172.28.1.10`: 4 of 4 replies, 0% loss.*

*A longer `ping -c 10` run from Gateway A to `172.28.2.10` keeps receiving replies.*

### Result summary

| Test | Result |
|---|---|
| Gateway A → `172.28.2.10` | 4/4 replies, 0% loss, avg ≈ 3.2 ms |
| Gateway A → `10.14.14.1` | 4/4 replies, 0% loss, avg ≈ 2.6 ms |
| Gateway B → `10.13.13.1` | 4/4 replies, 0% loss, avg ≈ 4.5 ms |
| Gateway B → `172.28.1.10` | 4/4 replies, 0% loss, avg ≈ 1.9 ms |

## 🔍 Step 9: Verify Encrypted Traffic with tcpdump

```bash
sudo tcpdump -ni any udp port 51820
```

While pings are running, only UDP traffic between the two gateways appears on the transport network:

| UDP length | Meaning |
|---|---|
| 148 bytes | WireGuard handshake initiation |
| 92 bytes | WireGuard handshake response |
| 32 bytes | Keepalive |
| 128 bytes | Encrypted data (carrying the ping packets) |

The ping contents are not readable on the transport network, which confirms the traffic is encrypted inside the tunnel.

You can also check the tunnel state on each gateway:

```bash
sudo docker exec vpn-gateway-a wg show
sudo docker exec vpn-gateway-b wg show
```

A recent handshake and increasing transfer counters show that traffic is passing through the tunnel.

---

## 🔒 Security Best Practices

- Never upload private keys or preshared keys to GitHub.
- Only share public keys. Check every screenshot before publishing.
- Keep the firewall on `FORWARD DROP` and allow only required flows.
- Use dedicated VPN subnets for each gateway.
- Update Docker images regularly.
- Restrict access to management interfaces.

## 🎓 Learning Outcomes

- WireGuard configuration: keys, peers, `AllowedIPs`, endpoints and keepalive.
- Docker bridge networks and fixed container IPs.
- Site-to-Site VPN architecture.
- Linux IP forwarding, NAT and default-deny firewall rules.
- Troubleshooting with `ping`, `wg`, `iptables` and `tcpdump`.

## 🚀 Future Improvements

- Add a dedicated Site A test client.
- Move firewall rules into `PostUp` so they persist across restarts.
- Allow specific services and ports instead of whole subnets.
- Add centralized logging and traffic monitoring.
- Connect additional remote sites.

## 👨‍💻 Author

Cybersecurity / Network Security Lab Project

Built using Kali Linux, Docker and WireGuard.

⭐ If you find this project useful, consider giving this repository a star!

`#CyberSecurity` `#WireGuard` `#Docker` `#VPN` `#ZeroTrust` `#NetworkSecurity` `#KaliLinux`
