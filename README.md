
# DNS Spoofing Forensics

**SBT-DF203 · Lab 8 · Digital Forensics Investigation Report**

![Course](https://img.shields.io/badge/course-SBT--DF203-1f6feb)
![Platform](https://img.shields.io/badge/platform-Kali%20Linux-557C94?logo=kalilinux&logoColor=white)
![Analysis](https://img.shields.io/badge/analysis-tshark-1679A7)
![Integrity](https://img.shields.io/badge/integrity-SHA--256-2ea44f)
![Environment](https://img.shields.io/badge/environment-isolated%20lab-orange)
![Purpose](https://img.shields.io/badge/purpose-educational-lightgrey)

A packet-level forensic investigation of a controlled DNS spoofing event in an isolated host-only network. The work detects forged DNS responses, correlates them with ARP poisoning evidence, confirms victim redirection, and verifies cleanup, with every evidence file hashed for integrity.

> **Authorised and educational use only.** All activity took place in an isolated lab under ICDFA authorisation, using the reserved training domain `portal.icdfa.test` and a harmless training page. Do not reproduce these techniques on systems you do not own or have written permission to test.

---

## Contents

1. [Executive Summary](#executive-summary)
2. [At a Glance](#at-a-glance)
3. [Case Information](#case-information)
4. [Scope and Authority](#scope-and-authority)
5. [Lab Environment](#lab-environment)
6. [Methodology](#methodology)
7. [Key Findings](#key-findings)
8. [Detection Indicators](#detection-indicators)
9. [Evidence Inventory](#evidence-inventory)
10. [Evidence Integrity](#evidence-integrity)
11. [Reproducing the Analysis](#reproducing-the-analysis)
12. [Cleanup and Verification](#cleanup-and-verification)
13. [Limitations](#limitations)
14. [Recommendations](#recommendations)
15. [Repository Structure](#repository-structure)
16. [Academic Integrity](#academic-integrity)
17. [Author and License](#author-and-license)

---

## Executive Summary

This laboratory exercise investigated a controlled DNS spoofing attack within an isolated host-only virtual network. The lab involved capturing baseline DNS and ARP traffic, executing an instructor-approved ARP poisoning and DNS spoofing simulation, analysing the resulting packet captures for forged DNS responses and ARP anomalies, correlating the spoofed DNS answer with the victim's subsequent HTTP connection, and verifying cleanup of all lab-created state.

The investigation confirmed that the victim machine received two conflicting DNS responses for the same transaction ID. The forged response arrived from the analyst VM's MAC address while claiming to be the legitimate resolver, and it directed the victim to the analyst-hosted training page at 192.168.20.129 rather than the legitimate host. ARP replies during the event mapped the gateway IP to the analyst's MAC address, confirming man-in-the-middle positioning. The victim then established an HTTP connection to the spoofed IP address. All lab-created processes, firewall rules, and ARP entries were cleaned up and verified after evidence collection.

---

## At a Glance

| | |
|---|---|
| **Question** | Was a victim redirected by forged DNS responses, and can the attack be proven from packet evidence? |
| **Answer** | Yes. Two conflicting DNS answers shared one transaction ID, the forged one came from the analyst MAC, and the victim then connected to the spoofed IP. |
| **Forged answer IP** | `192.168.20.129` (analyst VM) |
| **Legitimate answer IP** | `192.168.20.50` |
| **Confidence** | High |

---

## Case Information

| Field | Details |
|---|---|
| Case reference | SBT-DF203-Lab8-DNS-Spoofing-Forensics |
| Course | SBT-DF203, Basic Networking Skills for Digital Forensics |
| Investigator | Adekunle Ogunyemi |
| Instructor | Aminu Idris, AMCPN |
| Report date | 24 September 2026 |
| Evidence types | DNS and ARP packet captures, HTTP connection records, SHA-256 hashes, training page artefacts, firewall state snapshots |

---

## Scope and Authority

**Authority.** Work was performed as part of the SBT-DF203 practical laboratory exercise on systems assigned to the exercise, under ICDFA lab authorisation.

**In scope.** DNS spoofing detection, ARP poisoning correlation, baseline and controlled packet captures, HTTP connection analysis, firewall state documentation, ARP cache restoration, and associated forensic artefacts.

**Out of scope.** Any system beyond the assigned lab environment.

---

## Lab Environment

| Component | Details |
|---|---|
| Analyst VM | Kali Linux, `192.168.20.129` on `eth0` |
| Victim VM | Separate VM on the same host-only network (`192.168.20.131` in capture filters) |
| Gateway | `192.168.20.2`, MAC `00:50:56:ec:88:69` |
| Subnet | `192.168.20.0/24` |
| Web server | Apache2 hosting the training page on the analyst VM |
| Capture and analysis | `tshark` |
| Hashing | `sha256sum` |
| Training domain | `portal.icdfa.test` |

### Baseline Screenshots

**Lab directory structure**  

<img width="1367" height="466" alt="image" src="https://github.com/user-attachments/assets/551ee0b7-2564-4431-bcfb-88074abce56f" />


**Interface baseline** 

<img width="1575" height="310" alt="image" src="https://github.com/user-attachments/assets/1f8e9c34-0534-4b15-9fe7-d604bbda5b82" />


**Route baseline** 

<img width="1575" height="310" alt="image" src="https://github.com/user-attachments/assets/06cccccf-fabd-4d49-abaa-aa01e7fff24e" />


**ARP table before spoofing** 

<img width="1421" height="252" alt="image" src="https://github.com/user-attachments/assets/39e01188-e58b-4e2d-87a2-c3340340458c" />



---

## Methodology

| Phase | Activity |
|---|---|
| **A. Baseline** | Record interfaces, routes and ARP table; take a 30-second DNS/ARP capture; hash the capture and training page |
| **B. Preparation** | Record IP forwarding and `iptables` state; test the training page; review instructor-provided scripts |
| **C. Controlled capture** | Capture a 60-second spoofing event; hash the PCAP |
| **D. DNS analysis** | Extract query and response fields; compare transaction IDs, source MACs, answer IPs and TTLs |
| **E. Comparison** | Contrast baseline against spoofed behaviour |
| **F. ARP correlation** | Extract ARP replies and map gateway claims to MAC addresses |
| **G. Victim correlation** | Extract HTTP requests and TCP SYN packets to confirm redirection |
| **H. Cleanup** | Stop processes, restore state, and verify |

### Pre-Simulation Screenshots (Part B)

**IP forwarding state before change** 

<img width="1547" height="295" alt="image" src="https://github.com/user-attachments/assets/b4a1b0d6-949a-4203-930e-f5b840697659" />


**Firewall state snapshot, `iptables-save`** 

<img width="1482" height="647" alt="image" src="https://github.com/user-attachments/assets/d12c818f-bf72-484f-a6bb-939df64272e4" />


<img width="1430" height="731" alt="image" src="https://github.com/user-attachments/assets/3f0dfe55-4b1a-4c56-aee7-035709a0240c" />


<img width="1398" height="720" alt="image" src="https://github.com/user-attachments/assets/8d9a5b96-6720-4e77-930d-55ae096906e7" />


<img width="1580" height="715" alt="image" src="https://github.com/user-attachments/assets/c361c452-5dec-483f-9fb7-b56c9ad373d2" />


**Direct training page test** 

<img width="1572" height="360" alt="image" src="https://github.com/user-attachments/assets/cabacd78-9499-467a-9abe-ad73dbcd76a7" />


**Script review output, `dns_spoof.py` syntax error** (E-10)

<img width="1601" height="412" alt="image" src="https://github.com/user-attachments/assets/4bfb06cf-d042-4287-a681-cc3a98c78564" />


---

## Key Findings

| # | Finding | Result |
|---|---|---|
| 1 | DNS spoofing detection | Conflicting DNS answers for a single transaction ID |
| 2 | Forged response source | Analyst MAC address claiming to be the legitimate resolver |
| 3 | ARP poisoning | Gateway IP mapped to the analyst MAC during the event |
| 4 | Victim redirection | HTTP connection established to spoofed IP `192.168.20.129` |
| 5 | Transaction ID match | Forged and legitimate responses shared ID `0x1234` |
| 6 | TTL anomaly | Forged response TTL 60 s versus legitimate 300 s |
| 7 | Cleanup | ARP table and firewall state restored |
| 8 | Evidence integrity | Baseline and controlled PCAP SHA-256 hashes recorded |

The forged response arrived approximately 70 ms before the legitimate one, consistent with a DNS spoofing race condition.

---

## Detection Indicators

| Indicator | Baseline | Controlled spoof | Assessment |
|---|---|---|---|
| DNS responder IP | Legitimate resolver | Claimed legitimate resolver | Spoofed source IP |
| DNS responder MAC | Gateway MAC | Analyst MAC | Forged response source |
| Transaction ID | Single response | Two responses, same ID | Race condition |
| Answer IP | `192.168.20.50` | `192.168.20.129` | Redirected to attacker |
| TTL | 300 s | 60 s | Anomalous TTL |
| Response timing | Normal | Forged arrived first | Race indicator |
| Competing answers | None | Two conflicting answers | Spoofing confirmed |
| Gateway ARP mapping | Gateway MAC | Analyst MAC | ARP poisoning |
| HTTP destination | Legitimate host | `192.168.20.129` | Victim redirected |

### Alternative explanations considered

| Explanation | Assessment |
|---|---|
| DNS caching | Ruled out. Training DNS cache was cleared and two conflicting responses were seen for one transaction. |
| Split-horizon DNS | Ruled out. Forged response came from an unexpected MAC while claiming to be the resolver. |
| Multiple legitimate answers | Ruled out. Legitimate resolver returned a different IP with a normal TTL. |
| Network misconfiguration | Ruled out. ARP poisoning evidence and response timing indicate deliberate spoofing. |
| Script error | Noted. See [Limitations](#limitations). |

---

## Evidence Inventory

| ID | Description |
|---|---|
| E-01 | Lab directory structure confirmation |
| E-02 | Training page SHA-256 hash |
| E-03 | Interface and route baseline |
| E-04 | Initial ARP table (before spoofing) |
| E-05 | Baseline DNS/ARP packet capture |
| E-06 | Baseline capture SHA-256 hash |
| E-07 | IP forwarding state before change |
| E-08 | Firewall state snapshot (`iptables-save`) |
| E-09 | Direct training page test output |
| E-10 | Script review output and syntax error record |
| E-11 | Controlled spoofing PCAP hash |
| E-12 | DNS query and response extraction |
| E-13 | ARP claims during event extraction |
| E-14 | Victim HTTP connection extraction |
| E-15 | ARP table after cleanup |
| E-16 | Firewall state after cleanup |
| E-17 | Process cleanup verification |

---

## Evidence Integrity

Original captures were preserved unaltered. SHA-256 hashes were recorded at acquisition, and all analysis was performed on verified working copies.

| Artefact | SHA-256 |
|---|---|
| `evidence/dns_baseline.pcapng` | `21d4eb3cf466bfcd7a1badf38c1b834179d2ae5150c478d1460ae43a73b7cba4` |
| `evidence/dns_spoof_controlled.pcapng` | `67234562edc85922f4fd172a03e2800d88410aa32f78a62968d195e5bafebe92` |
| `/var/www/html/index.html` | `996ad035a8e146a61eced04db274ec6073309378a41c9e3256d0aab9d2a2df41` |

Verify a file yourself:

```bash
sha256sum evidence/dns_spoof_controlled.pcapng
```

**Training page hash** (E-02)

<img width="1762" height="227" alt="image" src="https://github.com/user-attachments/assets/92f14fd0-fb58-45b5-8216-86271dd8bafa" />


**Baseline capture hash** (E-06)

<img width="1615" height="152" alt="image" src="https://github.com/user-attachments/assets/db6c9d91-8c67-4d9d-b6c6-72a261c13473" />


**Controlled spoofing capture hash** (E-11)

<img width="1632" height="260" alt="image" src="https://github.com/user-attachments/assets/dced064e-653a-4f4f-a0e3-bb3807648e8b" />


---

## Reproducing the Analysis

### Requirements

- Kali Linux (or any Linux host with Wireshark/`tshark` installed)
- An isolated host-only virtual network with a victim VM
- Instructor-approved lab materials and written authorisation

### Capture

```bash
# Baseline (30 s)
sudo tshark -i eth0 -f 'host 192.168.20.131 and (arp or port 53 or tcp port 80)' \
  -a duration:30 -w evidence/dns_baseline.pcapng

# Controlled event (60 s)
sudo tshark -i eth0 -f "host 192.168.20.131 and (arp or port 53 or tcp port 80)" \
  -a duration:60 -w evidence/dns_spoof_controlled.pcapng
```

**Baseline capture in progress** (E-05)

<img width="1577" height="422" alt="image" src="https://github.com/user-attachments/assets/4aeb2465-9a69-43e2-a35c-286591ecc4a0" />


### Analyse

```bash
# DNS queries and responses
tshark -r evidence/dns_spoof_controlled.pcapng -Y 'dns' -T fields \
  -e frame.number -e frame.time_epoch -e eth.src -e eth.dst \
  -e ip.src -e udp.srcport -e ip.dst -e udp.dstport \
  -e dns.id -e dns.flags.response -e dns.qry.name -e dns.qry.type \
  -e dns.flags.rcode -e dns.a -e dns.resp.ttl
```

```bash
# ARP replies
tshark -r evidence/dns_spoof_controlled.pcapng -Y 'arp.opcode==2' -T fields \
  -e frame.number -e frame.time_epoch -e arp.src.proto_ipv4 -e arp.src.hw_mac \
  -e arp.dst.proto_ipv4 -e arp.dst.hw_mac
```

**ARP claims during the event** (E-13)

<img width="1603" height="345" alt="image" src="https://github.com/user-attachments/assets/2b5e2d48-be36-42fe-938f-1be41f7d91ad" />


```bash
# Victim HTTP connection
tshark -r evidence/dns_spoof_controlled.pcapng -Y 'http.request || tcp.flags.syn==1' -T fields \
  -e frame.number -e frame.time_epoch -e ip.src -e tcp.srcport \
  -e ip.dst -e tcp.dstport -e http.host -e http.request.uri
```

**Victim HTTP connection** (E-14)

<img width="1600" height="298" alt="image" src="https://github.com/user-attachments/assets/84f22c64-be36-4dfe-bc16-940deb59f776" />


---

## Cleanup and Verification

```bash
sudo pkill -f 'arp.py' 2>/dev/null || true
sudo pkill -f 'dns_spoof.py' 2>/dev/null || true
sudo sysctl -w net.ipv4.ip_forward=0
sudo iptables -L -n -v --line-numbers | tee reports/iptables_after_cleanup.txt
sudo ip neigh flush all
ip neigh show | tee reports/arp_after_cleanup.txt
ps aux | grep -E '[a]rp.py|[d]ns_spoof.py' | tee reports/process_cleanup_check.txt
```

ARP table after cleanup: `192.168.20.2 dev eth0 lladdr 00:50:56:ec:88:69 REACHABLE`

**Firewall state after cleanup** (E-16)

<img width="1733" height="937" alt="image" src="https://github.com/user-attachments/assets/d5e89f00-106d-4518-8bf9-31c2dc80f7f0" />


**ARP table after cleanup** (E-15)

<img width="1600" height="298" alt="image" src="https://github.com/user-attachments/assets/bab58d44-aa89-48fa-a51a-074698a1aadc" />


---

## Limitations

- The investigation was conducted in a controlled laboratory and may not represent all DNS spoofing scenarios.
- `dns_spoof.py` encountered a syntax error during execution, which limited the full demonstration of the DNS spoofing payload. ARP poisoning and DNS response anomalies were still captured and analysed.
- Some ARP extraction output was only partially captured in screenshots.
- DNS results may vary with caching behaviour, resolver configuration and request timing.
- A harmless training page and reserved domain were used, so no real credentials or production systems were affected.

---

## Recommendations

| # | Control | Purpose |
|---|---|---|
| 1 | DNSSEC | Detect forged DNS responses where supported |
| 2 | DNS-over-HTTPS / DNS-over-TLS | Protect queries from interception |
| 3 | Dynamic ARP Inspection | Prevent ARP poisoning on switches |
| 4 | DHCP Snooping | Build a trusted binding table |
| 5 | Switch port security | Limit MAC addresses per port |
| 6 | Network segmentation | Limit the blast radius of MITM attacks |
| 7 | HTTPS certificate validation | A forged DNS answer should not yield a valid certificate for the real service |
| 8 | Monitoring | Alert on conflicting DNS answers, unexpected responders and ARP anomalies |
| 9 | Evidence handling | Record resolver configuration first, preserve originals, analyse verified copies |
| 10 | Documentation | Log queries, responses, transaction IDs and timestamps |

---

## Repository Structure

```text
SBT-DF203-Lab8/
├── evidence/        # Original PCAP captures
├── reports/         # Command outputs, hashes, firewall and ARP snapshots
├── scripts/         # Instructor-provided lab scripts
├── exported/        # Exported analysis artefacts
├── working/         # Verified working copies
├── screenshots/     # 01_*.png to 19_*.png, referenced inline in this README
├── Report.pdf       # Full investigation report
└── README.md
```

---

## Academic Integrity

This repository describes work completed during the assigned SBT-DF203 practical laboratory exercise. Screenshots, command outputs, packet captures and hash values come from the author's own lab environment. Reference material does not replace the original evidence collected during the investigation.

---

## Author and License

**Adekunle Ogunyemi**  
Digital Forensics Student, SBT-DF203  
Instructor: Aminu Idris, AMCPN

Shared for educational reference.  
© 2026 Adekunle Ogunyemi. Please do not submit this work as your own.
