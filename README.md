# windows-disk-cleanup

A ZCode / Claude Code / Cursor skill for deep Windows C-drive cleanup — battle-tested over 4 real cleanup rounds that recovered **~82GB** on a developer machine (WSL2 + Docker + AI coding tools).

## What it covers

- 🔍 **Layered scanning** with baseline diffing (root → user profile → AppData → big files)
- 👻 **Ghost detection**: services pointing at deleted exes, orphan registry entries, dead scheduled tasks & Run keys
- 🗜️ **WSL/Docker vhdx compaction** — the exact working sequence (broad-pattern process kill → vmcompute restart → diskpart), including the `com.docker.backend` lock trap
- 🧠 **Invisible-space audit**: pagefile.sys bloat (35GB case!), hiberfil.sys revival after Windows updates, restore points, reserved storage
- 🛡️ **Safety-first workflow**: scan → report → per-item confirm → execute → verify, with junction protection and protected-path lists
- 🧹 **Cache regrowth playbook**: WPS pool, Chrome on-device AI model (policy-blocked, not just deleted), NVIDIA OTA leftovers
- 📦 **UWP bloatware removal** with provisioned-package cleanup

## Install

```bash
git clone https://github.com/davidluulc/windows-disk-cleanup.git
# ZCode / Claude Code / Cursor all discover skills in:
mkdir -p ~/.agents/skills && cp -r windows-disk-cleanup ~/.agents/skills/
```

## Usage

Just ask your agent things like:

- "C盘又满了,帮我看看"
- "我最近拉了几个 docker 项目,WSL 感觉变大了"
- "电脑怎么这么卡,是不是有垃圾软件"

The skill auto-triggers and walks the agent through scan → confirm → clean → verify.

## Structure

```
windows-disk-cleanup/
├── SKILL.md                  # main workflow & hard-won rules
└── references/
    ├── scan-recipes.md       # ready-to-adapt scan script patterns
    └── pitfalls.md           # vhdx locks, encoding traps, container restore, elevation pattern
```

## License

MIT — use it, fork it, improve it.
