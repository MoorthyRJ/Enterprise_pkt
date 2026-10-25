# Enterprise network in Cisco Packet Tracer

Two branch offices and a cloud services zone, joined by one ISP router. Every site has its own Cisco ASA firewall, and the two branches also have a site-to-site IPsec VPN between them.

I built this to practise subnetting, VLANs, DHCP, the different kinds of NAT, ACLs and IPsec in one project, instead of in separate small labs. I tried to make it behave like a small company network, so the access rules matter as much as the connectivity.

Built with Cisco Packet Tracer 9.0.0.

![Full topology](https://github.com/MoorthyRJ/Enterprise_pkt/blob/main/Photos/whole_Topology.png)

## What the network is supposed to do

- Both branches can reach the servers in the cloud zone.
- Each branch can reach the other branch's published servers, and nothing else there.
- The Server PC in each branch (50.0.0.100 and 60.0.0.100) can talk to the other branch over the IPsec VPN.
- Normal PCs in one branch cannot reach the PCs in the other branch.

## Layout

| Site | What is in it |
|---|---|
| Branch 1 | Router, ASA0, an office LAN (PC1 to PC3, Server3) and a server LAN (Server0, Server1, Server2, Server4, Server PC) |
| Branch 2 | Router, ASA2, two VLANs with office PCs, two servers and a Server PC |
| Cloud services | ASA1, three servers on 100.100.100.0/24, and a small client LAN behind its own router |
| ISP | One router with a link to each of the three ASAs |

## Addressing

| Where | Subnet | Notes |
|---|---|---|
| Branch 1 office LAN | 192.168.100.0/24 | Gateway 192.168.100.1 on the router |
| Branch 1 server LAN | 192.168.50.0/24 | Gateway 192.168.50.1 on the router |
| Branch 1 router to ASA0 | 50.0.0.0/24 | Router 50.0.0.1, ASA0 inside 50.0.0.2 |
| Branch 2 VLAN 10 | 200.200.100.0/24 | Gateway 200.200.100.1 |
| Branch 2 VLAN 20 | 200.200.200.0/24 | Gateway 200.200.200.1 |
| Branch 2 router to ASA2 | 60.0.0.0/24 | Router 60.0.0.1, ASA2 inside 60.0.0.2 |
| ASA0 to ISP | 20.0.0.0/30 | ASA0 is 20.0.0.1 |
| ASA2 to ISP | 20.0.1.0/30 | ASA2 is 20.0.1.1 |
| ASA1 to ISP | 20.0.2.0/30 | ASA1 is 20.0.2.1 |
| Cloud servers | 100.100.100.0/24 | Three servers, including 100.100.100.10 and 100.100.100.20 |

The 20.x, 50.x, 60.x, 100.x and 200.x ranges are public address space. I used them as lab addresses only, so please don't copy them into a real network.

## How a packet travels

I used two layers of NAT, one on the branch router and one on the ASA.

A normal PC in Branch 1 going to a cloud server:

1. The PC (192.168.x.x) sends the packet to the branch router.
2. The router uses PAT and changes the source to 50.0.0.1.
3. ASA0 uses PAT again and changes it to 20.0.0.1.
4. The ISP forwards it to ASA1, whose ACL allows 20.0.0.1.
5. The server answers and the reply follows the same path back.

A server behaves differently. The router has a static NAT entry for it (for example 192.168.50.50 to 50.0.0.50) and the ASA has an identity NAT entry, so it keeps the address 50.0.0.50 all the way through. That is why the cloud ASA allows 50.0.0.0/24 and 60.0.0.0/24 as well as the two ASA outside addresses.

A side effect I like: the ISP only needs three routes and never sees a 192.168.x.x or 200.200.x.x address.

## How each part is configured

### Branch routers

Branch 1 has three interfaces. Gi0/0 faces the ASA, Gi0/1 is the server LAN and Gi0/2 is the office LAN. DHCP runs on the router for each LAN.

```
interface GigabitEthernet0/0
 ip address 50.0.0.1 255.255.255.0
 ip nat outside
interface GigabitEthernet0/1
 ip address 192.168.50.1 255.255.255.0
 ip nat inside
interface GigabitEthernet0/2
 ip address 192.168.100.1 255.255.255.0
 ip nat inside
!
ip nat inside source list 10 interface GigabitEthernet0/0 overload
ip nat inside source static 192.168.50.100 50.0.0.100
ip route 0.0.0.0 0.0.0.0 50.0.0.2
access-list 10 permit 192.168.50.0 0.0.0.255
access-list 10 permit 192.168.100.0 0.0.0.255
```

There are six static NAT entries on this router, one for each server and the Server PC.

Branch 2 uses router-on-a-stick. Gi0/1 carries VLAN 10 and VLAN 20 as dot1Q subinterfaces, PAT uses ACL 1 (200.200.0.0/16), and there are three static NAT entries. The default route points at ASA2 (60.0.0.2).

### ASAs at the branches

ASA0 and ASA2 do three things. They PAT the whole inside network, they give every published server an identity NAT entry, and they have an outside ACL with one rule per service (FTP, HTTPS and RDP to the right servers).

```
object network INSIDE
 subnet 50.0.0.0 255.255.255.0
 nat (inside,outside) dynamic interface
object network server1
 host 50.0.0.10
 nat (inside,outside) static 50.0.0.10
```

Packet Tracer's ASA rejects the manual (twice) NAT command, so I used identity NAT on each host instead.

### Cloud ASA

ASA1 sits in front of the cloud servers. Its ACL only allows IP traffic from 20.0.0.1, 20.0.1.1, 50.0.0.0/24 and 60.0.0.0/24.

### IPsec VPN

The tunnel runs between ASA0 and ASA2. Phase 1 uses IKEv1 with AES, a pre-shared key and DH group 5. Phase 2 uses ESP with AES-256 and SHA. This is the ASA0 side, and ASA2 is the mirror image (peer 20.0.0.1 and a reversed crypto ACL).

```
crypto ikev1 enable outside
crypto ikev1 policy 10
 encr aes
 authentication pre-share
 group 5
crypto ipsec ikev1 transform-set TSET esp-aes 256 esp-sha-hmac
crypto map CMAP 10 match address L2L
crypto map CMAP 10 set peer 20.0.1.1
crypto map CMAP 10 set ikev1 transform-set TSET
crypto map CMAP interface outside
tunnel-group 20.0.1.1 type ipsec-l2l
tunnel-group 20.0.1.1 ipsec-attributes
 ikev1 pre-shared-key <pre-shared-key>
```

### ISP router

Only three static routes, one per site:

```
ip route 50.0.0.0 255.255.255.0 20.0.0.1
ip route 60.0.0.0 255.255.255.0 20.0.1.1
ip route 100.100.100.0 255.255.255.0 20.0.2.1
```

## Files in this repo

```
.
├── README.md
├── enterprise.pkt                  the Packet Tracer project
├── configs/
│   ├── branch1-router.txt
│   ├── branch2-router.txt
│   ├── asa0.txt
│   ├── asa1.txt
│   ├── asa2.txt
│   ├── isp-router.txt
│   └── cloud-router.txt
└── enterprise-network.pdf
└── photos/
    ├── topology.png
    ├── branch1.png
    ├── branch2.png
    ├── cloud.png
    └── ipsec-sa-proof.png
```

The snippets above are trimmed. The full running configs are in `configs/`.

## Opening the project

Open `enterprise.pkt` in Packet Tracer 9.0.0 or newer. All passwords and keys in the lab are dummy values, and in the text files in `configs/` I replaced them with placeholders.

## Checking it yourself

Useful commands on the ASAs:

```
show nat
show access-list
show route
show crypto ipsec sa
```

Things to try:

- Ping a cloud server (for example 100.100.100.10) from a normal PC in each branch.
- Ping 60.0.0.100 from the Branch 1 Server PC, then run `show crypto ipsec sa` on ASA0 and look at the encaps and decaps counters.
- Ping a normal PC in the other branch from a normal PC. This should fail.

Two tips for Packet Tracer. The first ping across a new path often times out because of ARP, so send a second one. And `show conn` does not exist on its ASA, so use the IPsec counters and Simulation mode to see what is happening.

## Problems I ran into

1. **Published servers were unreachable even though the ACL was right.** The ASA was still applying the PAT rule to their addresses. Giving each server its own identity NAT entry fixed it.
2. **VPN traffic reached the ASA and was dropped.** Packet Tracer checked the decrypted traffic against the outside ACL, so it needed an explicit permit.
3. **The first ping always failed.** It was ARP. Pinging the peer once before testing the tunnel got rid of it.
4. **object network group** Not available in Packet Tracker so creater a identity NAT for every IP
## What I would change in a real network

- Use private address space (RFC 1918) inside and a real public range only on the internet side.
- Use IKEv2 and a long random pre-shared key, or certificates.
- Tighten the outside ACLs so they allow specific hosts and ports, not whole IP ranges.
- Keep the configs in version control and push changes with a script.

## About

Made by `JEYA MOORTHY R`. If you see something I could do better, open an issue or message me on LinkedIn: `www.linkedin.com/in/jeyamoorthy-r`.
