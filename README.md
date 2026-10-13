# Information & Network Security — Practicals (Copy-Paste Ready)

All blocks are **direct copy-paste** — each starts with `conf t` and ends with `end`, so paste the whole block into the router CLI at the `R1#` / `R2#` / `R3#` prompt.

> 💡 If you're at `R1>`, type `en` first. If stuck, type `?` to see the next valid keyword.

---

## Practical 8 — Syslog, NTP and SSH

### 📋 Paste in R1
```
conf t
router ospf 1
network 192.168.1.0 0.0.0.255 area 0
network 10.1.1.0 0.0.0.3 area 0
area 0 authentication message-digest
exit
interface g0/1
ip ospf message-digest-key 1 md5 MD5pa55
exit
interface s0/0/0
ip ospf message-digest-key 1 md5 MD5pa55
exit
ntp server 192.168.1.5
ntp update-calendar
ntp authenticate
ntp trusted-key 1
ntp authentication-key 1 md5 NTPpa55
service timestamps log datetime msec
logging host 192.168.1.6
end
```

### 📋 Paste in R2
```
conf t
router ospf 1
network 10.1.1.0 0.0.0.3 area 0
network 10.2.2.0 0.0.0.3 area 0
area 0 authentication message-digest
exit
interface s0/0/0
ip ospf message-digest-key 1 md5 MD5pa55
exit
interface s0/0/1
ip ospf message-digest-key 1 md5 MD5pa55
exit
ntp server 192.168.1.5
ntp update-calendar
ntp authenticate
ntp trusted-key 1
ntp authentication-key 1 md5 NTPpa55
service timestamps log datetime msec
logging host 192.168.1.6
end
```

### 📋 Paste in R3 (includes SSH)
```
conf t
router ospf 1
network 192.168.3.0 0.0.0.255 area 0
network 10.2.2.0 0.0.0.3 area 0
area 0 authentication message-digest
exit
interface g0/1
ip ospf message-digest-key 1 md5 MD5pa55
exit
interface s0/0/1
ip ospf message-digest-key 1 md5 MD5pa55
exit
ntp server 192.168.1.5
ntp update-calendar
ntp authenticate
ntp trusted-key 1
ntp authentication-key 1 md5 NTPpa55
service timestamps log datetime msec
logging host 192.168.1.6
ip domain-name ccnasecurity.com
username SSHadmin privilege 15 secret ciscosshpa55
line vty 0 4
login local
transport input ssh
exit
crypto key generate rsa
end
```
> ⚠️ When `crypto key generate rsa` asks "How many bits": type **1024** and press Enter. Then continue:
```
conf t
ip ssh time-out 90
ip ssh authentication-retries 2
ip ssh version 2
end
```

**GUI:** PC-A → Services → NTP → On, Key `1`, Password `NTPpa55`. PC-B → Services → SYSLOG → On.
**Test:** from PC-C: `ssh -l SSHadmin 192.168.3.1` ✅ (telnet ❌). `show ntp status`, `show logging`.

---

## Practical 9 — AAA Authentication

### 🖱️ GUI first (right after OSPF)
- **TACACS+ Server** (192.168.2.2) → Services → AAA → Service **On**
  Client `shlok` | IP `192.168.2.1` | Secret `sycs` | Type **Tacacs** → Add. User `admin` / `sycs` → Add.
- **RADIUS Server** (192.168.3.2) → Services → AAA → Service **On**
  Client `shlok` | IP `192.168.3.1` | Secret `sycs` | Type **Radius** → Add. User `admin` / `sycs` → Add.

### 📋 Paste in R1 (Local)
```
conf t
username admin secret sycs
aaa new-model
aaa authentication login default local
line console 0
login authentication default
end
```

### 📋 Paste in R2 (TACACS+)
```
conf t
username admin secret sycs
tacacs-server host 192.168.2.2
tacacs-server key sycs
aaa new-model
aaa authentication login default group tacacs+ local
line console 0
login authentication default
end
```

### 📋 Paste in R3 (RADIUS)
```
conf t
username admin secret sycs
radius-server host 192.168.3.2
radius-server key sycs
aaa new-model
aaa authentication login default group radius local
line console 0
login authentication default
end
```

**Test:** on each router type `exit`, press Enter, log in with `admin` / `sycs`.

---

## Practical 10 — Extended ACLs (Scenario 1) — single router

### 📋 Paste in R1 (interface IPs, if not done)
```
conf t
interface g0/0
ip address 172.22.34.65 255.255.255.224
no shutdown
exit
interface g0/1
ip address 172.22.34.97 255.255.255.240
no shutdown
exit
interface g0/2
ip address 172.22.34.1 255.255.255.192
no shutdown
end
```

### 📋 Paste in R1 (both ACLs)
```
conf t
access-list 100 permit tcp 172.22.34.64 0.0.0.31 host 172.22.34.62 eq ftp
access-list 100 permit icmp 172.22.34.64 0.0.0.31 host 172.22.34.62
interface g0/0
ip access-group 100 in
exit
ip access-list extended HTTP_ONLY
permit tcp 172.22.34.96 0.0.0.15 host 172.22.34.62 eq www
permit icmp 172.22.34.96 0.0.0.15 host 172.22.34.62
exit
interface g0/1
ip access-group HTTP_ONLY in
end
```

**PC IPs:** Server `172.22.34.62/255.255.255.192` GW `.1` | PC1 `.66/255.255.255.224` GW `.65` | PC2 `.98/255.255.255.240` GW `.97`
**Test:** PC1 ping/ftp server ✅ (cisco/cisco), ping PC2 ❌. PC2 ping + browser to server ✅, ftp ❌.

---

## Practical 11 — Extended ACLs (Scenario 2)

### 📋 Paste in each switch (change IP per switch)
```
conf t
interface vlan 1
ip address 10.101.117.50 255.255.255.248
no shutdown
exit
ip default-gateway 10.101.117.49
end
```
> SWA: `10.101.117.50 /248` GW `.49` | SWB: `10.101.117.34 255.255.255.240` GW `.33` | SWC: `10.101.117.2 255.255.255.224` GW `.1`

### 📋 Paste in Router (RTA)
```
conf t
access-list 199 permit tcp 10.101.117.32 0.0.0.15 10.101.117.0 0.0.0.31 eq 22
access-list 199 permit icmp any any
interface g0/2
ip access-group 199 out
end
```

**Test:** PCB `ssh -l Admin 10.101.117.2` ✅ | PCA ssh to SWC ❌ | all pings ✅.

---

## Practical 12 — Zone-Based Policy Firewall — all on R3

### 📋 Paste in R3 — Step 1: license (then reload)
```
conf t
license boot module c1900 technology-package securityk9
```
> Type `yes` at ACCEPT → then:
```
end
wr
reload
```

### 📋 Paste in R3 — Step 2: whole ZPF config
```
conf t
zone security IN-ZONE
exit
zone security OUT-ZONE
exit
access-list 101 permit ip 192.168.3.0 0.0.0.255 any
class-map type inspect match-all IN-NET-CLASS-MAP
match access-group 101
exit
policy-map type inspect IN-2-OUT-PMAP
class type inspect IN-NET-CLASS-MAP
inspect
exit
exit
zone-pair security IN-2-OUT-ZPAIR source IN-ZONE destination OUT-ZONE
service-policy type inspect IN-2-OUT-PMAP
exit
interface g0/1
zone-member security IN-ZONE
exit
interface s0/0/1
zone-member security OUT-ZONE
end
wr
```

**Memory: Z-C-P-P-I** → Zones, Class-map, Policy-map, Pair, Interfaces.
**Test:** PC-C → PC-A ping/SSH/HTTP ✅. PC-A → PC-C ping ❌. Verify: `show policy-map type inspect zone-pair sessions`

---

## Practical 13 — IOS IPS — all on R1

### 📋 Paste in R1 — Step 1: license (then reload)
```
conf t
license boot module c1900 technology-package securityk9
```
> Type `yes` → then:
```
end
wr
reload
```

### 📋 Paste in R1 — Step 2: folder (exec mode, press Enter at prompt)
```
mkdir ipsdir
```

### 📋 Paste in R1 — Step 3: setup + signatures
```
conf t
ip ips config location flash:ipsdir
ip ips name iosips
ip ips notify log
service timestamps log datetime msec
logging host 192.168.1.50
ip ips signature-category
category all
retired true
exit
category ios_ips basic
retired false
exit
exit
```
> Press **Enter** at `[confirm]`, wait for ENGINE_READY, then:
```
ip ips signature-definition
signature 2004 0
status
retired false
enabled true
exit
engine
event-action produce-alert
event-action deny-packet-inline
exit
exit
exit
```
> Press **Enter** at `[confirm]`, then:
```
interface g0/1
ip ips iosips out
end
wr
```

**Chant:** *license–reload, mkdir–location, name, log, all off, basic on, 2004: wake–enable–deny, g0/1 out.*
**Test:** PC-C → ping PC-A ❌ fails. PC-A → ping PC-C ✅ works. Syslog server (192.168.1.50) shows `%IPS-4` alerts.

---

## Python Practicals (1–7)

| # | Topic | Library | Install |
|---|-------|---------|---------|
| 1 | Caesar & Monoalphabetic Cipher | `string`, `random` | — |
| 2 | Vernam Cipher (OTP) | `random`, `string` | — |
| 3 | Diffie-Hellman | — | — |
| 4 | DES | `Crypto.Cipher.DES` | `pip install pycryptodome` |
| 5 | AES | `Crypto.Cipher.AES` | `pip install pycryptodome` |
| 6 | RSA | `rsa` | `pip install rsa` |
| 7 | MD5 Message Digest | `hashlib` | — |

---

*TYBSc CS (Hons) Sem 5 — Information & Network Security, 2024-25*
