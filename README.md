# Jarkom-Modul-3-2026-K-40

## Kelompok: K-40


---

## SOAL 1: Konfigurasi IP & Routing


### Kode:
```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### Cara Cek:
```bash
cat /proc/sys/net/ipv4/ip_forward
```

### Hasil Benar:
```
1
```

### Status: ✅ SELESAI

---

## SOAL 2: Install DHCP Server

| Item | Detail |
|------|--------|
| **Node** | tengen |
| **Deskripsi** | Install dan konfigurasi isc-dhcp-server |

### Kode:
```bash
apt-get update
apt-get install -y isc-dhcp-server

cat > /etc/default/isc-dhcp-server <<EOF
INTERFACESv4="eth0"
EOF

cat > /etc/dhcp/dhcpd.conf <<EOF
subnet 192.231.4.0 netmask 255.255.255.0 {
    pool {
        range 192.231.4.200 192.231.4.210;
        deny unknown-clients;
    }
}
EOF

ip route del default 2>/dev/null
ip route add default via 192.231.4.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf

service isc-dhcp-server restart
update-rc.d isc-dhcp-server defaults
```

### Cara Cek:
```bash
service isc-dhcp-server status
```

### Hasil Benar:
```
Status of ISC DHCPv4 server: dhcpd is running.
```

### Status: ✅ SELESAI

---

## SOAL 3: Konfigurasi DHCP Relay

| Item | Detail |
|------|--------|
| **Node** | jichi |
| **Deskripsi** | Konfigurasi DHCP Relay ke tengen |

### Kode:
```bash
apt-get update
apt-get install -y isc-dhcp-relay

cat > /etc/default/isc-dhcp-relay <<EOF
SERVERS="192.231.4.10"
INTERFACES="eth2 eth3"
OPTIONS=""
EOF

service isc-dhcp-relay restart
update-rc.d isc-dhcp-relay defaults
```

### Cara Cek:
```bash
ps aux | grep dhcrelay
service isc-dhcp-relay status
```

### Hasil Benar:
```
root    xxx   0.0  /usr/sbin/dhcrelay -q -i eth2 -i eth3 192.231.4.10
```

### Status: ✅ SELESAI

---

## SOAL 4: DHCP Pool 1 & Pool 2

| Item | Detail |
|------|--------|
| **Node** | tengen |
| **Deskripsi** | Konfigurasi 2 pool DHCP untuk Divisi A |

### Kode:
```bash
cat > /etc/dhcp/dhcpd.conf <<'EOF'
option domain-name "jujutsu.k40.com";
option domain-name-servers 192.231.1.2, 192.231.1.3;

subnet 192.231.4.0 netmask 255.255.255.0 {
}

subnet 192.231.2.0 netmask 255.255.255.0 {
    option routers 192.231.2.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.2.255;
    option domain-name-servers 192.231.1.2, 192.231.1.3;
    option domain-search "jujutsu.k40.com";
    pool { range 192.231.2.10 192.231.2.35; }
    pool { range 192.231.2.70 192.231.2.95; }
}

subnet 192.231.3.0 netmask 255.255.255.0 {
    option routers 192.231.3.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.3.255;
}
EOF

service isc-dhcp-server restart
```

### Cara Cek:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
```

### Hasil Benar:
```
No error, dhcpd is running
```

### Status: ✅ SELESAI

---

## SOAL 5: Short Lease & Long Lease

| Item | Detail |
|------|--------|
| **Node** | tengen |
| **Deskripsi** | Konfigurasi lease time berbeda untuk Divisi B |

### Kode:
```bash
cat > /etc/dhcp/dhcpd.conf <<'EOF'
option domain-name "jujutsu.k40.com";
option domain-name-servers 192.231.1.2, 192.231.1.3;

subnet 192.231.4.0 netmask 255.255.255.0 {
}

subnet 192.231.2.0 netmask 255.255.255.0 {
    option routers 192.231.2.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.2.255;
    option domain-name-servers 192.231.1.2, 192.231.1.3;
    option domain-search "jujutsu.k40.com";
    pool { range 192.231.2.10 192.231.2.35; }
    pool { range 192.231.2.70 192.231.2.95; }
}

subnet 192.231.3.0 netmask 255.255.255.0 {
    option routers 192.231.3.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.3.255;
    option domain-name-servers 192.231.1.2, 192.231.1.3;
    option domain-search "jujutsu.k40.com";
    pool { range 192.231.3.10 192.231.3.89; default-lease-time 1800; max-lease-time 1800; }
    pool { range 192.231.3.100 192.231.3.199; default-lease-time 43200; max-lease-time 43200; }
}
EOF

service isc-dhcp-server restart
```

### Cara Cek:
```bash
grep -E "lease-time" /etc/dhcp/dhcpd.conf
```

### Hasil Benar:
```
default-lease-time 1800;
max-lease-time 1800;
default-lease-time 43200;
max-lease-time 43200;
```

### Status: ✅ SELESAI

---

## SOAL 6: Fixed Address Kugisaki

| Item | Detail |
|------|--------|
| **Node** | tengen |
| **Deskripsi** | Konfigurasi fixed IP untuk kugisaki |

### Kode:
```bash
cat > /etc/dhcp/dhcpd.conf <<'EOF'
option domain-name "jujutsu.k40.com";
option domain-name-servers 192.231.1.2, 192.231.1.3;

subnet 192.231.4.0 netmask 255.255.255.0 {
}

subnet 192.231.2.0 netmask 255.255.255.0 {
    option routers 192.231.2.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.2.255;
    option domain-name-servers 192.231.1.2, 192.231.1.3;
    option domain-search "jujutsu.k40.com";
    pool { range 192.231.2.10 192.231.2.35; }
    pool { range 192.231.2.70 192.231.2.95; }
}

subnet 192.231.3.0 netmask 255.255.255.0 {
    option routers 192.231.3.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.231.3.255;
    option domain-name-servers 192.231.1.2, 192.231.1.3;
    option domain-search "jujutsu.k40.com";

    host kugisaki {
        hardware ethernet 02:42:ac:11:00:99;
        fixed-address 192.231.3.99;
        default-lease-time 43200;
        max-lease-time 43200;
    }

    pool { range 192.231.3.10 192.231.3.89; default-lease-time 1800; max-lease-time 1800; }
    pool { range 192.231.3.100 192.231.3.199; default-lease-time 43200; max-lease-time 43200; }
}
EOF

service isc-dhcp-server restart
```

### Cara Cek:
```bash
grep -A5 "host kugisaki" /etc/dhcp/dhcpd.conf
```

### Hasil Benar:
```
host kugisaki {
    hardware ethernet 02:42:ac:11:00:99;
    fixed-address 192.231.3.99;
    default-lease-time 43200;
    max-lease-time 43200;
}
```

### Status: ✅ SELESAI

---

## SOAL 7: Validasi Konfigurasi

| Item | Detail |
|------|--------|
| **Node** | tengen |
| **Deskripsi** | Validasi semua konfigurasi DHCP |

### Kode:
```bash
cat /etc/dhcp/dhcpd.conf
```

### Cara Cek:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
```

### Checklist Hasil:

| Item | Expected | Status |
|------|----------|--------|
| Domain | jujutsu.k40.com | ✅ |
| DNS | 192.231.1.2, 192.231.1.3 | ✅ |
| Pool 1 (Net2) | 192.231.2.10-35 | ✅ |
| Pool 2 (Net2) | 192.231.2.70-95 | ✅ |
| Gateway Net2 | 192.231.2.1 | ✅ |
| Gateway Net3 | 192.231.3.1 | ✅ |
| Short Lease | 192.231.3.10-89, 1800s | ✅ |
| Long Lease | 192.231.3.100-199, 43200s | ✅ |
| Fixed (kugisaki) | 192.231.3.99 | ✅ |

### Status: ✅ SELESAI

---

## SOAL 8: Packet Capture DHCP

| Item | Detail |
|------|--------|
| **Node** | tengen, jichi, client |
| **Deskripsi** | Capture packet DHCP untuk verifikasi |

### Kode - tengen:
```bash
apt-get update
apt-get install -y tcpdump
tcpdump -i eth0 port 67 or port 68 -v
```

### Kode - jichi:
```bash
tcpdump -i eth2 port 67 or port 68 -v
```

### Kode - client:
```bash
ip addr flush dev eth0
ip addr add 192.231.2.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.231.2.1
```

### Cara Cek:
Lihat output tcpdump

### Hasil Benar:
```
DHCPDISCOVER     ← client broadcast mencari server
DHCPOFFER        ← server menawarkan IP
DHCPREQUEST      ← client meminta IP
DHCPACK          ← server konfirmasi IP
```

### Status: ✅ SELESAI

---

## SOAL 9: Gangguan & Recovery DHCP Relay

| Item | Detail |
|------|--------|
| **Node** | jichi |
| **Deskripsi** | Simulasi gangguan dan recovery relay |

### Kode:
```bash
# Stop Relay
service isc-dhcp-relay stop

# Cek gejala (client tidak dapat IP)

# Start Relay lagi
service isc-dhcp-relay start
```

### Cara Cek:
```bash
service isc-dhcp-relay status
```

### Hasil Benar:
```
# Saat STOP: relay tidak berjalan
# Saat START: relay berjalan dengan eth2, eth3 forwarding ke 192.231.4.10
```

### Status: ✅ SELESAI

---

## SOAL 10: Autostart Services

| Item | Detail |
|------|--------|
| **Node** | jichi, tengen |
| **Deskripsi** | Setup autostart service setelah reboot |

### Kode - jichi:
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ipforward.conf
sysctl -w net.ipv4.ip_forward=1

cat > /etc/init.d/autostart-nat.sh <<'EOF'
#!/bin/bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
EOF
chmod +x /etc/init.d/autostart-nat.sh
bash /etc/init.d/autostart-nat.sh

update-rc.d autostart-nat.sh defaults
update-rc.d isc-dhcp-relay defaults
```

### Kode - tengen:
```bash
update-rc.d isc-dhcp-server defaults
```

### Cara Cek:
```bash
# Cek di jichi
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L POSTROUTING -v -n
ls -la /etc/rc*.d/*dhcp*

# Cek di tengen
service isc-dhcp-server status
ls -la /etc/rc*.d/*dhcp*
```

### Hasil Benar:
```
IP Forward: 1
NAT MASQUERADE: Ada rule
Service: running
Autostart Links: rc0-rc6 ada
```

### Status: ✅ SELESAI

---

## TABEL RINGKASAN

| No | Node | Fungsi | Status |
|----|------|--------|--------|
| 1 | jichi | IP Forward + NAT | ✅ |
| 2 | tengen | DHCP Server | ✅ |
| 3 | jichi | DHCP Relay | ✅ |
| 4 | tengen | Pool 1 & 2 | ✅ |
| 5 | tengen | Short/Long Lease | ✅ |
| 6 | tengen | Fixed Address | ✅ |
| 7 | tengen | Validasi | ✅ |
| 8 | All | Packet Capture | ✅ |
| 9 | jichi | Gangguan/Recovery | ✅ |
| 10 | All | Autostart | ✅ |

---

## TOPOLOGI JARINGAN

```
INTERNET
    |
   NAT
    |
    | eth0 (DHCP)
    |
  [jichi]  ← Router & DHCP Relay
    |
    ├─ eth1 ── [Switch-1] ── utahime (.1.2), shoko (.1.3), gojo (.1.10)
    │
    ├─ eth2 ── [Switch-2] ── itadori, fushiguro, zenin, okkotsu (DHCP)
    │
    ├─ eth3 ── [Switch-4] ── ino, kusakabe, miwa, kugisaki (DHCP)
    │
    └─ eth4 ── [Switch-6] ── tengen (.4.10), geto (.4.20), dll
```

---

## TABEL IP

| Node | IP | Keterangan |
|------|-----|------------|
| jichi | eth1: 192.231.1.1, eth2: 192.231.2.1, eth3: 192.231.3.1, eth4: 192.231.4.1 | Router |
| tengen | 192.231.4.10 | DHCP Server |
| utahime | 192.231.1.2 | DNS Master |
| shoko | 192.231.1.3 | DNS Slave |
| gojo | 192.231.1.10 | Reverse Proxy |
| geto | 192.231.4.20 | Load Balancer |
| yaga | 192.231.4.51 | DB Master |
| inumaki | 192.231.4.52 | DB Slave |
| itadori | DHCP Pool 1 (192.231.2.10-35) | Client |
| fushiguro | DHCP Pool 1 (192.231.2.10-35) | Client |
| zenin | DHCP Pool 2 (192.231.2.70-95) | Client |
| okkotsu | DHCP Pool 2 (192.231.2.70-95) | Client |
| ino | DHCP Short Lease (192.231.3.10-89, 1800s) | Client |
| kusakabe | DHCP Short Lease (192.231.3.10-89, 1800s) | Client |
| miwa | DHCP Long Lease (192.231.3.100-199, 43200s) | Client |
| kugisaki | Fixed Address (192.231.3.99) | Client VIP |

---

**Laporan dibuat: 2026-10-13**
**Kelompok: K-40**
