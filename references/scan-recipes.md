# Scan Recipes (read when building scan scripts)

All scripts must: start with `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`, set `$ErrorActionPreference = 'SilentlyContinue'`, use English output labels (Chinese literals corrupt — see pitfalls.md), and be pure read-only.

## 1. Layered size scan

```powershell
function Size-GB { param($p) if(-not(Test-Path -LiteralPath $p)){return -1}; $s=(Get-ChildItem -LiteralPath $p -Recurse -Force -File -ErrorAction SilentlyContinue|Measure-Object Length -Sum).Sum; if($s){[math]::Round($s/1GB,2)}else{0} }
# C:\ root dirs >0.5GB → user profile dirs >0.3GB → AppData\Local & Roaming top-15 → ProgramData top-10
# big files: >300MB with LastWriteTime in last N days (finds what changed since baseline)
```

Note: non-admin scans under-count permission-protected dirs (e.g. Program Files\WindowsApps differs ~13GB between admin/non-admin). Compare like with like.

## 2. Invisible-space audit (when folders don't add up)

```powershell
Get-ChildItem 'C:\' -File -Force   # pagefile.sys / hiberfil.sys / swapfile.sys live here as FILES
fsutil volume diskfree C:          # true used/free + "Total Reserved bytes" (Reserved Storage ~5-7GB)
# vssadmin list shadowstorage      # restore-point usage (needs admin)
# vssadmin list shadows            # per-shadow-copy creation times
# check C:\Recovery (keep it — WinRE), C:\Windows.old, C:\$WINDOWS.~BT (update leftovers, deletable)
```

Gap analysis: sum(visible dirs) vs Used bytes. Difference = pagefile + hiberfil + SVI + reserved storage + permission-hidden dirs.

## 3. Ghost services

```powershell
$services = Get-CimInstance Win32_Service
foreach ($svc in $services) {
    $exe = $svc.PathName
    if ($path -match '^"([^"]+)"') { $exe = $Matches[1] }
    elseif ($path -match '^(\S+\.exe)') { $exe = $Matches[1] }
    if ($exe -notmatch '^[A-Z]:\\') { continue }
    if ($exe -match '\\Windows\\System32\\|\\SystemRoot\\') { continue }   # system stuff, skip
    if (-not (Test-Path -LiteralPath $exe)) { <# GHOST: report Name, State, StartMode, missing path #> }
}
```

## 4. Orphan uninstall entries + dead Run keys + ghost tasks

```powershell
# orphans: DisplayName set but InstallLocation gone → output PSChildName + root for EXACT-path deletion later
# dead Run keys: HKCU/HKLM ...\CurrentVersion\Run entries whose exe path no longer exists
# ghost scheduled tasks: Get-ScheduledTask | State -ne Disabled, actions whose Execute path is gone (match 'OurPlay' style by wildcard, Chinese task names break literals)
```

## 5. Cache-regrowth quick list

**The authoritative full list now lives in `cache-map.md` (read it every round — mandatory).** The entries below are the round-1 legacy subset, kept for backwards compatibility:

| Path | Note |
|---|---|
| `%APPDATA%\kingsoft\wps\addons\pool` | WPS plugin cache, regrows to 1-2GB in ~1 week |
| `%LOCALAPPDATA%\Google\Chrome\User Data\OptGuideOnDeviceModel` | must stay absent (policy-guarded) |
| `C:\ProgramData\NVIDIA Corporation\NVIDIA app\UpdateFramework\ota-artifacts` | driver installer leftovers |
| `%LOCALAPPDATA%\Temp` (skip dirs containing swap.vhdx), CrashDumps, C:\Windows\Temp/Logs | routine |
| Docker: `docker system df` + dangling images + volumes with LINKS=0 | anonymous orphan volumes |
