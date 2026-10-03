# Minecraft 26.2 Fabric Porting Playbook

A full-stack, reusable guideline for porting **any** mod to **MC 26.2 Fabric** from any source version/loader (Forge 1.20.1, NeoForge 26.x, a newer Fabric build, etc.). Grounded in the Wolfgang modpack's completed migration: **198 mods (1.20.1 Forge) → 233 mods (26.2 Fabric)**.

> This is a forward-looking playbook, not a retrospective. Each section ends with the rule you reuse next time, and uses real Wolfgang mods as case studies. Where a tool or workspace is named, the path is absolute so you can go straight to it.

---

## 0. The migration by the numbers (ground truth)

Measured live from the two instance `mods/` folders (`scratchpad/compare.py` + `compare2.py`):

| Metric | Count | Notes |
|---|---:|---|
| Old 1.20.1 real mods | **198** | Forge/Sinytra |
| New 26.2 real mods | **233** | Fabric |
| Carried over (same mod id) | **155 (78%)** | version-lifted or re-obtained |
| Dropped (old-only) | **43** | of which **7 pure backports**, 8 Forge perf mods, ~14 real content cuts, rest Forge libs/loader noise |
| Added (new-only) | **78** | 10 perf (sodium/lithium/c2me/krypton/noisium/scalablelux/ferritecore-class/chunky/spark), 9 fabric libs, ~55 content/replacements |
| **CF-native (official 26.2 release)** | **111** | `installedAddons` from `minecraftinstance.json` |
| **Ported / manual override** | **122** | every jar NOT provided by a CF file id — the hand-ports |

The single most important number: **122 of 233 jars have no official 26.2 CF release behind them.** Roughly half the pack is hand-work. That is why the decision tree below exists, and why distribution (§9) must carry overrides.

Instances of record:
- New (target): `%USERPROFILE%\curseforge\minecraft\Instances\Wolfgang Cute Craft\mods` (233 jars) — also the "Expanded Ports" / "Simplified Ports" lanes.
- Old (reference for texture/model recovery): `%USERPROFILE%\curseforge\minecraft\Instances\Wolfgang\mods` (199 jars), plus `Wolfgang (2)\mods` (a collaborator's bigger pack).

---

## 1. Port decision tree

Run this per mod **before writing a line of code.** Most mods exit at rung 1–3; only a few reach rung 5.

```
For each source mod:
│
├─1. Does an OFFICIAL 26.2 build exist (same loader = Fabric)?
│     → USE IT. Done. (111 of 233 Wolfgang mods.)
│     e.g. Ecologics, Waystones, AppleSkin, Clumps, Friends&Foes.
│
├─2. Does an official 26.2 build exist on the OTHER loader (NeoForge)?
│     ├─ Can you run Sinytra Connector + Forgified Fabric API? → consider it (§3),
│     │    but it is all-or-nothing for the pack and adds a heavy compat layer.
│     └─ Else → treat as "no native build" → rung 4/5.
│        ⚠ Some mods are NeoForge-ONLY with no feasible Fabric path short of Connector
│          (Sophisticated Storage — §3, §10).
│
├─3. Is this mod a BACKPORT of a feature that is now VANILLA in 26.2?
│     → DELETE it. Do not port. (7 Wolfgang mods — see list below.)
│
├─4. Is there a NATIVE-FABRIC equivalent / refork that does the same job?
│     → SWAP it. Cheaper than porting, already maintained.
│     e.g. Farmer's Delight (Forge) → Farmer's Delight Refabricated;
│          Cluttered → unofficial Fabric port; Curios → Accessories;
│          Corpse → Player Corpses; Starlight → DROP (vanilla absorbed it).
│
├─5. Does the mod's own repo have a 26.1/26.2/26.3 multiloader branch?
│     → This is a LOADER-GLUE port, not a version lift. Far cheaper (§3).
│     ALWAYS check the repo before assuming a big lift. Burned-and-corrected on
│     Patchouli, solcarrot, ApexCore/FantasyFurniture, YUNG's suite.
│
├─6. Is there published SOURCE (any version)?
│     → SOURCE-PORT it (§4–6). Cost scales with the version delta and GUI/render surface.
│
└─7. No source, no native build, no equivalent, feature not vanilla?
      → DROP it, and tell the user what is lost (concrete "what you lose" list).
      e.g. Haybale, Giant Mushroom/Swamp Tree blocks, Ender Storage (no 26.2 anywhere).
```

**Backport mods dropped (rung 3) — verified feature-present in the 26.2 jar:**
`backported_wolves` (WolfVariant), `copperandtuffbackport` (polished/chiseled tuff, copper bulb/grate), `golden_dandelion_backport`, `saddle_backport`, `shelfbackport` (ShelfBlock), `vanilla_discs_backport`, `vanillabackport`. Also `fireflybushport`, `steves_lava_chicken_music_disc` → vanilla.

**Real content cuts (rung 7, with reasons):** backpacked, betterfoliage, cfm (MrCrayfish Furniture), cosmeticarmorreworked, cozy_home (partially replaced), ding, mountsofmayhem, butterflies, trials, inventorysorter — no source/native path or superseded.

**Rule:** the decision tree converts "233 mods to port" into "~120 to actually touch, of which ~10 are hard." Spend the budget on rung 6–7; never hand-port a rung-1/3/4 mod.

---

## 2. Toolchain / environment

### The one fact that changes everything: 26.2 Fabric ships **mojmap at runtime**, not intermediary.
Verified by binary-scanning known-good 26.2 jars (Clumps, Friends&Foes, Waystones, CarryOn): **zero** `class_####` / `method_####` tokens.

- **Verify any jar's mapping:** `grep -aoE "class_[0-9]{3,}|method_[0-9]{4,}" <jar> | wc -l` — `0` = mojmap (correct for 26.2), nonzero = intermediary (wrong / needs remap).
  - ⚠ Raw-jar grep misses DEFLATE-compressed class bytes — extract first for a true scan.
- **Build implication:** compile against `loom.officialMojangMappings()` with **no refmap** and **no intermediary `remapJar` step**. The plain named-mapped jar is already runtime-correct.

### Loom setup that builds (canonical, proven across the whole pack)
- Gradle **9.5.1**, fabric-loom **1.16.2** (plain `net.fabricmc.fabric-loom` `1.17-SNAPSHOT` also resolves 26.2 mojmap — simpler single-module route).
- fabric-loader **0.19.5** (0.19.3 is too old — architectury/kotlin need ≥0.19.5), fabric-api **0.161.0+26.2** (earlier: 0.154.0+26.2), cloth 26.2.155, modmenu 20.0.3.
- JDK **25** (`C:\Program Files\Java\jdk-25.0.2`). `javap`/`javac` are NOT on PATH in git-bash — use the full `/c/Program Files/Java/jdk-25.0.2/bin/...`.
- `officialMojangMappings()`, **no `mappings()` line needed on single-module**, mixin json **no `refmap` key**, `compatibilityLevel JAVA_21` (or `JAVA_25` only if source uses Java 25 language features).
- **`--release 21` bytecode (major 65) runs fine on the Java 25 runtime** — the safe default. Only use `--release 25` when the source uses unnamed `_` vars etc.

### Architectury multiloader gotcha
The thin `:fabric:jar` holds only loader-specific classes. The complete mod (common + assets merged) is the **shadowJar** output (classifier `dev-shadow`). Deploy that renamed, not the thin jar.

### fabric.mod.json essentials (recurring boot failures)
- Depend key is **`fabric-api`**, not the legacy `fabric` alias (gone in 0.161) → a `"fabric": ">=0.92.0+1.20.1"` depend = `HARD_DEP_NO_CANDIDATE` at boot.
- Widen `"minecraft"` to **`">=26.1 <27"`** — an exact `"26.2"` or a template `"<26.2"` can EXCLUDE the real `26.2`/`26.2.0` runtime.
- JiJ any bundled config lib (fiber, CCA) whose setup runs at init, or it crashes.

### Rebuilding a jar-only mod (no source)
Decompile with **Vineflower** (`org.vineflower:vineflower:1.11.1`, `java -jar vineflower.jar <classesDir> <outDir>` — clean Fabric decompiles), scaffold a loom project from a known-good 26.2 build, drop sources + resources, add compile-only deps (JEI 26.2 is mojmap → no remap). Decompiled `(FunctionalIface)arg -> …` casts are valid — leave them. (Used for JEArchaeology, energy remap.)

**Rule:** mojmap-no-remap is the whole build model. If a dependency jar leaks intermediary tokens (e.g. team_reborn_energy shipped intermediary), re-remap it (§4) — do not fight it at the consumer.

---

## 3. Cross-loader strategy (Forge / NeoForge → Fabric)

### Three fundamentally different jobs — identify which you have
1. **Forge/Sinytra 1.20.1 → Fabric 26.2** = a *version lift* (biggest; vanilla APIs changed under you AND the loader changed). The Wolfgang baseline.
2. **NeoForge 26.2 → Fabric 26.2** = *loader-glue only* (vanilla APIs already correct; only `@Mod`/event-bus/registries/networking/config change). **Much cheaper** — always prefer finding this branch. (solcarrot, ApexCore/FantasyFurniture, blueprint-compat.)
3. **Fabric 1.21.1 → Fabric 26.2** = a *same-loader version bump* (only the vanilla-API delta, no loader work). (Sophisticated via Salandora's forks; Patchouli 26.1→26.2.)

### Loader-glue conversion map (NeoForge/Forge → fabric-api 0.161, all javap-verified)
| NeoForge/Forge | Fabric 26.2 |
|---|---|
| `@Mod` + ctor | `implements ModInitializer` / `ClientModInitializer` (entrypoints in fabric.mod.json) |
| `DeferredRegister.Items` / `registerItem` | `Registry.register(BuiltInRegistries.ITEM, id, new Item(props.setId(key)))` |
| `AttachmentType` / capability | `fabric-data-attachment-api-v1` `AttachmentRegistry...buildAndRegister`; access via `((AttachmentTarget)entity).getAttachedOrCreate(TYPE)` (CAST) |
| Creative tab `BuildCreativeModeTabContentsEvent` | `fabric-creative-tab-api-v1` `CreativeModeTabEvents.modifyOutputEvent(key)` (NOT the old item-group api) |
| `PayloadRegistrar.playToClient` | `PayloadTypeRegistry.clientboundPlay().register`; send `ServerPlayNetworking.send(player,payload)` |
| `RegisterCommandsEvent` | `CommandRegistrationCallback.EVENT` |
| `PlayerEvent.Clone` | `ServerPlayerEvents.COPY_FROM(old,new,alive)` (`!alive` == `isWasDeath()`) |
| `ItemTooltipEvent` | `ItemTooltipCallback.EVENT` |
| `ModConfigSpec` | self-contained Gson POJO in `FabricLoader.getConfigDir()` (Gson is on MC's classpath) |
| `FMLEnvironment.getDist().isClient()` | `level.isClientSide()` / `player.isLocalPlayer()` guard |
| `ModList.get().isLoaded(id)` | `FabricLoader.getInstance().isModLoaded(id)` |
| `neoforge:difference` recipe ingredient | `fabric:difference` (identical strings; discriminator key is `fabric:type`) |

### The big reusable shim: NeoForge/Forge-registries shim
Large Registrate/NeoForge content mods (Team Abnormals Blueprint, ApexCore/Registree, Neapolitan ~340 sites) are written against a `DeferredRegister`/`RegistryObject` DSL over hundreds of call sites. **Don't rewrite the call sites — shim the registry API once:**
1. A `RegistryObject<T>`/`DeferredHolder<T>` shim class (holds the Identifier + a resolved `T`; `.get()`/`.getKey()`/`.getId()`). **It CANNOT `implements Holder`** — vanilla seals `Holder` in 26.2 (NeoForge un-seals; Fabric does not). Expose `value()`/`get()` instead.
2. The helper `createItem/createBlock/...` does eager `Registry.register(...)` and returns the shim.
3. A `Block.Properties`/`Item.Properties`-ctor **mixin** reads a `ThreadLocal<ArrayDeque<ResourceKey>>` (the id being registered) and calls `.setId(...)` automatically — so bare `new Item.Properties()` call sites compile unchanged. **The ThreadLocal MUST be a per-thread stack (ArrayDeque), not a single slot** — nested registration (a block's Properties touching a sound-registry constant mid-construction) wipes a single slot and NPEs the outer ctor.
4. Datagen classes (`*.data.*`) never run at runtime — drop their framework deps; the generated JSON already ships.

### porting_lib — what its modules provide (the Sophisticated path)
A Forge-parity shim lib. The Wolfgang port published **11 modules @ `3.1.0+26.2`** to mavenLocal:
`core, transfer, fluids, model_loader, render_types` (Core's set) + `registry, resources, loot, blocks, level_events, item_abilities` (Backpacks' set). They re-expose Forge's DeferredRegister, loot-modifier, block/level event, item-ability, and fluid-container APIs on Fabric. **Caveat:** porting_lib mixins are string-targeted, so a module compiles green but its mixins fail in waves at datapack/world-load as each stale `@Shadow`/`@At` surfaces. When a mixin targets a removed/renamed 26.2 member AND the feature is unused by the consuming mod, **DROP it from its `*.mixins.json`** (honest, matches the source-port drop pattern).

### Sinytra Connector vs a true port
Connector + Forgified Fabric API lets official NeoForge jars run on Fabric. **It is all-or-nothing for the pack** (heavy compat layer, version-sensitive). Wolfgang chose true ports to keep the stack pure Fabric. The honest escape hatch: **Sophisticated Storage is NeoForge-only** (`sophisticatedstorage-26.2-1.5.116.2144.jar` = `neoforge.mods.toml`, needs `neoforge [26.2.0.53-beta,…]`). The pack runs an AI-port (`26.2-1.3.7.9.9999-SNAPSHOT`) whose chests/shulkers render via a real BER but whose item icons go through a broken porting_lib item-extension — "to actually fix it properly" is Connector + official NeoForge SS/Core/Backpacks. Keep that option in your pocket for the 1–2 mods that are genuinely NeoForge-bound.

**Rule:** before any cross-loader port, (a) check the repo for a 26.x multiloader branch to downgrade a lift into glue, (b) check for a native-Fabric refork to skip it entirely, (c) reach for the registries-shim before touching call sites, and (d) accept Connector only for the NeoForge-bound stragglers.

---

## 4. Recurring API changes + codemods (the cheatsheet)

The full, running catalog is in `fabric-262-api-changes.md` (memory). The high-frequency mechanical ones, as a table you can codemod:

### Renames / moves (pure mechanical)
| Old (≤1.21) | New (26.2) | Note |
|---|---|---|
| `ResourceLocation` | `net.minecraft.resources.Identifier` | `Identifier.fromNamespaceAndPath(ns,path)` / `parse` — no `.of()` |
| `ResourceKey.location()` | `.identifier()` | but `TagKey.location()` STAYS |
| `EntityType.ARMOR_STAND` | `EntityTypes.ARMOR_STAND` | ALL_CAPS consts moved to `*s` holders; same for `BlockEntityTypes` |
| `Blocks.WHITE_TERRACOTTA` | `Blocks.DYED_TERRACOTTA.pick(DyeColor.WHITE)` | ColorCollection: DYED_TERRACOTTA/GLAZED/BED/DYED_CANDLE/CONCRETE/WOOL/… |
| `Blocks.WAXED_COPPER_BLOCK` | `Blocks.COPPER_BLOCK.waxed().unaffected()` | WeatheringCopperCollection |
| `Registry.get(id)` | `getValue(id)` | `get(id)`→`Optional<Holder.Reference>`; use `getValue` for the value |
| `RegistryAccess.registryOrThrow` | `lookupOrThrow` | |
| `Level.getMinBuildHeight` | `getMinY` | `BlockPos.getCenter()`→`Vec3.atCenterOf(pos)` |
| `Direction.getNormal()` | `getUnitVec3i()` | |
| `Entity.moveTo` | `snapTo` | ⚠ NOT `PathNavigation.moveTo` |
| `MobSpawnType` | `EntitySpawnReason` | |
| `UseAnim` | `ItemUseAnimation` | |
| `FastColor`/`ARGB32` | `net.minecraft.util.ARGB` | |
| `ChatFormatting` color meta | `net.minecraft.network.chat.TextColor` | `getById(i)`→`values()[i]`, `getId()`→`ordinal()` |
| `ItemInteractionResult` | merged into `InteractionResult` | `sidedSuccess`→`SUCCESS` |
| `DirectionProperty` | `EnumProperty<Direction>` | |
| `noCollission()` | `noCollision()` | |
| advancements `criterion.*` | `predicates.*` + `triggers.*` | package split |
| `MultiBufferSource` | submit pipeline (`SubmitNodeCollector`) | render, §5 |

### Removed / replaced (needs rework, not rename)
- **`GuiGraphics` → `GuiGraphicsExtractor`** (retained-mode) — §5, the big one.
- **`Explosion` is now an INTERFACE** — custom explosions `extends Explosion` → `implements Explosion` or extend `ServerExplosion` (no `this.x/level/radius`, no `getToBlow()`/`finalizeExplosion`).
- **`Block` LOST `appendHoverText`** entirely (only `Item` has it). `Item.appendHoverText` sig = `(ItemStack, TooltipContext, TooltipDisplay, Consumer<Component>, TooltipFlag)`.
- **`Entity.hurt` is final** → override `hurtServer(ServerLevel, DamageSource, float)`. Same pattern: `customServerAiStep(ServerLevel)`, `doHurtTarget(ServerLevel,…)`.
- **NBT: `CompoundTag` in save/load → `ValueInput`/`ValueOutput`.** BE `loadAdditional/saveAdditional(ValueInput/ValueOutput)`; Entity `readAdditionalSaveData(ValueInput)`. Getters return Optional or `getIntOr(k,def)`. `putUUID`/`getUUID` removed → `store/read(key, UUIDUtil.CODEC)`; `NbtUtils.writeBlockPos` → `store(key, BlockPos.CODEC)`. ⚠ On-disk layout differs → fresh 26.2 world only, not backward-compatible.
- **`SpawnEggItem` ctor** dropped colors → `(Properties)` only; bind entity via `Item.Properties.spawnEgg(EntityType)`; colors = client tint.
- **`RecordItem` removed** → `JukeboxSong` datapack + `JUKEBOX_PLAYABLE` component. `ArmorItem` removed → `EQUIPPABLE` component. `ItemNameBlockItem`/`BannerPatternItem`/`EitherHolder`/`VariantHolder` removed.
- **`Ingredient` is final** (can't subclass); `.getItems()`→`items()`; `Ingredient.of(ItemStack)` gone → `of(ItemLike)`; tag ingredient binds eagerly → memoize in a `Supplier`.
- **`new ItemStack(Item)` / `Item.getDefaultInstance()` reads unbound components during recipe decode / `<clinit>`** → "Components not bound yet". Defer ALL recipe-result/static `ItemStack` construction to runtime (lazy `Supplier<ItemStack>`). **Never build an ItemStack in a `<clinit>` of any class that can load during resource reload** (BERs, entity renderers, models, particles) — §8.
- **26.2 food rework:** `.food(...)` no longer drives eating → add `.component(DataComponents.CONSUMABLE, Consumable.builder()...build())`.
- **Item id MUST be on Properties at construction:** `new Item(new Item.Properties().setId(ResourceKey.create(Registries.ITEM, id)))` or ctor throws `NullPointerException: Item id not set` (block: `Registries.BLOCK`). Codemod `setid_codemod.py`.

### Recipe/datapack JSON (silent — surfaces in client/data log, not boot)
- **Ingredient format:** `{"item":"minecraft:stick"}` → `"minecraft:stick"`; `{"tag":"x"}` → `"#x"`; multi-option → JSON array. Applies to shaped `key` + shapeless `ingredients`. Result stays `{"id":"...","count":N}`. Symptom: recipe silently doesn't exist. Codemod `fix_recipes.py`.
- **`minecraft:random_patch` FEATURE removed** → unwrap scattering into placed_feature placement modifiers. **`uniform` IntProvider flattened** (drop the `value` wrapper). **`minecraft:grass` block → `minecraft:short_grass`.** **`TreeConfiguration` dropped `dirt_provider` → requires `below_trunk_provider`.**

### Codemod discipline
- **Always `clean compileJava`, never trust an incremental error count.** javac short-circuits per file: unresolved IMPORTS hide BODY errors, so the count ticks UP as imports resolve, and a mangled statement (dangling `)`) aborts early printing a FAKE low count. Confirm progress by emitted `.class` count or a clean compile — never by a shrinking `errors.txt`.
- Codemod scripts live in the session scratchpad: `setid_codemod.py`, `fix_recipes.py`, `codemod262.py`, `gen_item_defs.py`.

**Rule:** rename-table changes are codemod fodder; the removed/replaced ones need per-file judgment. Measure with clean compiles only.

---

## 5. GUI / render cookbook (GuiGraphics → GuiGraphicsExtractor)

26.2 (1.21.6+) replaced immediate-mode `GuiGraphics` with a **retained-mode extract/render** pipeline. `GuiGraphics` is GONE. Every GUI-heavy mod hits this identically (Sophisticated Core/Storage/Backpacks, Supplementaries, Patchouli, Exposure). Full cookbook: `fabric-262-gui-render-port.md`.

**Paradigm:** widgets/screens no longer DRAW; they EXTRACT render state into a `GuiGraphicsExtractor`, and `GuiRenderer` renders it. `render(GuiGraphics)` bodies port mostly 1:1; the **signature** and a few calls change.

**Signature map (before → after):**
- `Renderable.render(GuiGraphics,int,int,float)` → `extractRenderState(GuiGraphicsExtractor,int,int,float)`.
- `AbstractContainerScreen` overrides: `renderBg`→`extractBackground`, `renderLabels`→`extractLabels`, `renderSlot`→`extractSlot`, `renderTooltip`→`extractTooltip`.
- `g.drawString(Font,t,x,y,c)` → `g.text(Font,t,x,y,c[,shadow])`; `drawCenteredString` → `centeredText`.
- `g.blit(Identifier,x,y,u,v,w,h,…)` → `g.blit(RenderPipeline, Identifier, x, y, float u, float v, w, h, texW, texH[,color])` — pass `RenderPipelines.GUI_TEXTURED` as arg 1. `blitSprite` likewise.
- `g.renderItem(stack,x,y)` → `g.item(stack,x,y[,seed])`; `renderFakeItem`→`fakeItem`.
- **`g.pose()` returns `org.joml.Matrix3x2fStack` (2D)** — `pushMatrix()/popMatrix()/translate/scale/rotate`, no z. `pushPose/popPose/last().pose()` don't exist.
- **Tooltips deferred:** `renderTooltip(...)` → `setTooltipForNextFrame(font, stack|Component|List, x, y)`.
- **Mouse/key events:** `mouseClicked(double,double,int)` → `mouseClicked(MouseButtonEvent, boolean doubleClick)`; `keyPressed(int,int,int)` → `keyPressed(KeyEvent)`. Extract `event.x()/y()/button()` at the top to reuse bodies.
- **Container clicks:** `ClickType` removed → `ContainerInput` (maps 1:1: PICKUP/QUICK_MOVE/SWAP/CLONE/THROW/QUICK_CRAFT/PICKUP_ALL).
- `imageWidth`/`imageHeight` are now **final** → add an access-widener `mutable field` line (2 lines, AW namespace `official`).

**fabric-api 0.161 removals (need rework, not rename):**
`BlockRenderLayerMap` (render_type is data-side), `BuiltinItemRendererRegistry` (→`SpecialModelRenderer`), `ColorProviderRegistry` / `ItemColor` (→ data-driven `ItemTintSource`), `LivingEntityFeatureRendererRegistrationCallback` (→`FeatureRendererRegistry`), `HudRenderCallback` (→`HudElementRegistry.addLast`), `ClientPickBlockApplyCallback` (no successor). `EntityModelLayerRegistry`→`ModelLayerRegistry`. `Minecraft.screen` field REMOVED → `mc.gui.screen()` / track your own `INSTANCE`.
- **`ExtendedScreenHandlerType` → `net.fabricmc.fabric.api.menu.v1.ExtendedMenuType`** (`fabric-screen-handler-api` is gone).

**In-world custom geometry — the submit pipeline** (separate from GUI; harder):
`MultiBufferSource` is gone. BER/EntityRenderer is now `createRenderState()` → `extractRenderState(...)` → `submit(state, PoseStack, SubmitNodeCollector, camera)`. Template on vanilla **`BeaconRenderer`** for raw geometry. Bridge: `SubmitNodeCollector.submitCustomGeometry(matrix, RenderType, (pose,buffer) -> …)` returns a real `VertexConsumer` and COPIES the pose — seed a fresh local `PoseStack` from it and your existing `ModelPart.render(...)` runs verbatim. **Wrap every `submit`/`extractRenderState` body in try/catch** — the dispatcher rethrows as a HARD crash; degrade to "not drawn + one log line."
- Moonlight 4.1.0 ships the reusable helper layer: `client.util.RenderUtil.submitBlockModel/getBlockModel/getBlockSprite`, `ClientHelper.addBlockEntityRenderersRegistration`, `TextUtil.submitLine`. Used to breach the Supplementaries render wall (Way Sign/Cannon/Clock/Bellows/Shelf now render in-world).

**Worked progress metric:** a GuiGraphicsExtractor codemod across 42 Sophisticated Core GUI files took it **620 → ~174 errors** in one pass — voluminous but tractable, NOT a from-scratch reimpl.

**Rule:** the GUI port is a mechanical signature pass (extract-* + call renames + 2D pose). The in-world submit pipeline is a genuine reimpl — template on BeaconRenderer and always try/catch.

---

## 6. Crash-guard pattern library

Every one of these was a real 26.2 port crash, and **all were centralized into two standalone mixin mods** so no content jar had to be re-edited:
- **`wolfgangclientfixes` (wcf)** — 18 common mixins + 6 client mixins + loot modifiers + client entrypoints. Workspace `%USERPROFILE%\mc-ports\wolfgangclientfixes` (offline `build-manual.sh`). **Its environment is `"*"` (not `"client"`)** — a client-only env means Fabric skips it on a dedicated server, so the common crash-guards never apply server-side (this is why ticking crashes kept recurring until it was flipped). Config `wolfgangclientfixes.mixins.json`: `JAVA_21`, no refmap, `defaultRequire:1`.
- **`wolfgangfixes`** — the generic `BlockEntityType.isValid` guard (separable, loads with any pack).

Each guard is a **symptom → root cause → mixin** recipe:

| # | Symptom | Root cause | Guard |
|---|---|---|---|
| 1 | `IllegalStateException: has not defined synched data value 8` on block PLACEMENT (server crash) | Ported entity's `defineSynchedData` override skips `super`, leaving inherited accessor id 8 undefined (exposure PhotographFrame, zetter Painting, fairylights FenceFastener) | `HangingEntityDirectionAccessor` + `HangingSubclassSynchedDataMixin` — define id 8 for the subclasses |
| 2 | `NoSuchElementException` / "No value present" at entity construction | `registry.get(KEY).orElseThrow()` on an empty/unshipped datapack registry (neapolitan Chimpanzee variant) | `ChimpanzeeVariantGuardMixin` — `@Redirect` the lookup → `getAny()` fallback |
| 3 | Client StackOverflow ("Render Frame" via Sodium chunk build) | Moonlight `MimicBlock.hasEmissiveRendering(state)` body is `return state.emissiveRendering()` → infinite recursion (sign posts, way signs, framed blocks) | `MimicBlockEmissiveMixin` — `@Inject HEAD cancellable`, `setReturnValue(false)` |
| 4 | Server-tick crash "Exception while updating neighbours" — `Cannot set property weathering … stone_brick_stairs` | Immersive Weathering fluid-generator calls `state.setValue(WEATHERING_STATE,…)` on a block lacking the property when lava spreads (caves/lava lakes — frequent random crash) | `ImmersiveWeatheringFluidGenGuardMixin` — `@WrapMethod` on `FluidGeneratorsHandler.applyGenerators`, catch `IllegalArgumentException` → `Optional.empty()` |
| 5 | "Ticking entity" — `Can't find attribute minecraft:tempt_range` | 26.2 `TemptGoal.canUse` reads the NEW `tempt_range` attribute; ported mobs built `AttributeSupplier` by hand without it (duckling, C&B) | `AttributeSupplierMixin` — guard `getValue`/`getBaseValue` HEAD, return `getDefaultValue()` when `!instances.containsKey(attr)` |
| 6 | `AttributeMap.getValue … supplier is null` on spawn | Ported EntityType registered with NO default AttributeSupplier | `AttributeMapMixin` — same guard when `supplier==null` |
| 7 | `IllegalStateException: Invalid block entity … got Block{X}` on placement/chunk-load | Mod adds a block variant (new wood, colored bottle, cross-mod food BE) but not to the BE type's `validBlocks` | `BlockEntityValidateGuardMixin` (+ `wolfgangfixes` generic) — `@Inject` RETURN of `isValid`, accept when the block's Class ==/isAssignableFrom any `validBlocks` member |
| 8 | "Unknown TooltipComponent" crash hovering an item | Mod ships a data-side `TooltipComponent` with no client renderer (supplementaries, sophisticatedcore, exposure) | `ClientTooltipComponentCallback` handler returns a zero-size `EmptyClientTooltip` for those class prefixes. **Do NOT mixin `ClientTooltipComponent` — it's an interface** (class-mixin fails apply) |
| 9 | ~54 entities crash with `NPE EntityRenderer.shouldRender … renderer is null` | Port cut the client package but still registers the entity; no renderer | `client.EntityRenderDispatcherMixin` — fill the renderer map with a `NoopRenderer` on reload (invisible but safe) |
| 10 | Moonlight "Failed to get registry access. This is a bug" → recipes silently uncraftable | Moonlight generates recipes off-thread at startup before any RegistryAccess is published (cannon_boat, way-signs) | `MoonlightRegistryFallbackMixin` — `@WrapMethod hackyGetRegistryAccess` returns captured RA else a lazily-built `RegistryAccess.fromRegistryOfRegistries(BuiltInRegistries.REGISTRY)` |
| 11 | Recipe-book "can't be placed due to empty ingredients" | Hand-ported recipe classes stub `placementInfo()->NOT_PLACEABLE` → 26.2 drops them | `Vinery*/Bakery*/FairyLights*PlacementMixin` — inject `placementInfo()` returning `PlacementInfo.create(ingredient)`, or force `isSpecial()->true` |

**Also covered: `AbstractArrowMixin`** (shroom_dealers arrow server-tick guard).

**When a NEW crash of these shapes appears:** grep the ported jars for the offending pattern (a `defineSynchedData` override lacking `invokespecial super`, an `orElseThrow` on a registry lookup, a `setValue` on a maybe-absent property), add the same mixin shape to wcf, rebuild with `build-manual.sh`, deploy to all packs (game CLOSED — §8). The 26.2 vanilla jar for signature checks: `%USERPROFILE%\curseforge\minecraft\Install\versions\26.2\26.2.jar`.

**build-manual.sh gotcha:** fabric-loader bundles Mixin/MixinExtras only as NESTED jars (javac can't read those) → every `@Inject/@At` fails "cannot find symbol". Add the standalone cache jars `sponge-mixin-0.17.4+mixin.0.8.7.jar` + `mixinextras-common-0.5.4.jar` (and the `_apilibs`) to `$CP`.

**Rule:** one centralized `targets=`-string mixin mod beats editing N content jars — it survives every content rebuild, builds with no dependency on the target mod, and deploys as a single jar. Make its env `"*"` so guards cover the dedicated server.

---

## 7. Validation stack (run in this order)

Each stage catches a class the previous one is BLIND to. Do not skip forward.

1. **Compile** — `./gradlew clean compileJava` (clean, every time; §4 masking rule). Green compile proves nothing about mixins or runtime.
2. **Boot-verify (dedicated server)** — stand up an isolated Fabric 26.2 server (fabric-api + moonlight + the mod only), boot to `Done`. This is the authoritative COMMON-mixin validator — it fails fast on the first bad common mixin with the exact target. Mixin faults surface in WAVES (registration crash hides datapack-load mixins hide world-load mixins) → check the log after EVERY fix. Confirm `wcf` loaded and `0 failed mixins`. Grep the log for `Found a remappable @Shadow | FAILED during APPLY | was not applied` — never trust boot-to-Done alone.
3. **Static mixin audit** — `scratchpad/mixin_audit.py` / `supp_mixin_check.py` parse the class constant pool for `@Mixin/@Shadow/@Inject/@Accessor/@Invoker` and check the merged jar. **NAME-LEVEL ONLY** — it catches target-class/`@Shadow`-field/method-name breaks but MISSES `@At` call-site + descriptor mismatches. First pass, not a verdict.
4. **⚠ CLIENT mixins are a BLIND SPOT for server boot** — a dedicated server never loads the `"client": []` array. A server-green port can still crash-cascade on the real client, one stale client-mixin at a time (each APPLY failure is fatal → whack-a-mole). Validate EVERY client mixin statically against the merged jar, or test on a real client.
5. **`bake_validate.py` (purple/invisible gate)** — `scratchpad/bake_validate.py scanjar <jar>` resolves blockstate→model→parent→textures exactly like Fabric's baker across the layered resource view. Flags BROKEN = missing blockstate/itemdef, a model with a Forge `"loader":` field + no vanilla geometry, a parent chain with no geometry, `parent:"builtin/entity"` (removed 26.2), or a missing non-particle texture. **This is the objective QC gate for any icon/model fix** — "purple/invisible" is a bake failure it decides. It does NOT catch runtime COLOR/tint failures (bake OK, wrong color — e.g. the backpack gray).
6. **Creative-inventory icon test** — items ship `models/item/*.json` but 26.2 REQUIRES a separate `assets/<ns>/items/<name>.json` model DEFINITION (`{"model":{"type":"minecraft:model","model":"<ns>:item/<name>"}}`). Mods built pre-1.21.4 ship zero `items/` defs → all icons purple. Also unguarded BEWLR / `builtin/entity` parents → purple icon. Generate one `items/` def per `models/item/` (`gen_item_defs.py`).
7. **RCON `execute if block <pos> <id>` phantom detection** — the block arg is parsed before the position-loaded check, so an UNregistered id throws `Unknown block type` read-only in ~2s. The ONLY reliable phantom detector (no offline heuristic works). `scratchpad/validate_registry.py` → `phantoms.txt`. (Of 25277 blockstate ids, 24896 real / 381 phantoms.) For mobs, the prune signal is **"Can't find element"** on summon, NOT "Unknown entity".
8. **In-client QA** — the last mile; boot-to-Done NEVER exercises client rendering. Walk a full block grid + spawn every mob. The datapack harness: `/function wolfgang:place_all_blocks` (every registered block, 1-block gap, per-mod square grids, doors/beds as whole multiblocks, forceload-tiled ≤256 chunks, default-gamerule-safe) + `summon_all_mobs`. Generator `scratchpad/gen_place_everything2.py`. The live client `logs/latest.log` is the game's own bake-error report (watcher `scratchpad/watch_client.py`).

**Rule:** compile → server-boot → static audit → client-mixin check → bake_validate → icon test → RCON phantom scan → in-client. Three independent "it works" illusions live here (green compile, boot-to-Done, asset-chain verify) — each is necessary and none is sufficient.

---

## 8. Common failure modes → fixes

| Symptom | Root cause | Fix |
|---|---|---|
| **Black screen** (render thread alive, CPU accruing, title never draws) | Startup resource-reload ABORTED → `clearResourcePacksOnError` rollback. Grep `latest.log` for **`Caught error loading resourcepacks, removing all selected resourcepacks`**; the `CompletionException` after it names the cause | Fix the underlying reload exception (below). Bisect by emptying `mods/` → vanilla renders → add back upward (libs first). Check the log after EVERY fix — a second cause hides behind the first |
| Black screen via atlas stitch — `Dest texture … not large enough` | Animated-texture `.png.mcmeta` with an EMPTY `"frames": []` (legal in 1.20.1, fatal in 26.2) | Delete the empty `frames` key (26.2 auto-detects from height). Scan rule: `animation.frames == []` |
| Black screen / purple — `pack_format > 64` rejected | `pack.mcmeta` missing `min_format`/`max_format` | Add `min_format`/`max_format` ([major,minor] arrays, e.g. `[88,0]`). Legacy `supported_formats` does NOT satisfy it |
| Black screen — "Failed to load required shader programs" | pre-26.2 `core/rendertype_text` shader renamed to `core/text` (Moonlight code-registered pipeline) | Alias the shader (inject `core/rendertype_text.vsh/.fsh` = copies of 26.2 `text.*`) |
| `NullPointerException: Components not bound yet` on reload | An `ItemStack` built in a `<clinit>` of a class that loads during resource reload (BER/renderer/model) — components not bound yet (sophisticatedstorage `DisplayItemRenderer` static init) | Make it a lazy static getter. **Never construct ItemStack / `getDefaultInstance()` in a `<clinit>` of a reload-loadable class** |
| Item/block renders **purple or invisible when PLACED** (icon looked fine) | Forge/NeoForge block model with a `"loader":` field (or apexcore runtime-gen model) → Fabric silently fails to parse/bake → invisible/purple. Icons use opaque fallback textures so they masked it | Register a Fabric model loader OR flatten geometry to a plain vanilla `elements` model. `bake_validate.py` is the gate |
| Backpack/tinted item renders GRAY | Relied on Forge/NeoForge tint sources that don't apply on Fabric → grayscale × white | Bake the tint into the texture (grayscale × color per-channel, alpha preserved), strip `tintindex`/`tints`; OR register a Fabric `ItemTintSource` via an `ID_MAPPER.put` mixin |
| False "crash" — `ZipException: bad LOC / zip END header not found` | A jar was rewritten (jar surgery) while the game was OPEN — live jars are locked/mapped | **Close the game before touching `mods/*.jar`.** Edit options.txt / mcmeta / jars only game-closed (MC rewrites options.txt on exit, clobbering live edits) |
| `ClassNotFoundException` at mixin PREPARE on client | mixins.json declares `client` mixin classes the port stripped | Remove the dangling `client` entries (keep only classes present in the jar) |
| `ClassNotFoundException` for an integration entrypoint when JEI/Jade present | fabric.mod.json lists entrypoints for deferred compat classes not in the jar | Trim those entrypoints |
| `validateAccessWidener` fails the build | AW lines target members removed in 26.2 (usually deferred client-render) | Clean compile ⇒ no live code uses them ⇒ strip exactly those flagged line numbers |

**Rule:** a black screen is almost always a resource-reload abort, not a render-backend failure — grep the one magic log line first. Everything with "purple/invisible" is a bake failure (`"loader":` field, missing `items/` def, `builtin/entity`), gated by `bake_validate.py`. And never edit a jar with the game open.

---

## 9. Distribution

**Distribute via CurseForge EXPORT ZIP (with overrides), NOT the share/import CODE.**
- The code is manifest-only — it references CF file-ids = STOCK files, so it **silently drops all ~122 hand-ported mods** (every mod with no real 26.2 CF release: supplementaries, sophisticated*, YUNG's, fantasyfurniture set, pam's, porting_lib_* modules, team_reborn energy, wolfgang* fixes) AND every jar you baked a fix into.
- The export ZIP includes the **overrides** folder → it carries all 233 jars 1:1 and is complete.
- Same trap applies to deploying to a host: **upload the actual jar FILES**, not a CF export. CF launcher "modified files" triangles are your intentional edits and do NOT travel to the server.

**CurseForge "edited files" management (111 hand-edited mods):**
- The "this project's files were modified" flags are YOUR edits. **Never click Update / Reinstall / Update-All** — it reverts the fix.
- Audit (`audit_edits.py`) split them: **101 asset-only** (lang/texture/model/blockstate → movable to the resource pack via the pack-override-to-free pattern) vs **10 real code-edits** (keep pinned): supplementaries, sophisticatedstorage/core/backpacks, neapolitan, herios_floral_expansion, farm_and_charm, exposure, chisels-and-bits, wolfgangclientfixes.
- The flag is cached as `isModified` in `minecraftinstance.json` and NOT auto-cleared even after a byte-identical restore. To clear without clicking reinstall: **edit `minecraftinstance.json` while CurseForge is fully CLOSED** — recompute CF's fingerprint (murmur2 seed=1 over the file bytes with tab/LF/CR/space `0x09/0a/0d/20` stripped) per addon; if it MATCHES the stored `packageFingerprint`, set `isModified:false` + each `modules[].invalidFingerprint:false` (genuinely-edited mods mismatch and correctly stay pinned).

**The "Wolfgang Core Fixes (DO NOT REMOVE)" resource pack is REQUIRED for distribution** — it holds the baked item-model defs, lang aliases, and model fixes. Push it to players via `server.properties` `resource-pack=<URL>` + `resource-pack-sha1` (recompute sha1 at deploy).

**Server host (Nodecraft) config — this is a Fabric 26.2 server, not the old 1.20.1 Forge one:**
- Loader = **Fabric 26.2**, Java = **25** (not 21), GC = **g1gc** (stale `concMarkSweep` = JVM won't start on Java 14+). Java 21 / Forge showing = still provisioned as 1.20.1.
- **Fresh world required** — a 1.20.1 or BOP-version-mismatched world fails `IllegalStateException: Overworld settings missing` (saved `blueprint:modded` biome source won't decode). Rename `world/` to force regen.
- Watch duplicate mods (two `biomesoplenty`/`shogi` loaded; loader picked newest but tidy them). The CF manifest pins stale BOP 26.1.2.0.40 — never let a CF import supply BOP; the tested 26.2.0.0.28 must win. Dedup check: scan each jar's `fabric.mod.json` id; clean = N jars / N distinct ids / 0 dups.
- Perf stack present (lithium/ferritecore/c2me/krypton/scalablelux/noisium/spark; ModernFix has NO 26.2 build — skip). Chunky pre-gen survived radius-2000 (63k chunks, 0 crashes).
- **Keep `wolfgang_loot_fixes` datapack** (server-side ash loot fix) when the checklist says "delete `wolfgang_qa`" (the QA block-placer) — they are different.

**Rule:** export-zip-with-overrides or raw files only; the import code drops every hand-port. The resource pack is a hard dependency for players. Never pre-deploy-update pinned versions.

---

## 10. Difficulty tiering (what makes a mod hard)

| Tier | Effort | What it is | Wolfgang case studies |
|---|---|---|---|
| **0 — Trivial** | minutes | Official 26.2 Fabric release exists → just install | Ecologics, Waystones, AppleSkin, Clumps, Friends&Foes (111 mods) |
| **0b — Delete** | minutes | Feature is now vanilla (backport) → drop | backported_wolves, shelfbackport, golden_dandelion_backport (7 mods) |
| **0c — Swap** | minutes | Native-Fabric equivalent/refork exists | Farmer's Delight → Refabricated; Curios → Accessories; Cluttered → Fabric fork; Starlight → drop |
| **1 — Loader-glue** | hours | Repo has a 26.x NeoForge branch → only `@Mod`/events/registries/config change; vanilla APIs already correct | solcarrot (1 compile error), Patchouli (26.1→26.2), ApexCore/FantasyFurniture 8 modules, Another Furniture |
| **2 — Asset-only edit** | hours | Code fine; just missing 26.2 `items/` defs, lang aliases, mcmeta, `grass`→`short_grass` | Let's Do family icon defs, Twigs item models, worldgen JSON migrations |
| **3 — Source port (self-contained)** | days | Forge/Fabric ≤1.21 → 26.2 version lift: API renames + NBT + maybe one GUI/render surface, but ONE module, no deep lib chain | Twigs, Creatures & Beasts, Artifacts, Neapolitan, Fairy Lights |
| **4 — Deep multi-mod port** | weeks / multi-session | A lib + its dependents, a sealed-Holder/registration-model rewrite, full GUI + in-world submit-pipeline render, cross-module mixin waves | **Sophisticated Core/Storage/Backpacks + 11 porting_lib modules**; **Supplementaries** (2845→0 errors) + Amendments; the whole render wall |

**What pushes a mod up the tiers (score these before committing):**
- **Registration model** — Forge `DeferredRegister`/Registrate DSL over hundreds of call sites + 26.2's new `Properties.setId` requirement + sealed `Holder` = a framework-shim port before you touch content (Tier 4).
- **GUI surface** — any screen/widget drags in the entire GuiGraphicsExtractor retained-mode pass (§5). Sophisticated Core was 620 GUI errors alone.
- **In-world custom geometry** — BER/entity renderers against the removed `MultiBufferSource` = the submit-pipeline reimpl (the genuine wall; everything else is mechanical).
- **Mixin depth** — string-targeted mixins compile green then fail in waves at datapack/world/client-load; client mixins are a server-boot blind spot. Deep block/level/entity mixins (porting_lib `blocks`) are the slow part.
- **Dependency chain** — porting_lib's `registry → {resources,blocks} → {loot,level_events,item_abilities}` cascade had to be ported bottom-up and published to mavenLocal before Backpacks could even compile.
- **NeoForge-bound APIs** — villager trades (`VillagerTradesEvent` has no fabric-api equal), custom cauldron interactions (private maps), tool-ability hooks = defer/cut features, or the mod stays NeoForge-only (Sophisticated Storage → Connector).

**Rule:** tier every mod at decision time. The pack's cost is dominated by the handful of Tier-4 mods — budget them as multi-session, keep the rest on the mechanical conveyor, and never let a Tier-0/1 mod accidentally get source-ported.

---

## Appendix — key paths

- **26.2 vanilla jar (signatures):** `%USERPROFILE%\curseforge\minecraft\Install\versions\26.2\26.2.jar`
- **Crash-guard mod:** `%USERPROFILE%\mc-ports\wolfgangclientfixes` (build `build-manual.sh`); deployed `wolfgangclientfixes-1.0.jar` + `wolfgangfixes-1.0.jar`
- **Port workspaces:** `%USERPROFILE%\Desktop\supp-port\supplementaries` (Supplementaries), `%USERPROFILE%\mc-ports\sophisticated-build\{core,storage,backpacks}` (+ porting_lib), `%USERPROFILE%\mc-ports\{solcarrot,patchouli,apexcore,blueprint,fairylights,herios-floral,pamhc2}`, `%USERPROFILE%\mc-ports\letsdo\out` (Let's Do family — reuse, don't re-port)
- **Validators / codemods (session scratchpad):** `bake_validate.py`, `validate_registry.py`, `mixin_audit.py` / `supp_mixin_check.py`, `fix_recipes.py`, `setid_codemod.py`, `gen_item_defs.py`, `gen_place_everything2.py`, `compare.py` / `compare2.py`
- **Old 1.20.1 jars (texture/model recovery):** `%USERPROFILE%\curseforge\minecraft\Instances\Wolfgang\mods`, `Wolfgang (2)\mods`
- **Deeper reference:** the `memory/` directory — `fabric-262-api-changes.md` (full API catalog), `fabric-262-gui-render-port.md`, `sophisticated-port-state.md`, `wolfgang-262-black-screen-rootcauses.md`, `wolfgang-262-distribution.md`, `curseforge-edited-mods-management.md`.
