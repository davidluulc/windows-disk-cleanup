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

A compose file with `${MINIO_LOCAL_IMAGE:-minio/minio:RELEASE...}` resolves the default when env is missing → "No such image" on locally-tagged stacks. Always restore via the project's own launcher:
```bash
wsl -d Ubuntu -- bash -lc "cd ~/ShieldAssist && SKIP_OCR_IMAGE=1 make -f infra/backend.mk local-infra-up"
```
Find the right target: `grep -n 'local-infra-up' Makefile infra/backend.mk`, read `COMPOSE_LOCAL := ... --env-file ... -f ...`. Stopping = same target's `-down` variant or `docker compose ... down` (volumes survive; never `down -v` unless user wants data gone).
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
- Chat app data (Feishu 5GB, WeChat, Bilibili) → clean INSIDE the app, never by folder deletion.
