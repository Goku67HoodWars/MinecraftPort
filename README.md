# MinecraftPort

A practical, technical how-to for **porting Minecraft mods** across game versions and mod loaders —
Forge → Fabric, NeoForge → Fabric, and version-to-version lifts.

## Contents
- **[`docs/Minecraft-Mod-Porting-Playbook.md`](docs/Minecraft-Mod-Porting-Playbook.md)** — the
  guide. A reusable, full-stack playbook: a port decision tree (triage before you code), toolchain
  and mapping setup, a cross-loader conversion map, an API-change cheatsheet, a GUI/render porting
  cookbook, a crash-guard pattern library, a layered validation stack, common failure modes →
  fixes, distribution, and difficulty tiering. The methodology is version-agnostic; the worked
  examples use the 1.21.x/Forge/NeoForge → Minecraft 26.2 Fabric transition for concrete detail.
- `ports/` — a reference archive of example ports produced with this approach (jars kept for study,
  not as a maintained distribution).

## License / redistribution
The documentation is original work. The jars in `ports/` are modified and/or third-party mods; each
retains its original author's license (several are all-rights-reserved) — **do not redistribute them
publicly without checking each mod's license.**
