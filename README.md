# 🚀 Otomasi Remote ExamBrowser Client via Action1 RMM

Repositori ini berisi panduan lengkap dan kumpulan skrip PowerShell siap pakai untuk menjalankan (*start & auto RUN*) serta menutup (*graceful exit*) aplikasi **ExamBrowser Client** (ANBK/CBT/SNBT/TKA) secara massal dan simultan melalui cloud dashboard **Action1 RMM**.

> **💡 Info Kuota Gratis:** Action1 menyediakan paket gratis permanen (*free tier*) hingga **200 endpoint/klien** dengan fitur lengkap, sangat ideal untuk kebutuhan laboratorium komputer sekolah maupun kampus.

---

## 📑 Daftar Isi

1. [Fitur Utama](#-fitur-utama)
2. [Setup Action1 (Akun & Agen)](#-setup-action1-akun--agen)
   * [1. Pendaftaran Akun](#1-pendaftaran-akun)
   * [2. Unduh Installer Agent](#2-unduh-installer-agent)
   * [3. Instalasi Agent di PC Klien Lab](#3-instalasi-agent-di-pc-klien-lab)
     * [Opsi A: Instalasi 1-Klik via Flashdisk (Batch Script)](#opsi-a-instalasi-1-klik-via-flashdisk-batch-script)
     * [Opsi B: Deploy via PSEXEC (Jaringan Lokal)](#opsi-b-deploy-via-psexec-jaringan-lokal)
     * [Opsi C: Active Directory (Group Policy / GPO)](#opsi-c-active-directory-group-policy--gpo)
   * [4. Pengelompokan Klien (Endpoint Group)](#4-pengelompokan-klien-endpoint-group)
3. [Konfigurasi Direktori (Wajib untuk Publik)](#-konfigurasi-direktori-wajib-untuk-publik)
4. [Skrip Otomasi](#-skrip-otomasi)
   * [Skrip 1: Buka & Auto-RUN ExamBrowser](#skrip-1-buka--auto-run-exambrowser)
   * [Skrip 2: Tutup Aman ExamBrowser (Ctrl+C+B)](#skrip-2-tutup-aman-exambrowser-ctrlcb)
5. [Cara Eksekusi Massal dari Dashboard](#-cara-eksekusi-massal-dari-dashboard)
6. [Bonus: Skrip Shutdown Massal Klien Lab via Action1](#-bonus-skrip-shutdown-massal-klien-lab-via-action1)
   * [1. Skrip PowerShell: Shutdown dengan Peringatan 30 Detik](#1-skrip-powershell-shutdown-dengan-peringatan-30-detik)
   * [2. Skrip Pembatalan: Cancel Shutdown](#2-skrip-pembatalan-cancel-shutdown-jika-salah-eksekusi)
   * [3. Cara Penggunaan di Dashboard Action1](#3-cara-penggunaan-di-dashboard-action1)
7. [Troubleshooting](#-troubleshooting)
8. [Lisensi](#-lisensi)

---

## 📌 Fitur Utama

* **Otomasi Klik RUN Tanpa Campur Tangan Siswa:** Membuka `ExamBrowser.exe`, menekan tombol konfirmasi update jika muncul, dan otomatis mengklik tombol **RUN** via UI Automation & simulasi koordinat.
* **Kustomisasi Direktori Publik:** Variabel path terisolasi di baris teratas, memudahkan admin lab mana pun untuk menyesuaikan lokasi file di komputer mereka.
* **Sesi Interaktif Pengguna:** Skrip dijalankan di dalam sesi desktop pengguna aktif menggunakan *Scheduled Task helper*, sehingga antarmuka ExamBrowser tampil di layar siswa.
* **Tutup Aman (Graceful Exit):** Mengirim kombinasi tombol standar `Ctrl + C + B` untuk menutup sesi ujian secara resmi tanpa merusak data sesi atau mematikan paksa (*force kill*).

---

## 🛠️ Setup Action1 (Akun & Agen)

### 1. Pendaftaran Akun
1. Buka laman resmi [Action1 Sign Up](https://www.action1.com/signup/).
2. Isi formulir pendaftaran:
   * **Work Email**: Masukkan email aktif (email instansi atau domain organisasi).
   * **Organization Name**: Nama sekolah, universitas, atau lab komputer.
   * **Region Data Storage**: Pilih wilayah server terdekat (misal: *Europe/Germany* atau *US*).
3. Buka email verifikasi, klik tautan aktivasi, lalu buat kata sandi akun Anda.
4. Aktifkan **Two-Factor Authentication (2FA)** untuk mengamankan akses ke dashboard.

### 2. Unduh Installer Agent
1. Masuk ke dashboard Action1.
2. Di panel navigasi sebelah kiri, buka menu **Endpoints**.
3. Klik tombol **Add Endpoints** di kanan atas.
4. Pilih **Download Agent Installer (.msi / .exe)**. File installer ini sudah otomatis dipasangi token organisasi Anda (*pre-configured*).

### 3. Instalasi Agent di PC Klien Lab

Pilih salah satu metode berikut sesuai sarana lab Anda:

#### Opsi A: Instalasi 1-Klik via Flashdisk (Batch Script)

Untuk mempercepat instalasi pada puluhan PC tanpa perlu membuka Command Prompt (CMD) manual di tiap komputer, gunakan file launcher `.bat` dengan auto-elevation administrator.

##### 1. Persiapan File di Flashdisk
Simpan file `action1_agent.msi` dan buat file baru bernama `INSTALL_ACTION1.bat` di direktori yang sama (misal di root flashdisk):

```
📁 FLASHDISK (D:)
├── 📄 action1_agent.msi
└── ⚙️ INSTALL_ACTION1.bat
```

##### 2. Isi Skrip `INSTALL_ACTION1.bat`
Salin kode berikut ke dalam file `INSTALL_ACTION1.bat`:

```bat
@echo off
:: Otomatis meminta hak akses Administrator (UAC Prompt) jika belum Run as Admin
net session >nul 2>&1
if %errorLevel% neq 0 (
    powershell -Command "Start-Process '%~f0' -Verb RunAs"
    exit /b
)

:: Pindah ke direktori tempat file BAT ini berada di flashdisk
cd /d "%~dp0"

echo [1/2] Menginstall Action1 Agent secara latar belakang (silent)...
msiexec /i action1_agent.msi /qn

echo [2/2] Menunggu verifikasi service Windows...
timeout /t 5 /nobreak >nul

sc query "Action1 Agent" | find "RUNNING" >nul
if %errorLevel% equ 0 (
    echo.
    echo ==============================================
    echo SUKSES: Action1 Agent berhasil terpasang dan aktif!
    echo ==============================================
) else (
    echo.
    echo Instalasi selesai, service sedang sinkronisasi ke cloud Action1.
)

timeout /t 3 >nul
exit
```

##### 3. Cara Penggunaan di PC Klien Lab
1. Tancapkan flashdisk ke PC klien yang menyala.
2. Buka isi flashdisk, lalu **klik ganda (double-click)** pada file `INSTALL_ACTION1.bat`.
3. Klik **Yes** pada jendela konfirmasi User Account Control (UAC).
4. Skrip akan langsung memasang agent secara hening di latar belakang, memverifikasi status servicenya, lalu jendela konsol tertutup sendiri dalam 3 detik.

#### Opsi B: Deploy via PSEXEC (Jaringan Lokal)
Jika lab memiliki koneksi LAN dengan kredensial administrator lokal yang sama:
```powershell
psexec.exe \\CLIENT-01 -u Administrator -p PasswordLab -c action1_agent.msi msiexec /i action1_agent.msi /qn
```

#### Opsi C: Active Directory (Group Policy / GPO)
Jika komputer lab terhubung ke domain Windows Server:
1. Simpan `action1_agent.msi` pada folder network share (misal: `\\SERVER\Deploy$`).
2. Buka **Group Policy Management** > buat GPO baru.
3. Masuk ke **Computer Configuration** > **Policies** > **Software Settings** > **Software installation**.
4. Tambahkan package `.msi` tersebut agar terpasang otomatis saat komputer menyala (*startup*).

### 4. Pengelompokan Klien (Endpoint Group)
1. Setelah agen terpasang, pastikan PC muncul di menu **Endpoints** dengan status **Online** (hijau).
2. Masuk ke menu **Groups** > klik **+ Add Group**.
3. Beri nama grup, misalnya `Lab Komputer - Klien Ujian`.
4. Masukkan seluruh PC peserta ujian ke dalam grup ini agar memudahkan pengiriman instruksi sekaligus.

---

## ⚙️ Konfigurasi Direktori (Wajib untuk Publik)

Sebelum menggunakan skrip pembuka, publik/admin lab wajib menyesuaikan path direktori aplikasi di bagian konfigurasi paling atas:

```powershell
# ==========================================
# KONFIGURASI DIREKTORI (SESUAIKAN DENGAN PC ANDA)
# ==========================================
$Dir       = 'C:\Laboran Lab. DKV\ExambrowserClient'   # Folder tempat ExamBrowser.exe berada
$SearchDir = 'C:\Laboran Lab. DKV'                     # Folder root pencarian jika path utama tidak ditemukan
# ==========================================
```

* `$Dir`: Alamat absolut tempat folder aplikasi `ExamBrowser.exe` diekstrak.
* `$SearchDir`: Folder induk yang akan dipindai secara rekursif jika file tidak ditemukan di `$Dir`.

---

## 📜 Skrip Otomasi

### Skrip 1: Buka & Auto-RUN ExamBrowser

Daftarkan skrip ini di Action1 pada menu **Automation** > **Custom Scripts** dengan nama `Launch and Run ExamBrowser`.

```powershell
# ==========================================
# KONFIGURASI DIREKTORI (SESUAIKAN DENGAN PC ANDA)
# ==========================================
$Dir       = 'C:\Path\Folder\ExamBrowserClient'   # Folder tempat ExamBrowser.exe berada
$SearchDir = 'C:\Path\Folder'                     # Folder root pencarian jika path utama tidak ditemukan
# ==========================================

$Exe = Join-Path $Dir 'ExamBrowser.exe'

if (-not (Test-Path $Exe)) {
    $f = Get-ChildItem $SearchDir -Filter 'ExamBrowser.exe' -Recurse -ErrorAction SilentlyContinue | Select-Object -First 1
    if (-not $f) { Write-Output 'ExamBrowser.exe tidak ditemukan'; exit 1 }
    $Exe = $f.FullName; $Dir = $f.DirectoryName
}

if (Get-Process -Name 'ExamBrowser' -ErrorAction SilentlyContinue) {
    Write-Output 'ExamBrowser sudah berjalan'; exit 0
}

$User = (Get-CimInstance Win32_ComputerSystem).UserName
if (-not $User) { Write-Output 'Tidak ada user yang login'; exit 1 }

$HelperDir  = 'C:\ProgramData\Action1Scripts'
$HelperPath = Join-Path $HelperDir 'OpenExamBrowser.ps1'
New-Item -ItemType Directory -Path $HelperDir -Force | Out-Null

$helper = @'
param($Exe, $Dir)
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes
Add-Type @"
using System;
using System.Text;
using System.Runtime.InteropServices;
public class W {
  public delegate bool EP(IntPtr h, IntPtr l);
  [DllImport("user32.dll")] public static extern bool EnumWindows(EP p, IntPtr l);
  [DllImport("user32.dll")] public static extern bool EnumChildWindows(IntPtr h, EP p, IntPtr l);
  [DllImport("user32.dll", CharSet=CharSet.Unicode)] public static extern int GetClassName(IntPtr h, StringBuilder s, int n);
  [DllImport("user32.dll", CharSet=CharSet.Unicode)] public static extern int GetWindowText(IntPtr h, StringBuilder s, int n);
  [DllImport("user32.dll")] public static extern bool IsWindowVisible(IntPtr h);
  [DllImport("user32.dll")] public static extern IntPtr SendMessage(IntPtr h, int m, IntPtr w, IntPtr l);
  [DllImport("user32.dll")] public static extern bool SetCursorPos(int x, int y);
  [DllImport("user32.dll")] public static extern void mouse_event(int f, int dx, int dy, int d, int e);
  public static void Click(int x, int y) {
    SetCursorPos(x, y);
    mouse_event(0x02, 0, 0, 0, 0);
    mouse_event(0x04, 0, 0, 0, 0);
  }
  public static bool ClickOk(string key) {
    bool done = false;
    EnumWindows((h, l) => {
      if (!IsWindowVisible(h)) return true;
      var c = new StringBuilder(64); GetClassName(h, c, 64);
      if (c.ToString() != "#32770") return true;
      bool match = false; IntPtr btn = IntPtr.Zero;
      EnumChildWindows(h, (ch, l2) => {
        var t = new StringBuilder(512); GetWindowText(ch, t, 512);
        var cc = new StringBuilder(64); GetClassName(ch, cc, 64);
        if (t.ToString().IndexOf(key, StringComparison.OrdinalIgnoreCase) >= 0) match = true;
        if (cc.ToString() == "Button" && t.ToString().Replace("&", "").Trim() == "OK") btn = ch;
        return true;
      }, IntPtr.Zero);
      if (match && btn != IntPtr.Zero) {
        SendMessage(btn, 0x00F5, IntPtr.Zero, IntPtr.Zero);
        done = true; return false;
      }
      return true;
    }, IntPtr.Zero);
    return done;
  }
}
"@

$AE = [System.Windows.Automation.AutomationElement]
function Click-Run($tries) {
    $wins = $AE::RootElement.FindAll('Children', [System.Windows.Automation.Condition]::TrueCondition)
    foreach ($w in $wins) {
        if ($w.Current.Name -notmatch '(?i)exambrowser\s+client') { continue }
        foreach ($b in $w.FindAll('Descendants', [System.Windows.Automation.Condition]::TrueCondition)) {
            if ($b.Current.Name -match '^\s*RUN\s*$') {
                $p = $null
                if ($b.TryGetCurrentPattern([System.Windows.Automation.InvokePattern]::Pattern, [ref]$p)) {
                    $p.Invoke()
                } else {
                    $r = $b.Current.BoundingRectangle
                    [W]::Click([int]($r.X + $r.Width / 2), [int]($r.Y + $r.Height / 2))
                }
                return $true
            }
        }
        # Cadangan: jika tombol belum terbaca UIA, klik area bawah jendela
        if ($tries -ge 10) {
            $r = $w.Current.BoundingRectangle
            [W]::Click([int]($r.X + $r.Width / 2), [int]($r.Y + $r.Height * 0.90))
            return $true
        }
    }
    return $false
}

Start-Process -FilePath $Exe -WorkingDirectory $Dir

$popupDone = $false
for ($i = 0; $i -lt 180; $i++) {
    Start-Sleep -Seconds 1
    if (-not $popupDone) { $popupDone = [W]::ClickOk('autoupdate') }
    if (Click-Run $i) { break }
}
'@
Set-Content -Path $HelperPath -Value $helper -Encoding UTF8

$Task     = 'Action1_OpenExamBrowser'
$taskArgs = "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$HelperPath`" -Exe `"$Exe`" -Dir `"$Dir`""
$act = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument $taskArgs
$pri = New-ScheduledTaskPrincipal -UserId $User -LogonType Interactive -RunLevel Highest
$set = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit (New-TimeSpan -Minutes 10)

Unregister-ScheduledTask -TaskName $Task -Confirm:$false -ErrorAction SilentlyContinue
Register-ScheduledTask -TaskName $Task -Action $act -Principal $pri -Settings $set | Out-Null
Start-ScheduledTask -TaskName $Task
Write-Output "ExamBrowser dijalankan untuk $User"
```

---

### Skrip 2: Tutup Aman ExamBrowser (Ctrl+C+B)

Daftarkan skrip ini di menu **Automation** > **Custom Scripts** dengan nama `Close ExamBrowser (Ctrl+C+B)`.

```powershell
$User = (Get-CimInstance Win32_ComputerSystem).UserName
if (-not $User) { Write-Output 'Tidak ada user yang login'; exit 1 }

$HelperDir  = 'C:\ProgramData\Action1Scripts'
$HelperPath = Join-Path $HelperDir 'CloseExamBrowser.ps1'
New-Item -ItemType Directory -Path $HelperDir -Force | Out-Null

$helper = @'
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class K {
  [DllImport("user32.dll")] public static extern void keybd_event(byte vk, byte scan, int flags, int extra);
  [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);
  [DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr h, int c);
}
"@

# Fokuskan jendela ExamBrowser (jika ada), lalu kirim kombinasi tombol Ctrl + C + B
$p = Get-Process -Name 'ExamBrowser', 'runningexam' -ErrorAction SilentlyContinue |
     Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1
if ($p) {
    [K]::ShowWindow($p.MainWindowHandle, 9) | Out-Null
    [K]::SetForegroundWindow($p.MainWindowHandle) | Out-Null
    Start-Sleep -Milliseconds 500
}

$keys = 0x11, 0x43, 0x42    # Virtual Key Codes: Ctrl, C, B
foreach ($k in $keys) { [K]::keybd_event($k, 0, 0, 0); Start-Sleep -Milliseconds 100 }
Start-Sleep -Milliseconds 500
[array]::Reverse($keys)
foreach ($k in $keys) { [K]::keybd_event($k, 0, 2, 0); Start-Sleep -Milliseconds 50 }
'@
Set-Content -Path $HelperPath -Value $helper -Encoding UTF8

$Task     = 'Action1_CloseExamBrowser'
$taskArgs = "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$HelperPath`""
$act = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument $taskArgs
$pri = New-ScheduledTaskPrincipal -UserId $User -LogonType Interactive -RunLevel Highest
$set = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit (New-TimeSpan -Minutes 2)

Unregister-ScheduledTask -TaskName $Task -Confirm:$false -ErrorAction SilentlyContinue
Register-ScheduledTask -TaskName $Task -Action $act -Principal $pri -Settings $set | Out-Null
Start-ScheduledTask -TaskName $Task

Start-Sleep -Seconds 5
Unregister-ScheduledTask -TaskName $Task -Confirm:$false -ErrorAction SilentlyContinue
Write-Output "Ctrl+C+B dikirim ke sesi $User"
```

---

## 🚀 Cara Eksekusi Massal dari Dashboard

1. Buka menu **Endpoints** atau **Groups** di dashboard Action1.
2. Centang grup komputer target (misal: `Lab Komputer - Klien Ujian`).
3. Klik tombol **Run Action** di bar atas > pilih **Run Script**.
4. Pilih skrip yang dibutuhkan:
   * **Saat Ujian Mulai**: Pilih `Launch and Run ExamBrowser`.
   * **Saat Ujian Selesai**: Pilih `Close ExamBrowser (Ctrl+C+B)`.
5. Klik **Run Now**.
6. Pantau jendela status Action1:
   * Hasil sukses buka: `ExamBrowser dijalankan untuk [NamaUser]`
   * Hasil sukses tutup: `Ctrl+C+B dikirim ke sesi [NamaUser]`

---

## 🎁 Bonus: Skrip Shutdown Massal Klien Lab via Action1

Setelah sesi ujian berakhir dan aplikasi ExamBrowser ditutup, Anda dapat mematikan seluruh PC klien lab secara serempak langsung dari dashboard Action1 dengan dialog peringatan hitung mundur (*countdown*).

### 1. Skrip PowerShell: Shutdown dengan Peringatan 30 Detik

Daftarkan skrip berikut di Action1 melalui menu **Automation** > **Custom Scripts** dengan nama `Shutdown Lab Client with Warning`:

```powershell
# Parameter shutdown Windows:
# /s : Mematikan komputer (shutdown)
# /f : Memaksa aplikasi tertutup tanpa menunggu konfirmasi (force)
# /t 30 : Waktu jeda hitung mundur 30 detik
# /c : Pesan dialog yang muncul di layar monitor klien

$msg = "Perhatian: Komputer akan dimatikan dalam 30 detik. Silakan simpan pekerjaan Anda!"
shutdown.exe /s /f /t 30 /c $msg

Write-Output "Perintah shutdown hitung mundur 30 detik berhasil dikirim."
```

### 2. Skrip Pembatalan: Cancel Shutdown (Jika Salah Eksekusi)

Jika sewaktu-waktu Anda tidak sengaja mengirim perintah shutdown atau perlu menundanya, buat skrip darurat di Action1 dengan nama `Cancel Shutdown`:

```powershell
# Membatalkan proses shutdown yang sedang hitung mundur
shutdown.exe /a

Write-Output "Perintah shutdown berhasil dibatalkan."
```

### 3. Cara Penggunaan di Dashboard Action1

1. Masuk ke menu **Endpoints** atau **Groups**.
2. Centang grup PC lab yang dituju (misal: `Lab Komputer - Klien Ujian`).
3. Klik tombol **Run Action** di bar atas > pilih **Run Script**.
4. Pilih skrip `Shutdown Lab Client with Warning`, lalu klik **Run Now**.
5. Di layar monitor seluruh siswa/klien akan muncul dialog bawaan Windows dengan pesan peringatan dan hitung mundur sebelum PC mati otomatis.

---

## ❓ Troubleshooting

| Gejala Masalah | Kemungkinan Penyebab | Solusi |
| :--- | :--- | :--- |
| `Tidak ada user yang login` | Komputer klien masih berada di layar Lock Screen atau belum login Windows. | Pastikan siswa/proktor sudah login desktop sebelum aksi dijalankan. |
| `ExamBrowser.exe tidak ditemukan` | Folder aplikasi tidak sesuai dengan path yang didefinisikan. | Periksa letak folder di komputer klien dan sesuaikan variabel `$Dir` atau `$SearchDir` di skrip. |
| Tombol RUN tidak terklik otomatis | UI ExamBrowser merender lambat atau terhalang dialog lain. | Skrip memiliki batas tunggu hingga 180 detik dan otomatis beralih ke simulasi koordinat klik di detik ke-10. |
| Agent Action1 berstatus Offline | PC mati, jaringan LAN/internet lab putus, atau service terhenti. | Cek status service Windows `Action1 Agent` di client atau cek sambungan internet lab. |

---

## 📄 Lisensi

Proyek ini didistribusikan di bawah lisensi **MIT License**. Silakan gunakan, modifikasi, dan bagikan secara bebas untuk kebutuhan laboratorium dan instansi pendidikan Anda.
