# 🐧 Linux + Docker Networking Notes (Today's Learning)

These are the core networking concepts

---

## ✅ 1. Container = Just a Linux Process (Not a VM)

- A container is not a virtual machine
- It is a **process** running in **isolated namespaces**
- Docker does not virtualize hardware

```
Container = Process + Namespaces + Cgroups + OverlayFS + Capabilities
```

---

## ✅ 2. Host Namespace vs Container Namespace

| Area     | Host Namespace              | Container Namespace          |
|----------|-----------------------------|------------------------------|
| PIDs     | sees all processes          | sees only its own PIDs       |
| Network  | sees host NICs, bridges     | sees only its eth0           |
| FS       | full filesystem             | isolated root filesystem     |
| Users    | real root                   | root mapped (not real root)  |

---

## ✅ 3. Virtual Ethernet (veth) = Virtual Cable

- Real world: NIC ↔ Ethernet Cable ↔ Switch
- Linux world: eth0 ↔ **veth pair** ↔ docker0 bridge
- A veth pair = a **virtual Ethernet cable**

---

## ✅ 4. docker0 = Virtual Switch

- Acts like a Layer-2 Linux bridge
- All container NICs plug into it
- Provides intra-container communication (same subnet)
- Default subnet: `172.17.0.0/16`

---

## ✅ 5. Outbound Traffic = SNAT / MASQUERADE

```
Container IP (172.x) → NAT → Host IP (public) → Internet
```

iptables:
```
POSTROUTING / MASQUERADE
```

Outbound `.md` diagram:  
`docker_network_outbound.md`

---

## ✅ 6. Inbound Traffic (Port Mapping) = DNAT

```
Host:80 → DNAT → Container:80
```

iptables:
```
PREROUTING / DNAT
```

Inbound `.md` diagram:  
`docker_network_inbound.md`

---

## ✅ 7. Multiple Containers on Same Bridge

- Each gets its own veth pair
- All share docker0
- They can talk to each other directly (L2 switching)
- Only internet traffic is NATed, not container-to-container traffic

---

## ✅ 8. Why Port Publishing Exists

Because container lives in **isolated namespace**, it cannot bind `host:80`.

Docker solves it via DNAT in PREROUTING.

---

Next:  
- inbound + outbound combined doc
- host network vs bridge network
- custom docker networks (frontend/backend isolation)
- DNS inside containers
- manual namespace creation via `ip netns`

## Docker Bridge Networking + NAT (Outbound Flow)

```
                                INTERNET
                                    ▲
                                    │  (public traffic)
                                    │
                             +----------------+
                             |   Host eth0    |   ← Real NIC (public IP / LAN IP)
                             +----------------+
                                      ▲
                           (MASQUERADE / SNAT by iptables)
                                      │
                              +------------------+
                              |   NAT TABLE      |   ← packet rewrite (container IP → host IP)
                              +------------------+
                                      ▲
                                      │
                       +----------------------------------+
                       |            docker0              |   ← Linux bridge (switch)
                       +----------------------------------+
                           │       │       │       │        │
                        vethA   vethB   vethC   vethD    vethE   ← host-side of veth pairs
                           │       │       │       │        │
         -------------------------------------------------------------------
         |                |           |           |           |            |
         |                |           |           |           |            |
   +-------------+ +-------------+ +-------------+ +-------------+ +-------------+
   | eth0 (C1)   | | eth0 (C2)   | | eth0 (C3)   | | eth0 (C4)   | | eth0 (C5)   |
   | 172.17.0.2  | | 172.17.0.3  | | 172.17.0.4  | | 172.17.0.5  | | 172.17.0.6  |
   +-------------+ +-------------+ +-------------+ +-------------+ +-------------+
```

## 🔄 Docker Port Mapping (Inbound Traffic) – DNAT Flow

When you run:  
`docker run -p 80:80 nginx`

Docker installs a **DNAT rule** in the `PREROUTING` chain, which forwards host:80 → container:80.

```
 Client (Browser)
        |
        |   HTTP Request (Host_Public_IP:80)
        v
+-------------------+
|    Host eth0      |   ← Real NIC receiving packet
+-------------------+
        |
        v
+--------------------------------+
| NAT TABLE (PREROUTING / DNAT)  |
| "dest:80 → 172.17.0.2:80"      |
+--------------------------------+
        |
        v
+------------------------------+
|     docker0 (bridge)         |  ← Linux software switch
+------------------------------+
        |
      vethX (host-side veth)
        |
      vethY (container-side)
        |
+------------------------------+
|  Container eth0:80           |
|  172.17.0.2 (nginx)          |
+------------------------------+
        |
  (Response generated)
        |
   Packet travels BACK the same path
   (DNAT undone → SNAT → delivered to client)
```

### ✅ Key iptables Location

Outbound: POSTROUTING (SNAT/MASQUERADE)  
Inbound:  PREROUTING  (DNAT/port forwarding)

You can verify with:
```
sudo iptables -t nat -L PREROUTING -n --line-numbers
```
