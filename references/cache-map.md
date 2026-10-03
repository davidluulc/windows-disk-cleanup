# Cache Map (full regression checklist — read BEFORE every scan)

> MANDATORY: check EVERY entry below during Phase 1's cache-regrowth step.
> `Confirm date` = last verified real finding. New hotspots get appended in Phase 6.

## High-frequency regressors (confirmed multiple times)

| Path | What it is | Typical size | Regrow cycle | Confirm date |
|---|---|---|---|---|
| `%APPDATA%\kingsoft\wps\addons\pool` | WPS plugin installer cache (safe; active plugin data is in `addons\data` — do NOT touch) | 0.6-2.2 GB | ~1 week | 2026-09-26 (876MB) |
| `C:\ProgramData\NVIDIA Corporation\NVIDIA app\UpdateFramework\ota-artifacts` | Driver/app installer leftovers after install | 0.9-3.7 GB | after each driver/app update | 2026-09-26 (874MB) |
| `%LOCALAPPDATA%\Temp` | Generic temp; SKIP dirs containing `swap.vhdx` (active WSL swap) | 0.4-3.7 GB | continuous | 2026-09-26 (413MB) |
| `%LOCALAPPDATA%\CrashDumps` | Crash dumps | 45-450 MB | after crashes | 2026-09-26 (46MB) |
| `C:\Windows\Logs` | System logs (admin) | 50-220 MB | continuous | 2026-09-26 (71MB) |
| `C:\$Recycle.Bin` | Preview contents before emptying (`Clear-RecycleBin -DriveLetter C -Force`) | 0-5 GB | user deletions | 2026-09-26 (693MB) |
| `C:\Windows\SoftwareDistribution\Download` | Windows Update leftovers (admin; stop wuauserv first) | 0-50 MB | after updates | 2026-08-06 |
| `%LOCALAPPDATA%\Temp\<GUID>\swap.vhdx` (stale ones) | WSL swap remnants after shutdown — only deletable when WSL is fully down | 0.04-2 GB | every WSL session | 2026-09-09 |

## App version coexistence (delete OLD versions, keep newest)

| Path | Pattern | Confirm date |
|---|---|---|
| `%LOCALAPPDATA%\KOOK\app-*` | Multiple versions coexist; delete all but the newest (`app-0.95.1` 461MB deleted 2026-09-26, newest was 0.110.0) | 2026-09-26 |
| `%LOCALAPPDATA%\Kingsoft\WPS Office\<version>` | WPS self-cleans old versions after updates — verify before touching; do NOT delete the version whose wps.exe is running | 2026-08-06 |

## AI-tool caches (regrow fast on this machine)

| Path | What it is | Typical size | Confirm date |
|---|---|---|---|
| `%USERPROFILE%\.cache\codex-runtimes` | Codex runtime binaries (auto-redownloads; Codex must be closed; `config.toml` lives in `~/.codex` and is untouched) | 1.3-1.6 GB | 2026-09-26 (1602MB) |
| `%APPDATA%\ZCode\remote-assets-cache` | ZCode UI remote resources, grows every update (components + releases). ZCode must be CLOSED. Session history lives in `%APPDATA%\ZCode\session` — NEVER touch | 0-3 GB | 2026-09-19 (3028MB) |
| `%LOCALAPPDATA%\ms-playwright` | Playwright test browsers; `npx playwright install` restores | 0.9-1.2 GB | 2026-08-06 (926MB) |
| `%USERPROFILE%\.codex` | Codex data: SAFE to clean = `.tmp`, `logs_2.sqlite`(+wal/shm, two locations incl. `sqlite\`), `backups`, `backups_state`, `archived_sessions`; NEVER: `sessions`, `auth.json`, `config.toml`, `plugins` | 1.2-2.2 GB | 2026-09-02 (1000MB) |

## GPU / game shader caches (regrow on every game session; check game NOT running)

| Path | What it is | Typical size | Confirm date |
|---|---|---|---|
| `%LOCALAPPDATA%\NVIDIA\DXCache` | DirectX shader cache — grows fast with new games; deleting causes shader recompile stutter on next launch | 0.5-2.1 GB | 2026-09-26 (2100MB, skipped: RainbowSix running) |
| `%LOCALAPPDATA%\NVIDIA\GLCache`, `%LOCALAPPDATA%\D3DSCache` | OpenGL/D3D shader caches | 10-500 MB | 2026-08-06 |

## Net-disk / media app caches (prefer in-app cleaning; folder deletion loses downloads)

| Path | What it is | Typical size | Confirm date |
|---|---|---|---|
| `%APPDATA%\LarkShell\aha\users\<uid>\profile_main\WebStorage` | Feishu embedded-web (workbench/mini-app) local storage — **the real 2GB bomb**; standard Cache dirs only hold ~550MB | 2 GB | 2026-09-30 (2005MB) |
| `%APPDATA%\LarkShell\aha\users\<uid>\profile_explorer\Service Worker` | Feishu SW resource cache — pure web cache, zero-risk delete | 1.7 GB | 2026-09-30 (1740MB) |
| `%APPDATA%\LarkShell\aha\users\<uid>\doubao\sandbox_envs_dir` | Feishu built-in AI (Doubao) sandbox runtime copy — auto-rebuilds on next use | 0.8 GB | 2026-09-30 (761MB) |
| `%APPDATA%\baidu\BaiduNetdisk` | ⚠️ TRAP: this is the PROGRAM INSTALL DIR (BaiduNetdisk.exe, kernel.dll, module\=1.27GB components). Only `AutoUpdate` (~58MB) is cleanable. Do NOT delete module or the exe files | 1.8 GB total, ~0.06 cleanable | 2026-09-30 |
| `%APPDATA%\bilibili\IndexedDB` | Bilibili client playback/preload data — the 1.1GB bomb (login stays in Local Storage) | 1.1 GB | 2026-09-30 (1123MB) |
| `%APPDATA%\Tencent` | QQ data — in-app clean only | 2.4-3.3 GB | 2026-09-26 (2983MB) |
| `%APPDATA%\bilibili`, `%APPDATA%\douyin` | Media app caches (bilibili structure: ffmpeg/player/resource = program, do not touch) | 0.7-1.3 GB each | 2026-09-19 |
| `%APPDATA%\Telegram Desktop` | Telegram cache — in-app clean | ~1 GB | 2026-09-19 |
| `%USERPROFILE%\xwechat_files` | WeChat files — in-app clean (settings → file management). USER PROHIBITED from automated cleaning (2026-09-30) | 2.9-4.1 GB | 2026-09-19 |

### Feishu deep-cleaning pattern (learned 2026-09-30)

Feishu's `aha\users\<uid>\profile_*` dirs are Chromium profiles; the obvious `Cache` subdirs hold only ~550MB. The real mass hides in non-standard dirs: **WebStorage** (embedded-web storage), **Service Worker** (SW cache), and doubao's **sandbox_envs_dir**. Standard-cache-name matching misses 4.5GB of 5.9GB. When an Electron app's known caches come up short, drill one level into the profiles and chase the biggest subdirs by NAME CATEGORY: SW caches and sandbox/runtime copies = safe; WebStorage = resets embedded-app local state only (chat/login unaffected, login lives in Cookies). Total achievable: 5.9GB → 0.77GB (−87%).

## WSL-internal caches (clean INSIDE Ubuntu before compacting vhdx)

| Path (inside Ubuntu) | Command | Typical size | Confirm date |
|---|---|---|---|
| `~/.cache/codex-runtimes` | `rm -rf ~/.cache/codex-runtimes` | 1.3-2.9 GB | 2026-09-26 |
| `~/.npm/_cacache` | `rm -rf ~/.npm/_cacache` | 0.7 GB | 2026-09-26 |
| `/var/cache/apt` | needs sudo (`sudo apt-get clean`) — skip if passwordless sudo unavailable | ~100 MB | 2026-09-26 |
| `~/.cache` other subdirs | list first (`du -sh ~/.cache/*`), only delete known caches | varies | — |

## One-shot / rare items (verify cheaply each round)

| Path | Note | Confirm date |
|---|---|---|
| `C:\eSupport` | ASUS factory driver backup — safe to delete, drivers already installed | 2026-09-02 (5GB deleted) |
| `%LOCALAPPDATA%\Google\Chrome\User Data\OptGuideOnDeviceModel` | Chrome on-device AI model — must stay ABSENT (policy-guarded; see pitfalls) | 2026-09-26 (OK absent) |
| `%LOCALAPPDATA%\Ubisoft Game Launcher\cache` | Ubisoft launcher webcache (game configs in r6s/r6siege dirs are NOT cache — keep) | 2026-09-02 (607MB) |
| `%LOCALAPPDATA%\Google\Chrome\User Data` | Browser data — grows steadily; clean browsing cache in-browser, keep bookmarks/passwords | 4.2 GB @ 2026-09-26 |
