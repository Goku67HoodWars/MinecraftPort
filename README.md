# MinecraftPort

Mod ports and porting documentation for the Wolfgang modpack's migration to Minecraft 26.2 (Fabric).

## Contents
- `ports/` — 122 mods supplied for the 26.2 Fabric build: hand-ports and manual builds of mods with no official 26.2 Fabric release on CurseForge. The rest of the pack comes straight from CurseForge and is not duplicated here.
- `docs/Wolfgang-262-Porting-Playbook.md` — a reusable playbook for porting any mod to 26.2 Fabric from Forge / NeoForge / older Fabric: decision tree, toolchain, cross-loader strategy, API codemods, GUI/render cookbook, crash-guard recipes, validation stack, difficulty tiering.

## License / redistribution
The jars in `ports/` are modified and/or third-party mods; each retains its original author's license (several are all-rights-reserved). This is a private archive — do not redistribute publicly without checking each mod's license. The documentation is original work.
