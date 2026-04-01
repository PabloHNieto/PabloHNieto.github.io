---
permalink: /linux/vpn&local_wifi
collection: linux
title: "Fix on VPN and dead local wifi"
author_profile: false
excerpt: 'Using FortiClient with wifi hotspot resulted in no local internet. Here we fix it.'
tags:
  - linux
  - nentwork
  - DNS
---


# 🛠️ Troubleshooting Report: VPN + Smartphone Hotspot DNS Failure on Ubuntu

## 📌 Summary
When connecting a VPN on an Ubuntu laptop using a smartphone Wi‑Fi hotspot, the user experienced *loss of internet access*.  
Diagnostics revealed that **connectivity through the VPN remained intact**, but **DNS queries were rejected** when the hotspot was active.

The root cause was a **DNS resolution conflict**, worsened by a mismatch between the system hostname and the `/etc/hosts` file.

After correcting the hostname entry and reconfiguring DNS, the issue was fully resolved.

---

## 🧩 Observed Symptoms


### 1. Internet connectivity still works
Running:
Shellping -c 4 1.1.1.1Show more lines
returned successful replies both with and without the VPN connected.
This confirmed:

The tunnel was active,
Routing was correct,
Only DNS was broken.


```bash
(base) pablo@gyptaeus 09:43 ~ ping -c 4 1.1.1.1 SIN VPN
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=49.0 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=40.1 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=55.4 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=54 time=47.9 ms

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 40.102/48.085/55.386/5.427 ms
(base) pablo@gyptaeus 09:45 ~ ping -c 4 1.1.1.1 CON VPN
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=45.6 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=51.6 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=51.0 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=54 time=50.9 ms

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 45.554/49.756/51.627/2.443 ms
```

### **2. DNS resolution fails with VPN + hotspot**
Running:

```bash
dig google.com
```

Returned 
```bash
## WITH VPN CONNECTED
(base) pablo@gyptaeus 09:43 ~ dig google.com

; <<>> DiG 9.18.39-0ubuntu0.22.04.3-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 19273
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.com.			IN	A

;; Query time: 71 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Wed Apr 01 09:43:44 CEST 2026
;; MSG SIZE  rcvd: 39

## WITHOUT VPN CONNECTED
(base) pablo@gyptaeus 09:43 ~ dig google.com

; <<>> DiG 9.18.39-0ubuntu0.22.04.3-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49557
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		278	IN	A	172.217.171.46

;; Query time: 225 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Wed Apr 01 09:43:49 CEST 2026
;; MSG SIZE  rcvd: 55
```
returned:

```bash
status: REFUSED
SERVER: 127.0.0.53#53
```

This indicated that:

DNS requests were reaching systemd-resolved,
but the DNS server assigned by the VPN refused the query.

## Solution

### Step 1 — Edit resolved.conf:
```bash
sudo cp /etc/systemd/resolved.conf /etc/systemd/resolved_bck.conf ## For backup
sudo vi /etc/systemd/resolved.conf
```

Add the following lines
```bash
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9
DNSStubListener=no
```
### Step 2 — Restart systemd‑resolved
```bash
sudo systemctl restart systemd-resolved
```
### Step 3 — Point /etc/resolv.conf to the correct file:
```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
However that gave me te following problem

```bash
unable to resolve host pc-csm-004: Name or service not known
```
And I fixed with the followign.
```bash
sudo vi /etc/hosts
```

And adding the following line below `127.0.1.1 localhost`
```bash
127.0.1.1   pc-csm-004
```

The above solved my issue.
