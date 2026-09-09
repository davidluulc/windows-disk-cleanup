---
name: windows-disk-cleanup
description: Windows C盘深度清理与空间回收的完整工作流。Use whenever the user mentions C盘满了/空间不足/清理磁盘/释放空间/disk cleanup/磁盘爆了/"又该清理了"/电脑变卡/清理垃圾 — covers temp files, WSL/Docker vhdx compaction, ghost services, orphan registry entries, pagefile/hiberfil bloat, rogue software, UWP bloatware, and cache regrowth. Even if the user just asks "看看有什么能删的" without saying "cleanup" explicitly.
---

# Windows Disk Cleanup (Windows 磁盘深度清理)

A battle-tested workflow for reclaiming C-drive space on a developer's Windows 11 machine (WSL2 + Docker + AI coding tools). Built from four real cleanup rounds recovering ~82GB total.

## Core principles (read first)

1. **Scan → Report → Confirm per-item → Execute → Verify.** Never delete anything without the user explicitly approving each category. This is destructive work; the user's data outranks reclaiming space.
2. **Read-only scans only** until cleanup is confirmed. All scan scripts are pure queries.
3. **Protect lists are sacred.** Before any deletion, identify and exclude: dev environments (`.codex`, `.vscode`, `.claude`, `.zcode`, `.gradle`, WSL/Docker data), game installs, chat app data (QQ/WeChat/Feishu history), the user's projects (look for `.git`, `node_modules`, Makefiles before calling a folder "junk"), Steam games.
4. **Prefer official uninstallers over deleting folders.** Deleting a program folder leaves registry entries, services, and scheduled tasks behind (they become "ghosts"). Use the UninstallString from the registry; `Remove-AppxPackage` for UWP.
5. **Scripts are disposable.** Write each PowerShell script to the workspace, syntax-check it (`[System.Management.Automation.Language.Parser]::ParseFile`), run it, then delete it. Do not inline PowerShell in bash — bash eats `$vars`.

## Workflow

### Phase 1 — Scan (read-only, ~10 min)

Run these in order; each maps to a recipe in `references/scan-recipes.md`:

1. **Quick triage**: disk free %, `docker system df`, vhdx file sizes vs their last-known size.
2. **Layered size scan**: C:\ root → user profile → AppData\Local/Roaming top-N → biggest files >300MB modified recently. Compare against previous-round numbers if known (the agent should keep/rebuild a baseline).
3. **Invisible-space audit** when visible folders don't explain disk usage: `Get-ChildItem C:\ -File -Force` reveals pagefile.sys/hiberfil.sys (these are FILES, missed by directory scans); `vssadmin list shadowstorage`; `fsutil volume diskfree C:`; check `C:\Recovery`, `$Recycle.Bin`. A gap between "sum of visible folders" and "used bytes" usually means pagefile/hiberfil/restore points/permission-hidden dirs (admin reads more than non-admin — same folder can differ by 10GB+).
4. **Ghost detection**: services whose exe is missing (recipe in scan-recipes.md), orphan uninstall entries whose InstallLocation no longer exists, scheduled tasks pointing at deleted exes, Run-key entries with dead targets.
5. **Cache-regrowth check** for previously-cleaned items (WPS addons\pool, Chrome AI model, NVIDIA OTA, eSupport) — they come back; report but don't auto-delete.

### Phase 2 — Report

Categorize findings into: (A) system files to fix (pagefile fixed-size, hiberfil off), (B) safe junk (temp/crash dumps/logs/recycle bin), (C) app caches (WPS pool, KOOK old versions, Ubisoft webcache), (D) needs-user-judgment (chat app data → clean inside the app, games, projects), (E) protected/keep. Always show expected GB per item and total. Use AskUserQuestion for per-category decisions — the user has overturned my categorization before and was right to.

### Phase 3 — Execute

- User runs admin scripts themselves when elevation is needed (pattern: write script → syntax check → give exact one-liner `& "path.ps1"` → user pastes output back). `Start-Process -Verb RunAs` UAC popups get dismissed; don't rely on them.
- Batch independent deletions in one script with per-item logging (`before/after/freed MB`), LinkType guard on every Remove-Item (junctions like "Application Data" must never be deleted), and a final verify section.
- For registry deletion: first *query exact key paths* (PSChildName), then delete by exact path — never by keyword match on DisplayName (Chinese names break, keywords over-match).

### Phase 4 — Verify & close

Re-measure disk free, re-check each target is gone, list what was kept and why, remind about pending reboots (pagefile resize needs one) and how to restore services (see pitfalls.md for the Docker/WSL restore procedure).

## Hard-won rules (the short list)

- **pagefile.sys ballooning to 20-35GB** is caused by sustained RAM pressure + "system managed size". Fix: disable AutomaticManagedPagefile, set Win32_PageFileSetting Initial 8192 / Max 16384, reboot. A bloated pagefile is also a signal to discuss RAM upgrades.
- **hiberfil.sys reappears after major Windows updates** even if previously disabled. Re-run `powercfg /h off`.
- **Deleting Chrome's on-device AI model (weights.bin ~4GB) is pointless alone** — it re-downloads. Set policy `HKLM:\SOFTWARE\Policies\Google\Chrome\OptimizationGuideOnDeviceModelDefault=0` first, then delete.
- **vhdx compaction order matters**: kill Docker processes with a BROAD pattern (`Name -match 'docker|vmmem|wslhost|wslservice|^wsl'` — exact-match regex misses `com.docker.backend`, the usual lock holder), restart `vmcompute` service, `wsl --shutdown`, then `diskpart compact vdisk`. See pitfalls.md.
- **Never bare `docker compose up` to restore a stopped compose stack** — env vars like `${MINIO_LOCAL_IMAGE:-...}` resolve to defaults and break local-image stacks. Use the project's make target (e.g. `SKIP_OCR_IMAGE=1 make -f infra/backend.mk local-infra-up`).
- **Stuck service (Stop-Pending forever)**: `taskkill /F /PID <pid>` then `sc.exe delete`. Stop-Service politely waits forever on broken services.
- **Rogue software signature**: registry Publisher/DisplayName containing mojibake garbage = bundled malware-ish; its uninstaller may delete files but leave registry — clean both.
- **Chinese literals in .ps1 files corrupt on save/run** (this environment). Write script labels in English; build Chinese paths via `[char]0xXXXX` concatenation or match by listing+filtering, not by typing the literal.
- **Don't fight unwinnable UI automation**: elevated Task Manager blocks UIPI reads; its per-second refresh stales screenshot frames. Use PowerShell as the same data source instead.

## When the user says "scan again" (repeat rounds)

Rebuild the comparison baseline from the last round's report. The usual suspects in order: vhdx growth (if they pulled images/projects), pagefile/hiberfil (after updates), WPS pool, temp regrowth, a new app's cache, games updating. Historical fix-points (Chrome AI policy, ghost services) rarely recur — verify cheaply, then focus on what's new.
