# Digital Forensics Artifacts — Windows & Linux Cheat Sheet

> Ini beda sama cheat sheet Volatility/TSK sebelumnya (yang isinya *command*). Yang ini isinya **konsep & lokasi artefak** — "ini tuh apa", "ini nyimpen apa", "buat apa" — biar pas ketemu istilah kayak Amcache, Prefetch, SRUM, dsb, langsung ngerti fungsinya sebelum masuk ke tool command-nya.
>
> Format: **Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat**

## Windows Forensics Artifacts

### Registry Hives (inti dari hampir semua artefak Windows)

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **SAM** | Menyimpan akun user lokal & password hash. | `C:\Windows\System32\config\SAM` | Daftar user lokal, hash password (butuh SYSTEM hive buat decrypt), grup |
| **SECURITY** | Menyimpan kebijakan keamanan lokal & cached credential. | `C:\Windows\System32\config\SECURITY` | LSA secrets, cached domain logon |
| **SYSTEM** | Konfigurasi inti sistem: service, driver, network, ShimCache. | `C:\Windows\System32\config\SYSTEM` | Daftar service/driver, ShimCache, komputer name, timezone, USB history (via `ControlSetXXX\Enum\USBSTOR`) |
| **SOFTWARE** | Konfigurasi software terinstal & OS. | `C:\Windows\System32\config\SOFTWARE` | Installed programs, versi OS, network profile, App Paths |
| **NTUSER.DAT** | Registry hive per-user; nyimpen preferensi & aktivitas user. | `C:\Users\<user>\NTUSER.DAT` | UserAssist, RunMRU, TypedPaths, RecentDocs, ShellBags (sebagian) |
| **UsrClass.dat** | Registry hive per-user untuk shell/COM class; sumber utama ShellBags. | `C:\Users\<user>\AppData\Local\Microsoft\Windows\UsrClass.dat` | ShellBags (folder yang pernah dibuka), file association |

### Eksekusi Program (Program Execution Evidence)

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **Prefetch (.pf)** | File yang dibuat Windows untuk mempercepat loading aplikasi; dibuat tiap kali program dijalankan. | `C:\Windows\Prefetch\*.pf` | Nama executable, jumlah run, **timestamp run pertama & terakhir**, file/DLL yang diakses |
| **Amcache.hve** | Registry hive yang mencatat metadata setiap executable yang pernah *ada* di sistem (dijalankan atau tidak). | `C:\Windows\AppCompat\Programs\Amcache.hve` | Path executable, SHA1 hash, timestamp compile, install date — bagus buat bukti keberadaan file walau sudah dihapus |
| **ShimCache / AppCompatCache** | Cache kompatibilitas aplikasi; mencatat file executable yang pernah diakses (belum tentu dijalankan). | Registry `SYSTEM` hive: `ControlSet00X\Control\Session Manager\AppCompatCache` | Path file, ukuran file, last modified time — urutan entry menunjukkan urutan eksekusi relatif |
| **UserAssist** | Mencatat program yang dijalankan via GUI Explorer (bukan command line). | `NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\UserAssist` (nama key di-ROT13) | Nama program, jumlah run, last run time |
| **BAM / DAM** (Background/Desktop Activity Moderator) | Fitur throttling yang mencatat aplikasi yang jalan di background, termasuk timestamp eksekusi terakhir. | Registry `SYSTEM` hive: `ControlSet00X\Services\bam\State\UserSettings\<SID>` | Path executable, **timestamp eksekusi terakhir per user** |
| **SRUM (System Resource Usage Monitor)** | Database yang mencatat penggunaan resource (CPU, network, energi) per aplikasi per 60 menit. | `C:\Windows\System32\sru\SRUDB.dat` | Aplikasi yang berjalan, network usage (bytes sent/received), berguna walau aplikasinya sudah dihapus |

### File & Folder Access Evidence

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **ShellBags** | Menyimpan preferensi tampilan folder (size, posisi) — efek sampingnya, mencatat folder yang pernah dibuka, termasuk yang sudah dihapus/di drive removable. | `UsrClass.dat` & `NTUSER.DAT` → `...\Shell\Bags` & `BagMRU` | Path folder yang pernah diakses, timestamp akses folder |
| **LNK files (.lnk)** | Shortcut file, otomatis dibuat Windows saat user membuka file lewat Explorer/Recent. | `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\` | Path asli file target, MAC time file asal, serial number volume asal (berguna kalau file dari USB) |
| **Jump Lists** | Daftar file/aksi yang baru diakses per-aplikasi, muncul saat klik-kanan icon di taskbar. | `AutomaticDestinations`: `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\AutomaticDestinations\`<br>`CustomDestinations`: folder sebelah | File yang dibuka per aplikasi, timestamp akses |
| **RecentDocs** | Daftar dokumen terakhir dibuka via Explorer. | `NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs` | Nama file, ekstensi, urutan akses |
| **RunMRU** | Riwayat command yang diketik di Run dialog (Win+R). | `NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU` | Command yang pernah dijalankan lewat Run |
| **TypedPaths** | Riwayat path yang diketik manual di address bar Explorer. | `NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\TypedPaths` | Path folder/network share yang pernah diketik |
| **Thumbnail Cache** | Cache thumbnail gambar/video, bisa jadi bukti keberadaan file walau sudah dihapus. | `C:\Users\<user>\AppData\Local\Microsoft\Windows\Explorer\thumbcache_*.db` | Bukti visual (thumbnail) file yang pernah ada, walau file asli sudah dihapus |
| **Windows Search DB** | Index pencarian Windows Search. | `C:\ProgramData\Microsoft\Search\Data\Applications\Windows\Windows.edb` | Path file, isi email (Outlook), metadata file yang pernah diindex |
| **Windows Timeline** | Fitur riwayat aktivitas (Windows 10) — aplikasi & dokumen yang dibuka. | `C:\Users\<user>\AppData\Local\ConnectedDevicesPlatform\...\ActivitiesCache.db` | Riwayat aplikasi/dokumen yang dibuka beserta timestamp |

### Filesystem (NTFS)

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **$MFT (Master File Table)** | Struktur inti NTFS; setiap file/folder punya entry di sini. | Root volume (metadata file, tidak terlihat langsung) | Nama file, timestamp MACB (Modified/Accessed/Created/Born), ukuran, lokasi data, termasuk file yang sudah dihapus |
| **$LogFile** | Transaction log NTFS untuk memastikan integritas filesystem. | Root volume | Rekonstruksi perubahan filesystem baru-baru ini (rename, create, delete) |
| **$UsnJrnl ($Extend\$UsnJrnl)** | Update Sequence Number Journal — mencatat setiap perubahan pada file/folder. | Root volume, `$Extend\$UsnJrnl` | History perubahan file (create, rename, delete, overwrite) beserta timestamp |
| **$I30 (INDX)** | Index attribute NTFS untuk direktori (B-tree index nama file). | Bagian dari entry direktori di $MFT | Bisa menyimpan nama file yang sudah dihapus dari direktori tapi belum ter-overwrite di index |
| **Recycle Bin** | Tempat file yang dihapus user (belum permanen). | `C:\$Recycle.Bin\<SID>\` — pasangan `$I` (metadata) dan `$R` (data) | Nama asli file, path asal, waktu dihapus, ukuran (dari file `$I`) |
| **Volume Shadow Copy (VSS)** | Snapshot otomatis Windows dari waktu ke waktu (System Restore/backup). | Tersembunyi, diakses via `vssadmin` atau mount | Versi lama file/registry dari waktu snapshot dibuat — bisa "membuka waktu" ke kondisi sebelum insiden |
| **Pagefile.sys / Hiberfil.sys / Swapfile.sys** | File swap memory (pagefile), hibernasi (hiberfil), dan UWP app swap (swapfile). | Root drive (`C:\pagefile.sys`, dsb, biasanya hidden) | Sisa data dari RAM — password, string, fragment proses yang pernah aktif |

### Log & Eksekusi Terjadwal

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **Windows Event Logs (.evtx)** | Log event sistem, security, dan aplikasi. | `C:\Windows\System32\winevt\Logs\*.evtx` | Login/logout (Event ID 4624/4625), service start (7045), proses (4688 jika audit aktif), dsb |
| **Scheduled Tasks** | Task terjadwal, sering dipakai malware untuk persistence. | `C:\Windows\System32\Tasks\` & registry `SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache` | Nama task, command yang dijalankan, jadwal, author |
| **Run / RunOnce Keys** | Registry key untuk auto-start program saat boot/login. | `HKLM/HKCU\Software\Microsoft\Windows\CurrentVersion\Run(Once)` | Program yang auto-jalan saat startup — titik persistence umum |
| **Services** | Daftar Windows service beserta konfigurasinya. | Registry `SYSTEM\CurrentControlSet\Services` | Path binary service, start type — sering disalahgunakan buat persistence malware |
| **WMI Repository** | Database WMI, bisa dipakai attacker untuk persistence (WMI event subscription). | `C:\Windows\System32\wbem\Repository\OBJECTS.DATA` | WMI event filter/consumer — teknik persistence fileless |
| **Windows Error Reporting (WER)** | Crash dump & report saat aplikasi crash. | `C:\ProgramData\Microsoft\Windows\WER\` | Bisa mengandung path & info proses yang crash, termasuk malware yang gagal jalan |

## Linux Forensics Artifacts

### Log Sistem & Autentikasi

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **auth.log / secure** | Log autentikasi (login, sudo, SSH). | Debian/Ubuntu: `/var/log/auth.log`<br>RHEL/CentOS: `/var/log/secure` | Login berhasil/gagal, penggunaan `sudo`, koneksi SSH masuk |
| **syslog / messages** | Log sistem umum (kernel, service, daemon). | Debian/Ubuntu: `/var/log/syslog`<br>RHEL/CentOS: `/var/log/messages` | Event sistem umum, error service, aktivitas daemon |
| **systemd journal** | Log terpusat systemd (binary format), pengganti syslog di distro modern. | `/var/log/journal/` (persistent) atau in-memory | Semua log unit systemd, boot log — dibaca pakai `journalctl` |
| **wtmp** | Riwayat login/logout user beserta durasi sesi. | `/var/log/wtmp` | Riwayat login (dibaca via `last`) |
| **btmp** | Riwayat percobaan login yang **gagal**. | `/var/log/btmp` | Percobaan login gagal (dibaca via `lastb`) — indikasi brute force |
| **utmp** | Sesi login yang **sedang aktif** saat ini. | `/var/run/utmp` | User yang sedang login saat ini (dibaca via `who`/`w`) |
| **lastlog** | Waktu login terakhir per user. | `/var/log/lastlog` | Timestamp login terakhir tiap akun |
| **audit.log (auditd)** | Log audit kernel-level yang detail (syscall, file access, dsb) jika `auditd` aktif. | `/var/log/audit/audit.log` | Syscall yang dipanggil, file access, eksekusi command — sangat detail kalau rule audit-nya lengkap |

### Aktivitas User & Shell

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **Bash history** | Riwayat command yang diketik user di shell. | `~/.bash_history` (per user), `/root/.bash_history` | Command yang dijalankan attacker/user — bisa dihapus/dimanipulasi, cross-check dengan memory (`linux.bash.Bash`) |
| **Shell profile files** | Script yang dijalankan saat shell/login dimulai; sering disalahgunakan buat persistence. | `~/.bashrc`, `~/.profile`, `/etc/profile.d/`, `~/.bash_profile` | Command/alias yang auto-jalan tiap login — titik persistence umum |
| **SSH artifacts** | Konfigurasi & kunci SSH user. | `~/.ssh/authorized_keys`, `~/.ssh/known_hosts`, `~/.ssh/id_rsa*` | Public key yang diizinkan login (indikasi backdoor kalau ada key asing), riwayat host yang pernah diakses |
| **Recently used files** | Riwayat file yang dibuka lewat aplikasi GTK/desktop environment. | `~/.local/share/recently-used.xbel` | File yang baru-baru dibuka user via GUI |

### Identitas & Kredensial

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **/etc/passwd** | Daftar akun user beserta info dasar (UID, GID, shell, home dir). | `/etc/passwd` | Daftar user lokal, termasuk user mencurigakan yang baru dibuat (UID 0 tapi bukan root = red flag) |
| **/etc/shadow** | Password hash user (butuh root untuk baca). | `/etc/shadow` | Hash password, tanggal expired — bisa dicoba crack offline |
| **/etc/group** | Keanggotaan grup user. | `/etc/group` | User yang punya privilege grup tertentu (misalnya `sudo`/`wheel`) |
| **sudoers** | Konfigurasi siapa yang boleh `sudo` dan sejauh apa. | `/etc/sudoers`, `/etc/sudoers.d/` | Privilege escalation path, user dengan akses root |

### Persistence & Eksekusi Terjadwal

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **Cron jobs** | Task terjadwal level user/sistem. | `/etc/crontab`, `/etc/cron.d/`, `/var/spool/cron/crontabs/<user>` | Command terjadwal — titik persistence umum di Linux |
| **systemd timers/units** | Alternatif cron di systemd; unit service/timer custom. | `/etc/systemd/system/`, `/lib/systemd/system/` | Service/timer custom yang dibuat attacker untuk persistence |
| **rc.local / init scripts** | Script yang dijalankan saat boot (metode lama). | `/etc/rc.local`, `/etc/init.d/` | Command yang auto-jalan saat sistem boot |
| **LD_PRELOAD** | Environment variable untuk memaksa loading shared library tertentu lebih dulu — teknik hooking umum. | `/etc/ld.so.preload`, env var per proses | Indikasi library berbahaya yang di-inject ke semua proses |
| **Kernel modules** | Module kernel yang dimuat, termasuk yang auto-load saat boot. | `/etc/modules`, `/lib/modules/`, `/etc/modprobe.d/` | Module mencurigakan (potensi rootkit level kernel) |

### Filesystem & Package Management

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **dpkg/apt log** | Riwayat instalasi/upgrade paket (Debian-based). | `/var/log/dpkg.log`, `/var/log/apt/history.log` | Paket yang diinstall/dihapus/upgrade beserta waktunya |
| **rpm/yum/dnf log** | Riwayat paket (RedHat-based). | `/var/log/yum.log`, `rpm -qa --last` | Sama seperti dpkg, versi RHEL |
| **/tmp & /var/tmp** | Direktori temporary yang world-writable — sering jadi tempat staging malware. | `/tmp/`, `/var/tmp/` | File payload/staging attacker, sering dihapus setelah eksekusi tapi bisa dicari via carving |
| **Filesystem timestamps (MAC time)** | Metadata waktu tiap file (Modify, Access, Change). | Semua file, dibaca via `stat` | Kapan file dibuat/diubah/diakses — dasar timeline forensik |
| **Mount info** | Filesystem apa saja yang ter-mount, termasuk removable media. | `/etc/fstab`, `/proc/mounts` | Bukti device eksternal/USB yang pernah dipasang |

### Jaringan & Service

| Artefak | Apa itu & kegunaan | Lokasi | Info yang bisa didapat |
| --- | --- | --- | --- |
| **/etc/hosts** | Resolusi hostname manual — bisa dimanipulasi untuk redirect domain. | `/etc/hosts` | Entry hostname mencurigakan (misalnya domain bank di-redirect ke IP attacker) |
| **/etc/resolv.conf** | Konfigurasi DNS server yang dipakai sistem. | `/etc/resolv.conf` | DNS server yang dipakai — bisa jadi indikasi DNS hijacking |
| **Web server logs** | Log akses & error web server. | `/var/log/apache2/access.log`, `/var/log/nginx/access.log`, dsb | Request HTTP masuk — bukti web shell/exploit attempt |
| **Mail logs** | Log aktivitas mail server. | `/var/log/mail.log` | Aktivitas SMTP — bukti spam relay/phishing dari server yang dikompromi |

## Cara pakai cheat sheet ini

1. Pas nemu artefak di lab/CTF (misalnya "ini file Amcache.hve isinya apa?") → cek tabel di atas buat tau **fungsi & info apa yang bisa diambil**.
2. Kalau udah paham konsepnya, baru pakai command dari cheat sheet **Volatility 3** (buat baca dari memory) atau **The Sleuth Kit** (buat baca dari disk image) yang sudah dibuat sebelumnya buat benar-benar ekstrak datanya.
3. Kombinasi umum: Prefetch + ShimCache + Amcache = triangulasi bukti eksekusi program; $MFT + $UsnJrnl + $LogFile = rekonstruksi aktivitas file; auth.log + wtmp/btmp + bash history = rekonstruksi aktivitas user Linux.

> **Catatan:** path/lokasi bisa sedikit berbeda tergantung versi Windows/distro Linux. Untuk parsing artefak-artefak ini biasanya dipakai tool tambahan seperti RegRipper, Eric Zimmerman Tools (untuk Windows), atau Plaso/log2timeline (cross-platform) — kalau mau, bilang aja nanti dibikinin cheat sheet tool parsing-nya juga.
