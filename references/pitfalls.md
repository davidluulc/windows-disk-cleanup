# Pitfalls & Fix Procedures (read before executing compaction/uninstalls)

## 1. vhdx compaction — the exact working sequence

Failure mode: diskpart "file in use by another process" after `wsl --shutdown`. Root cause: `com.docker.backend` survives GUI close and re-launches WSL; `vmcompute` holds handles.

Working order (admin):
```powershell
# 1. broad-pattern kill (exact-match regex MISSES com.docker.backend)
Get-Process | Where-Object { $_.Name -match 'docker|vmmem|wslhost|wslservice|^wsl' } | Stop-Process -Force
# vmmemWSL may refuse "Access denied" — fine, vmcompute restart handles it
# 2. restart the host compute service (releases vhdx handles)
Restart-Service vmcompute -Force
# 3. shutdown WSL again, then compact each vhdx
wsl --shutdown; Start-Sleep 8
$dp = "select vdisk file=`"$vhdx`"`nattach vdisk readonly`ncompact vdisk`ndetach vdisk`nexit"
$dp | Out-File $tmp -Encoding ASCII; diskpart /s $tmp
```
Expected recovery: 1-9GB per vhdx (ext4 ~2-4GB, docker_data can give 6-9GB after image churn).
**Reality check**: freed = (vhdx size) − (df inside WSL) only partly; ext4 metadata isn't compactable. 1.1GB freed on a 3GB gap is normal — don't over-promise.

## 2. Restoring containers after compaction — NEVER bare `docker compose up`

A compose file with `${MINIO_LOCAL_IMAGE:-minio/minio:RELEASE...}` resolves the default when env is missing → "No such image" on locally-tagged stacks. Always restore via the project's own launcher, e.g.:
```bash
wsl -d <distro> -- bash -lc "cd ~/your-project && make -f infra/backend.mk <your-infra-target>"
```
Find the right target: `grep -n 'infra-up' Makefile infra/*.mk`, read the `COMPOSE_LOCAL := ... --env-file ... -f ...` line to see which env file injects the image variables. Stopping = same target's `-down` variant or `docker compose ... down` (volumes survive; never `down -v` unless user wants data gone).
Verification: `docker ps` shows all `(healthy)`; migrations log "already applied" (data intact).

## 3. Elevation pattern

`Start-Process -Verb RunAs` UAC popups get dismissed/missed. Instead: write script → syntax-check → hand user the one-liner `& "C:\path\script.ps1"` for their open admin PowerShell → they paste output back. Admin script skeleton: check `[Security.Principal.WindowsBuiltInRole]::Administrator` first, return with red message if not.

## 4. Chinese text in .ps1 corrupts (this environment)

Symptoms: parser errors pointing INSIDE string literals (`意外的标记`), mojibake in output. Fixes that work:
- English labels/strings only in scripts
- Chinese paths built via code points: `$name = [char]0x76F4 + [char]0x64AD + ...` (直播伴侣 = 76F4 64AD 4F34 4FA3)
- Or discover by listing: `Get-ChildItem $dir | Where-Object { $_.Name -match 'pattern' }` with non-Chinese patterns
- bash also eats `$vars` in inline `powershell -Command "..."` — always use script files

## 5. Safe deletion guards

```powershell
function Remove-Safe {
    param($Label, $Path)
    $item = Get-Item -LiteralPath $Path -Force
    if ($item.LinkType) { skip }        # junctions: "Application Data", Cookies, Recent... NEVER delete
    ...Remove-Item -Recurse -Force...
}
```
Locked files (dll still loaded): stop owning process first; if still locked, `Rename-Item` to a `_dead_<date>` tombstone — deletable after reboot.

## 6. Uninstall specifics

- **流氓软件 detection**: mojibake Publisher/DisplayName in registry → its uninstaller may leave registry; delete exact key `HKLM:\SOFTWARE\WOW6432Node\...\Uninstall\<key>` after files gone.
- **UWP**: `Remove-AppxPackage` + `Remove-AppxProvisionedPackage -Online`. Keep XboxIdentityProvider (game logins); XboxGameCallableUI is undeletable (SystemApps, error 0x80073CFA) — accept and move on, it's inert without Game Bar.
- **Stuck service**: Stop-Pending forever → `taskkill /F /PID` + `sc.exe delete <name>`.
- **WPS cache**: kill `wps,wpscloudsvr,wpscenter` first or files are locked.

## 7. computer-use limits on Windows

Elevated Task Manager: UIPI blocks both accessibility reads and window activation from non-elevated agent; per-second refresh makes screenshot frames stale before clicks land. Don't retry loops — switch to PowerShell (`Get-Process` grouped by Name with WorkingSet64 sums = same data).

## 8. Maintenance cadence advice for the user

- After "I pulled some docker images/projects" → compact vhdx (recipe 1+2).
- After big Windows updates → check hiberfil.sys revival + restore points + `C:\Windows.old`.
- Pagefile re-bloat after fixing = RAM pressure; discuss 32GB upgrade or lighter startup.
- Chat app data (Feishu 6GB, WeChat, Bilibili) → clean INSIDE the app, never by folder deletion.

## 9. Lessons from round 6 (2026-09-26 A/B experiment: skill flow vs free scan)

- **Shader caches while a game runs = skip.** Rainbow Six was running; deleting DXCache (2.1GB) mid-game risks rendering corruption. Pre-check must enumerate game processes (`ELDEN|DARKSOULS|RainbowSix|...`) before touching GPU caches.
- **App version coexistence is a recurring pattern**: KOOK had shipped a new version (0.110.0) while old (0.95.1, 461MB) remained. Always list `app-*` siblings, verify the newest is what runs, delete only the old ones.
- **codex-runtimes** (`~/.cache/codex-runtimes`, 1.6GB) is a runtime binary cache — safe to delete when Codex is closed; it redownloads on next start. Codex config/auth live in `~/.codex` and are unaffected.
- **pagefile.sys can read 0 bytes right after a reboot** even when the setting is correct — verify via `Win32_PageFileUsage` (AllocatedBaseSize) instead of file size.
- **WSL-internal cleanup must happen BEFORE vhdx compaction**, while the distro is running; then `wsl --shutdown` and compact. Compacting without internal cleanup frees almost nothing (0-1GB on a 17GB vhdx).
- **Progressive-disclosure trap**: a reference file that is only "read on demand" will be SKIPPED under time pressure — 4GB of findings were missed in the A/B test because the cache table lived in references and was never read. Fix applied: mandatory read instruction in SKILL.md + dedicated `cache-map.md`.
- **Attributed growth ≠ junk**: Program Files (x86) +28GB was a new Steam game (DARK SOULS III). Deep-dive to attribute, then reclassify as protected user content.
- **Free-form scan complements the fixed checklist**: pure memory-based lookups can hit stale targets (ms-playwright/Douyin already deleted), but they caught all four NEW hotspot types the static checklist missed. The write-back loop (Phase 6) is what merges the two strengths.
