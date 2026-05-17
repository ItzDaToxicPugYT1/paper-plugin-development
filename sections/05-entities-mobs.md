---
name: paper-plugin-dev-entities-mobs
description: >
  Expansion to paper-plugin-dev.md covering Paper entity & mob development end-to-end:
  the Entity / LivingEntity / Mob / Animals / Monster / Display hierarchy, spawning
  (World#spawn with pre-spawn function, EntitySpawnEvent, PreCreatureSpawnEvent,
  PreSpawnerSpawnEvent), Attribute API (post-1.21 deobfuscation, AttributeModifier
  deprecation, NamespacedKey-based modifiers), Mob Goal API + Pathfinder API for
  custom AI, Display entities (Block, Item, Text) with transformation interpolation,
  PersistentDataContainer on entities, the new 1.20+/1.21+ entity surface (Allay,
  Camel, Sniffer, Armadillo, Bogged, Creaking), removal events (EntityRemoveEvent +
  cause), Folia EntityScheduler, async-safety contract, and a comprehensive entity
  failure cookbook with reference implementation.
---

# 5. ENTITIES & MOBS — DEEP DIVE

This document is a comprehensive companion to §6–§8 of `paper-plugin-dev.md`. The base
file lists `EntityType` values and shows a basic `World#spawn` call. This file expands
into the full live-entity surface available in modern Paper: spawning, AI, attributes,
display entities, persistence, removal, and Folia thread-affinity.

All snippets target **Paper 1.21.4+** (Mojang-mapped runtime). Where 1.20.5 / 1.21.x add
new types or rename surfaces they are flagged inline. Authoritative sources are linked
inline plus a [References](#5z-references) appendix.

---

## 5.0 MENTAL MODEL — THE ENTITY HIERARCHY

```
Entity                                         (base — has UUID, location, vehicle, PDC)
├── Display                                    (BlockDisplay, ItemDisplay, TextDisplay)
├── Item                                       (dropped items)
├── ExperienceOrb
├── Projectile                                 (Arrow, Snowball, Egg, ...)
├── Hanging                                    (ItemFrame, Painting)
├── Vehicle                                    (Boat, Minecart, ...)
├── AreaEffectCloud
├── Marker                                     (no-op marker entity)
├── Interaction                                (1.19.4+ click target with no model)
├── ArmorStand
├── EnderCrystal
├── EvokerFangs
├── FallingBlock
├── Firework
├── ...
└── LivingEntity                               (has health, attributes, equipment)
    ├── Player                                 (special — see §5.X)
    ├── ArmorStand (yes, both branches)
    └── Mob                                    (has AI, target, pathfinding)
        ├── Ambient (Bat)
        ├── Animals
        │    ├── Tameable                      (Wolf, Cat, Parrot, Horse, ...)
        │    ├── Breedable                     (Cow, Pig, ...)
        │    └── WaterMob                      (Fish, Squid, Dolphin, ...)
        ├── Monster
        │    ├── Skeleton / AbstractSkeleton (→ Bogged 1.21)
        │    ├── Zombie / ZombieVillager / Husk / Drowned
        │    ├── Creaking (1.21.4+)
        │    └── ...
        ├── NPC                                 (Villager, WanderingTrader)
        ├── Slime / MagmaCube
        └── Allay (1.19+) / Camel (1.20+) / Sniffer (1.20+) / Armadillo (1.21+)
```

When you receive an entity from an event or `World#getEntities()`, **always** narrow
to the most specific interface you need before calling type-specific methods. Use
`instanceof` patterns:

```java
if (entity instanceof Tameable tame && tame.isTamed()) { ... }
if (entity instanceof Mob mob)                          { mob.setTarget(player); }
if (entity instanceof Damageable d)                     { d.damage(2.0, src); }
if (entity instanceof Display display)                  { display.setTransformation(...); }
```

References: [Paper Javadoc — Entity](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/Entity.html),
[LivingEntity](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/LivingEntity.html),
[Mob (uses-of)](https://jd.papermc.io/paper/1.21.1/org/bukkit/entity/class-use/Mob.html).

---

## 5.1 SPAWNING — THE FIVE WAYS

### 5.1.1 `World#spawn(Location, Class)` — the modern default

```java
Zombie zombie = world.spawn(loc, Zombie.class);
```

Paper enriched this with a **pre-spawn function** that runs *before* `EntitySpawnEvent`,
so listeners see the entity in the configured state. Use it for everything you'd
previously have done in a sync follow-up:

```java
Zombie zombie = world.spawn(loc, Zombie.class, z -> {
    z.setBaby();
    z.setCanPickupItems(false);
    z.setRemoveWhenFarAway(false);
    z.customName(Component.text("Tiny Tim", NamedTextColor.RED));
    z.setCustomNameVisible(true);
    z.getEquipment().setHelmet(new ItemStack(Material.GOLDEN_HELMET));
    z.getPersistentDataContainer().set(myKey, PersistentDataType.STRING, "spawned-by-myplugin");
});
```

This is the right place for everything. Don't do `world.spawn(...)` then mutate, because:

1. Other plugins' `EntitySpawnEvent` listeners run *between* spawn and your mutation.
2. On Folia, the entity may be ticked before your follow-up runs.

### 5.1.2 `World#spawnEntity(Location, EntityType)` — type-erased

```java
Entity zombie = world.spawnEntity(loc, EntityType.ZOMBIE);
```

Returns `Entity`; you must cast. Useful when the type comes from config, but you lose
compile-time safety. Prefer `spawn(loc, Class)` when possible.

### 5.1.3 `World#spawn(Location, Class, SpawnReason, Consumer)` — Paper extension

```java
Skeleton skel = world.spawn(loc, Skeleton.class,
    CreatureSpawnEvent.SpawnReason.CUSTOM,
    s -> s.setShouldBurnInDay(false));
```

Lets you pass a `SpawnReason` so listeners can distinguish your spawn from natural ones.
**Always pass `SpawnReason.CUSTOM`** for plugin-spawned mobs — natural mob caps and
spawn rules ignore CUSTOM by design. References:
[CreatureSpawnEvent.SpawnReason](https://jd.papermc.io/paper/1.21.8/org/bukkit/event/entity/CreatureSpawnEvent.SpawnReason.html).

### 5.1.4 Spawning with NBT — `EntityType.tryCast` + NMS

For some edge cases (a pre-built entity from a structure NBT) you'll need NMS access via
paperweight-userdev. Generally avoid this; prefer the API.

### 5.1.5 Display / Marker / Interaction entities

Same `World#spawn(loc, Class)` pattern, but the consumer typically configures
transformation / brightness / view range:

```java
TextDisplay td = world.spawn(loc, TextDisplay.class, e -> {
    e.text(Component.text("WARP", NamedTextColor.GOLD));
    e.setBillboard(Display.Billboard.CENTER);
    e.setShadowed(false);
    e.setSeeThrough(false);
    e.setBackgroundColor(Color.fromARGB(0xC0_000000));
    e.setViewRange(64f);
});
```

### 5.1.6 The spawn-event chain

When you call `World#spawn(...)`, the following events fire in order (when applicable):

1. `PreCreatureSpawnEvent` — Paper-only, fires before the entity is constructed. Cancelable
   with no allocation cost. Use this to block spawns from spawners / natural sources.
2. `PreSpawnerSpawnEvent` — only for spawner blocks; a more specific override of #1.
3. **Pre-spawn `Consumer`** runs (your code from `spawn(loc, Class, Consumer)`).
4. `EntitySpawnEvent` (or `CreatureSpawnEvent` subclass for mobs).
5. `EntityAddToWorldEvent` (Paper) — entity is now in the world.

Cancel `EntitySpawnEvent` to abort *after* the entity is built (wasteful). Cancel
`PreCreatureSpawnEvent` to abort *before* (cheap). References:
[PreCreatureSpawnEvent](https://jd.papermc.io/paper/1.16/com/destroystokyo/paper/event/entity/PreCreatureSpawnEvent.html),
[EntitySpawnEvent](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/entity/EntitySpawnEvent.html).

---

## 5.2 ATTRIBUTES — THE 1.21 RENAME

In 1.21 the Bukkit `Attribute` enum was renamed to drop the `GENERIC_` / `HORSE_` /
`ZOMBIE_` prefixes and switch to `NamespacedKey`-based identity. Old plugins compiled
against `Attribute.GENERIC_MAX_HEALTH` will fail to link on 1.21+.

```java
// Pre-1.21 (BROKEN on 1.21+):
double max = entity.getAttribute(Attribute.GENERIC_MAX_HEALTH).getValue();

// 1.21+ (correct):
double max = entity.getAttribute(Attribute.MAX_HEALTH).getValue();
```

The full rename map is in [Attribute Javadoc](https://jd.papermc.io/paper/1.21.1/org/bukkit/attribute/Attribute.html).
Common ones:

| Pre-1.21 | 1.21+ |
|---|---|
| `GENERIC_MAX_HEALTH` | `MAX_HEALTH` |
| `GENERIC_MOVEMENT_SPEED` | `MOVEMENT_SPEED` |
| `GENERIC_ATTACK_DAMAGE` | `ATTACK_DAMAGE` |
| `GENERIC_KNOCKBACK_RESISTANCE` | `KNOCKBACK_RESISTANCE` |
| `GENERIC_FOLLOW_RANGE` | `FOLLOW_RANGE` |
| `GENERIC_ARMOR` | `ARMOR` |
| `HORSE_JUMP_STRENGTH` | `JUMP_STRENGTH` |
| `ZOMBIE_SPAWN_REINFORCEMENTS` | `SPAWN_REINFORCEMENTS` |

> **paperweight-userdev hazard:** if you use `Bootstrap` to access plugin code very early,
> the old `Attribute.GENERIC_*` symbols are sometimes mapped strangely depending on the
> dev-bundle version. If you see "GENERIC_MAX_HEALTH not a member of Attribute", upgrade
> paperweight-userdev to ≥ 2.0-beta.19. See
> [Paper forum thread](https://forums.papermc.io/threads/generic_max_health-disappear-after-using-bootstrapper.1513/).

### 5.2.1 AttributeModifier — UUID → NamespacedKey

In 1.21 `AttributeModifier` constructors that take `UUID` are deprecated for removal.
Use the `NamespacedKey` overload.

```java
// Old (deprecated):
new AttributeModifier(UUID.randomUUID(), "myplugin.bonus", 5.0,
    AttributeModifier.Operation.ADD_NUMBER);

// New (1.21+):
NamespacedKey key = new NamespacedKey(plugin, "speed-boost");
AttributeModifier mod = new AttributeModifier(
    key, 0.05, AttributeModifier.Operation.ADD_NUMBER, EquipmentSlotGroup.ANY);

entity.getAttribute(Attribute.MOVEMENT_SPEED).addModifier(mod);
```

`EquipmentSlotGroup` (1.20.6+) replaces the older `EquipmentSlot`. See
[AttributeModifier Javadoc](https://jd.papermc.io/paper/1.21.7/org/bukkit/attribute/AttributeModifier.html).

### 5.2.2 Operations

| Operation | Effect | Stacking order |
|---|---|---|
| `ADD_NUMBER` | base + sum(adds) | First |
| `ADD_SCALAR` | result *= (1 + sum(scalars)) | Second |
| `MULTIPLY_SCALAR_1` | result *= (1 + scalar), per modifier | Third |

Example: max-health 20, +10 ADD_NUMBER, +0.5 ADD_SCALAR, +0.1 MULTIPLY_SCALAR_1 →
((20 + 10) * (1 + 0.5)) * (1 + 0.1) = **49.5**.

### 5.2.3 Persistence of modifiers

Modifiers added programmatically **persist with the entity** unless removed. Restart-safe.
Modifiers added to player items via `ItemMeta#addAttributeModifier` persist with the item.
Modifiers added directly to a `Player`'s attribute container do **not** persist across
sessions — re-apply on `PlayerJoinEvent`.

---

## 5.3 MOB GOAL API — THE BUKKIT-NATIVE AI HOOK

Paper's `MobGoals` (under `com.destroystokyo.paper.entity.ai`) lets you add/remove AI
goals **without touching NMS**. This is the supported way to customize mob brains.

```java
import com.destroystokyo.paper.entity.ai.MobGoals;
import com.destroystokyo.paper.entity.ai.GoalKey;
import com.destroystokyo.paper.entity.ai.GoalType;
import com.destroystokyo.paper.entity.ai.Goal;

MobGoals goals = Bukkit.getMobGoals();

goals.addGoal(zombie, 0, new Goal<Zombie>() {
    @Override public boolean shouldActivate()                  { return zombie.getHealth() < 10; }
    @Override public boolean shouldStayActive()                { return zombie.getHealth() < 15; }
    @Override public void start()                              { zombie.getWorld().playSound(...); }
    @Override public void stop()                               { /* cleanup */ }
    @Override public void tick()                               { /* per-tick logic */ }
    @Override public GoalKey<Zombie> getKey() {
        return GoalKey.of(Zombie.class, new NamespacedKey(plugin, "panic-when-low-health"));
    }
    @Override public EnumSet<GoalType> getTypes()              { return EnumSet.of(GoalType.MOVE); }
});
```

### 5.3.1 Goal types

`GoalType` is a flag set; goals at the same priority **conflict** if their types overlap:

| GoalType | Meaning |
|---|---|
| `MOVE` | Goal moves the mob |
| `LOOK` | Goal turns the mob's head |
| `JUMP` | Goal makes the mob jump |
| `TARGET` | Goal selects a combat target |
| `UNKNOWN_BEHAVIOR` | Catch-all (used by some vanilla goals) |

Two `MOVE` goals at the same priority will fight. Lower priority wins (yes, "lower" =
"more important", inherited from vanilla).

### 5.3.2 Removing vanilla goals

Use `Bukkit.getMobGoals().removeAllGoals(mob)` to wipe everything, or
`removeGoal(mob, GoalKey)` to remove a specific one. Vanilla goal keys are exposed via
`VanillaGoal`:

```java
goals.removeGoal(zombie, VanillaGoal.MELEE_ATTACK);
goals.removeGoal(zombie, VanillaGoal.RANDOM_LOOK_AROUND);
```

References: [Paper Mob Goal API docs](https://docs.papermc.io/paper/dev/mob-goals/),
[MobGoals Javadoc](https://jd.papermc.io/paper/1.21/com/destroystokyo/paper/entity/ai/MobGoals.html).

---

## 5.4 PATHFINDER API — DIRECT MOVEMENT CONTROL

When you don't need a full goal but just want "go here", use `Mob#getPathfinder()`:

```java
Pathfinder pf = zombie.getPathfinder();

pf.moveTo(targetLocation, 1.0);            // speed multiplier
pf.moveTo(targetEntity, 1.2);

if (pf.hasPath()) {
    Pathfinder.PathResult result = pf.getCurrentPath();
    List<Location> nodes = result.getPoints();
    int next = result.getNextPointIndex();
}

pf.stopPathfinding();

pf.setCanOpenDoors(true);
pf.setCanPassDoors(true);
pf.setCanFloat(true);                      // swims in water instead of sinking
```

### 5.4.1 Finding a path without committing

```java
Pathfinder.PathResult result = pf.findPath(targetLocation);
if (result == null) {
    // No path exists — handle gracefully
} else if (result.getFinalPoint() != null
        && result.getFinalPoint().distanceSquared(targetLocation) < 1.0) {
    pf.moveTo(result, 1.0);
}
```

References: [Paper Entity Pathfinder API docs](https://docs.papermc.io/paper/dev/entity-pathfinder/).

### 5.4.2 When pathfinding API is not enough

If you need finer control (custom navigators, swimming behavior overrides, fly nav), drop
to NMS via paperweight-userdev. The classes you want:

- `net.minecraft.world.entity.ai.navigation.PathNavigation`
- `net.minecraft.world.level.pathfinder.WalkNodeEvaluator`

Wrap NMS in your plugin so the rest of your code stays API-clean.

---

## 5.5 DISPLAY ENTITIES (1.19.4+) — THE NEW HOLOGRAM SUBSTRATE

Display entities replaced ArmorStand-based holograms. They have no AI, no hitbox, no
collision, and are pure visual. Three flavors: `TextDisplay`, `ItemDisplay`, `BlockDisplay`.

### 5.5.1 Common API on `Display`

```java
display.setBillboard(Display.Billboard.CENTER);          // always face camera
display.setBrightness(new Display.Brightness(15, 15));   // override block / sky light
display.setViewRange(48f);                               // distance in blocks the client renders it
display.setShadowRadius(0f);                             // entity shadow disc
display.setShadowStrength(0f);
display.setTransformation(new Transformation(
    new Vector3f(0, 0, 0),                                // translation
    new AxisAngle4f(0, 0, 0, 0),                          // left rotation (joml)
    new Vector3f(1, 1, 1),                                // scale
    new AxisAngle4f(0, 0, 0, 0)                           // right rotation
));
display.setInterpolationDuration(20);                    // 20-tick smooth transition
display.setInterpolationDelay(0);                        // start immediately
display.setTeleportDuration(5);                          // smooth teleport over 5 ticks (1.20.2+)
```

### 5.5.2 Animating a `Display`

To animate a transformation, set the *next* state and the interpolation timing **in the
same tick**:

```java
display.setInterpolationDelay(0);
display.setInterpolationDuration(40);
display.setTransformation(newTransformation);
// Client smooths the change over 40 ticks.
```

If you skip `setInterpolationDelay(0)`, the client uses whatever delay is currently set,
which may be in the middle of a previous animation. Always reset.

References: [Paper display-entities docs](https://docs.papermc.io/paper/dev/display-entities/),
[Display Javadoc](https://jd.papermc.io/paper/1.21.10/org/bukkit/entity/Display.html),
[Minecraft Wiki — Display](https://minecraft.wiki/w/Display).

### 5.5.3 `TextDisplay` specifics

```java
TextDisplay td = world.spawn(loc, TextDisplay.class);
td.text(MiniMessage.miniMessage().deserialize("<gradient:gold:red>Boss Spawn!</gradient>"));
td.setBillboard(Display.Billboard.CENTER);
td.setSeeThrough(true);
td.setBackgroundColor(Color.fromARGB(0));      // fully transparent background
td.setLineWidth(200);                          // pixels before wrap
td.setAlignment(TextDisplay.TextAlignment.CENTER);
td.setShadowed(true);
td.setTextOpacity((byte) 255);
```

### 5.5.4 `ItemDisplay` specifics

```java
ItemDisplay id = world.spawn(loc, ItemDisplay.class);
id.setItemStack(new ItemStack(Material.NETHERITE_SWORD));
id.setItemDisplayTransform(ItemDisplay.ItemDisplayTransform.GROUND);
```

`ItemDisplayTransform` mirrors the `display` JSON in resource packs: `THIRDPERSON_LEFTHAND`,
`HEAD`, `GUI`, `GROUND`, `FIXED`, `FIRSTPERSON_RIGHTHAND`, etc. Choose `FIXED` for "show
the item exactly as the model defines it".

### 5.5.5 `BlockDisplay` specifics

```java
BlockDisplay bd = world.spawn(loc, BlockDisplay.class);
bd.setBlock(Material.OAK_PLANKS.createBlockData());
// Or with state:
bd.setBlock(Bukkit.createBlockData("minecraft:oak_stairs[facing=north,half=top]"));
```

> **No glow color on BlockDisplay** — block displays don't render their own outline.
> Use `entity.setGlowing(true)` + scoreboard team color via Adventure if you need a tint.

### 5.5.6 `Interaction` entity (clickable hitbox without a model)

```java
Interaction inter = world.spawn(loc, Interaction.class);
inter.setInteractionHeight(2.0f);
inter.setInteractionWidth(1.0f);
inter.setResponsive(true);                   // arm-swing animation on click
```

Listen for `PlayerInteractEntityEvent` to detect clicks. Pair with a `Display` to make a
"clickable hologram" — the display shows visuals, the Interaction handles input.

---

## 5.6 PERSISTENT DATA CONTAINER (PDC) ON ENTITIES

Every entity (and every block-entity, every item) is a `PersistentDataHolder` with a
`PersistentDataContainer`.

```java
NamespacedKey OWNER_KEY = new NamespacedKey(plugin, "owner");
NamespacedKey LEVEL_KEY = new NamespacedKey(plugin, "level");

PersistentDataContainer pdc = entity.getPersistentDataContainer();
pdc.set(OWNER_KEY, PersistentDataType.STRING, player.getUniqueId().toString());
pdc.set(LEVEL_KEY, PersistentDataType.INTEGER, 5);

if (pdc.has(OWNER_KEY, PersistentDataType.STRING)) {
    UUID owner = UUID.fromString(pdc.get(OWNER_KEY, PersistentDataType.STRING));
}
```

### 5.6.1 Built-in `PersistentDataType` values

`STRING`, `BYTE`, `SHORT`, `INTEGER`, `LONG`, `FLOAT`, `DOUBLE`, `BYTE_ARRAY`,
`INTEGER_ARRAY`, `LONG_ARRAY`, `TAG_CONTAINER`, `LIST` (1.20.6+, with sub-types like
`ListPersistentDataType.STRING`), `BOOLEAN` (1.21+).

### 5.6.2 Custom types via `PersistentDataType`

Implement the interface for any type that has a stable byte representation:

```java
public final class UUIDDataType implements PersistentDataType<byte[], UUID> {
    @Override public Class<byte[]> getPrimitiveType() { return byte[].class; }
    @Override public Class<UUID> getComplexType()     { return UUID.class; }

    @Override public byte[] toPrimitive(UUID complex, PersistentDataAdapterContext ctx) {
        ByteBuffer bb = ByteBuffer.allocate(16);
        bb.putLong(complex.getMostSignificantBits());
        bb.putLong(complex.getLeastSignificantBits());
        return bb.array();
    }
    @Override public UUID fromPrimitive(byte[] primitive, PersistentDataAdapterContext ctx) {
        ByteBuffer bb = ByteBuffer.wrap(primitive);
        return new UUID(bb.getLong(), bb.getLong());
    }
}
```

Now `pdc.set(OWNER_KEY, new UUIDDataType(), uuid)` works.

### 5.6.3 Persistence rules

- **Entities:** PDC saves whenever the entity saves (chunk unload, server stop). Survives
  restart. **Does not** survive `entity.remove()` — by design.
- **Players:** PDC saves on `playerdata/<uuid>.dat`. Survives logout/login.
- **Block entities:** PDC saves with the chunk. Survives restart.
- **Items:** PDC is part of `ItemMeta`, saves whenever the item is serialized.

Reference: [Paper PDC docs](https://docs.papermc.io/paper/dev/pdc).

---

## 5.7 ENTITY EVENTS WORTH KNOWING

| Event | Fires when | Async-safe? |
|---|---|---|
| `EntitySpawnEvent` | Any entity spawns (super of `CreatureSpawnEvent`, `ItemSpawnEvent`, etc.) | No |
| `CreatureSpawnEvent` | Any LivingEntity spawns; carries `SpawnReason` | No |
| `PreCreatureSpawnEvent` (Paper) | Before mob is constructed; cheap cancel | No |
| `PreSpawnerSpawnEvent` (Paper) | Specifically for spawner spawns | No |
| `EntityAddToWorldEvent` (Paper) | Entity actually added (after chunk load) | No |
| `EntityRemoveEvent` | Entity removed; carries `Cause` (enum incl. DEATH, UNLOAD, DISCARD, ...) | No |
| `EntityRemoveFromWorldEvent` (Paper) | Entity removed from a world (covers chunk unload too) | No |
| `EntityDeathEvent` | Living entity died; modify drops via `getDrops()` | No |
| `EntityResurrectEvent` | Totem of Undying triggered | No |
| `EntityTargetEvent` / `EntityTargetLivingEntityEvent` | Mob picked a target | No |
| `EntityPathfindEvent` (Paper) | Pathfinding initiated | No |
| `EntityMoveEvent` (Paper) | Non-player entity moved (does NOT fire for Player) | No |
| `EntityTeleportEvent` | Entity teleported | No |
| `EntityChangeBlockEvent` | Entity altered a block (Enderman, Silverfish, ...) | No |
| `EntityToggleSwimEvent` | Mob entered/left swim state | No |

### 5.7.1 The `EntityRemoveEvent` cause matrix

`EntityRemoveEvent.Cause` (1.20.5+) values include:

| Cause | Meaning | Should you persist data? |
|---|---|---|
| `DEATH` | Health hit 0 | No — entity gone forever |
| `KILL` | `entity.remove()` call | No |
| `DISCARD` | `kill` command or vanilla cleanup | No |
| `DESPAWN` | Persistence rules removed it | Yes (it may respawn) |
| `UNLOAD` | Chunk unloaded | Yes (PDC handles this) |
| `OUT_OF_WORLD` | Fell into the void | No |
| `EXPLODE` | TNT or creeper destroyed it (e.g. ItemFrame) | No |
| `ENTER_BLOCK` | Silverfish entered stone | No |
| `DROP` | Block / dust dropped its display | No |
| `PLAYER_QUIT` | Player left | Yes (player rejoin restores) |
| `TRANSFORMATION` | Mob transformed (Zombie → Drowned) | Yes — copy PDC to new entity |

Reference: [EntityRemoveEvent.Cause Javadoc](https://jd.papermc.io/paper/1.21.0/org/bukkit/event/entity/EntityRemoveEvent.Cause.html).

### 5.7.2 Don't remove entities during `ChunkUnloadEvent`

This is a known issue; entity removal during unload can leave dangling references and
crash servers. See [Paper issue #8448](https://github.com/PaperMC/Paper/issues/8448). If
you must conditionally remove, use `EntityRemoveEvent` with cause `UNLOAD` and call
`entity.remove()` on the *previous* tick instead.

---

## 5.8 NEW 1.20+/1.21+ MOBS — API COVERAGE

| Mob | Added in | Specific API methods |
|---|---|---|
| `Allay` | 1.19 | `getInventory()`, `getDuplicationCooldown()` |
| `Frog` / `Tadpole` | 1.19 | `Frog#setVariant(Frog.Variant)` |
| `Camel` | 1.20 | `getDashCooldown()`, `setSitting(boolean)` |
| `Sniffer` | 1.20 | `getState()`, `dig()`, `removeExploredLocation()` |
| `Bogged` | 1.21 | Skeleton subclass; shears to give Skeleton + Mushroom |
| `Armadillo` | 1.21 | `setState(Armadillo.State)` (IDLE / ROLLING / SCARED / UNROLLING) |
| `Creaking` | 1.21.4 | tied to `CreakingHeart` block; `setHomePos(Location)` |
| `Wind Charge` | 1.21 | Projectile with `setExplosionPower(...)` |

References: [EntityType Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/EntityType.html),
[Bogged](https://jd.papermc.io/paper/1.21.0/org/bukkit/entity/Bogged.html),
[Sniffer](https://jd.papermc.io/paper/1.19/org/bukkit/entity/Sniffer.html).

---

## 5.9 FOLIA THREAD-AFFINITY FOR ENTITIES

On Folia, *every entity* belongs to one **owning region** at a time. The thread that ticks
that region is the only thread allowed to touch that entity.

### 5.9.1 The `EntityScheduler`

```java
entity.getScheduler().run(plugin, task -> {
    entity.teleport(target);
}, /* retired */ () -> {
    plugin.getLogger().info("entity removed before task ran");
});
```

Behavior:
- "Follows" the entity. If the entity is teleported between regions / worlds, the task
  runs on whatever region the entity ends up in.
- The retired callback runs if the entity was removed before the task could execute.
- Returns immediately; never blocks.

### 5.9.2 Don't use `RegionScheduler` for entities

`RegionScheduler` schedules by *location*, not entity. An entity might move to a different
region after you scheduled, and your task will run on the *original* region's thread —
which **does not own** the entity anymore. That's a Folia crash.

```java
// WRONG on Folia:
plugin.getServer().getRegionScheduler().run(plugin, entity.getLocation(),
    task -> entity.teleport(target));

// CORRECT:
entity.getScheduler().run(plugin, task -> entity.teleport(target), null);
```

### 5.9.3 Cross-entity operations

To do something with two entities (e.g. attach one to another), schedule on *both*
schedulers and synchronize via a `CompletableFuture`:

```java
CompletableFuture<Boolean> ready1 = new CompletableFuture<>();
CompletableFuture<Boolean> ready2 = new CompletableFuture<>();

entity1.getScheduler().run(plugin, t -> { /* prep */ ready1.complete(true); }, null);
entity2.getScheduler().run(plugin, t -> { /* prep */ ready2.complete(true); }, null);

CompletableFuture.allOf(ready1, ready2).thenRun(() -> {
    entity1.getScheduler().run(plugin, t -> entity1.addPassenger(entity2), null);
});
```

References: [Folia EntityScheduler Javadoc](https://jd.papermc.io/folia/1.19/io/papermc/paper/threadedregions/scheduler/EntityScheduler.html),
[Folia RegionScheduler caveat](https://jd.papermc.io/folia/1.20/io/papermc/paper/threadedregions/scheduler/RegionScheduler.html),
[Paper Folia-support docs](https://docs.papermc.io/paper/dev/folia-support/).

---

## 5.10 ASYNC-SAFETY MATRIX

| Operation | Main thread | Async thread | Folia entity scheduler |
|---|---|---|---|
| `entity.getLocation()` | OK | UNSAFE | OK |
| `entity.teleport(loc)` | OK | UNSAFE | OK if loc owned by same region |
| `entity.remove()` | OK | UNSAFE | OK |
| `entity.getPersistentDataContainer().get(...)` | OK | OK (read) | OK |
| `entity.getPersistentDataContainer().set(...)` | OK | UNSAFE | OK |
| `world.getEntities()` | OK | UNSAFE | NOT REGIONAL — risky on Folia |
| `world.getNearbyEntities(loc, x, y, z)` | OK | UNSAFE | OK if loc owned |
| `Bukkit.getEntity(uuid)` | OK | UNSAFE | OK if owning region |
| `entity.getType()` | OK | OK | OK |
| `entity.getUniqueId()` | OK | OK | OK |

**Rule of thumb for Paper (non-Folia):** main-thread for all entity mutations. Async-thread
read access is *technically* unsafe but works for primitives like UUID/type. Wrap in
`Bukkit.getScheduler().runTask(...)` to be safe.

**Rule for Folia:** never assume the main thread; always go through the entity scheduler.

---

## 5.11 COMMON ENTITY-DEV FAILURES

| Symptom | Cause | Fix |
|---|---|---|
| `Attribute.GENERIC_MAX_HEALTH` does not exist | Using 1.21+ paper-api | Rename to `MAX_HEALTH` |
| Entity spawned but EntitySpawnEvent listener doesn't see configured state | Configured *after* `spawn()` | Use the `Consumer<T>` overload |
| `setHealth(25)` clamped to 20 | MAX_HEALTH attribute not raised | Set `Attribute.MAX_HEALTH` first |
| `Mob#getPathfinder().moveTo(...)` does nothing | Mob is in vehicle / has higher-priority goal | Remove conflicting goals |
| Custom Goal never activates | Returned same `GoalType` set as a higher-priority goal | Use unique types or lower priority number |
| `ArmorStand` invisible | `setMarker(true)` set | Toggle off, or use Display entity instead |
| `Display` doesn't appear past 16 blocks | Default view range too small | `setViewRange(64f)` |
| TextDisplay text breaks lines weirdly | `setLineWidth` defaulted to 200 | Increase to fit, or use `\n` in text |
| Entity not saving PDC | `entity.remove()` was called | PDC dies with entity by design |
| Entities I spawn count toward mob caps | Used wrong SpawnReason | Pass `SpawnReason.CUSTOM` |
| `entity.teleport(loc)` no-ops on Folia | Cross-region teleport from wrong thread | Use `entity.getScheduler()` |
| Entity goes invisible periodically | Client unloaded chunk | Increase `entity-tracking-range` in spigot.yml |
| Listener for `EntityDeathEvent` doesn't fire | Entity was *removed* (`Cause.KILL`) not *killed* | Listen for `EntityRemoveEvent` too |
| AttributeModifier doesn't stack as expected | Two ADD_NUMBER modifiers with same NamespacedKey | Use distinct keys |

---

## 5.12 PERFORMANCE NOTES

- **`world.getEntities()`** is O(n) over every entity in the world. For per-tick logic,
  cache a filtered subset and refresh it on `EntityAddToWorldEvent` /
  `EntityRemoveFromWorldEvent` instead.
- **`getNearbyEntities(...)`** with a bounding box ≥ 16 blocks scans many chunks. Prefer
  `world.getNearbyLivingEntities(loc, r, type)` (Paper) which is a faster typed variant.
- **Display-entity transformation updates** are full entity-data packets per change.
  Don't tween in Java — use `setInterpolationDuration(20)` and let the *client* tween.
- **PDC reads** are cheap. PDC writes serialize the whole map; batch sets when possible.
- **MobGoals** run every tick. Cheap goals (cached state) are fine; goals doing IO will
  tank server TPS.

---

## 5.13 REFERENCE IMPLEMENTATION

A "boss zombie" plugin showing every pattern above:

```java
package com.example.boss;

import com.destroystokyo.paper.entity.ai.Goal;
import com.destroystokyo.paper.entity.ai.GoalKey;
import com.destroystokyo.paper.entity.ai.GoalType;
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import org.bukkit.*;
import org.bukkit.attribute.*;
import org.bukkit.entity.*;
import org.bukkit.event.*;
import org.bukkit.event.entity.*;
import org.bukkit.inventory.*;
import org.bukkit.persistence.*;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.util.Transformation;
import org.joml.AxisAngle4f;
import org.joml.Vector3f;

import java.util.EnumSet;
import java.util.UUID;

public final class BossPlugin extends JavaPlugin implements Listener {

    private NamespacedKey bossKey;
    private NamespacedKey rageGoalKey;

    @Override
    public void onEnable() {
        bossKey      = new NamespacedKey(this, "is-boss");
        rageGoalKey  = new NamespacedKey(this, "boss-rage");

        Bukkit.getPluginManager().registerEvents(this, this);

        // /spawnboss command
        getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, e -> {
            e.registrar().register(
                Commands.literal("spawnboss")
                    .requires(s -> s.getSender() instanceof Player p && p.isOp())
                    .executes(ctx -> {
                        Player p = (Player) ctx.getSource().getSender();
                        spawnBoss(p.getLocation());
                        return 1;
                    }).build()
            );
        });
    }

    private void spawnBoss(Location loc) {
        Zombie boss = loc.getWorld().spawn(loc, Zombie.class,
            CreatureSpawnEvent.SpawnReason.CUSTOM,
            z -> {
                z.customName(Component.text("Lord of Decay", NamedTextColor.DARK_RED));
                z.setCustomNameVisible(true);
                z.setRemoveWhenFarAway(false);
                z.setCanPickupItems(false);

                // Stats via Attribute API
                AttributeInstance maxHp = z.getAttribute(Attribute.MAX_HEALTH);
                maxHp.addModifier(new AttributeModifier(
                    new NamespacedKey(this, "boss-hp"),
                    180.0, AttributeModifier.Operation.ADD_NUMBER, EquipmentSlotGroup.ANY));
                z.setHealth(maxHp.getValue());

                AttributeInstance speed = z.getAttribute(Attribute.MOVEMENT_SPEED);
                speed.addModifier(new AttributeModifier(
                    new NamespacedKey(this, "boss-speed"),
                    0.5, AttributeModifier.Operation.ADD_SCALAR, EquipmentSlotGroup.ANY));

                z.getEquipment().setHelmet(new ItemStack(Material.NETHERITE_HELMET));

                // PDC tag
                z.getPersistentDataContainer().set(bossKey, PersistentDataType.BYTE, (byte) 1);
            });

        // Custom AI goal: rage when below 50% HP
        Bukkit.getMobGoals().addGoal(boss, 0, new RageGoal(boss));

        // Floating health label via TextDisplay
        TextDisplay label = loc.getWorld().spawn(loc.clone().add(0, 2.4, 0), TextDisplay.class, td -> {
            td.text(Component.text("Lord of Decay", NamedTextColor.DARK_RED));
            td.setBillboard(Display.Billboard.CENTER);
            td.setSeeThrough(false);
            td.setBackgroundColor(Color.fromARGB(0xC0_220000));
            td.setViewRange(64f);
            td.setShadowed(false);
        });
        label.addPassenger(boss);                        // attach so it moves with boss
        // (or use entity scheduler / TickAttachment for non-passenger setups)
    }

    @EventHandler
    public void onDeath(EntityDeathEvent ev) {
        if (ev.getEntity().getPersistentDataContainer().has(bossKey, PersistentDataType.BYTE)) {
            ev.getDrops().clear();
            ev.getDrops().add(new ItemStack(Material.NETHERITE_INGOT, 4));
            ev.setDroppedExp(500);
            ev.getEntity().getWorld().sendMessage(
                Component.text("The boss has fallen!", NamedTextColor.GOLD));
        }
    }

    @EventHandler
    public void onRemove(EntityRemoveEvent ev) {
        // Clean up the floating label when boss disappears
        if (ev.getEntity().getPersistentDataContainer().has(bossKey, PersistentDataType.BYTE)) {
            ev.getEntity().getPassengers().stream()
                .filter(e -> e instanceof TextDisplay)
                .forEach(Entity::remove);
        }
    }

    private final class RageGoal implements Goal<Zombie> {
        private final Zombie z;
        private boolean raged;
        RageGoal(Zombie z) { this.z = z; }

        @Override public boolean shouldActivate()   { return !raged && z.getHealth() < 90; }
        @Override public boolean shouldStayActive() { return raged && !z.isDead(); }
        @Override public void start() {
            raged = true;
            z.getWorld().strikeLightningEffect(z.getLocation());
            z.setGlowing(true);
            // Add a temporary speed boost
            z.getAttribute(Attribute.MOVEMENT_SPEED).addModifier(new AttributeModifier(
                new NamespacedKey(BossPlugin.this, "rage-speed"),
                0.3, AttributeModifier.Operation.ADD_SCALAR, EquipmentSlotGroup.ANY));
        }
        @Override public void tick() { /* per-tick rage effect */ }
        @Override public void stop() { z.setGlowing(false); }
        @Override public GoalKey<Zombie> getKey() {
            return GoalKey.of(Zombie.class, rageGoalKey);
        }
        @Override public EnumSet<GoalType> getTypes() {
            return EnumSet.of(GoalType.UNKNOWN_BEHAVIOR);
        }
    }
}
```

`paper-plugin.yml`:

```yaml
name: BossPlugin
main: com.example.boss.BossPlugin
api-version: '1.21'
authors: ['You']
```

---

## 5.X SELF-REVIEW CHECKLIST

- [x] Full entity hierarchy diagram with `instanceof` narrowing pattern
- [x] All 5 spawning methods (with the pre-spawn `Consumer` heavily emphasized)
- [x] Spawn-event chain (PreCreatureSpawnEvent → consumer → EntitySpawnEvent → AddToWorld)
- [x] 1.21 `Attribute` rename table (full migration map)
- [x] `AttributeModifier` UUID-deprecation + `EquipmentSlotGroup`
- [x] AttributeModifier operations + math example
- [x] Persistence rules for modifiers (entity vs item vs player)
- [x] MobGoals API with custom Goal example
- [x] GoalType priority/conflict semantics
- [x] Removing vanilla goals via VanillaGoal
- [x] Pathfinder API (moveTo, hasPath, findPath)
- [x] When to drop to NMS (PathNavigation)
- [x] Display entity common API (transformation, brightness, viewRange)
- [x] Display interpolation pattern (delay=0 reset)
- [x] TextDisplay / ItemDisplay / BlockDisplay specifics
- [x] Interaction entity for clickable hitboxes
- [x] PDC entity persistence rules (saves with entity, dies with `remove()`)
- [x] Custom PersistentDataType example (UUID)
- [x] Entity event matrix
- [x] EntityRemoveEvent.Cause matrix with persistence guidance
- [x] Don't-remove-during-ChunkUnload caveat
- [x] New 1.20/1.21 mob coverage (Allay, Camel, Sniffer, Bogged, Armadillo, Creaking)
- [x] Folia EntityScheduler vs RegionScheduler pitfall
- [x] Cross-entity scheduling pattern with CompletableFuture
- [x] Async-safety matrix (Paper main vs async vs Folia)
- [x] 14-row failure cookbook
- [x] Performance notes (getEntities cost, display interpolation, PDC writes, MobGoals tick)
- [x] Complete reference implementation (boss zombie with all patterns)

---

## 5.Z REFERENCES

- [Paper docs — Mob Goal API](https://docs.papermc.io/paper/dev/mob-goals/)
- [Paper docs — Entity Pathfinder API](https://docs.papermc.io/paper/dev/entity-pathfinder/)
- [Paper docs — Display entities](https://docs.papermc.io/paper/dev/display-entities/)
- [Paper docs — Persistent Data Container (PDC)](https://docs.papermc.io/paper/dev/pdc)
- [Paper docs — Folia support](https://docs.papermc.io/paper/dev/folia-support/)
- [Paper Javadoc — Entity (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/Entity.html)
- [Paper Javadoc — LivingEntity (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/LivingEntity.html)
- [Paper Javadoc — Attribute (1.21.1)](https://jd.papermc.io/paper/1.21.1/org/bukkit/attribute/Attribute.html)
- [Paper Javadoc — AttributeModifier (1.21.7)](https://jd.papermc.io/paper/1.21.7/org/bukkit/attribute/AttributeModifier.html)
- [Paper Javadoc — MobGoals (1.21)](https://jd.papermc.io/paper/1.21/com/destroystokyo/paper/entity/ai/MobGoals.html)
- [Paper Javadoc — Display (1.21.10)](https://jd.papermc.io/paper/1.21.10/org/bukkit/entity/Display.html)
- [Paper Javadoc — TextDisplay (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/entity/TextDisplay.html)
- [Paper Javadoc — EntityRemoveEvent.Cause (1.21)](https://jd.papermc.io/paper/1.21.0/org/bukkit/event/entity/EntityRemoveEvent.Cause.html)
- [Paper Javadoc — PreCreatureSpawnEvent](https://jd.papermc.io/paper/1.16/com/destroystokyo/paper/event/entity/PreCreatureSpawnEvent.html)
- [Paper Javadoc — CreatureSpawnEvent.SpawnReason (1.21.8)](https://jd.papermc.io/paper/1.21.8/org/bukkit/event/entity/CreatureSpawnEvent.SpawnReason.html)
- [Folia Javadoc — EntityScheduler](https://jd.papermc.io/folia/1.19/io/papermc/paper/threadedregions/scheduler/EntityScheduler.html)
- [Folia Javadoc — RegionScheduler](https://jd.papermc.io/folia/1.20/io/papermc/paper/threadedregions/scheduler/RegionScheduler.html)
- [Minecraft Wiki — Display entities](https://minecraft.wiki/w/Display)
- [Paper forum — GENERIC_MAX_HEALTH bootstrapper issue](https://forums.papermc.io/threads/generic_max_health-disappear-after-using-bootstrapper.1513/)
- [Paper issue #8448 — entity removal during ChunkUnloadEvent](https://github.com/PaperMC/Paper/issues/8448)

Compliance note: information from external sources was paraphrased to comply with content
licensing.

---

## See also

- `01-build-tooling.md` §1.B — paperweight-userdev (needed if you drop to NMS for
  navigation overrides).
- `02-velocity-folia-bedrock.md` §2 — Folia regional threading model in depth.
- `03-events-extras.md` — generic event-listener patterns; entity-specific events covered
  here in §5.7.
- `04-commands-extras.md` §4.2.5 — `ArgumentTypes.entity()` / `entities()` for command
  selectors.
- `paper-plugin-dev.md` §6–§8 — original short entity / mob reference.
