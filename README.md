# MinecraftPort

A practical, technical how-to for **porting Minecraft mods** across game versions and mod loaders —
Forge → Fabric, NeoForge → Fabric, and version-to-version lifts — plus a worked, end-to-end case
study and a greenfield companion.

## Docs
- **[`docs/Minecraft-Mod-Porting-Playbook.md`](docs/Minecraft-Mod-Porting-Playbook.md)** — the guide.
  A reusable, loader/version-agnostic playbook: a port decision tree (triage before you code),
  toolchain and mapping setup, a cross-loader conversion map, an API-change cheatsheet, a GUI/render
  cookbook, a crash-guard pattern library, a layered validation stack, common failure modes → fixes,
  distribution, and difficulty tiering. The worked examples use the 1.21.x/Forge/NeoForge → Minecraft
  26.2 Fabric transition for concrete detail.
- **[`docs/Useful-Ribbits-Port-Case-Study.md`](docs/Useful-Ribbits-Port-Case-Study.md)** — a true 1:1
  account of porting **Useful Ribbits** from 1.20.1 Forge to 26.2 on all three loaders, including the
  hard bugs (and the deep-dive on the Forge GUI-preview collapse). The resulting mod and its jars live
  at **<https://github.com/Goku67HoodWars/Useful-Ribbits-Remastered>**.
- **[`docs/Writing-A-26.2-Mod-From-Scratch.md`](docs/Writing-A-26.2-Mod-From-Scratch.md)** — a
  greenfield companion: clean multiloader layout, the platform seam, modern registration, the
  data-component + retained-render model — for starting a new 26.2 mod correctly rather than porting.

## ports/
A reference archive of example ports produced with this approach (jars kept for study, not as a
maintained distribution).

## License / redistribution
The documentation is original work. The jars in `ports/` are modified and/or third-party mods; each
retains its original author's license (several are all-rights-reserved) — **do not redistribute them
publicly without checking each mod's license.**
