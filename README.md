# 🚀 Panduan Automasi ExamBrowser Menggunakan Action1 RMM

Repositori/panduan ini berisi kumpulan skrip PowerShell siap pakai untuk menjalankan (*start & run*) serta menutup (*graceful exit*) aplikasi **ExamBrowser Client** (ANBK/CBT) secara massal melalui dashboard **Action1**.

---

## 📌 Ringkasan Fitur

1. **Buka & Otomatisasi Tombol RUN (`OpenExamBrowser.ps1`)**:
   - Mendeteksi user Windows yang sedang login (interaktif).
   - Menjalankan file executable `ExamBrowser.exe`.
   - Mengabaikan/menekan konfirmasi dialog (seperti notifikasi *autoupdate*).
   - Mengklik tombol **RUN** otomatis via UI Automation & simulasi koordinat klik.
   - Dapat disesuaikan foldernya oleh publik.
2. **Keluar / Tutup Aman (`CloseExamBrowser.ps1`)**:
   - Membawa jendela ExamBrowser ke latar depan (*foreground*).
   - Mengirim kombinasi tombol standar `Ctrl + C + B` untuk keluar secara normal tanpa *force kill* yang merusak sesi ujian.

---

## ⚙️ Prasyarat (Prerequisites)

- Endpoint klien Windows 10/11 terhubung ke **Action1 Agent**.
- Pengguna/Siswa sudah login ke Windows desktop (karena UI Automation membutuhkan sesi interaktif pengguna).
- Hak akses Administrator di dashboard Action1.

---

## 🛠️ Persiapan & Kustomisasi Direktori (Wajib Dibaca)

Sebelum mendistribusikan skrip pembuka, pastikan publik/admin lab menyesuaikan path direktori aplikasi di bagian **paling atas** skrip `OpenExamBrowser.ps1`:

```powershell
# ==========================================
# KONFIGURASI DIREKTORI (SESUAIKAN DENGAN PC ANDA)
# ==========================================
$Dir       = 'C:\Laboran Lab. DKV\ExambrowserClient'   # Folder tempat ExamBrowser.exe berada
$SearchDir = 'C:\Laboran Lab. DKV'                     # Folder root pencarian jika path utama tidak ditemukan
# ==========================================
```

> **Catatan:**
> - `$Dir`: Alamat absolut tempat folder ExamBrowser Client diekstrak.
> - `$SearchDir`: Folder induk untuk pencarian darurat jika folder utama bergeser atau berbeda nama subfolder.

---

## 📋 Langkah-Langkah Deploy di Action1

### Langkah 1: Buat Script "Jalankan ExamBrowser"

1. Masuk ke dashboard **Action1**.
2. Buka menu **Automation** > **Custom Scripts** (atau via **Run PowerShell Script**).
3. Klik **+ Add Script**.
4. Isi detail:
   - **Name**: `Launch and Run ExamBrowser`
   - **Description**: `Membuka ExamBrowser dan otomatis klik RUN di sesi user yang aktif`
5. Salin dan tempel kode berikut:

<details>
<summary>▶ Klik untuk melihat skrip Jalankan ExamBrowser</summary>

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
        # Cadangan jika tombol tidak terbaca UIA
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

</details>

---

### Langkah 2: Buat Script "Tutup ExamBrowser (Ctrl+C+B)"

1. Di menu **Custom Scripts**, klik **+ Add Script**.
2. Isi detail:
   - **Name**: `Close ExamBrowser (Ctrl+C+B)`
   - **Description**: `Mengirim shortcut Ctrl+C+B secara interaktif ke jendela ExamBrowser`
3. Salin dan tempel kode berikut:

<details>
<summary>▶ Klik untuk melihat skrip Tutup ExamBrowser</summary>

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

# Fokuskan jendela ExamBrowser (jika ada), lalu kirim Ctrl + C + B
$p = Get-Process -Name 'ExamBrowser', 'runningexam' -ErrorAction SilentlyContinue |
     Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1
if ($p) {
    [K]::ShowWindow($p.MainWindowHandle, 9) | Out-Null
    [K]::SetForegroundWindow($p.MainWindowHandle) | Out-Null
    Start-Sleep -Milliseconds 500
}

$keys = 0x11, 0x43, 0x42    # Virtual-Key codes: Ctrl, C, B
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

</details>

---

## 🚀 Cara Menjalankan ke Klien Lab

1. Dari dashboard Action1, masuk ke menu **Endpoints**.
2. Pilih komputer lab target (atau pilih grup komputer lab).
3. Klik tombol **Run Action** > **Run Script**.
4. Pilih script:
   - Pilih `Launch and Run ExamBrowser` saat ujian hendak dimulai.
   - Pilih `Close ExamBrowser (Ctrl+C+B)` saat sesi ujian telah berakhir.
5. Klik **Run Now**.
6. Pantau output status pada panel Action1:
   - Output sukses buka: `ExamBrowser dijalankan untuk [NamaUser]`
   - Output sukses tutup: `Ctrl+C+B dikirim ke sesi [NamaUser]`

---

## ❓ Troubleshooting

| Gejala Masalah | Penyebab | Solusi |
|---|---|---|
| `Tidak ada user yang login` | Komputer klien masih berada di layar login Windows (Lock/Logoff). | Pastikan siswa/proktor sudah login ke desktop sebelum skrip dijalankan. |
| `ExamBrowser.exe tidak ditemukan` | Folder penempatan aplikasi berbeda dengan isi variabel `$Dir` atau `$SearchDir`. | Pastikan path pada bagian konfigurasi script sudah sesuai dengan letak folder di komputer klien. |
| Tombol RUN tidak terklik otomatis | Resolusi layar ekstrem atau UI lambat merender tombol. | Skrip memiliki toleransi tunggu hingga 180 detik dan otomatis melakukan klik koordinat cadangan di detik ke-10 ke atas. |
