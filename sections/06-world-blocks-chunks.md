---
name: paper-plugin-dev-world-blocks-chunks
description: >
  Expansion to paper-plugin-dev.md covering Paper world / block / chunk APIs end-to-end:
  Block vs BlockData vs BlockState vs TileState distinction, the new 1.18+ height range
  (-64 to 320), createBlockData parsing, waterlogged + multi-part blocks, BlockState#update
  semantics, TileState PDC persistence, chunk API (getChunkAt, isLoaded, ChunkSnapshot
  for async safety, plugin chunk tickets), async chunk loading via getChunkAtAsync +
  Moonrise, world creation via WorldCreator + ChunkGenerator + BiomeProvider + LimitedRegion,
  Biome registry (post-1.20 deobfuscation, BiomeKeys, custom biomes), RegionAccessor as
  the unified read/write surface, block events (BlockPlaceEvent, BlockBreakEvent,
  BlockPhysicsEvent, BlockGrowEvent), Folia chunk-thread affinity, and a comprehensive
  failure cookbook with reference implementation.
---

# 6. WORLDS, BLOCKS & CHUNKS — DEEP DIVE

This document is a comprehensive companion to §9–§13 of `paper-plugin-dev.md`. The base
file mentions `World#getBlockAt`, `setType`, and chunks in passing. This file goes
through the entire spatial API: how blocks, chunks, and worlds relate, when each piece
mutates state vs reads state, and how to do all of it without crashing on Folia.

All snippets target **Paper 1.21.4+** (Mojang-mapped runtime). Where 1.18 / 1.20.5 added
new height ranges or registry surfaces they're flagged inline. Authoritative sources are
linked inline plus a [References](#6z-references) appendix.

---

## 6.0 MENTAL MODEL — THE FOUR BLOCK ABSTRACTIONS

Bukkit/Paper has **four** different "block" types and they are not interchangeable.
Choosing the wrong one is the #1 source of confusion for new plugin devs.

```
World ──── (chunk grid 16×16×worldHeight)
  │
  ├── Chunk (16×16×worldHeight slab)
  │     └── ChunkSnapshot (immutable, thread-safe copy)
  │
  └── Block (live cell at x,y,z)         <- a *handle*; cheap, doesn't carry data
        │
        ├── BlockData                    <- the *type + state* (e.g. "oak_stairs[facing=north]")
        │     │                              immutable, equals/hashCode by content
        │     ├── Waterlogged
        │     ├── Directional / Orientable
        │     ├── Stairs / Slab / Chest / Bed
        │     └── ... ~120 specialized interfaces
        │
        ├── BlockState                   <- a *snapshot* of the block's full state
        │     │                              copy when read; .update() to write back
        │     ├── Container (Chest, Furnace, Barrel, ShulkerBox, ...)
        │     ├── TileState              <- BlockState that has a block-entity (NBT + PDC)
        │     │     ├── Sign / SignSide
        │     │     ├── Skull / Beacon / Banner / Lectern / Spawner
        │     │     └── Banner / Beehive / DecoratedPot
        │     └── ...
        │
        └── Biome (read via World#getBiome / Block#getBiome)
```

**Quick decision tree:**
- "What block is this and what direction does it face?" → `BlockData`
- "Set this to oak stairs facing north, also be waterlogged" → `BlockData`
- "Open the chest's inventory" → `BlockState` (cast to `Chest`)
- "Read/write the sign's text or the spawner's mob type" → `TileState` subclass
- "Store custom plugin data on this block" → `TileState#getPersistentDataContainer()`
- "Iterate every block in this chunk safely from another thread" → `ChunkSnapshot`

References: [Paper Block Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/block/Block.html),
[BlockData (1.13+)](https://jd.papermc.io/paper/1.13/org/bukkit/block/data/BlockData.html),
[BlockState](https://jd.papermc.io/paper/1.20.3/org/bukkit/block/BlockState.html),
[TileState](https://jd.papermc.io/paper/1.21.1/org/bukkit/block/TileState.html).

---

## 6.1 `Block` — THE LIGHTWEIGHT HANDLE

`Block` is a *pointer* to a position in a world. It does not store the block type — every
call goes back to the world to read live state. This means `Block` is cheap to hold but
**holding it across ticks is fine while holding `BlockState` across ticks is not.**

```java
World world = Bukkit.getWorld("world");
Block block = world.getBlockAt(100, 64, 100);

// Type-only operations:
Material type = block.getType();
block.setType(Material.STONE);                 // updates physics
block.setType(Material.STONE, /* applyPhysics */ false);

// Data-aware operations:
BlockData data = block.getBlockData();
block.setBlockData(data, /* applyPhysics */ true);

// Position queries:
Location loc = block.getLocation();
Chunk chunk  = block.getChunk();
Biome biome  = block.getBiome();
int light    = block.getLightLevel();
boolean solid = block.isSolid();
boolean liquid = block.isLiquid();
```

`setType` vs `setBlockData`: `setType` resets the block to its default state (e.g. setting
to `STAIRS` gives you a default-facing stair). `setBlockData` lets you specify the exact
state. **Always prefer `setBlockData` for anything that has orientation / state.**

### 6.1.1 `applyPhysics` — when to disable

By default, `setType(...)` schedules a physics update so torches drop, redstone propagates,
sand falls, etc. For batch operations (filling a region, placing a structure) this can
cause cascading lag. Pass `false`:

```java
for (Block b : largeRegion) {
    b.setType(Material.STONE, false);
}
// One big physics flush at the end if needed:
world.refreshChunk(chunkX, chunkZ);
```

> **Caveat:** disabling physics on liquid-adjacent blocks can cause client desync until the
> next chunk reload. Acceptable for static structures, not for live edits in player view.

References: [Paper issue #1842 — setBlockData with tile entity](https://github.com/PaperMC/Paper/issues/1842).

---

## 6.2 `BlockData` — THE STATE-CARRYING IMMUTABLE

`BlockData` is the modern (1.13+) successor to legacy "byte data". It's an immutable
description of a block's full state, including all the type-specific properties.

### 6.2.1 Creating BlockData

```java
// From Material default:
BlockData data = Material.OAK_STAIRS.createBlockData();

// From a string (Mojang's "block-state" format):
BlockData stairs = Bukkit.createBlockData(
    "minecraft:oak_stairs[facing=north,half=top,waterlogged=true]");

// From an existing block (clone):
BlockData clone = block.getBlockData().clone();
```

The string format matches `/setblock` and the F3 debug screen, so you can copy-paste from
in-game.

### 6.2.2 Casting to specialized interfaces

```java
BlockData raw = Material.OAK_STAIRS.createBlockData();
if (raw instanceof Stairs stairs) {
    stairs.setFacing(BlockFace.NORTH);
    stairs.setHalf(Bisected.Half.BOTTOM);
    stairs.setShape(Stairs.Shape.STRAIGHT);
}
if (raw instanceof Waterlogged wl) {
    wl.setWaterlogged(true);
}
block.setBlockData(raw);
```

Common specialized interfaces:

| Interface | Used by | Property |
|---|---|---|
| `Directional` / `Orientable` | Pistons, Hoppers, Dispensers | `facing` |
| `Bisected` | Stairs, Tall blocks, Doors | `half` (top/bottom) |
| `Waterlogged` | Stairs, Slabs, Fences, Walls | `waterlogged` |
| `Stairs` | Stairs | `shape` (straight, inner_left, ...) |
| `Slab` | Slabs | `type` (top, bottom, double) |
| `Bed` | Beds | `part` (head/foot), `occupied` |
| `Chest` (data) | Chests | `type` (single, left, right) |
| `Door` | Doors | `open`, `hinge` |
| `Powerable` | Buttons, Pressure plates | `powered` |
| `Lightable` | Furnaces, Campfires, Redstone lamps | `lit` |
| `Light` (1.18+) | Light blocks | `level` (0–15) |

References: [BlockData Javadoc](https://jd.papermc.io/paper/1.13/org/bukkit/block/data/BlockData.html),
[Waterlogged](https://jd.papermc.io/paper/1.21.10/org/bukkit/block/data/Waterlogged.html),
[Light](https://jd.papermc.io/paper/1.21.3/org/bukkit/block/data/type/Light.html).

### 6.2.3 Equality & cloning

`BlockData` has proper `equals`/`hashCode` on its serialized form, so:

```java
if (block.getBlockData().matches(otherData)) { ... }   // pattern matching with wildcards
if (block.getBlockData().equals(otherData)) { ... }    // exact match
```

`matches(...)` (1.20+) is useful for "is this any oak stairs regardless of facing?" — it
treats unspecified properties as wildcards.

---

## 6.3 `BlockState` — THE READ-MUTATE-COMMIT PATTERN

`BlockState` is a *snapshot* of a block: its data **plus** any block-entity NBT. It's
copy-on-read; mutating it does nothing until you call `.update()`.

```java
BlockState state = block.getState();        // copy
state.setType(Material.GOLD_BLOCK);         // mutate the copy

// Block in the world is still unchanged:
assert block.getType() != Material.GOLD_BLOCK;

state.update(/* force */ true, /* applyPhysics */ true);
// Now the world matches.
```

### 6.3.1 When to use BlockState over BlockData

- The block has a **block entity** (chest, sign, spawner, beehive). You need `BlockState`
  to access its NBT-backed state.
- You want to mutate multiple fields atomically and apply once.
- You're holding the snapshot to apply later (e.g. building schematic preview).

**Don't** hold `BlockState` across ticks; the block in the world may change underneath you,
and `update()` will overwrite those changes silently. Re-read with `block.getState()`.

### 6.3.2 The `Container` family

```java
BlockState state = block.getState();
if (state instanceof Chest chest) {
    Inventory inv = chest.getInventory();
    inv.addItem(new ItemStack(Material.DIAMOND, 5));
    chest.update();
}
if (state instanceof Furnace furnace) {
    furnace.setBurnTime((short) 200);
    furnace.setCookTime((short) 50);
    furnace.update();
}
```

Containers used to need explicit `update()` to commit inventory changes; on modern Paper
inventory mutations write through to the live block automatically. Keep `update()` for
non-inventory state changes (custom name, lock).

---

## 6.4 `TileState` — BLOCK-ENTITY SUBCLASS WITH PDC

`TileState` is `BlockState` that has a backing block-entity (chest, sign, beehive, etc.).
The big addition: it has a `PersistentDataContainer`.

```java
BlockState state = block.getState();
if (state instanceof TileState ts) {
    PersistentDataContainer pdc = ts.getPersistentDataContainer();
    pdc.set(myKey, PersistentDataType.STRING, "owner-of-this-shop");
    ts.update();              // critical — PDC is part of the snapshot
}
```

### 6.4.1 PDC persistence rules for blocks

- PDC saves with the chunk on next world save.
- Survives chunk unload/reload (PDC lives in the block-entity NBT).
- Survives server restart.
- **Does not** survive `block.setType(Material.AIR)` — destroying the block-entity removes
  the PDC.

### 6.4.2 Common TileState subclasses

| Subclass | Purpose |
|---|---|
| `Sign` | Reading/writing sign text (now supports both faces — `SignSide` 1.20+) |
| `Skull` | Player-head texture / owner |
| `Beehive` | Entities currently inside |
| `Lectern` | Page number, the held book |
| `CreatureSpawner` | Mob type, spawn delay |
| `Beacon` | Active effects, tier |
| `Banner` | Pattern list |
| `DecoratedPot` (1.20+) | Sherd pattern |
| `Conduit` | Active state |
| `EnderChest` | (Paper) PDC-friendly variant |

References: [TileState Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/block/TileState.html).

### 6.4.3 Sign API (1.20+ — both sides)

```java
Sign sign = (Sign) block.getState();
SignSide front = sign.getSide(Side.FRONT);
front.line(0, Component.text("Welcome", NamedTextColor.GOLD));
front.line(1, Component.text("Player", NamedTextColor.WHITE));
front.setGlowingText(true);
front.setColor(DyeColor.YELLOW);
sign.update();
```

`Side.BACK` lets you write to the back face independently (1.20 added back-face text).

---

## 6.5 CHUNKS — THE LOAD/UNLOAD UNIT

A `Chunk` is a 16×16×worldHeight slab. The whole world is a sparse grid of these. Two
properties matter:

1. **Loaded** — chunk is in memory and ticking.
2. **Generated** — chunk has had terrain populated (regardless of load state).

```java
Chunk chunk = world.getChunkAt(x, z);          // ensures loaded; sync if not
boolean loaded    = world.isChunkLoaded(x, z);
boolean generated = world.isChunkGenerated(x, z);

chunk.load();                                  // sync load if not loaded
chunk.unload(/* save */ true);                 // sync unload + save
```

### 6.5.1 Plugin chunk tickets (modern keep-loaded)

```java
world.addPluginChunkTicket(chunkX, chunkZ, plugin);
// Chunk stays loaded until removeTicket or plugin disable.
world.removePluginChunkTicket(chunkX, chunkZ, plugin);
```

Chunk tickets replace the legacy `chunk.load()` / `setChunkForceLoaded(true)` patterns.
They're the official, refcounted way to keep a chunk loaded and they automatically clear
when your plugin disables.

References: [Chunk#addPluginChunkTicket](https://jd.papermc.io/paper/1.17.1/org/bukkit/Chunk.html),
[World#addPluginChunkTicket](https://jd.papermc.io/paper/1.21.8/org/bukkit/World.html).

### 6.5.2 Async chunk loading (Paper extension)

```java
CompletableFuture<Chunk> future = world.getChunkAtAsync(x, z);
future.thenAccept(chunk -> {
    // Runs on main thread once chunk is loaded.
    Block b = chunk.getBlock(0, 64, 0);
    b.setType(Material.STONE);
});

// Or with generation gen:
world.getChunkAtAsync(x, z, /* gen */ true).thenAccept(...);

// urgent:
world.getChunkAtAsyncUrgently(x, z).thenAccept(...);
```

Paper's chunk system (Moonrise, default since ~1.20.5) does multi-threaded chunk I/O and
generation. The future completes on the main thread by design. Never block waiting for
it; chain via `thenAccept`.

References: [Paper Chunk loading optimization (Moonrise)](https://papermc-paper.mintlify.app/optimization/chunk-loading),
[PaperLib (legacy compat shim)](https://github.com/PaperMC/PaperLib).

### 6.5.3 ChunkSnapshot — the async-safe view

When you need to read a chunk's blocks from another thread (heatmap generation, world
analysis), copy it into a `ChunkSnapshot`. Snapshots are immutable and thread-safe.

```java
ChunkSnapshot snap = chunk.getChunkSnapshot(/* includeMaxBlockY */ true,
                                            /* includeBiome */     true,
                                            /* includeBiomeTemp */ false);

CompletableFuture.runAsync(() -> {
    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            int y = snap.getHighestBlockYAt(x, z);
            BlockData data = snap.getBlockData(x, y, z);
            // ... heavy computation here ...
        }
    }
});
```

References: [ChunkSnapshot Javadoc](https://jd.papermc.io/paper/1.21.11/org/bukkit/ChunkSnapshot.html).

### 6.5.4 Iterating chunks

```java
// Loaded chunks only:
for (Chunk c : world.getLoadedChunks()) { ... }

// Force-loaded chunks (ticket-pinned):
for (Chunk c : world.getForceLoadedChunks()) { ... }

// All entity-bearing chunks (Paper):
for (Chunk c : world.getEntities().stream().map(Entity::getChunk).distinct().toList()) { ... }
```

`world.getLoadedChunks()` returns a snapshot array; safe to iterate without CME.

---

## 6.6 WORLD HEIGHT (post-1.18)

Since 1.18, dimensions have a vertical range:

| Dimension | Min Y | Max Y | Build limit |
|---|---|---|---|
| Overworld | -64 | 320 | 319 |
| Nether | 0 | 256 | 255 |
| End | 0 | 256 | 255 |
| Custom | data-driven | data-driven | data-driven |

Read it via `World#getMinHeight()` / `getMaxHeight()`:

```java
for (int y = world.getMinHeight(); y < world.getMaxHeight(); y++) {
    Block b = world.getBlockAt(x, y, z);
    // ...
}
```

**Never hard-code 0 or 256** for vertical loops — old plugins break in custom dimensions.
Anti-Xray and other Paper world configs read these dynamic ranges. See
[Paper world-configuration docs](https://docs.papermc.io/paper/reference/world-configuration).

---

## 6.7 BIOMES — POST-1.20 DEOBFUSCATION

In 1.20.5+ the `Biome` enum became deprecated for removal. Custom datapacks can register
biomes that aren't in the enum. The replacement is the registry API.

```java
import io.papermc.paper.registry.RegistryAccess;
import io.papermc.paper.registry.RegistryKey;
import io.papermc.paper.registry.keys.BiomeKeys;

// Vanilla biome via type-safe key (preferred):
Biome plains = RegistryAccess.registryAccess()
    .getRegistry(RegistryKey.BIOME)
    .get(BiomeKeys.PLAINS);

// Custom datapack biome:
Biome myBiome = RegistryAccess.registryAccess()
    .getRegistry(RegistryKey.BIOME)
    .get(NamespacedKey.fromString("mypack:scorched_plains"));

// Set it on a block:
block.setBiome(plains);
```

`BiomeKeys` (in `io.papermc.paper.registry.keys`) lists every vanilla biome with type-safe
constants. Use it instead of the deprecated `Biome.PLAINS` enum.

References: [Paper Registry API docs](https://papermc-paper.mintlify.app/api/registry),
[BiomeKeys Javadoc](https://jd.papermc.io/folia/io/papermc/paper/registry/keys/BiomeKeys.html),
[Biome (deprecated note)](https://jd.papermc.io/paper/1.21.11/org/bukkit/block/Biome.html).

---

## 6.8 `RegionAccessor` — THE UNIFIED INTERFACE

`World` extends `RegionAccessor`, which is also implemented by `LimitedRegion` (used in
chunk generation). It's the unified read/write surface for blocks, biomes, and entities.

```java
public interface RegionAccessor {
    Material getType(int x, int y, int z);
    BlockData getBlockData(int x, int y, int z);
    void setType(int x, int y, int z, Material type);
    void setBlockData(int x, int y, int z, BlockData data);
    BlockState getBlockState(int x, int y, int z);
    Biome getBiome(int x, int y, int z);
    void setBiome(int x, int y, int z, Biome biome);
    <T extends Entity> T spawn(Location loc, Class<T> clazz);
    // ...
}
```

Code that takes `RegionAccessor` works equally well with a live `World` and a generation
`LimitedRegion`. Use this when writing utilities.

References: [RegionAccessor Javadoc](https://jd.papermc.io/paper/1.21.0/org/bukkit/RegionAccessor.html).

---

## 6.9 WORLD CREATION — `WorldCreator` + `ChunkGenerator`

```java
WorldCreator creator = new WorldCreator(NamespacedKey.fromString("myplugin:arena"))
    .environment(World.Environment.NORMAL)
    .type(WorldType.FLAT)
    .generateStructures(false)
    .seed(42L)
    .generator(new ArenaGenerator())
    .biomeProvider(new SinglesBiomeProvider(Biome.PLAINS));

World world = creator.createWorld();
```

### 6.9.1 ChunkGenerator (custom terrain)

```java
public final class ArenaGenerator extends ChunkGenerator {
    @Override
    public void generateNoise(WorldInfo info, Random rng, int chunkX, int chunkZ, ChunkData data) {
        for (int x = 0; x < 16; x++)
        for (int z = 0; z < 16; z++) {
            data.setBlock(x, info.getMinHeight(), z, Material.BEDROCK);
            for (int y = info.getMinHeight() + 1; y < 64; y++) {
                data.setBlock(x, y, z, Material.STONE);
            }
            data.setBlock(x, 64, z, Material.GRASS_BLOCK);
        }
    }

    @Override
    public boolean shouldGenerateNoise() { return false; }   // we did it ourselves
    @Override public boolean shouldGenerateSurface() { return false; }
    @Override public boolean shouldGenerateBedrock() { return false; }
    @Override public boolean shouldGenerateCaves() { return false; }
    @Override public boolean shouldGenerateDecorations() { return false; }
    @Override public boolean shouldGenerateMobs() { return true; }
    @Override public boolean shouldGenerateStructures() { return false; }
}
```

`ChunkGenerator` has multiple overrideable phases (noise, surface, bedrock, caves,
decoration, structures). Override only what you need. References:
[ChunkGenerator Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/generator/ChunkGenerator.html),
[ChunkData Javadoc](https://jd.papermc.io/paper/1.21.0/org/bukkit/generator/ChunkGenerator.ChunkData.html).

### 6.9.2 BiomeProvider (custom biome map)

```java
public final class SinglesBiomeProvider extends BiomeProvider {
    private final Biome biome;
    public SinglesBiomeProvider(Biome b) { this.biome = b; }

    @Override
    public Biome getBiome(WorldInfo info, int x, int y, int z) {
        return biome;
    }
    @Override
    public List<Biome> getBiomes(WorldInfo info) {
        return List.of(biome);
    }
}
```

References: [BiomeProvider Javadoc](https://jd.papermc.io/paper/1.21.11/org/bukkit/generator/BiomeProvider.html).

### 6.9.3 LimitedRegion — accessing neighbours during generation

`ChunkGenerator#generateDecorations` gets a `LimitedRegion` that exposes a buffer of
adjacent chunks (typically 3×3 around the current). Use this to place decorations that
straddle chunk boundaries (vines, large trees, structures) without crashing on chunk-load
ordering.

```java
@Override
public void generateDecorations(WorldInfo info, Random rng, int chunkX, int chunkZ, LimitedRegion region) {
    int worldX = chunkX * 16 + 8;
    int worldZ = chunkZ * 16 + 8;
    if (region.getBlock(worldX, 64, worldZ).getType() == Material.GRASS_BLOCK) {
        // Place a tree extending up to 6 blocks away — LimitedRegion has the buffer
        for (int y = 65; y < 75; y++) region.setType(worldX, y, worldZ, Material.OAK_LOG);
    }
}
```

References: [LimitedRegion Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/generator/LimitedRegion.html).

### 6.9.4 plugin.yml registration (legacy alternative)

For plugins that ship a generator and want to be selectable via `bukkit.yml`:

```yaml
# plugin.yml
name: ArenaPlugin
main: com.example.ArenaPlugin
api-version: '1.21'

# bukkit.yml — managed externally, but plugin must implement getDefaultWorldGenerator
# in JavaPlugin
```

```java
@Override
public ChunkGenerator getDefaultWorldGenerator(String worldName, String id) {
    return new ArenaGenerator();
}
```

Now `bukkit.yml` can specify `generator: ArenaPlugin` for any world.

---

## 6.10 BLOCK & WORLD EVENTS WORTH KNOWING

| Event | Fires when | Cancelable? |
|---|---|---|
| `BlockPlaceEvent` | Player places a block | Yes |
| `BlockBreakEvent` | Player breaks a block | Yes |
| `BlockBurnEvent` | Block consumed by fire | Yes |
| `BlockIgniteEvent` | Block ignited (lava, lightning, fire spread, ...) | Yes |
| `BlockSpreadEvent` | Block grew sideways (mushroom, fire, vines) | Yes |
| `BlockGrowEvent` | Block grew in place (sapling -> tree pre-update) | Yes |
| `BlockFromToEvent` | Liquid flowed | Yes |
| `BlockPhysicsEvent` | Physics tick (very high frequency!) | Yes |
| `BlockExplodeEvent` | Block exploded (TNT, end crystal) | Yes (filter `getBlockList()`) |
| `BlockFadeEvent` | Block faded (snow melt, ice -> water) | Yes |
| `BlockDispenseEvent` | Dispenser fired | Yes |
| `BlockRedstoneEvent` | Redstone signal change | Yes (mutate `setNewCurrent`) |
| `LeavesDecayEvent` | Leaves decayed | Yes |
| `BlockFormEvent` | Block formed (snow, ice, frost walker) | Yes |
| `ChunkLoadEvent` | Chunk loaded | No |
| `ChunkUnloadEvent` | Chunk unloaded | No (in 1.14+) |
| `ChunkPopulateEvent` | Chunk decorations placed (pre-1.18 surface) | No |
| `WorldLoadEvent` / `WorldUnloadEvent` | World loaded / unloaded | Unload yes |
| `WorldSaveEvent` | World save started | No |
| `WorldInitEvent` | World initialized (very early) | No |

> **`BlockPhysicsEvent` is high-frequency.** Don't do heavy work in its handler. Filter
> early on `event.getBlock().getType()` and return.

---

## 6.11 FOLIA THREAD-AFFINITY FOR WORLDS / CHUNKS / BLOCKS

On Folia, each chunk belongs to a region; only the region's owning thread may touch its
blocks.

### 6.11.1 RegionScheduler for block operations

```java
plugin.getServer().getRegionScheduler().run(plugin, location, task -> {
    location.getBlock().setType(Material.STONE);
});

// Or by chunk:
plugin.getServer().getRegionScheduler().run(plugin, world, chunkX, chunkZ, task -> {
    Chunk c = world.getChunkAt(chunkX, chunkZ);
    // ...
});
```

### 6.11.2 The cross-region trap

A region of blocks may straddle multiple owning regions (if you're modifying a 1000-block
square). Schedule per-chunk, not per-region:

```java
for (int cx = minCx; cx <= maxCx; cx++) {
    for (int cz = minCz; cz <= maxCz; cz++) {
        final int chunkX = cx, chunkZ = cz;
        plugin.getServer().getRegionScheduler().run(plugin, world, chunkX, chunkZ, task -> {
            // Mutations limited to chunk (chunkX, chunkZ) here
            for (int x = 0; x < 16; x++)
            for (int z = 0; z < 16; z++)
            for (int y = world.getMinHeight(); y < 70; y++) {
                world.getBlockAt(chunkX*16 + x, y, chunkZ*16 + z).setType(Material.STONE);
            }
        });
    }
}
```

References: [Folia RegionScheduler Javadoc](https://jd.papermc.io/folia/io/papermc/paper/threadedregions/scheduler/RegionScheduler.html),
[Folia chunk async load callback](https://jd.papermc.io/folia/1.21/org/bukkit/World.ChunkLoadCallback.html).

### 6.11.3 World listing + global ops

```java
// Always safe (even on Folia):
List<World> worlds = Bukkit.getWorlds();
String name        = world.getName();
UUID id            = world.getUID();
WorldType type     = world.getWorldType();

// Need scheduler on Folia:
//   - any block mutation
//   - chunk load/unload
//   - entity spawn
//   - setSpawnLocation
```

---

## 6.12 ASYNC-SAFETY MATRIX

| Operation | Main thread | Async thread | Folia region scheduler |
|---|---|---|---|
| `block.getType()` | OK | UNSAFE | OK |
| `block.setType(...)` | OK | UNSAFE | OK |
| `block.getState()` | OK | UNSAFE | OK |
| `state.update()` | OK | UNSAFE | OK |
| `chunk.getChunkSnapshot()` | OK | UNSAFE (must call from main first) | OK |
| Reading from a `ChunkSnapshot` | OK | OK (snapshots are thread-safe) | OK |
| `world.getBlockAt(...)` | OK | UNSAFE | OK if region owns chunk |
| `world.getChunkAtAsync(...)` | OK (returns immediately) | OK (callback runs on main) | OK |
| `world.getLoadedChunks()` | OK | UNSAFE | NOT regional |
| `world.getName()` / `getUID()` | OK | OK | OK |
| `Bukkit.getWorlds()` | OK | OK (immutable list) | OK |
| `world.getMinHeight()` / `getMaxHeight()` | OK | OK | OK |

---

## 6.13 COMMON WORLD/BLOCK FAILURES

| Symptom | Cause | Fix |
|---|---|---|
| Sign text not saving | Forgot `sign.update()` | Always `update()` after mutations on TileState |
| `setBlockData` removes block-entity contents | Default `applyPhysics=true` cleared NBT | Use Paper's setType overload that preserves; see [#1842](https://github.com/PaperMC/Paper/issues/1842) |
| Loop iterating `0..255` finds no blocks | World minHeight=-64 in 1.18+ | Use `world.getMinHeight()` / `getMaxHeight()` |
| `IllegalStateException: chunk not loaded` | Sync access to unloaded chunk on Folia | Use `getChunkAtAsync` or RegionScheduler |
| Async block read crashes | Reading live block from async | Use `ChunkSnapshot` |
| Plugin chunk ticket not removed on disable | Ticket leak after reload | `addPluginChunkTicket` auto-clears on disable; if using legacy `setChunkForceLoaded` you must remove |
| `Biome.PLAINS` flagged deprecated | 1.20.5+ post-deobfuscation | Use `RegistryAccess.registryAccess().getRegistry(RegistryKey.BIOME).get(BiomeKeys.PLAINS)` |
| Custom datapack biome returns null from `Biome.valueOf` | Custom biomes aren't enum constants | Resolve via registry, not enum |
| Waterlogged stairs reset on `setType(Material.OAK_STAIRS)` | setType uses default state | Use `setBlockData` with explicit waterlogged=true |
| `BlockPhysicsEvent` ate TPS | Handler doing heavy work per-event | Early-return on type check; avoid this event for "block changed" |
| `applyPhysics=false` left floating sand | Disabled physics block updates | Re-enable on perimeter blocks or call `world.refreshChunk` |
| Large structure place lags server | Each `setType` schedules light + physics | Batch with `applyPhysics=false`, then flush |
| Chunk listener doesn't see entities | `ChunkLoadEvent` fires before entity load | Use `EntitiesLoadEvent` (Paper) instead |
| `getChunkAtAsync` future never completes | Callback exception swallowed | `.exceptionally(e -> { e.printStackTrace(); return null; })` |
| `BlockData#matches` returns false unexpectedly | Wrong key/value comparison | Use `equals` for exact, `matches` only for partial |
| `ChunkGenerator` produces bedrock at y=0 | Custom dimension has minY=-64 | Use `info.getMinHeight()` not 0 |

---

## 6.14 PERFORMANCE NOTES

- **`world.getBlockAt`** is O(1) but goes through a chunk lookup. Cache the `Chunk`
  reference if iterating tightly within one chunk.
- **`block.getState()`** allocates and copies block-entity NBT. Avoid in hot paths;
  prefer `BlockData` if you don't need block-entity fields.
- **`ChunkSnapshot`** allocates a full chunk's worth of state. Reuse the snapshot for
  all reads; don't request a new one per block.
- **`BlockPhysicsEvent`** can fire millions of times. Filter aggressively at the top.
- **`addPluginChunkTicket`** is cheap, but loaded chunks tick. Don't keep 10,000 chunks
  loaded. Use timed unload or radius-based logic.
- **Chunk save frequency**: Paper auto-saves chunks; `World#save()` from a plugin forces
  a save on the main thread and stalls. Avoid except at shutdown.
- **WorldEdit-style fills**: prefer `RegionAccessor` + raw `setBlockData(x,y,z,data)`
  over loops of `getBlockAt(...).setBlockData(...)`. Same end state but skips a method
  hop per block.

---

## 6.15 REFERENCE IMPLEMENTATION

A "magical region" plugin: claim a chunk, store owner in PDC on a magic block, and
prevent anyone else from breaking blocks inside it.

```java
package com.example.region;

import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import org.bukkit.*;
import org.bukkit.block.*;
import org.bukkit.entity.Player;
import org.bukkit.event.*;
import org.bukkit.event.block.*;
import org.bukkit.persistence.*;
import org.bukkit.plugin.java.JavaPlugin;

import java.util.UUID;

public final class RegionPlugin extends JavaPlugin implements Listener {
    private NamespacedKey ownerKey;

    @Override
    public void onEnable() {
        ownerKey = new NamespacedKey(this, "owner");

        Bukkit.getPluginManager().registerEvents(this, this);

        getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, e -> {
            e.registrar().register(
                Commands.literal("claim")
                    .requires(s -> s.getSender() instanceof Player)
                    .executes(ctx -> {
                        Player p = (Player) ctx.getSource().getSender();
                        return claimChunk(p);
                    }).build()
            );
        });
    }

    private int claimChunk(Player p) {
        Chunk chunk = p.getLocation().getChunk();
        Block beacon = chunk.getBlock(8, p.getWorld().getMinHeight() + 64, 8);

        // Pin the chunk so the beacon stays loaded
        p.getWorld().addPluginChunkTicket(chunk.getX(), chunk.getZ(), this);

        // Place a magic beacon (BlockData ensures correct default state)
        beacon.setBlockData(Material.BEACON.createBlockData(), false);

        // Tag block-entity with owner UUID via TileState PDC
        BlockState state = beacon.getState();
        if (state instanceof TileState ts) {
            ts.getPersistentDataContainer().set(ownerKey, PersistentDataType.STRING,
                p.getUniqueId().toString());
            ts.update();
        }

        p.sendMessage(Component.text("Chunk claimed!", NamedTextColor.GREEN));
        return 1;
    }

    @EventHandler
    public void onBreak(BlockBreakEvent ev) {
        Chunk chunk = ev.getBlock().getChunk();
        UUID owner = chunkOwner(chunk);
        if (owner == null) return;
        if (!owner.equals(ev.getPlayer().getUniqueId())) {
            ev.setCancelled(true);
            ev.getPlayer().sendMessage(Component.text("This chunk is claimed.", NamedTextColor.RED));
        }
    }

    @EventHandler
    public void onPlace(BlockPlaceEvent ev) {
        Chunk chunk = ev.getBlock().getChunk();
        UUID owner = chunkOwner(chunk);
        if (owner == null) return;
        if (!owner.equals(ev.getPlayer().getUniqueId())) {
            ev.setCancelled(true);
        }
    }

    private UUID chunkOwner(Chunk chunk) {
        // Re-read the block-entity (don't cache TileState across calls)
        Block beacon = chunk.getBlock(8, chunk.getWorld().getMinHeight() + 64, 8);
        if (beacon.getType() != Material.BEACON) return null;
        BlockState state = beacon.getState();
        if (!(state instanceof TileState ts)) return null;
        String s = ts.getPersistentDataContainer().get(ownerKey, PersistentDataType.STRING);
        if (s == null) return null;
        try { return UUID.fromString(s); } catch (Exception e) { return null; }
    }
}
```

`paper-plugin.yml`:

```yaml
name: RegionPlugin
main: com.example.region.RegionPlugin
api-version: '1.21'
authors: ['You']
```

This shows: Block + BlockData + BlockState + TileState + PDC + chunk ticket + events all
working together.

---

## 6.16 LOCATION / VECTOR / BOUNDING-BOX UTILITIES

### 6.16.1 `Location` — world + (x,y,z) + yaw + pitch

```java
Location loc = new Location(world, 100.5, 64.0, 200.5, /* yaw */ 90f, /* pitch */ 0f);
loc.add(1, 0, 0);                                  // mutates in place
Location offset = loc.clone().add(0, 1, 0);        // immutable variant via clone

Block b   = loc.getBlock();
Vector v  = loc.toVector();                        // strip yaw/pitch
Location centered = loc.toCenterLocation();        // x.5, y.5, z.5

double dist  = loc.distance(other);                // sqrt — slow
double dsq   = loc.distanceSquared(other);         // prefer for comparisons

Vector dir = loc.getDirection();                   // unit vector from yaw/pitch
```

**Pitfall:** `Location` mutators (`add`, `subtract`, `setX`) modify the receiver. Always
`.clone()` before mutating a Location you got from someone else's API.

### 6.16.2 `Vector` — pure 3D math, no world

```java
Vector v = new Vector(1, 0, 0);
v.normalize();
v.multiply(2.5);
v.add(new Vector(0, 1, 0));
double len = v.length();
double lensq = v.lengthSquared();

// Rotate around Y axis by 30 degrees:
v.rotateAroundY(Math.toRadians(30));
```

`Vector` is widely used for projectile velocity, knockback, particle direction. It has no
world reference — combine with a `Location` only when you need positioning.

### 6.16.3 `BoundingBox` — AABB queries

```java
BoundingBox box = BoundingBox.of(loc1, loc2).expand(2.0);
boolean inside  = box.contains(player.getLocation().toVector());

// Find entities in a box:
Collection<Entity> hits = world.getNearbyEntities(box);

// Block raycast through a box:
RayTraceResult hit = world.rayTraceBlocks(start, direction, /* maxDistance */ 50);
if (hit != null && hit.getHitBlock() != null) {
    Block b = hit.getHitBlock();
    BlockFace face = hit.getHitBlockFace();
}
```

### 6.16.4 `World#rayTrace` — entity + block raycast

```java
RayTraceResult hit = world.rayTrace(
    eyeLocation,
    direction,
    /* maxDistance */ 30,
    FluidCollisionMode.NEVER,
    /* ignorePassableBlocks */ true,
    /* raySize */ 0.1,
    /* entityFilter */ e -> e instanceof LivingEntity && !(e instanceof Player)
);
```

Returns the first hit (block or entity). Useful for crosshair targeting, AOE origin, line
of sight.

---

## 6.17 WORLD STATE — TIME, WEATHER, BORDER, GAMERULES

```java
// Time + weather
long ticks = world.getTime();           // 0 .. 23999
world.setTime(6000);                    // noon
world.setStorm(true);
world.setThundering(true);
world.setWeatherDuration(20 * 60 * 5);  // 5 minutes

// Game rules
world.setGameRule(GameRule.DO_DAYLIGHT_CYCLE, false);
world.setGameRule(GameRule.KEEP_INVENTORY, true);
world.setGameRule(GameRule.MOB_GRIEFING, false);
world.setGameRule(GameRule.RANDOM_TICK_SPEED, 6);

// Spawn / difficulty / view distance
world.setSpawnLocation(0, 80, 0);
world.setDifficulty(Difficulty.HARD);
world.setViewDistance(8);                          // per-world override (1.21+)
world.setSimulationDistance(6);

// World border
WorldBorder border = world.getWorldBorder();
border.setCenter(0, 0);
border.setSize(2000);
border.setSize(500, /* timeSec */ 60);             // shrink over 60s
border.setWarningDistance(10);
border.setDamageBuffer(2.0);
border.setDamageAmount(0.5);
```

References: [Paper World Javadoc — game rules / weather / border](https://jd.papermc.io/paper/1.21.8/org/bukkit/World.html).

---

## 6.18 STRUCTURES — `StructureManager` + Schematics

Paper exposes vanilla Structure-block snapshots (`.nbt`) via `StructureManager`:

```java
StructureManager sm = Bukkit.getStructureManager();

// Load a structure from disk
Structure s = sm.loadStructure(NamespacedKey.fromString("minecraft:village/plains/houses/plains_small_house_1"));

// Place into the world:
s.place(loc,
    /* includeEntities */ true,
    StructureRotation.CLOCKWISE_90,
    Mirror.NONE,
    /* palette */ 0,
    /* integrity */ 1.0f,
    new Random());

// Capture an existing region:
Structure capture = sm.createStructure();
capture.fill(corner1, corner2, /* includeEntities */ false);
sm.saveStructure(NamespacedKey.fromString("myplugin:my_arena"), capture);
```

For non-vanilla schematics (WorldEdit `.schem` / `.schematic`) use the WorldEdit API
directly — Paper's Structure system reads only the vanilla NBT format.

References: [Paper StructureManager docs](https://docs.papermc.io/paper/dev/structures/) (where available).

---

## 6.19 PARTICLES, SOUNDS, BLOCK EFFECTS

### 6.19.1 Particles

```java
world.spawnParticle(Particle.FLAME, loc, /* count */ 20,
    /* offsetX */ 0.5, /* offsetY */ 0.5, /* offsetZ */ 0.5,
    /* extra */ 0.01,
    /* data */ null);

// Visible only to one player:
player.spawnParticle(Particle.NOTE, loc, 1, 0, 0, 0, 1.0);

// Dust with custom color:
world.spawnParticle(Particle.DUST, loc, 1, 0, 0, 0, 0,
    new Particle.DustOptions(Color.fromRGB(255, 0, 0), 1.0f));
```

Particles are client-side; spawning them async is unsafe (entity tracker). Always main
or region scheduler.

### 6.19.2 Sounds

```java
world.playSound(loc, Sound.BLOCK_ANVIL_LAND, 1.0f, 1.0f);
world.playSound(loc, "minecraft:block.note_block.harp", SoundCategory.BLOCKS, 1.0f, 1.5f);

// Play to a specific player only:
player.playSound(loc, Sound.UI_TOAST_CHALLENGE_COMPLETE, 0.7f, 1.0f);
```

`SoundCategory` lets the user mute via slider settings — use `BLOCKS`, `MUSIC`,
`HOSTILE`, etc. as appropriate.

### 6.19.3 Block break / damage animation

```java
// Show fake break stage (0..9) to one player:
player.sendBlockDamage(loc, /* progress 0.0..1.0 */ 0.5f);

// Send a fake block change without altering server state:
player.sendBlockChange(loc, Material.DIAMOND_BLOCK.createBlockData());
```

Useful for previews and ghost outlines.

---

## 6.X SELF-REVIEW CHECKLIST

- [x] Mental model — four block abstractions + decision tree
- [x] Block (lightweight handle) and `applyPhysics` flag
- [x] BlockData creation (Material#createBlockData, string, clone) + matches/equals
- [x] All common BlockData specialized interfaces tabulated
- [x] BlockState read-mutate-update pattern + when not to cache
- [x] Container family (Chest, Furnace) example
- [x] TileState PDC persistence rules
- [x] TileState subclass list
- [x] Sign API both faces (1.20+)
- [x] Chunk loaded vs generated
- [x] Plugin chunk tickets + auto-clear semantics
- [x] Async chunk loading (`getChunkAtAsync`) + Moonrise reference
- [x] ChunkSnapshot for thread-safe reads
- [x] Iterating loaded / force-loaded chunks
- [x] World height (1.18+ minY=-64, dynamic via `getMinHeight`)
- [x] Biome registry (post-1.20 deobfuscation, BiomeKeys)
- [x] RegionAccessor as unified interface
- [x] WorldCreator + ChunkGenerator + BiomeProvider example
- [x] LimitedRegion in `generateDecorations`
- [x] plugin.yml `getDefaultWorldGenerator` for `bukkit.yml`-driven generators
- [x] Block/world event matrix incl. `BlockPhysicsEvent` warning
- [x] Folia RegionScheduler patterns + cross-region trap
- [x] Async-safety matrix
- [x] 16-row failure cookbook
- [x] Performance notes
- [x] Reference implementation (chunk claim plugin) tying everything together
- [x] Location/Vector/BoundingBox utility coverage + raycasting
- [x] World state — time, weather, gamerules, world border
- [x] StructureManager + place/capture API
- [x] Particles, sounds, fake block changes

---

## 6.Z REFERENCES

- [Paper Block (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/block/Block.html)
- [Paper BlockData (1.13)](https://jd.papermc.io/paper/1.13/org/bukkit/block/data/BlockData.html)
- [Paper Waterlogged](https://jd.papermc.io/paper/1.21.10/org/bukkit/block/data/Waterlogged.html)
- [Paper Light block](https://jd.papermc.io/paper/1.21.3/org/bukkit/block/data/type/Light.html)
- [Paper BlockState (1.20.3)](https://jd.papermc.io/paper/1.20.3/org/bukkit/block/BlockState.html)
- [Paper TileState (1.21.1)](https://jd.papermc.io/paper/1.21.1/org/bukkit/block/TileState.html)
- [Paper World (1.21.8)](https://jd.papermc.io/paper/1.21.8/org/bukkit/World.html)
- [Paper Chunk (1.17.1)](https://jd.papermc.io/paper/1.17.1/org/bukkit/Chunk.html)
- [Paper ChunkSnapshot (1.21.11)](https://jd.papermc.io/paper/1.21.11/org/bukkit/ChunkSnapshot.html)
- [Paper RegionAccessor (1.21)](https://jd.papermc.io/paper/1.21.0/org/bukkit/RegionAccessor.html)
- [Paper ChunkGenerator (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/generator/ChunkGenerator.html)
- [Paper LimitedRegion (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/generator/LimitedRegion.html)
- [Paper BiomeProvider (1.21.11)](https://jd.papermc.io/paper/1.21.11/org/bukkit/generator/BiomeProvider.html)
- [Paper WorldCreator (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/WorldCreator.html)
- [Paper Biome (deprecated note)](https://jd.papermc.io/paper/1.21.11/org/bukkit/block/Biome.html)
- [Paper BiomeKeys](https://jd.papermc.io/folia/io/papermc/paper/registry/keys/BiomeKeys.html)
- [Paper Registry API docs](https://papermc-paper.mintlify.app/api/registry)
- [Paper world configuration](https://docs.papermc.io/paper/reference/world-configuration)
- [Paper chunk-loading optimization (Moonrise)](https://papermc-paper.mintlify.app/optimization/chunk-loading)
- [Folia RegionScheduler Javadoc](https://jd.papermc.io/folia/io/papermc/paper/threadedregions/scheduler/RegionScheduler.html)
- [Folia World.ChunkLoadCallback](https://jd.papermc.io/folia/1.21/org/bukkit/World.ChunkLoadCallback.html)
- [Paper issue #1842 — setBlockData with tile entity](https://github.com/PaperMC/Paper/issues/1842)
- [Paper issue #2705 — waterlogged chests block-entity](https://github.com/PaperMC/Paper/issues/2705)
- [PaperLib (legacy compat shim)](https://github.com/PaperMC/PaperLib)

Compliance note: information from external sources was paraphrased to comply with content
licensing.

---

## See also

- `01-build-tooling.md` §1.B — paperweight-userdev for NMS chunk access.
- `02-velocity-folia-bedrock.md` §2 — Folia regional threading model.
- `03-events-extras.md` — generic event-listener patterns.
- `05-entities-mobs.md` §5.6 — PDC on entities (block PDC works the same way).
- `paper-plugin-dev.md` §9–§13 — original short world/block reference.
