# Homelab Kubernetes over KVM/Wi-Fi: Network & Gateway Architecture

## 1. Overview & Problem Statement
Standard Linux bridging (`br0`) directly on Wi-Fi client interfaces (`wlp2s0`) is prohibited by the IEEE 802.11 standard due to 3-address frame limitations (the AP refuses frames with source MAC addresses not negotiated during authentication). 

To run a multi-node Kubernetes cluster inside KVM/libvirt VMs on an Ubuntu host connected solely via Wi-Fi, the cluster runs on an isolated virtual subnet (`192.168.122.0/24`) attached to `virbr0`. The host serves as a dual L3 router and transparent gateway, utilizing destination NAT (DNAT), source NAT (SNAT/MASQUERADE), and packet forwarding.

```
+-----------------------------------------------------------------------------------+
| PHYSICAL LAN (192.168.1.0/24)                                                     |
| Router / Gateway: 192.168.1.1                                                     |
|                                                                                   |
|  [ Mobile / LAN Client ]  --(DNS: 53 UDP/TCP)--> [ Host IP: 192.168.1.151 ]        |
|                           --(HTTP/S: 80/443)---> [ (wlp2s0 Interface)     ]        |
+---------------------------------------------------------|-------------------------+
                                                          | (iptables DNAT / FORWARD)
+---------------------------------------------------------v-------------------------+
| KVM / LIBVIRT HOST (virbr0: 192.168.122.1)                                        |
|                                                                                   |
|  OUTPUT Chain: Catches host-local browser/DNS traffic                              |
|  PREROUTING Chain: Catches incoming LAN traffic                                   |
|  FORWARD Chain: Bypasses default libvirt bridge isolation filters                 |
+---------------------------------------------------------|-------------------------+
                                                          |
+---------------------------------------------------------v-------------------------+
| VIRTUAL CLUSTER SUBNET (192.168.122.0/24)                                         |
|                                                                                   |
|  - Control Plane: 192.168.122.10                                                  |
|  - Worker Nodes:  192.168.122.11, 192.168.122.12                                 |
|  - MetalLB Pool:  192.168.122.200 - 192.168.122.210 (Layer 2 Mode)               |
|                                                                                   |
|  Ingress & Service Endpoints:                                                     |
|    * Traefik Ingress LB:  192.168.122.200 (Ports 80, 443)                         |
|    * AdGuard Home LB:     192.168.122.201 (Port 53 UDP/TCP)                       |
|    * CoreDNS ClusterIP:   10.43.0.10                                              |
+-----------------------------------------------------------------------------------+
```

---

## 2. Network Specifications

| Parameter | Value / Allocation | Description |
| :--- | :--- | :--- |
| **Physical LAN Subnet** | `192.168.1.0/24` | Primary home network |
| **LAN Gateway** | `192.168.1.1` | Physical router / DHCP upstream |
| **Host Wi-Fi Interface** | `wlp2s0` (`192.168.1.151`) | Ingress gateway for all LAN devices |
| **KVM Bridge Interface** | `virbr0` (`192.168.122.1`) | Default libvirt NAT network |
| **Cluster Node IPs** | `192.168.122.10 - 12` | K3s / Kube VMs (Master & Workers) |
| **MetalLB IP Pool** | `192.168.122.200 - 210` | L2 mode virtual load balancer IPs |
| **Traefik Ingress VIP** | `192.168.122.200` | HTTP (80), HTTPS (443) endpoint |
| **AdGuard Home VIP** | `192.168.122.201` | DNS (53 UDP/TCP) authoritative & sinkhole |
| **Internal CoreDNS** | `10.43.0.10` | Kubernetes cluster DNS |

---

## 3. End-to-End Traffic Flow Logic

### A. Resolution Phase (`https://adguard.dc.home`)
1. **Client Query:** Client sends a DNS request for `adguard.dc.home` to `192.168.1.151:53`.
2. **NAT Redirection:** Host NAT table redirects port 53 traffic to AdGuard at `192.168.122.201:53`.
3. **AdGuard Rewrite:** AdGuard resolves `*.dc.home` to `192.168.1.151` and responds to the client.

### B. Ingress HTTP/S Phase
1. **Client HTTP/S Request:** Client makes a TLS connection to `https://192.168.1.151:443` with SNI/Host header `adguard.dc.home`.
2. **Host DNAT & Filter:**
   - External clients hit the `PREROUTING` chain $
ightarrow$ DNAT to `192.168.122.200:443`.
   - Host-local processes hit the `OUTPUT` chain $
ightarrow$ DNAT to `192.168.122.200:443`.
   - The `FORWARD` chain allows LAN packets to traverse into `virbr0`.
3. **Traefik Ingress Routing:** Traefik receives the request at `192.168.122.200`, matches the `Host(`adguard.dc.home`)` rule, terminates TLS using the `cert-manager` secret, and proxies traffic to the backend AdGuard Web UI service.

---

## 4. Host System Configuration

### 4.1. Enable Kernel IP Forwarding
Ensure the host kernel forwards packets across interfaces:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-kubernetes-nat.conf
```

### 4.2. Resolve Host Port 53 Conflicts (`systemd-resolved`)
Ubuntu's default `systemd-resolved` binds to `127.0.0.53:53` or intercepts outbound DNS queries. To ensure clean forwarding and prevent local resolver pollution:

1. Edit `/etc/systemd/resolved.conf`:
   ```ini
   [Resolve]
   DNS=192.168.122.201
   DNSStubListener=no
   ```
2. Restart and unlink the stub resolver:
   ```bash
   sudo systemctl restart systemd-resolved
   sudo rm -f /etc/resolv.conf
   echo "nameserver 192.168.1.151" | sudo tee /etc/resolv.conf
   ```

---

## 5. Complete iptables Routing & Firewall Script

Save this script as `/usr/local/bin/apply-k8s-nat.sh`:

```bash
#!/usr/bin/env bash
# ==============================================================================
# Script: apply-k8s-nat.sh
# Purpose: Configure iptables NAT & Filter rules for Wi-Fi KVM Kubernetes Cluster
# Target Host: Ubuntu Linux (192.168.1.151)
# ==============================================================================

set -euo pipefail

HOST_IP="192.168.1.151"
TRAEFIK_IP="192.168.122.200"
ADGUARD_IP="192.168.122.201"
WIFI_IFACE="wlp2s0"
BRIDGE_IFACE="virbr0"
K8S_SUBNET="192.168.122.0/24"

echo "[*] Configuring iptables rules for Kubernetes NAT Gateway..."

# ------------------------------------------------------------------------------
# 1. PREROUTING: External LAN -> Host IP -> VM Endpoints
# ------------------------------------------------------------------------------
iptables -t nat -C PREROUTING -i "$WIFI_IFACE" -p tcp --dport 80 -j DNAT --to-destination "$TRAEFIK_IP:80" 2>/dev/null || iptables -t nat -A PREROUTING -i "$WIFI_IFACE" -p tcp --dport 80 -j DNAT --to-destination "$TRAEFIK_IP:80"

iptables -t nat -C PREROUTING -i "$WIFI_IFACE" -p tcp --dport 443 -j DNAT --to-destination "$TRAEFIK_IP:443" 2>/dev/null || iptables -t nat -A PREROUTING -i "$WIFI_IFACE" -p tcp --dport 443 -j DNAT --to-destination "$TRAEFIK_IP:443"

iptables -t nat -C PREROUTING -i "$WIFI_IFACE" -p udp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53" 2>/dev/null || iptables -t nat -A PREROUTING -i "$WIFI_IFACE" -p udp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53"

iptables -t nat -C PREROUTING -i "$WIFI_IFACE" -p tcp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53" 2>/dev/null || iptables -t nat -A PREROUTING -i "$WIFI_IFACE" -p tcp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53"

# ------------------------------------------------------------------------------
# 2. OUTPUT: Local Host Browser / CLI -> Host IP -> VM Endpoints
# ------------------------------------------------------------------------------
iptables -t nat -C OUTPUT -d "$HOST_IP" -p tcp --dport 80 -j DNAT --to-destination "$TRAEFIK_IP:80" 2>/dev/null || iptables -t nat -A OUTPUT -d "$HOST_IP" -p tcp --dport 80 -j DNAT --to-destination "$TRAEFIK_IP:80"

iptables -t nat -C OUTPUT -d "$HOST_IP" -p tcp --dport 443 -j DNAT --to-destination "$TRAEFIK_IP:443" 2>/dev/null || iptables -t nat -A OUTPUT -d "$HOST_IP" -p tcp --dport 443 -j DNAT --to-destination "$TRAEFIK_IP:443"

iptables -t nat -C OUTPUT -d "$HOST_IP" -p udp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53" 2>/dev/null || iptables -t nat -A OUTPUT -d "$HOST_IP" -p udp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53"

iptables -t nat -C OUTPUT -d "$HOST_IP" -p tcp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53" 2>/dev/null || iptables -t nat -A OUTPUT -d "$HOST_IP" -p tcp --dport 53 -j DNAT --to-destination "$ADGUARD_IP:53"

# ------------------------------------------------------------------------------
# 3. FORWARD: Bypass libvirt bridge isolation for permitted ports
# ------------------------------------------------------------------------------
iptables -C FORWARD -d "$ADGUARD_IP" -p tcp --dport 53 -j ACCEPT 2>/dev/null || iptables -I FORWARD 1 -d "$ADGUARD_IP" -p tcp --dport 53 -j ACCEPT

iptables -C FORWARD -d "$ADGUARD_IP" -p udp --dport 53 -j ACCEPT 2>/dev/null || iptables -I FORWARD 1 -d "$ADGUARD_IP" -p udp --dport 53 -j ACCEPT

iptables -C FORWARD -d "$TRAEFIK_IP" -p tcp --dport 80 -j ACCEPT 2>/dev/null || iptables -I FORWARD 1 -d "$TRAEFIK_IP" -p tcp --dport 80 -j ACCEPT

iptables -C FORWARD -d "$TRAEFIK_IP" -p tcp --dport 443 -j ACCEPT 2>/dev/null || iptables -I FORWARD 1 -d "$TRAEFIK_IP" -p tcp --dport 443 -j ACCEPT

# ------------------------------------------------------------------------------
# 4. POSTROUTING: Masquerade traffic exiting to LAN and virtual bridge
# ------------------------------------------------------------------------------
iptables -t nat -C POSTROUTING -s "$K8S_SUBNET" -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -s "$K8S_SUBNET" -j MASQUERADE

iptables -t nat -C POSTROUTING -o "$BRIDGE_IFACE" -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o "$BRIDGE_IFACE" -j MASQUERADE

echo "[+] iptables configuration successfully applied."
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/apply-k8s-nat.sh
sudo /usr/local/bin/apply-k8s-nat.sh
```

---

## 6. Persistence across Reboots

To ensure rules persist after reboot or network interface cycling, install `iptables-persistent`:

```bash
sudo apt-get update
sudo apt-get install -y iptables-persistent netfilter-persistent
```

Save the active ruleset:
```bash
sudo netfilter-persistent save
```

Alternatively, create a systemd one-shot service:
```ini
# /etc/systemd/system/k8s-nat-routing.service
[Unit]
Description=Apply Kubernetes NAT & Forwarding Rules for KVM/Wi-Fi
After=network.target libvirtd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/apply-k8s-nat.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```
Enable the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now k8s-nat-routing.service
```

---

## 7. Client & Mobile Device Configuration

Modern mobile OS features (Apple iCloud Private Relay, Android Private DNS / DoH) actively bypass local DHCP DNS servers. Follow these settings for seamless access:

### Apple iOS / iPadOS
1. Open **Settings** $
ightarrow$ **Wi-Fi** $
ightarrow$ Tap **(i)** next to Home Wi-Fi.
2. Set **Limit IP Address Tracking** $
ightarrow$ **Off**.
3. Under **Configure DNS**, select **Manual**:
   - Remove existing entries.
   - Add: `192.168.1.151`.
4. If using iCloud+: **Settings** $
ightarrow$ **[Apple ID]** $
ightarrow$ **iCloud** $
ightarrow$ **Private Relay** $
ightarrow$ **Off** (or pause for home network).

### Android
1. Open **Settings** $
ightarrow$ **Network & Internet** $
ightarrow$ **Private DNS**.
2. Select **Off** (disables hardcoded DoT/DoH upstream resolvers).
3. Under Wi-Fi Details, verify the gateway/DNS assigned by DHCP is `192.168.1.151`.

---

## 8. Verification & Troubleshooting Runbook

| Test Objective | Command (from Host or LAN) | Expected Output | Root Cause if Failing |
| :--- | :--- | :--- | :--- |
| **AdGuard DNS Query** | `nslookup adguard.dc.home 192.168.1.151` | Address: `192.168.1.151` | Port 53 UDP missing in `PREROUTING`/`OUTPUT` or AdGuard rewrite missing. |
| **Traefik Raw IP Probe** | `curl -Iv http://192.168.1.151` | `HTTP/1.1 404 Not Found` (from Traefik) | Forward chain blocking port 80 or DNAT missing. |
| **Full Ingress HTTPS** | `curl -kv https://adguard.dc.home` | `HTTP/2 200` or `HTTP/1.1 200 OK` | Host header mismatch or cert-manager secret issue. |
| **Monitor Packet Hits** | `sudo iptables -t nat -L -n -v` | Packet count increments on DNAT rules | Traffic hitting wrong interface or dropped before NAT. |