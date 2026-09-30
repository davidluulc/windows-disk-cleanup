---
name: windows-disk-cleanup
description: Windows C盘深度清理与空间回收的完整工作流。Use whenever the user mentions C盘满了/空间不足/清理磁盘/释放空间/disk cleanup/磁盘爆了/"又该清理了"/电脑变卡/清理垃圾 — covers temp files, WSL/Docker vhdx compaction, ghost services, orphan registry entries, pagefile/hiberfil bloat, rogue software, UWP bloatware, and cache regrowth. Even if the user just asks "看看有什么能删的" without saying "cleanup" explicitly.
---

# Windows Disk Cleanup (Windows 磁盘深度清理)

A battle-tested workflow for reclaiming C-drive space on a developer's Windows 11 machine (WSL2 + Docker + AI coding tools). Built from six real cleanup rounds recovering ~90GB total, validated by an A/B experiment (skill flow vs free-form scan).

## Core principles (read first)

1. **Scan → Report → Safety Pre-check → Confirm per-item → Execute → Verify → Write-back.** Never delete anything without the user explicitly approving each category. This is destructive work; the user's data outranks reclaiming space.
2. **Read-only scans only** until cleanup is confirmed. All scan scripts are pure queries.
3. **Protect lists are sacred.** Before any deletion, identify and exclude: dev environments (`.codex`, `.vscode`, `.claude`, `.zcode`, `.gradle`, WSL/Docker data), game installs, chat app data (QQ/WeChat/Feishu history), the user's projects (look for `.git`, `node_modules`, Makefiles before calling a folder "junk"), Steam games.
4. **Prefer official uninstallers over deleting folders.** Deleting a program folder leaves registry entries, services, and scheduled tasks behind (they become "ghosts"). Use the UninstallString from the registry; `Remove-AppxPackage` for UWP.
5. **Scripts are disposable.** Write each PowerShell script to the workspace, syntax-check it (`[System.Management.Automation.Language.Parser]::ParseFile`), run it, then delete it. Do not inline PowerShell in bash — bash eats `$vars`.
6. **READ `references/cache-map.md` BEFORE Phase 1.** This is mandatory, not "read on demand" — an execution trap found in validation was skipping the reference file and missing 4GB of real findings (KOOK old version, NVIDIA DXCache, codex-runtimes, WSL-internal caches). The cache map is the full regression checklist; the list in this file is only the high-frequency subset.
7. **Privileged programs cannot be audited from a non-elevated agent** (UIPI). Elevated Task Manager also stales screenshot frames. Use PowerShell as the data source instead of fighting it.

## Workflow

### Phase 1 — Scan (read-only, ~10 min)

Run these in order; scripts follow the patterns in `references/scan-recipes.md`:

1. **Quick triage**: disk free %, `docker system df` (if Docker runs), vhdx file sizes vs `references/baseline.md`.
2. **Layered size scan**: C:\ root → user profile → AppData\Local/Roaming top-N → biggest files >300MB modified recently. Compare against `references/baseline.md`.
3. **MANDATORY deep-dive on growth**: any top-level directory that grew **>5GB since baseline** must be drilled into its next level until the growth is attributed (e.g. Program Files (x86) +28GB turned out to be a new Steam game; stopping at the top level would have hidden it). Attributed growth that is user content (games, projects) is NOT junk — report it, then move on.
4. **Invisible-space audit** when visible folders don't explain disk usage: `Get-ChildItem C:\ -File -Force` (pagefile.sys/hiberfil.sys are FILES), `vssadmin list shadowstorage`, `fsutil volume diskfree C:`, check `C:\Recovery`, `$Recycle.Bin`. A gap between "sum of visible folders" and "used bytes" means pagefile/hiberfil/restore points/permission-hidden dirs (admin reads more than non-admin — same folder can differ by 10GB+).
5. **Ghost detection**: services whose exe is missing, orphan uninstall entries whose InstallLocation no longer exists, scheduled tasks pointing at deleted exes, Run-key entries with dead targets.
6. **Cache-regrowth check — TWO layers**:
   - High-frequency subset (below) is quick to run every round;
   - **Full map: read `references/cache-map.md` and check EVERY entry listed there.** New hotspots are confirmed by writing them into the map (Phase 6).
   - High-frequency subset: WPS `addons\pool`, NVIDIA `ota-artifacts`, User/Windows Temp, CrashDumps, Win-Logs, Recycle Bin, Chrome-AI model (policy-guarded), eSupport.

### Phase 2 — Report

Categorize findings into: (A) system files to fix (pagefile fixed-size, hiberfil off), (B) safe junk (temp/crash dumps/logs/recycle bin), (C) app caches (per cache-map), (D) needs-user-judgment (chat app data → clean inside the app, games, projects), (E) protected/keep. Always show expected GB per item and total. Use AskUserQuestion for per-category decisions — the user has overturned my categorization before and was right to.

### Phase 3 — Safety Pre-check (MANDATORY before any deletion)

Run a read-only pre-check script and show the user the output. For EVERY item about to be deleted:
1. **Sample the content**: list top files/subdirs inside — confirm it is cache/installer leftovers, NOT user data (e.g. WPS pool contains installer cache only; active plugin data lives elsewhere and is excluded).
2. **Check running processes**: the owning app (WPS, KOOK, games for shader caches, Codex for runtimes) must not be running, or must be closed first. Deleting shader caches while a game runs causes rendering corruption. If a protected process is running, SKIP and defer.
3. **Verify protected neighbors are intact** (e.g. WPS `addons\data`, Codex `config.toml`).
4. **Preview the Recycle Bin** before emptying (file count + size; it is a one-way trip for `Clear-RecycleBin`).
5. Confirm version-coexistence traps: keep the NEWEST app version dir, delete only old `app-x.y.z` siblings (verified against the running/exe version).

Only after the user sees this pre-check output and confirms do you proceed.

### Phase 4 — Execute

- User runs admin scripts themselves when elevation is needed (pattern: write script → syntax check → give exact one-liner `& "path.ps1"` → user pastes output back). `Start-Process -Verb RunAs` UAC popups get dismissed; don't rely on them.
- Batch independent deletions in one script with per-item logging (`before/after/freed MB`), LinkType guard on every Remove-Item (junctions like "Application Data" must never be deleted), active-swap skip in Temp, and a final verify section.
- For registry deletion: first *query exact key paths* (PSChildName), then delete by exact path — never by keyword match on DisplayName (Chinese names break, keywords over-match).
- Script counting vs disk delta may differ (regenerating caches, NTFS overhead) — always re-measure disk free and report the honest number.

### Phase 5 — Verify & close

Re-measure disk free, re-check each target is gone, list what was kept and why (with reasons: "game running", "user data"), remind about pending reboots (pagefile resize needs one) and how to restore services (pitfalls.md has the Docker/WSL restore procedure).

### Phase 6 — Write-back (keeps the skill evolving)

The junk ecosystem evolves — every round finds NEW hotspot types (round 6: NVIDIA DXCache grew 2.1GB from a new game; codex-runtimes 1.6GB; KOOK old-version coexistence). After each confirmed round:
1. Append newly confirmed cache/regrowth points to `references/cache-map.md`.
2. Update `references/baseline.md` with this round's directory sizes.
3. Add new pitfalls (locked-file cases, version traps) to `references/pitfalls.md`.
4. If the user keeps the skill on GitHub, push the updates.

## Hard-won rules (the short list)

- **pagefile.sys ballooning to 20-35GB** is caused by sustained RAM pressure + "system managed size". Fix: disable AutomaticManagedPagefile, set Win32_PageFileSetting Initial 8192 / Max 16384, reboot. A bloated pagefile is also a signal to discuss RAM upgrades. Note: pagefile.sys shows 0 bytes in scripts when freshly recreated — don't panic, verify via Win32_PageFileUsage.
- **hiberfil.sys reappears after major Windows updates** even if previously disabled. Re-run `powercfg /h off`.
- **Deleting Chrome's on-device AI model (weights.bin ~4GB) is pointless alone** — it re-downloads. Set policy `HKLM:\SOFTWARE\Policies\Google\Chrome\OptimizationGuideOnDeviceModelDefault=0` first, then delete. Policy survives updates; deletion alone does not.
- **vhdx compaction order matters**: kill Docker processes with a BROAD pattern (`Name -match 'docker|vmmem|wslhost|wslservice|^wsl'` — exact-match regex misses `com.docker.backend`, the usual lock holder), restart `vmcompute` service, `wsl --shutdown`, then `diskpart compact vdisk`. Clean WSL-internal caches BEFORE compacting (internal ~/.cache, npm, apt), or you compact nothing. See pitfalls.md.
- **Never bare `docker compose up` to restore a stopped compose stack** — env vars like `${MINIO_LOCAL_IMAGE:-...}` resolve to defaults and break local-image stacks. Use the project's make target or the compose file's `--env-file`.
- **Stuck service (Stop-Pending forever)**: `taskkill /F /PID <pid>` then `sc.exe delete`. Stop-Service politely waits forever on broken services.
- **Rogue software signature**: registry Publisher/DisplayName containing mojibake garbage = bundled malware-ish; its uninstaller may delete files but leave registry — clean both.
- **Chinese literals in .ps1 files corrupt on save/run** (this environment). Write script labels in English; build Chinese paths via `[char]0xXXXX` concatenation or match by listing+filtering, not by typing the literal.
- **Don't fight unwinnable UI automation**: elevated Task Manager blocks UIPI reads; its per-second refresh stales screenshot frames. Use PowerShell as the same data source instead.
- **Electron apps bloat in the DATA layer, not the install layer** (install dirs update in place, no version leftovers; but `remote-assets-cache`-style caches grow every update). Check the app's Roaming data dir for cache subdirs.

## When the user says "scan again" (repeat rounds)

Load `references/baseline.md` + `references/cache-map.md` FIRST, then follow Phase 1. The usual suspects in order: vhdx growth (if they pulled images/projects), pagefile/hiberfil (after updates), WPS pool, temp regrowth, a new app's cache, games updating (watch shader caches!), new AI-tool runtimes. Historical fix-points (Chrome AI policy, ghost services) rarely recur — verify cheaply, then focus on what's new.
