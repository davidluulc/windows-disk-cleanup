# Baseline (directory sizes per round — update in Phase 6)

> Machine: Windows 11 dev laptop, 16GB RAM, C: 449.5GB total.
> All sizes in GB unless noted. "~" = user profile. Measured with admin PowerShell unless noted.

## 2026-09-26 (round 6, disk free 84.8 pre / 89.5 post)

| Location | Size |
|---|---|
| C:\Program Files (x86) | 159.27 (Steam=139.8: ELDEN RING 52.15, R6 53.95, DARK SOULS III 24.82 [new 9/26], wallpaper_engine 2.07) |
| C:\Users | 100.20 |
| C:\WINDOWS | 37.67 |
| C:\Program Files | 15.86 |
| C:\ProgramData | 9.80 |
| ~\AppData | 79.31 (Local: wsl 17.5, Programs 5.8, Packages 5.2, Google 4.3, Docker 3.3, Microsoft 3.2, Kingsoft 2.9, NVIDIA 2.1; Roaming: LarkShell 6.1, Tencent 3.0, baidu 1.9, Code 1.6, kingsoft 1.4, bilibili 1.3) |
| ~\Videos | 2.89 |
| ~\.vscode | 1.70 |
| ~\.zcode | 1.65 (cli 1.23) |
| ~\.cache | 1.56 (all codex-runtimes, deleted) |
| ~\.codex | 1.38 |
| pagefile.sys | 15.21 (fixed 8-16GB setting, hit max — RAM pressure) |
| ext4.vhdx | 17.12 → 16.05 (post-compact) |
| docker_data.vhdx | 2.98 |
| X: none | D/E/F absent |

## 2026-09-19 (round 5, disk free 91.9)

| Location | Size |
|---|---|
| Users | 109.25 |
| Program Files (x86) | 131.14 (Steam: ELDEN 52.15, R6 51.97) |
| ext4.vhdx | 20.53 / docker_data.vhdx 5.44 |
| pagefile.sys | not yet re-allocated (fixed setting just applied) |
| AppData Roaming: LarkShell 5.93, ZCode 3.43, Tencent 3.16, Code 1.40, bilibili 1.32, kingsoft 1.15, Telegram 1.04, douyin 0.72 |

## 2026-09-09 (round 4, disk free 121.2 post-reboot)

| Location | Size |
|---|---|
| pagefile.sys | fixed 8192-16384MB applied (was 35.26 system-managed!) |
| hiberfil.sys | deleted (powercfg /h off) |
| disk free | 121.2 (round recovered 37.2) |

## Historical anchors

- 2026-08-06 (round 1 start): disk free 85GB, used 82%
- 2026-08-28 (round 2): free 104.9 → post-compact
- 2026-09-02 (round 3): free 110.1 (eSupport 5GB + webcast_mate uninstalled)
- Steam: R6 seasonal rewrites ~+2GB; new games = big deltas (DARK SOULS III +24.8GB)
- RAM: 15.2GB, commit demand ~27GB — pagefile hits 16GB cap under load; RAM upgrade discussion pending
