# Evidence index

Screenshots and captures for the site-to-site IPsec VPN lab. File names are numbered in the order of the story: baseline, VPN verification, packet capture, then troubleshooting.

## 01-baseline (before the VPN)

| File | Command / action | Shows |
|---|---|---|
| 01-hq-interfaces.png | `show ip interface brief` on HQ-R1 | HQ addressing, interfaces up |
| 02-isp-interfaces.png | `show ip interface brief` on ISP | ISP links and Loopback0 (8.8.8.8) |
| 03-br-interfaces.png | `show ip interface brief` on BR-R2 | Branch addressing |
| 04-pc1-ip.png | IP and route on PC1 | Host IP and default gateway |
| 05-pc2-ip.png | IP and route on PC2 | Host IP and default gateway |
| 06-hq-route-table.png | `show ip route` on HQ-R1 | Default route to the ISP |
| 07-isp-route-table.png | `show ip route` on ISP | No routes to the private LANs |
| 08-pc1-ping-internet.png | PC1 pings 8.8.8.8 | Internet access through PAT |
| 09-hq-nat-translations.png | `show ip nat translations` on HQ-R1 | PAT translating PC1 traffic |
| 10-hq-ping-br-wan.png | HQ-R1 pings 198.51.100.2 | WAN path works via the ISP |

## 02-vpn-verification (after the VPN)

| File | Command / action | Shows |
|---|---|---|
| 01-pc1-ping-pc2-success.png | PC1 pings PC2 | LAN-to-LAN traffic works through the tunnel |
| 02-isakmp-sa-hq.png | `show crypto isakmp sa` on HQ-R1 | Phase 1 `QM_IDLE` / `ACTIVE` |
| 03-isakmp-sa-br.png | `show crypto isakmp sa` on BR-R2 | Same from the branch side |
| 04-ipsec-sa-hq.png | `show crypto ipsec sa` on HQ-R1 | Phase 2 SAs, encaps/decaps counters |
| 05-ipsec-sa-br.png | `show crypto ipsec sa` on BR-R2 | Mirror counters |
| 06-crypto-map-hq.png | `show crypto map` on HQ-R1 | Map `CMAP`, peer, ACL, interface |
| 07-crypto-acl.png | `show access-lists VPN-TRAFFIC` | ACL match counters |
| 08-isakmp-policy.png | `show crypto isakmp policy` | Phase 1 policy parameters |
| 09-traceroute.png | `traceroute` from PC1 to PC2 | ISP not visible as a hop |
| 10-internet-still-works.png | PC1 pings 8.8.8.8 | NAT exemption does not break internet access |
| 11-nat-translations-after.png | `show ip nat translations` on HQ-R1 | Only internet traffic is translated |

## 03-packet-capture

| File | Shows |
|---|---|
| 01-esp-capture.png | Capture on the HQ-R1 to ISP link: ISAKMP negotiation, then ESP |
| 02-esp-packet-detail.png | An ESP packet expanded: payload not readable |
| 03-plain-icmp-capture.png | Plain ICMP to 8.8.8.8 for comparison |
| vpn-esp.pcap | Raw capture of the VPN traffic (open in Wireshark) |
| internet-icmp.pcap | Raw capture of the internet traffic |

## 04-troubleshooting

Reserved for fault-injection evidence, one subfolder per fault. To be added.
