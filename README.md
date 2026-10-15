# Jarkom-Modul-3-2026-K-40

Laporan konfigurasi **DHCP Server & DHCP Relay** menggunakan GNS3, dengan Router **Jichi**, DHCP Server **Tengen**, dan client **alpinet** yang mendapat IP secara dinamis.

Prefix IP yang digunakan: `192.231.x.x`

| Interface Jichi | Subnet         | Node                                                                |
| --------------- | -------------- | ------------------------------------------------------------------- |
| eth0            | DHCP           | Internet Publik (NAT GNS3)                                          |
| eth1            | 192.231.1.0/24 | Utahime, Shoko, Gojo                                                |
| eth2            | 192.231.2.0/24 | Itadori, Fushiguro, Zenin, Okkotsu (Divisi A - DHCP)                |
| eth3            | 192.231.3.0/24 | Ino, Kusakabe, Miwa, Kugisaki (Divisi B - DHCP)                     |
| eth4            | 192.231.4.0/24 | Tengen, Geto, Maki, Panda, Nanami, Todo, Kirara, Yaga, Inumaki      |

---

## 1. Topologi, Forwarding & NAT

![topologi](images/topologi.png)

### Konfigurasi IP (`/etc/network/interfaces` / Edit config GNS3)

**Router Jichi**

```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.231.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.231.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.231.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 192.231.4.1
    netmask 255.255.255.0
```

**Host statis** (contoh Tengen, ganti `address` & `gateway` sesuai tabel di bawah)

```
auto eth0
iface eth0 inet static
    address 192.231.4.10
    netmask 255.255.255.0
    gateway 192.231.4.1
```

| Node    | IP           | Node    | IP           |
| ------- | ------------ | ------- | ------------ |
| Utahime | 192.231.1.2  | Panda   | 192.231.4.32 |
| Shoko   | 192.231.1.3  | Nanami  | 192.231.4.41 |
| Gojo    | 192.231.1.4  | Todo    | 192.231.4.42 |
| Tengen  | 192.231.4.10 | Kirara  | 192.231.4.43 |
| Geto    | 192.231.4.20 | Yaga    | 192.231.4.51 |
| Maki    | 192.231.4.31 | Inumaki | 192.231.4.52 |

**Client alpinet** (8 client, isinya sama)

```
auto eth0
iface eth0 inet dhcp
```

> Isi juga di menu **Edit config** GNS3, karena file ini ditulis ulang setiap node di-Start.

Aktifkan forwarding dan NAT di **Jichi**:

```
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-forward.conf
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Set DNS sementara di host statis:

```
echo "nameserver 192.168.122.1" > /etc/resolv.conf
```

**Verifikasi:** dari host statis jalankan `ping -c 2 <gateway>` dan `ping -c 2 8.8.8.8`, harus dapat balasan.

![no1](images/no1.png)

---

## 2. DHCP Server di Tengen

```
apt-get update && apt-get install -y isc-dhcp-server
echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
```

Isi `/etc/dhcp/dhcpd.conf` (versi dasar, dipecah jadi pool di soal 4-5):

```
authoritative;
option domain-name "k40.com";
option domain-name-servers 192.231.1.2, 192.231.1.3;
option domain-search "k40.com";
default-lease-time 600;
max-lease-time 7200;

subnet 192.231.4.0 netmask 255.255.255.0 {
}

subnet 192.231.2.0 netmask 255.255.255.0 {
    option routers 192.231.2.1;
    option broadcast-address 192.231.2.255;
    range 192.231.2.10 192.231.2.35;
}

subnet 192.231.3.0 netmask 255.255.255.0 {
    option routers 192.231.3.1;
    option broadcast-address 192.231.3.255;
    range 192.231.3.10 192.231.3.89;
}
```

> Subnet `192.231.4.0` (tanpa range) wajib ada agar dhcpd tidak error `No subnet declaration`.

```
touch /var/lib/dhcp/dhcpd.leases
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
```

**Verifikasi:** `service isc-dhcp-server status` harus `running`.

![no2](images/no2.png)

---

## 3. DHCP Relay di Jichi

```
apt-get update && apt-get install -y isc-dhcp-relay
```

Isi `/etc/default/isc-dhcp-relay`:

```
SERVERS="192.231.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
```

> `eth2` & `eth3` = sisi client, `eth4` = sisi Tengen (agar balasan server ikut diteruskan).

```
service isc-dhcp-relay restart
```

Di **setiap client**, buat script untuk meminta lease:

```
cat << 'EOF' > /root/client_dhcp.sh
#!/bin/sh
pkill udhcpc 2>/dev/null
ip addr flush dev eth0
ip link set eth0 up
udhcpc -i eth0 -n -q -t 5 -T 2 -x hostname:$(hostname)
ip -4 addr show eth0 | grep inet
ip route
cat /etc/resolv.conf
EOF
chmod +x /root/client_dhcp.sh
sh /root/client_dhcp.sh
```

**Verifikasi:** `pgrep -a dhcrelay` di Jichi harus muncul, dan client menampilkan `lease of 192.231.x.x obtained from 192.231.4.10`.

![no3](images/no3.png)

---

## 4. Pool Berbasis MAC (Divisi A)

| Pool   | Client             | Range               |
| ------ | ------------------ | ------------------- |
| Pool 1 | Itadori, Fushiguro | 192.231.2.10 - .35  |
| Pool 2 | Zenin, Okkotsu     | 192.231.2.70 - .95  |
| -      | (tidak dialokasikan) | 192.231.2.36 - .69 |

Cek MAC tiap client dengan `ip link show eth0 | grep ether`, lalu tambahkan di `/etc/dhcp/dhcpd.conf` Tengen:

```
class "pool1" {
    match if substring(hardware, 1, 6) = 02:42:e2:07:da:00
          or substring(hardware, 1, 6) = 02:42:46:07:2d:00;
}
class "pool2" {
    match if substring(hardware, 1, 6) = 02:42:cb:b2:d2:00
          or substring(hardware, 1, 6) = 02:42:fb:4c:da:00;
}

subnet 192.231.2.0 netmask 255.255.255.0 {
    option routers 192.231.2.1;
    option broadcast-address 192.231.2.255;
    pool {
        allow members of "pool1";
        range 192.231.2.10 192.231.2.35;
    }
    pool {
        allow members of "pool2";
        range 192.231.2.70 192.231.2.95;
    }
}
```

```
dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
```

Jalankan `sh /root/client_dhcp.sh` di keempat client.

**Verifikasi** di Tengen:

```
grep -E "^lease|hardware ethernet|client-hostname" /var/lib/dhcp/dhcpd.leases
```

![no4](images/no4.png)

---

## 5. Short Lease & Long Lease (Divisi B)

| Pool       | Client        | Range                | Lease               |
| ---------- | ------------- | -------------------- | ------------------- |
| Short      | Ino, Kusakabe | 192.231.3.10 - .89   | 1800 detik (30 menit) |
| Long       | Miwa          | 192.231.3.100 - .199 | 43200 detik (12 jam)  |

Tambahkan class dan ganti blok subnet `192.231.3.0` di `/etc/dhcp/dhcpd.conf` Tengen (isi MAC sesuai client masing-masing):

```
class "short-lease" {
    match if substring(hardware, 1, 6) = 02:42:c7:7f:09:00
          or substring(hardware, 1, 6) = <MAC_KUSAKABE>;
}
class "long-lease" {
    match if substring(hardware, 1, 6) = <MAC_MIWA>;
}

subnet 192.231.3.0 netmask 255.255.255.0 {
    option routers 192.231.3.1;
    option broadcast-address 192.231.3.255;
    pool {
        allow members of "short-lease";
        range 192.231.3.10 192.231.3.89;
        default-lease-time 1800;
        max-lease-time 1800;
    }
    pool {
        allow members of "long-lease";
        range 192.231.3.100 192.231.3.199;
        default-lease-time 43200;
        max-lease-time 43200;
    }
}
```

```
dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
```

**Verifikasi:** jalankan `sh /root/client_dhcp.sh` di Ino, Kusakabe, dan Miwa. Output `udhcpc` menampilkan `lease time 1800` (Ino, Kusakabe) dan `lease time 43200` (Miwa).

![no5](images/no5.png)

---

## 6. Fixed Address Kugisaki

Tambahkan di akhir `/etc/dhcp/dhcpd.conf` Tengen:

```
host kugisaki {
    hardware ethernet 02:42:a8:e3:32:00;
    fixed-address 192.231.3.99;
}
```

> `.99` berada di luar pool short (`.10-.89`) dan pool long (`.100-.199`), jadi tidak bentrok.

```
dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
```

**Verifikasi:** jalankan `sh /root/client_dhcp.sh` di Kugisaki dua kali, IP harus selalu `192.231.3.99` dengan gateway `192.231.3.1`.

![no6](images/no6.png)

---

## 7. Validasi Konfigurasi Client & Lease

Di **setiap client**:

```
sh /root/client_dhcp.sh
```

| Client             | IP                   | Gateway     | Broadcast     |
| ------------------ | -------------------- | ----------- | ------------- |
| Itadori, Fushiguro | 192.231.2.10 - .35   | 192.231.2.1 | 192.231.2.255 |
| Zenin, Okkotsu     | 192.231.2.70 - .95   | 192.231.2.1 | 192.231.2.255 |
| Ino, Kusakabe      | 192.231.3.10 - .89   | 192.231.3.1 | 192.231.3.255 |
| Miwa               | 192.231.3.100 - .199 | 192.231.3.1 | 192.231.3.255 |
| Kugisaki           | 192.231.3.99 (fixed) | 192.231.3.1 | 192.231.3.255 |

Semua client harus mendapat netmask `/24`, DNS `192.231.1.2` & `192.231.1.3`, dan `search k40.com` di `/etc/resolv.conf`.

Di **Tengen**, cocokkan dengan file lease:

```
grep -E "^lease|starts|ends|hardware ethernet|client-hostname|binding state" /var/lib/dhcp/dhcpd.leases
```

**Verifikasi:**

- 7 lease aktif (semua client kecuali Kugisaki)
- Tidak ada IP/MAC dobel dan tidak ada lease di range `192.231.2.36 - .69`
- Kugisaki tidak muncul di `dhcpd.leases` (normal untuk `fixed-address`)

![no7](images/no7.png)

---

## 8. Packet Capture DHCP (DORA + giaddr)

Di **Tengen**, mulai capture 40 detik:

```
apt-get install -y tcpdump
timeout 40 tcpdump -i eth0 -n -vv -w /root/dhcp_net3.pcap 'udp port 67 or udp port 68'
```

Dalam 40 detik tersebut, di **Ino** jalankan:

```
sh /root/client_dhcp.sh
```

Baca hasil capture (difilter berdasarkan MAC Ino):

```
tcpdump -n -vv -r /root/dhcp_net3.pcap 'udp[36:4]=0x0242c77f and udp[40:2]=0x0900' 2>/dev/null \
  | grep -E "DHCP-Message|Gateway-IP|Your-IP|Server-ID|Lease-Time"
```

| Yang dicari        | Hasil                                                    |
| ------------------ | -------------------------------------------------------- |
| Urutan             | `Discover` → `Offer` → `Request` → `ACK`                 |
| Alamat server      | `Server-ID 192.231.4.10`                                 |
| Alamat diberikan   | `Your-IP 192.231.3.x`                                    |
| Lease time         | `Lease-Time 1800`                                        |
| giaddr             | `Gateway-IP 192.231.3.1` (interface eth3 Jichi)          |

![no8](images/no8.png)

---

## 9. Simulasi Gangguan DHCP Relay

Matikan relay di **Jichi**:

```
service isc-dhcp-relay stop
pkill dhcrelay
```

Di **Itadori**, minta lease baru:

```
pkill udhcpc; ip addr flush dev eth0
udhcpc -i eth0 -n -q -t 3 -T 2
ip -4 addr show eth0
```

| Lokasi       | Gejala                                                     |
| ------------ | ---------------------------------------------------------- |
| Itadori      | `no lease, failing`, eth0 tidak punya IP & default route   |
| Tengen       | `tcpdump -i eth0` tidak menangkap paket DHCP sama sekali   |
| Jichi (eth2) | DISCOVER dari client terlihat, tetapi tidak ada OFFER      |

Pulihkan relay di **Jichi**:

```
service isc-dhcp-relay restart
pgrep -a dhcrelay
```

**Verifikasi:** jalankan `sh /root/client_dhcp.sh` di Itadori, harus kembali dapat IP di Pool 1 (`192.231.2.10 - .35`) dengan gateway `192.231.2.1`.

![no9](images/no9.png)

---

## 10. Autostart & Uji Restart

Di **Tengen**, buat script start dan autostart:

```
cat << 'EOF' > /root/start_services.sh
#!/bin/bash
ip route replace default via 192.231.4.1
touch /var/lib/dhcp/dhcpd.leases
rm -f /var/run/dhcpd.pid
service isc-dhcp-server restart
EOF
chmod +x /root/start_services.sh

update-rc.d isc-dhcp-server defaults

cat << 'EOF' > /root/init.sh
#!/bin/sh
( sleep 10; /root/start_services.sh > /root/start_services.log 2>&1 ) &
EOF
chmod +x /root/init.sh
```

Di **Jichi**:

```
cat << 'EOF' > /root/start_services.sh
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
rm -f /var/run/dhcrelay.pid
service isc-dhcp-relay restart
EOF
chmod +x /root/start_services.sh

update-rc.d isc-dhcp-relay defaults

cat << 'EOF' > /root/init.sh
#!/bin/sh
( sleep 15; /root/start_services.sh > /root/start_services.log 2>&1 ) &
EOF
chmod +x /root/init.sh
```

> Container debinet menjalankan `/root/init.sh` otomatis setiap node di-Start, sehingga service ikut hidup setelah restart.

Di GNS3: **Stop** lalu **Start** Tengen dan Jichi, tunggu ±30 detik.

**Verifikasi:**

```
# Tengen
pgrep -a dhcpd
# Jichi
pgrep -a dhcrelay
iptables -t nat -L POSTROUTING -n
# Setiap client
sh /root/client_dhcp.sh
ping -c 2 192.231.4.10
```

Semua client harus kembali mendapat IP sesuai pool masing-masing (Pool 1, Pool 2, short lease, long lease, dan fixed address Kugisaki).

![no10](images/no10.png)
