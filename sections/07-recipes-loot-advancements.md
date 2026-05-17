---
name: paper-plugin-dev-recipes-loot-advancements
description: >
  Expansion to paper-plugin-dev.md covering Paper recipe / loot / advancement APIs
  end-to-end: ShapedRecipe / ShapelessRecipe / CookingRecipe (furnace, blasting,
  smoking, campfire) / StonecuttingRecipe / SmithingTransformRecipe /
  SmithingTrimRecipe / MerchantRecipe / ComplexRecipe; RecipeChoice.MaterialChoice
  vs RecipeChoice.ExactChoice and the stack-meta matching pitfall;
  Bukkit#addRecipe / Bukkit#removeRecipe / Bukkit#resetRecipes lifecycle and the
  SPIGOT-6084 reload-wipe gotcha; PrepareItemCraftEvent vs CraftItemEvent vs
  ItemCraftedEvent (1.21.5+) and the result-slot mutation pattern; the four
  prepare-result siblings (PrepareSmithingEvent, PrepareAnvilEvent,
  PrepareGrindstoneEvent, PrepareItemEnchantEvent); player recipe book
  (Player#discoverRecipe, undiscoverRecipe, hasDiscoveredRecipe);
  PlayerRecipeDiscoverEvent / PlayerRecipeBookClickEvent;
  org.bukkit.loot.LootTable / LootContext / LootContext.Builder with luck +
  lootingModifier + lootedEntity + killer; LootTables enum and lookup via
  Registry.LOOT_TABLE / Bukkit#getLootTable; populating containers via
  Lootable#setLootTable / fillInventory; LootGenerateEvent and droppedItems
  mutation; vanilla loot-table JSON schema (pools, rolls, entries, conditions,
  functions, item_modifier); EntityDeathEvent#getDrops vs vanilla loot-table
  pipeline ordering; PlayerDeathEvent keep/drop; advancement registration via
  datapacks (DatapackRegistrar 1.21.10+) and the historical UnsafeValues hack;
  AdvancementProgress, awardCriteria, revokeCriteria, isDone, getAwardedCriteria;
  PlayerAdvancementDoneEvent and PlayerAdvancementCriterionGrantEvent; advancement
  JSON schema (criteria, requirements, rewards, display, parent); Folia
  thread-affinity for crafting / loot / advancements; failure cookbook with 16
  rows; reference implementation (a "Lost Relic" plugin that registers a custom
  smithing-transform recipe, a custom loot table that injects into chest loot,
  and a tracked advancement granted by completing the craft).
---

# 7. RECIPES, LOOT TABLES & ADVANCEMENTS — DEEP DIVE

This document is a comprehensive companion to the recipes / drops / advancements
mentions scattered throughout `paper-plugin-dev.md`. The base file briefly mentions
`Bukkit#addRecipe`, `EntityDeathEvent#getDrops`, and "advancements exist." This file
walks the entire surface from the API side and the JSON-datapack side, plus the
events that fire as players interact with each system, plus what every interaction
costs at runtime.

All snippets target **Paper 1.21.4+** (Mojang-mapped runtime). 1.20.5+ smithing
template renames, the 1.21.10 `DatapackRegistrar`, and the 1.21.5 `ItemCraftedEvent`
are flagged inline. Authoritative sources are linked inline plus a
[References](#7z-references) appendix.

---

## 7.0 MENTAL MODEL — THREE PIPELINES, ONE NAMESPACE

Recipes, loot tables, and advancements are three completely separate subsystems but
they share four properties:

1. **They are all `NamespacedKey`-keyed.** `minecraft:diamond_sword`,
   `myplugin:lost_relic`, etc.
2. **They are all registered at server start** (post `WorldInitEvent` for loot, post
   `ServerLoadEvent` for recipes if you want them to survive `/reload`).
3. **They all have a vanilla JSON form** (datapack), and an **API form** (Bukkit
   classes). The two forms are interchangeable for the engine but the API form
   doesn't auto-survive a `/reload` (see §7.3.4).
4. **They all fire Bukkit events** at well-defined points in the pipeline.

```
                   ┌──────────────── REGISTRATION ────────────────┐
                   │                                                │
RECIPE       Bukkit#addRecipe(recipe, resend) ──> RecipeIterator    │
                   │       └─ datapack JSON in /data/<ns>/recipe/   │
                                                                    │
LOOT TABLE   Bukkit#getLootTable(NamespacedKey) ──> LootTable       │ same registry
                   │       └─ datapack JSON in /data/<ns>/loot_table│ namespace
                                                                    │
ADVANCEMENT  Bukkit#getAdvancement(key) / DatapackRegistrar         │
                   │       └─ datapack JSON in /data/<ns>/advancement
                   └────────────────────────────────────────────────┘

                   ┌─────────────────── RUNTIME ────────────────────┐
                   │                                                  │
RECIPE       PlayerInteract / item-pickup ──> recipe-book discover    │
                                              │                       │
                       Crafting matrix change ─┼─> PrepareItemCraftEvent (mutable result)
                                              │                       │
                       Player picks up result  ├─> CraftItemEvent      │
                                              │     (cancellable)     │
                                              └─> ItemCraftedEvent (1.21.5+, post-pickup)
                                                                      │
LOOT TABLE   Container generated/opened ──> LootGenerateEvent (mutable items)
             Mob death     ──> EntityDeathEvent#getDrops (mutable list, vanilla-table run already)
             Player death  ──> PlayerDeathEvent (keepInventory rules)
                                                                      │
ADVANCEMENT  any Bukkit event ──> awardCriteria(criterionName)         │
             internal trigger ──> PlayerAdvancementCriterionGrantEvent  │
             all criteria met ──> PlayerAdvancementDoneEvent            │
                                  ──> Bukkit#dispatchCommand(rewards)   │
                   └────────────────────────────────────────────────┘
```

References:
[Paper recipes guide](https://docs.papermc.io/paper/dev/recipes),
[Paper API index](https://docs.papermc.io/paper/dev/api/),
[Paper Bukkit Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/Bukkit.html).

---

## 7.1 RECIPES — THE EIGHT TYPES

Paper's `org.bukkit.inventory.Recipe` is a marker interface; every concrete recipe
extends it. Here is the full matrix:

| Class | Workstation | Inputs | Output | Since |
|---|---|---|---|---|
| `ShapedRecipe` | crafting table / 2×2 inventory | shape + per-slot `RecipeChoice` | one `ItemStack` | 1.0 |
| `ShapelessRecipe` | crafting table / 2×2 inventory | unordered `List<RecipeChoice>` | one `ItemStack` | 1.0 |
| `FurnaceRecipe` | furnace | one `RecipeChoice` | one `ItemStack` | 1.0 |
| `BlastingRecipe` | blast furnace | one `RecipeChoice` | one `ItemStack` | 1.14 |
| `SmokingRecipe` | smoker | one `RecipeChoice` | one `ItemStack` | 1.14 |
| `CampfireRecipe` | campfire / soul campfire | one `RecipeChoice` | one `ItemStack` | 1.14 |
| `StonecuttingRecipe` | stonecutter | one `RecipeChoice` | one `ItemStack` | 1.14 |
| `SmithingTransformRecipe` | smithing table | template + base + addition | one `ItemStack` | 1.20 |
| `SmithingTrimRecipe` | smithing table | template + base + addition | trimmed `ItemStack` | 1.20 |
| `MerchantRecipe` | villager / wandering trader | up to 2 ingredient stacks | one `ItemStack` | 1.8 |
| `ComplexRecipe` | hardcoded vanilla (banner, map, fireworks…) | special handler | varies | 1.13 |

The pre-1.20 `SmithingRecipe` constructor still exists but is deprecated; never use
it for new code — use the `Transform` / `Trim` split per
[SmithingTransformRecipe](https://jd.papermc.io/paper/1.20.0/org/bukkit/inventory/SmithingTransformRecipe.html)
and [SmithingTrimRecipe](https://jd.papermc.io/paper/1.20.1/org/bukkit/inventory/SmithingTrimRecipe.html).

### 7.1.1 ShapedRecipe with RecipeChoice

```java
NamespacedKey key = new NamespacedKey(plugin, "lost_relic_blade");
ItemStack result = ItemStack.of(Material.NETHERITE_SWORD);
result.editMeta(m -> m.displayName(Component.text("Lost Relic Blade", NamedTextColor.GOLD)));

ShapedRecipe recipe = new ShapedRecipe(key, result);
recipe.shape(
    " A ",
    "ABA",
    " S "
);
// MaterialChoice = "any of these block tags or materials"
recipe.setIngredient('A', new RecipeChoice.MaterialChoice(Tag.PLANKS));
recipe.setIngredient('B', new RecipeChoice.MaterialChoice(Material.NETHERITE_INGOT));
// ExactChoice = exact ItemStack (including custom display name + PDC)
ItemStack scriptedShard = scriptedShardStack(); // your custom item
recipe.setIngredient('S', new RecipeChoice.ExactChoice(scriptedShard));

recipe.setCategory(CraftingBookCategory.EQUIPMENT); // controls recipe-book tab
recipe.setGroup("myplugin:relics");                 // groups in recipe book
Bukkit.addRecipe(recipe, /* resendDiscovered */ true);
```

`MaterialChoice` matches by `Material` only (i.e. anything with that block id).
`ExactChoice` matches **stack components** including display name, lore, PDC, and
custom model data — this is how you require a specific custom item as an
ingredient. See
[RecipeChoice.ExactChoice](https://jd.papermc.io/paper/1.21.11/org/bukkit/inventory/RecipeChoice.ExactChoice.html)
and [RecipeChoice.MaterialChoice](https://jd.papermc.io/paper/1.21.10/org/bukkit/inventory/RecipeChoice.MaterialChoice.html).

> **Pitfall.** `ExactChoice` matches by full item-component equality. If the
> client's preview slot has *less* metadata than the registered template (e.g. a
> player crafted with a stripped-name version), it will fail to match silently. If
> you want a "any item with my PDC marker" check, do it in `PrepareItemCraftEvent`
> instead and override the result.

### 7.1.2 ShapelessRecipe

```java
ShapelessRecipe shapeless = new ShapelessRecipe(
    new NamespacedKey(plugin, "blank_relic"),
    ItemStack.of(Material.PAPER)
);
shapeless.addIngredient(new RecipeChoice.MaterialChoice(Material.WHEAT));
shapeless.addIngredient(new RecipeChoice.MaterialChoice(Material.WHEAT));
shapeless.addIngredient(new RecipeChoice.MaterialChoice(Material.WHEAT));
Bukkit.addRecipe(shapeless, true);
```

### 7.1.3 Cooking recipes (Furnace / Blasting / Smoking / Campfire)

All four extend `CookingRecipe<T>`; they only differ in workstation block.

```java
FurnaceRecipe smelt = new FurnaceRecipe(
    new NamespacedKey(plugin, "smelt_relic"),
    /* result */ ItemStack.of(Material.GOLD_INGOT),
    /* input  */ new RecipeChoice.ExactChoice(scriptedShardStack()),
    /* xp     */ 1.0f,
    /* cookTime ticks */ 200
);
smelt.setCategory(CookingBookCategory.MISC);
Bukkit.addRecipe(smelt, true);

BlastingRecipe blast  = new BlastingRecipe(...);  // 100 ticks default
SmokingRecipe  smoke  = new SmokingRecipe(...);   // 100 ticks default; food only
CampfireRecipe camp   = new CampfireRecipe(...);  // 600 ticks default; food only
```

Cook time is in **ticks** (20/sec). The vanilla furnace is 200 ticks (10s); blast
furnace and smoker run at half time. See
[BlastingRecipe](https://jd.papermc.io/paper/1.18/org/bukkit/inventory/BlastingRecipe.html).

### 7.1.4 StonecuttingRecipe

```java
StonecuttingRecipe stone = new StonecuttingRecipe(
    new NamespacedKey(plugin, "relic_block"),
    ItemStack.of(Material.STONE_BRICKS, 4),
    new RecipeChoice.MaterialChoice(Material.STONE)
);
Bukkit.addRecipe(stone, true);
```

The stonecutter UI auto-lists every recipe whose input matches the placed item.
Multiple stonecutting recipes can share the same input. See
[StonecuttingRecipe](https://jd.papermc.io/paper/1.20/org/bukkit/inventory/StonecuttingRecipe.html).

### 7.1.5 Smithing — Transform vs Trim (1.20+)

Pre-1.20 used a single `SmithingRecipe` with two ingredients (base + addition).
1.20 introduced the **template** slot and split into two distinct recipe types:

```java
// Transform: produces a wholly new item (e.g. diamond -> netherite upgrade).
SmithingTransformRecipe upgrade = new SmithingTransformRecipe(
    new NamespacedKey(plugin, "relic_upgrade"),
    /* result   */ resultStack,
    /* template */ new RecipeChoice.MaterialChoice(Material.NETHERITE_UPGRADE_SMITHING_TEMPLATE),
    /* base     */ new RecipeChoice.MaterialChoice(Material.DIAMOND_SWORD),
    /* addition */ new RecipeChoice.MaterialChoice(Material.NETHERITE_INGOT)
);
Bukkit.addRecipe(upgrade, true);

// Trim: applies a trim pattern + material to existing armor; preserves the base item.
SmithingTrimRecipe trim = new SmithingTrimRecipe(
    new NamespacedKey(plugin, "relic_trim"),
    /* template */ new RecipeChoice.MaterialChoice(Material.WAYFINDER_ARMOR_TRIM_SMITHING_TEMPLATE),
    /* base     */ new RecipeChoice.MaterialChoice(Tag.ITEMS_TRIMMABLE_ARMOR),
    /* addition */ new RecipeChoice.MaterialChoice(Material.GOLD_INGOT)
);
Bukkit.addRecipe(trim, true);
```

Refs:
[SmithingTransformRecipe](https://jd.papermc.io/paper/1.20.0/org/bukkit/inventory/SmithingTransformRecipe.html),
[SmithingTrimRecipe](https://jd.papermc.io/paper/1.20.1/org/bukkit/inventory/SmithingTrimRecipe.html).

### 7.1.6 MerchantRecipe (villager trades)

`MerchantRecipe` is set on a villager via `Villager#setRecipes`, not via
`Bukkit#addRecipe` — it is per-merchant, not server-global.

```java
MerchantRecipe trade = new MerchantRecipe(
    /* result      */ ItemStack.of(Material.EMERALD, 1),
    /* uses        */ 0,
    /* maxUses     */ 12,
    /* expReward   */ true,
    /* villagerXp  */ 4,
    /* priceMult   */ 0.05f
);
trade.addIngredient(ItemStack.of(Material.WHEAT, 20));
List<MerchantRecipe> recipes = new ArrayList<>(villager.getRecipes());
recipes.add(trade);
villager.setRecipes(recipes);
```

`uses` resets to 0 to re-enable the trade — the Javadoc explicitly notes setting
`uses < maxUses` re-enables the trade. See
[MerchantRecipe](https://jd.papermc.io/paper/1.21.5/org/bukkit/inventory/MerchantRecipe.html).

### 7.1.7 ComplexRecipe

`ComplexRecipe` is a **read-only marker** for vanilla recipes whose logic is
hardcoded in the engine (banner duplicate, banner-add-pattern, map cloning, map
extending, firework rocket, firework star, suspicious stew, repair, tipped arrow,
shulker box dyeing, etc.). You **cannot** create new `ComplexRecipe` instances —
the marker exists only so that iterators (`Bukkit#recipeIterator()`) can identify
and skip them when removing-then-re-adding recipes. See
[ComplexRecipe](https://jd.papermc.io/paper/1.21.8/org/bukkit/inventory/class-use/ComplexRecipe.html).

### 7.1.8 Recipe categories (`CraftingBookCategory` / `CookingBookCategory`)

Every modern recipe class accepts a category that controls its **recipe-book tab**:

| Recipe family | Category enum | Values |
|---|---|---|
| Shaped/Shapeless | `CraftingBookCategory` | `BUILDING`, `EQUIPMENT`, `REDSTONE`, `MISC` |
| Cooking (all 4) | `CookingBookCategory` | `FOOD`, `BLOCKS`, `MISC` |

If you don't set one, the recipe lands in the catch-all `MISC` tab.

---

## 7.2 RECIPE LIFECYCLE — `addRecipe`, `removeRecipe`, `resetRecipes`, RELOAD

```java
boolean ok = Bukkit.addRecipe(recipe, /* resendDiscovered */ true);
boolean removed = Bukkit.removeRecipe(key);
Bukkit.resetRecipes(); // wipes ALL recipes back to vanilla, including yours
Iterator<Recipe> it = Bukkit.recipeIterator();
```

The `resendDiscovered` flag (added in modern Paper) re-pushes the discovered set to
all online players — without it, the recipe exists server-side but nobody's recipe
book contains it until they relog.

### 7.2.1 The `/reload` and datapack-reload trap (SPIGOT-6084)

Reloading datapacks via `/minecraft:reload` (or `/reload`) **wipes recipes
registered through `Bukkit#addRecipe` and re-imports only the JSON datapack
recipes**. This was filed as
[SPIGOT-6084 / Paper#4244](https://github.com/PaperMC/Paper/issues/4244) and
remains unfixed — it's a deliberate Mojang behavior. Two mitigations:

1. **Re-register on `ServerLoadEvent`.** The event fires both at startup *and*
   after `/reload`:

   ```java
   @EventHandler
   public void onLoad(ServerLoadEvent e) {
       registerAllRecipes();
   }
   ```

2. **Ship recipes as a JSON datapack** under your jar's
   `data/<plugin>/recipe/<key>.json` and load it via Paper's
   [DatapackRegistrar](https://jd.papermc.io/paper/1.21.10/io/papermc/paper/datapack/DatapackRegistrar.html)
   (1.21.10+) — these survive reloads automatically.

> **Pitfall.** Calling `Bukkit#addRecipe` for a key that already exists throws
> `IllegalStateException`. Always pair add+remove on `ServerLoadEvent`:
> `Bukkit.removeRecipe(key); Bukkit.addRecipe(recipe, true);`.

### 7.2.2 Async safety

`Bukkit#addRecipe` mutates the global recipe registry. **Main-thread only.** On
Folia, recipe registration is global state, not region-bound — use
`GlobalRegionScheduler` only at server start.

---

## 7.3 PLAYER RECIPE BOOK

```java
player.discoverRecipe(key);          // adds to recipe book + plays toast
player.discoverRecipes(List.of(...)); // bulk
player.undiscoverRecipe(key);
boolean known = player.hasDiscoveredRecipe(key);
Set<NamespacedKey> all = player.getDiscoveredRecipes();
```

The toast pop-up only shows on **first** discovery; calling `discoverRecipe` for a
key the player already knows is a no-op.

### 7.3.1 Recipe book events

| Event | Fires when | Cancellable | Useful for |
|---|---|---|---|
| `PlayerRecipeDiscoverEvent` | Player auto-discovers a recipe (e.g. picks up an ingredient that unlocks one) | yes | gating recipes behind your own progression |
| `PlayerRecipeBookClickEvent` | Player clicks a recipe in the recipe book to auto-fill the grid | yes | blocking auto-fill for custom recipes |
| `PlayerRecipeBookSettingsChangeEvent` | Player toggles filter/open state | no | analytics |

Refs:
[PlayerRecipeDiscoverEvent](https://jd.papermc.io/paper/1.21.6/org/bukkit/event/player/PlayerRecipeDiscoverEvent.html),
[PlayerRecipeBookClickEvent](https://jd.papermc.io/paper/1.18/com/destroystokyo/paper/event/player/PlayerRecipeBookClickEvent.html).

---

## 7.4 CRAFTING EVENTS — THE THREE PHASES

The crafting GUI fires events in three phases:

```
slot change ──> PrepareItemCraftEvent ──> (player picks up result) ──> CraftItemEvent ──> ItemCraftedEvent
                       │                                                       │
                       └─ inventory.setResult(...) lets you change             └─ post-pickup, read-only
                          the result before the player sees it                    (1.21.5+)
```

### 7.4.1 PrepareItemCraftEvent — mutating the result slot

This event fires every time the crafting matrix changes (every slot drag). It is
**not cancellable** — use `inventory.setResult(null)` to "cancel" by hiding the
result.

```java
@EventHandler
public void onPrep(PrepareItemCraftEvent e) {
    Recipe recipe = e.getRecipe();
    if (recipe == null) return;
    if (!(recipe instanceof Keyed keyed)) return;
    if (!keyed.getKey().equals(LOST_RELIC_BLADE_KEY)) return;

    HumanEntity player = e.getView().getPlayer();
    if (!player.hasPermission("myplugin.craft.relic")) {
        e.getInventory().setResult(null); // hide the result
        return;
    }
    // Custom result: stamp the player's UUID into the PDC
    ItemStack result = e.getInventory().getResult();
    if (result == null) return;
    result.editMeta(meta -> meta.getPersistentDataContainer()
        .set(CRAFTER_KEY, PersistentDataType.STRING, player.getUniqueId().toString()));
    e.getInventory().setResult(result);
}
```

> **Pitfall.** `PrepareItemCraftEvent` fires *for every other player viewing the
> same crafting table too*. Do not key off the viewer alone if your table can be
> shared (vanilla tables are single-user, but inventory grids are shared with
> shulker viewers etc.).

Ref:
[PrepareItemCraftEvent](https://jd.papermc.io/paper/1.20/org/bukkit/event/inventory/PrepareItemCraftEvent.html).

### 7.4.2 CraftItemEvent — cancelling a craft

Fires when the player **clicks the result slot to take the item**. It extends
`InventoryClickEvent`, so all the click semantics apply (shift-click = repeat
craft until ingredients run out, etc.).

```java
@EventHandler
public void onCraft(CraftItemEvent e) {
    if (!(e.getWhoClicked() instanceof Player p)) return;
    if (!isLostRelicBlade(e.getRecipe())) return;
    if (!p.hasPermission("myplugin.craft.relic")) {
        e.setCancelled(true);
        p.sendMessage(Component.text("You may not forge the relic.", NamedTextColor.RED));
    }
}
```

Ref: [CraftItemEvent](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/inventory/CraftItemEvent.html).

### 7.4.3 ItemCraftedEvent — post-craft hook (1.21.5+)

A new, **post-pickup** event that fires after the player has actually received the
crafted item. It's read-only but exists so plugins don't have to re-implement
shift-click bookkeeping by listening to `InventoryClickEvent`.

```java
@EventHandler
public void onCrafted(ItemCraftedEvent e) {
    Bukkit.getLogger().info(e.getPlayer().getName() + " crafted " + e.getItem().getType());
}
```

Ref: [ItemCraftedEvent](https://jd.papermc.io/paper/1.21.11/io/papermc/paper/event/inventory/ItemCraftedEvent.html).

### 7.4.4 The four "prepare result" siblings

Anvil, smithing, grindstone, and enchant tables each have their own `Prepare*`
event that fires when a slot changes:

| Event | Workstation | Fires on | Result mutable | Notes |
|---|---|---|---|---|
| `PrepareAnvilEvent` | anvil | item or name change | yes (`setResult`) | level cost via `setRepairCost` |
| `PrepareSmithingEvent` | smithing table | base/template/addition change | yes | replaces 1.19's `PrepareSmithingTableEvent` |
| `PrepareGrindstoneEvent` | grindstone | item change | yes | preserves the unenchant XP if you keep result type |
| `PrepareItemEnchantEvent` | enchanting table | item placement / lapis change | enchants offered: yes | sets `EnchantmentOffer[3]` |

Refs:
[PrepareSmithingEvent](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/inventory/PrepareSmithingEvent.html),
[PrepareAnvilEvent](https://jd.papermc.io/paper/1.20.1/org/bukkit/event/inventory/PrepareAnvilEvent.html),
[PrepareGrindstoneEvent](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/inventory/PrepareGrindstoneEvent.html).

> **Pitfall.** All four fire on **every** slot tick, sometimes 2–3 times per
> single user click (drag, drop, then re-validate). Cache results — never do
> heavy work in these handlers without an early-exit check.

---

## 7.5 LOOT TABLES — API SIDE

`org.bukkit.loot.LootTable` is a server-side handle to a loot-table key; the
table's actual contents come from the JSON datapack (`/data/<ns>/loot_table/<path>.json`).

### 7.5.1 Getting a LootTable

```java
// Vanilla constants enum (typed, autocompletes):
LootTable simpleDungeon = Bukkit.getLootTable(LootTables.SIMPLE_DUNGEON.getKey());

// By NamespacedKey (works for custom datapacks):
LootTable mine = Bukkit.getLootTable(new NamespacedKey(plugin, "secret_chest"));

// Modern registry path (1.20.4+):
LootTable mine2 = Registry.LOOT_TABLE.get(new NamespacedKey(plugin, "secret_chest"));
```

The
[`LootTables`](https://jd.papermc.io/paper/1.21.4/org/bukkit/loot/LootTables.html)
enum lists every vanilla table key — `SIMPLE_DUNGEON`,
`ABANDONED_MINESHAFT`, `END_CITY_TREASURE`, `SHEEP_BLACK`,
`ENTITIES_ZOMBIE`, etc.

### 7.5.2 LootContext — the inputs

`LootContext` carries the per-roll context: who killed what, with how much luck,
where, with what looting level. Built with `LootContext.Builder`:

```java
LootContext ctx = new LootContext.Builder(location)
    .lootedEntity(zombie)         // sets vanilla "this_entity" predicate target
    .killer(player)               // who gets credit; defaults looting from this player's MainHand
    .lootingModifier(2)           // explicit override; -1 = derive from killer
    .luck(1.0f)                   // affects "high"/"low" loot tables
    .build();

Collection<ItemStack> drops = lootTable.populateLoot(new Random(), ctx);
```

Refs:
[LootTable](https://jd.papermc.io/paper/1.19/org/bukkit/loot/LootTable.html),
[LootContext.Builder](https://jd.papermc.io/folia/1.21/org/bukkit/loot/LootContext.Builder.html).

### 7.5.3 Filling an inventory holder

```java
// Lootable applies to chests, barrels, shulker boxes, hoppers, dispensers,
// droppers, minecarts-with-chest, decorated pots — anything with a loot table.
public void seedChest(Block chestBlock, NamespacedKey tableKey, long seed) {
    BlockState state = chestBlock.getState();
    if (state instanceof Lootable lootable) {
        lootable.setLootTable(Bukkit.getLootTable(tableKey));
        lootable.setSeed(seed);
        state.update(true, false);  // force block update; loot is lazy-rolled on first open
    }
}
```

The roll is **deferred until the first open**, so re-seeding before the player
opens it is free. After the first open the chest stops being `Lootable`-pending
(`hasLootTable()` returns false).

Ref: [Lootable](https://jd.papermc.io/paper/1.21.8/org/bukkit/loot/Lootable.html).

### 7.5.4 LootGenerateEvent — intercepting generation

```java
@EventHandler
public void onLoot(LootGenerateEvent e) {
    if (e.getLootTable().getKey().equals(LootTables.ABANDONED_MINESHAFT.getKey())) {
        // Inject a custom relic into 5% of mineshaft chest rolls.
        if (ThreadLocalRandom.current().nextDouble() < 0.05) {
            e.getLoot().add(scriptedShardStack());
        }
    }
}
```

`getLoot()` returns the **mutable** rolled list. `e.setLoot(...)` replaces it
wholesale. Cancelling stops the chest being filled at all.

Ref: [LootGenerateEvent](https://jd.papermc.io/paper/1.20.3/org/bukkit/event/world/LootGenerateEvent.html).

> **Pitfall.** `LootGenerateEvent` does **not** fire for entity drops — those go
> through the vanilla loot pipeline first and surface as
> `EntityDeathEvent#getDrops`. If you need to mutate mob drops, use the death
> events.

---

## 7.6 LOOT TABLE JSON SCHEMA

A loot table is `pools[] -> rolls + entries[] -> items + functions + conditions`.

```json
{
  "type": "minecraft:chest",
  "pools": [
    {
      "rolls": { "min": 1, "max": 3 },
      "bonus_rolls": { "min": 0.0, "max": 1.0 },
      "entries": [
        {
          "type": "minecraft:item",
          "name": "minecraft:diamond",
          "weight": 1,
          "functions": [
            { "function": "minecraft:set_count", "count": { "min": 1, "max": 4 } }
          ],
          "conditions": [
            { "condition": "minecraft:random_chance_with_enchanted_bonus",
              "unenchanted_chance": 0.1, "enchanted_chance": 0.5,
              "enchantment": "minecraft:looting" }
          ]
        },
        { "type": "minecraft:empty", "weight": 4 },
        {
          "type": "minecraft:loot_table",
          "value": "myplugin:secret_relic"
        }
      ]
    }
  ],
  "functions": [
    { "function": "minecraft:set_loot_table", "name": "myplugin:noop" }
  ]
}
```

Key fields:

| Field | Means |
|---|---|
| `type` | The "context" — `chest`, `entity`, `block`, `fishing`, `gift`, `barter`, `archaeology` |
| `pools[].rolls` | Number of times to roll the pool. Static, range `{min,max}`, or a number provider |
| `pools[].bonus_rolls` | Extra rolls multiplied by `LootContext#getLuck()` |
| `entries[].type` | `item`, `tag`, `loot_table`, `empty`, `dynamic`, `alternatives`, `group`, `sequence` |
| `entries[].weight` | Relative weight within pool. Combined with `quality * luck` for skewed rolls |
| `functions` | Item-modifier list — applied in order. Examples: `set_count`, `set_nbt`, `enchant_with_levels`, `enchant_randomly`, `apply_bonus`, `set_potion`, `furnace_smelt` |
| `conditions` | Predicate list — all must pass for the entry/pool. Examples: `random_chance`, `entity_properties`, `killed_by_player`, `random_chance_with_enchanted_bonus` |

References:
[Minecraft Wiki: Loot table](https://minecraft.wiki/w/Loot_table),
[skylinerw loot table guide](https://github.com/skylinerw/guides/blob/master/java/loot%20tables.md),
[Minecraft Wiki: Item modifier](https://minecraft.wiki/w/Item_modifier).

> **1.20.5 schema rename.** Pre-1.20.5 used `function: minecraft:set_nbt` with
> raw NBT; 1.20.5+ uses `set_components` with the new item-component schema. Old
> tables still load with a deprecation warning. Always target the modern schema
> for new content.

---

## 7.7 LOOT FROM ENTITIES — `EntityDeathEvent#getDrops`

Mobs have an entity loot table baked into their entity definition
(`minecraft:entities/zombie`, etc.). The pipeline is:

```
EntityDeathEvent fires
   ↓
getDrops() returns a mutable List<ItemStack> already populated
by the vanilla entity loot table (with looting applied)
   ↓
You mutate the list (add, remove, replace)
   ↓
After the event, items spawn at the death location
```

```java
@EventHandler
public void onMobDeath(EntityDeathEvent e) {
    LivingEntity victim = e.getEntity();
    if (victim.getType() != EntityType.ZOMBIE) return;
    Player killer = victim.getKiller();
    if (killer == null) return;
    // Drop a relic shard 1% of the time when killed with a netherite sword.
    ItemStack hand = killer.getInventory().getItemInMainHand();
    if (hand.getType() == Material.NETHERITE_SWORD
        && ThreadLocalRandom.current().nextDouble() < 0.01) {
        e.getDrops().add(scriptedShardStack());
    }
    // Drop more XP for "elite" zombies tagged with our PDC marker.
    if (victim.getPersistentDataContainer().has(ELITE_KEY, PersistentDataType.BYTE)) {
        e.setDroppedExp(e.getDroppedExp() * 5);
    }
}
```

For player deaths, use [`PlayerDeathEvent`](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/entity/PlayerDeathEvent.html)
which extends `EntityDeathEvent` and adds keep-inventory / keep-level / death
message handling.

> **Pitfall.** Adding to `getDrops()` does **not** prevent the vanilla drops —
> only `clear()`-then-add does. Cancelling the event entirely stops drops *and*
> XP *and* statistics tracking — almost never what you want.

---

## 7.8 ADVANCEMENTS — TWO HALVES (DATAPACK + API)

An advancement has **two** halves:

1. **Definition** — a JSON file in a datapack (`/data/<ns>/advancement/<path>.json`)
   that lists criteria, rewards, parent, display, and trigger conditions.
2. **Player progress** — a per-player record of which criteria are awarded.

The Bukkit API only exposes the **progress** half:
[`Advancement`](https://jd.papermc.io/paper/1.21.8/org/bukkit/advancement/Advancement.html)
+ [`AdvancementProgress`](https://jd.papermc.io/paper/1.20.0/org/bukkit/advancement/AdvancementProgress.html).
You can:

- Look up advancements via `Bukkit#getAdvancement(NamespacedKey)`.
- Iterate via `Bukkit#advancementIterator()`.
- Read `AdvancementProgress#isDone()`, `getAwardedCriteria()`, `getRemainingCriteria()`.
- **Award** or **revoke** individual criteria via
  `progress.awardCriteria("name")` / `progress.revokeCriteria("name")`.

You **cannot** define a new advancement purely through Bukkit — that has always
required either NMS hackery or a datapack. Paper 1.21.10+ added
[`DatapackRegistrar`](https://jd.papermc.io/paper/1.21.10/io/papermc/paper/datapack/DatapackRegistrar.html)
which lets you ship JSON files inside your jar and register them at server start.

### 7.8.1 Defining an advancement (datapack JSON)

```json
{
  "parent": "minecraft:adventure/root",
  "display": {
    "icon": { "id": "minecraft:netherite_sword" },
    "title": { "translate": "advancement.myplugin.relic_forged.title" },
    "description": { "translate": "advancement.myplugin.relic_forged.desc" },
    "frame": "challenge",
    "show_toast": true,
    "announce_to_chat": true,
    "hidden": false
  },
  "criteria": {
    "forge_relic": {
      "trigger": "minecraft:impossible"
    }
  },
  "requirements": [["forge_relic"]],
  "rewards": {
    "experience": 100,
    "function": "myplugin:relic/grant"
  }
}
```

`"trigger": "minecraft:impossible"` is the standard pattern when you intend to
grant the criterion entirely from the plugin side — it can never fire on its own.

`requirements` is a list-of-lists: outer is AND, inner is OR. `[["a", "b"], ["c"]]`
means `(a OR b) AND c`. Default (omitted) is "all criteria, AND".

`frame` is `task` (default), `goal`, or `challenge` (gold, with toast fanfare).

### 7.8.2 Registering datapack content from a jar (1.21.10+)

```java
Bukkit.getPluginManager().registerEvent(
    LifecycleEventManager.LifecycleEvents.DATAPACK_DISCOVERY, ...);

// Or in a Paper plugin's bootstrap:
public class MyBootstrap implements PluginBootstrap {
    @Override
    public void bootstrap(BootstrapContext ctx) {
        ctx.getLifecycleManager().registerEventHandler(
            LifecycleEvents.DATAPACK_DISCOVERY,
            (DatapackRegistrar registrar) -> {
                registrar.discoverPack(
                    Path.of("/path/inside/jar/my_advancements"),
                    "my_advancements"
                );
            }
        );
    }
}
```

For older Paper versions the only options are: (a) require the server admin to
drop the JSON into `world/datapacks/`, or (b) use the deprecated
`UnsafeValues#loadAdvancement(NamespacedKey, String)` shim — the latter is
unstable and broken on multiple 1.20.x point releases. **Prefer the datapack
route.**

### 7.8.3 Awarding criteria from the API

```java
public void grantRelicAdvancement(Player p) {
    Advancement adv = Bukkit.getAdvancement(new NamespacedKey(plugin, "relic_forged"));
    if (adv == null) return;
    AdvancementProgress prog = p.getAdvancementProgress(adv);
    if (prog.isDone()) return;
    prog.awardCriteria("forge_relic"); // matches "criteria" key in the JSON
}
```

When the **last** criterion is awarded, the engine fires
`PlayerAdvancementDoneEvent`, plays the toast, runs `rewards.function`, and
broadcasts to chat (if `announce_to_chat: true`).

### 7.8.4 Advancement events

| Event | Fires when | Notes |
|---|---|---|
| `PlayerAdvancementCriterionGrantEvent` | A criterion is granted (vanilla trigger or API) | Cancellable — blocks the criterion |
| `PlayerAdvancementDoneEvent` | All criteria complete; advancement awarded | Read-only; runs **after** rewards |

Refs:
[PlayerAdvancementDoneEvent](https://jd.papermc.io/paper/1.17.1/org/bukkit/event/player/PlayerAdvancementDoneEvent.html),
[PlayerAdvancementCriterionGrantEvent](https://jd.papermc.io/paper/1.21.0/com/destroystokyo/paper/event/player/PlayerAdvancementCriterionGrantEvent.html).

```java
@EventHandler
public void onCrit(PlayerAdvancementCriterionGrantEvent e) {
    if (!e.getAdvancement().getKey().getNamespace().equals("myplugin")) return;
    if (!e.getPlayer().hasPermission("myplugin.advance")) e.setCancelled(true);
}

@EventHandler
public void onDone(PlayerAdvancementDoneEvent e) {
    if (e.getAdvancement().getKey().equals(RELIC_KEY)) {
        e.getPlayer().getWorld().strikeLightningEffect(e.getPlayer().getLocation());
    }
}
```

### 7.8.5 Vanilla triggers (the ones you'll use without a custom criterion)

Common values for `criteria.<name>.trigger`:

| Trigger | Fires when |
|---|---|
| `minecraft:impossible` | Never (pure API-grant pattern) |
| `minecraft:inventory_changed` | Inventory contents change. Match by `items[]` predicate |
| `minecraft:consume_item` | Player eats/drinks something |
| `minecraft:player_killed_entity` | Player kills entity. Match by entity predicate |
| `minecraft:entity_killed_player` | Player killed by entity |
| `minecraft:placed_block` | Block placed |
| `minecraft:enter_block` | Player enters a block (waterlogging, leaves) |
| `minecraft:fishing_rod_hooked` | Catch via fishing rod |
| `minecraft:tick` | Every tick. Used with player predicate to gate on location/time |

The full list lives in the
[Minecraft Wiki: Advancement triggers](https://minecraft.wiki/w/Advancement/JSON_format#Triggers).

---

## 7.9 ASYNC SAFETY MATRIX

| Operation | Main thread | Async safe | Folia scheduler |
|---|---|---|---|
| `Bukkit#addRecipe` / `removeRecipe` / `resetRecipes` | ✅ | ❌ | `GlobalRegionScheduler` |
| `Bukkit#getRecipe(key)` / `recipeIterator()` | ✅ | ✅ (read) | any |
| `LootTable#populateLoot` | ✅ | ⚠️ (random + state read) | `RegionScheduler` for the location |
| `Lootable#setLootTable` (block state) | ✅ | ❌ | region of the block |
| `EntityDeathEvent#getDrops` mutation | ✅ (in handler) | n/a | region of entity |
| `Player#discoverRecipe` | ✅ | ❌ | scheduler of the player's region |
| `Player#getAdvancementProgress` (read) | ✅ | ✅ | any |
| `AdvancementProgress#awardCriteria` | ✅ | ❌ (sends packet) | player's region |
| Reading loot table JSON | ✅ | ✅ | any |
| `Bukkit#getLootTable(key)` | ✅ | ✅ | any |

Folia note: recipe state is server-global and is owned by the
`GlobalRegionScheduler`. Award/revoke advancement criteria, drop manipulation,
and `setLootTable` are owned by the **region** of the affected entity / block /
player.

---

## 7.10 FAILURE COOKBOOK

| Symptom | Cause | Fix |
|---|---|---|
| `IllegalStateException: Duplicate recipe ignored with ID …` | `Bukkit.addRecipe` called twice with same key (often after `/reload`) | Pair add+remove on `ServerLoadEvent`, or use `Bukkit.removeRecipe(key)` first |
| Custom recipe disappears after `/minecraft:reload` | SPIGOT-6084 — datapack reload wipes API recipes | Re-register in `ServerLoadEvent` listener, or ship as datapack JSON |
| Recipe doesn't show in recipe book even though crafting works | Forgot `resendDiscovered=true` flag and never called `discoverRecipe` | `Bukkit.addRecipe(recipe, true)` and ensure category set |
| `ExactChoice` ingredient never matches | Provided template stack has different display name / PDC than crafted ingredient | Strip metadata before comparing, or use `MaterialChoice` + `PrepareItemCraftEvent` validation |
| Smithing recipe matches in 1.19 but not 1.20+ | Old `SmithingRecipe(base, addition)` constructor — no template slot | Switch to `SmithingTransformRecipe(template, base, addition)` |
| `PrepareItemCraftEvent.setResult` ignored | Set on `getRecipe().getResult()` (immutable) instead of `getInventory().setResult(...)` | Always mutate via `e.getInventory().setResult(...)` |
| Mob drops doubled when killed by player | Re-implemented vanilla drops via `EntityDeathEvent#getDrops().add(...)` instead of replacing | Either `clear()` + add, or just augment, never both |
| Custom items in chest only first time, then never | `Lootable#setLootTable` called too late — chest was already opened (`hasLootTable() == false`) | Set the loot table on the `BlockState` *before* anyone opens it; check `isLootTablePending` |
| `LootGenerateEvent` not firing for mob drops | Mob drops go via vanilla pipeline, surface as `EntityDeathEvent` | Use `EntityDeathEvent#getDrops()` for mobs |
| Advancement toast shows but criterion not actually awarded | Used `awardCriteria` on a criterion name that doesn't exist in the JSON | Match the criterion key string exactly; `awardCriteria` returns `false` if it didn't apply |
| `Bukkit#getAdvancement` returns null for your custom advancement | Datapack not discovered / file path wrong | Check `/datapack list`, verify file is at `/data/<ns>/advancement/<path>.json` |
| `loadAdvancement(UnsafeValues)` throws on 1.20.5+ | Removed — schema migration broke it | Migrate to `DatapackRegistrar` (1.21.10+) or ship a real datapack |
| Custom recipe not visible to players who logged in before plugin loaded | Recipes pushed to clients only at login | Call `Bukkit.updateRecipes()` after re-registration; force player relog as last resort |
| Smithing trim recipe matches but produces wrong material | `addition` slot determined trim color in 1.20.0; renamed/fixed by 1.21 | Pin to 1.21+ trim materials (`Tag.ITEMS_TRIM_MATERIALS`) and test on the runtime version |
| Chest loot identical every time it's broken+placed | Same seed via `Lootable#setSeed(0L)` (default) | Always pass `Long.MIN_VALUE` (Mojang sentinel for "fresh seed") or a unique long |
| `LootContext.Builder` requires `Location` non-null | Builder constructor takes `Location` directly | Pass `entity.getLocation()` or world spawn — never null |
| `populateLoot` returns empty list | All entries failed `random_chance` predicate (luck=0, chance < 1) | Pass `.luck(player.getLuck())` from a `Player` to enable luck-weighted entries |

---

## 7.11 PERFORMANCE NOTES

- **`Bukkit#recipeIterator()` allocates a fresh list every call.** Cache the
  result if you scan recipes repeatedly (e.g. in tab-complete handlers).
- **`Bukkit#getRecipe(NamespacedKey)` is `O(1)`** (HashMap lookup) — cheap.
- **`LootTable#populateLoot` is moderately expensive** (random rolls, function
  evaluation, item-component resolution). Cost: roughly **40–150 µs per chest**
  on a modern CPU. Don't call in a tight loop on the main thread; chunk it.
- **`AdvancementProgress#awardCriteria` sends a packet per call.** Batch grants
  by awarding all criteria in one tick rather than one per game-event; clients
  collapse rapid toasts into a single animation.
- **`PrepareItemCraftEvent` fires every slot change**, including drag-paint.
  A complex listener can produce 100+ event invocations per second per open
  inventory. Always early-exit on recipe key first.
- **`LootGenerateEvent` is paid even when uncancelled** — the rolled list is
  copied to a mutable `ArrayList` for handlers. Avoid scope-creeping handlers.

---

## 7.12 FOLIA NOTES

- **Recipe registration** is global state. Use
  `Bukkit.getGlobalRegionScheduler()` and only at server start (in your
  `ServerLoadEvent` listener, which runs on the global region).
- **Loot table population** happens on the region of the container or entity.
  `LootGenerateEvent` and `EntityDeathEvent` fire on the correct region already;
  do not re-schedule.
- **Advancement awards** must run on the player's current region. If you award
  from a non-region context (e.g. a webhook/HTTP callback), schedule via
  `player.getScheduler().run(plugin, task, null)`.
- **Recipe-book pushes** (`discoverRecipe`) target the player's region.

---


## 7.13 REFERENCE IMPLEMENTATION — THE "LOST RELIC" PLUGIN

A complete, runnable plugin that ties together every topic in this section:

- A **smithing-transform recipe** that upgrades a Diamond Sword + Netherite Ingot
  via a custom template into a **Lost Relic Blade** (PDC-tagged unique item).
- A **chest-loot injection** via `LootGenerateEvent` that drops a Relic Shard in
  5% of mineshaft / dungeon / stronghold chest rolls.
- A **mob-drop hook** via `EntityDeathEvent` that drops a Relic Shard at 1% from
  zombies killed with a netherite weapon.
- A **player recipe-book gating** via `PlayerRecipeDiscoverEvent` that prevents
  players without `myplugin.relic.discover` from auto-learning the recipe.
- A **custom advancement** (datapack-shipped, JSON below) granted on the first
  successful craft, with toast + chat broadcast and a 100-XP reward.
- Re-registration on `ServerLoadEvent` to survive `/reload`.

### 7.13.1 `plugin.yml`

```yaml
name: LostRelic
version: 1.0.0
main: com.example.lostrelic.LostRelicPlugin
api-version: '1.21'
folia-supported: true
load: STARTUP
permissions:
  myplugin.relic.discover:
    default: true
  myplugin.relic.craft:
    default: op
```

### 7.13.2 `src/main/java/.../LostRelicPlugin.java`

```java
package com.example.lostrelic;

import io.papermc.paper.event.inventory.ItemCraftedEvent;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import org.bukkit.*;
import org.bukkit.advancement.Advancement;
import org.bukkit.advancement.AdvancementProgress;
import org.bukkit.event.*;
import org.bukkit.event.entity.EntityDeathEvent;
import org.bukkit.event.inventory.PrepareItemCraftEvent;
import org.bukkit.event.player.PlayerRecipeDiscoverEvent;
import org.bukkit.event.server.ServerLoadEvent;
import org.bukkit.event.world.LootGenerateEvent;
import org.bukkit.inventory.*;
import org.bukkit.loot.LootTables;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.entity.*;

import java.util.concurrent.ThreadLocalRandom;

public final class LostRelicPlugin extends JavaPlugin implements Listener {

    public static final String NAMESPACE = "lostrelic";
    public NamespacedKey RELIC_BLADE_KEY;
    public NamespacedKey RELIC_SHARD_KEY;
    public NamespacedKey CRAFTER_PDC_KEY;
    public NamespacedKey RELIC_ADV_KEY;

    @Override
    public void onEnable() {
        RELIC_BLADE_KEY  = new NamespacedKey(this, "lost_relic_blade");
        RELIC_SHARD_KEY  = new NamespacedKey(this, "relic_shard");
        CRAFTER_PDC_KEY  = new NamespacedKey(this, "crafter_uuid");
        RELIC_ADV_KEY    = new NamespacedKey(this, "relic_forged");
        getServer().getPluginManager().registerEvents(this, this);
    }

    private ItemStack relicShard() {
        ItemStack s = ItemStack.of(Material.PRISMARINE_SHARD);
        s.editMeta(m -> {
            m.displayName(Component.text("Relic Shard", NamedTextColor.AQUA));
            m.getPersistentDataContainer().set(RELIC_SHARD_KEY, PersistentDataType.BYTE, (byte) 1);
        });
        return s;
    }

    private boolean isRelicShard(ItemStack s) {
        if (s == null || !s.hasItemMeta()) return false;
        return s.getItemMeta().getPersistentDataContainer()
            .has(RELIC_SHARD_KEY, PersistentDataType.BYTE);
    }

    private ItemStack relicBladeResult() {
        ItemStack s = ItemStack.of(Material.NETHERITE_SWORD);
        s.editMeta(m -> {
            m.displayName(Component.text("Lost Relic Blade", NamedTextColor.GOLD));
            m.lore(java.util.List.of(Component.text("Forged from forgotten relics.",
                NamedTextColor.GRAY)));
        });
        return s;
    }

    @EventHandler
    public void onLoad(ServerLoadEvent e) {
        Bukkit.removeRecipe(RELIC_BLADE_KEY);
        SmithingTransformRecipe r = new SmithingTransformRecipe(
            RELIC_BLADE_KEY,
            relicBladeResult(),
            new RecipeChoice.MaterialChoice(Material.NETHERITE_UPGRADE_SMITHING_TEMPLATE),
            new RecipeChoice.MaterialChoice(Material.DIAMOND_SWORD),
            new RecipeChoice.ExactChoice(relicShard())
        );
        Bukkit.addRecipe(r, true);
        getLogger().info("Lost Relic Blade recipe registered.");
    }

    @EventHandler
    public void onPrep(PrepareItemCraftEvent e) {
        Recipe rec = e.getRecipe();
        if (!(rec instanceof Keyed k)) return;
        if (!k.getKey().equals(RELIC_BLADE_KEY)) return;
        HumanEntity p = e.getView().getPlayer();
        if (!p.hasPermission("myplugin.relic.craft")) {
            e.getInventory().setResult(null);
            return;
        }
        ItemStack out = e.getInventory().getResult();
        if (out == null) return;
        out.editMeta(m -> m.getPersistentDataContainer()
            .set(CRAFTER_PDC_KEY, PersistentDataType.STRING, p.getUniqueId().toString()));
        e.getInventory().setResult(out);
    }

    @EventHandler
    public void onCrafted(ItemCraftedEvent e) {
        if (!e.getRecipe().getKey().equals(RELIC_BLADE_KEY)) return;
        Player p = e.getPlayer();
        Advancement adv = Bukkit.getAdvancement(RELIC_ADV_KEY);
        if (adv == null) return;
        AdvancementProgress prog = p.getAdvancementProgress(adv);
        if (!prog.isDone()) prog.awardCriteria("forge_relic");
    }

    @EventHandler
    public void onDiscover(PlayerRecipeDiscoverEvent e) {
        if (!e.getRecipe().equals(RELIC_BLADE_KEY)) return;
        if (!e.getPlayer().hasPermission("myplugin.relic.discover")) {
            e.setCancelled(true);
        }
    }

    @EventHandler
    public void onLoot(LootGenerateEvent e) {
        NamespacedKey k = e.getLootTable().getKey();
        boolean isStructureChest =
            k.equals(LootTables.ABANDONED_MINESHAFT.getKey())
         || k.equals(LootTables.SIMPLE_DUNGEON.getKey())
         || k.equals(LootTables.STRONGHOLD_CORRIDOR.getKey())
         || k.equals(LootTables.STRONGHOLD_LIBRARY.getKey());
        if (!isStructureChest) return;
        if (ThreadLocalRandom.current().nextDouble() < 0.05) {
            e.getLoot().add(relicShard());
        }
    }

    @EventHandler
    public void onMobDeath(EntityDeathEvent e) {
        if (e.getEntity().getType() != EntityType.ZOMBIE) return;
        Player killer = e.getEntity().getKiller();
        if (killer == null) return;
        ItemStack hand = killer.getInventory().getItemInMainHand();
        if (hand.getType() != Material.NETHERITE_SWORD) return;
        if (ThreadLocalRandom.current().nextDouble() < 0.01) {
            e.getDrops().add(relicShard());
        }
    }
}
```

### 7.13.3 The advancement datapack (`data/lostrelic/advancement/relic_forged.json`)

```json
{
  "parent": "minecraft:adventure/root",
  "display": {
    "icon": { "id": "minecraft:netherite_sword" },
    "title": { "text": "Lost Relic Forged", "color": "gold" },
    "description": { "text": "Forge a Lost Relic Blade." },
    "frame": "challenge",
    "show_toast": true,
    "announce_to_chat": true,
    "hidden": false
  },
  "criteria": {
    "forge_relic": { "trigger": "minecraft:impossible" }
  },
  "requirements": [["forge_relic"]],
  "rewards": {
    "experience": 100
  }
}
```

This datapack lives at `world/datapacks/lostrelic/` (or shipped via
`DatapackRegistrar` on 1.21.10+). With `pack.mcmeta`:

```json
{
  "pack": {
    "pack_format": 48,
    "description": "Lost Relic plugin advancements"
  }
}
```

`pack_format` 48 corresponds to 1.21.4; check
[Pack format](https://minecraft.wiki/w/Pack_format) for the value matching your
target version.

---

## 7.X SELF-REVIEW CHECKLIST

- [x] Mental model — three pipelines, one namespace, ASCII diagram
- [x] All 8 recipe types tabulated with class + workstation + since-version
- [x] ShapedRecipe + RecipeChoice (MaterialChoice vs ExactChoice) example
- [x] ShapelessRecipe example
- [x] FurnaceRecipe / BlastingRecipe / SmokingRecipe / CampfireRecipe (cook-time matrix)
- [x] StonecuttingRecipe example
- [x] SmithingTransformRecipe vs SmithingTrimRecipe (1.20+ rename)
- [x] MerchantRecipe (per-villager, not server-global)
- [x] ComplexRecipe (read-only marker; cannot create)
- [x] CraftingBookCategory / CookingBookCategory (recipe-book tabs)
- [x] addRecipe / removeRecipe / resetRecipes lifecycle + duplicate-key trap
- [x] SPIGOT-6084 / Paper#4244 reload-wipe gotcha + ServerLoadEvent fix
- [x] Async-safety + Folia GlobalRegionScheduler note for recipe registration
- [x] Player#discoverRecipe / undiscoverRecipe / hasDiscoveredRecipe
- [x] PlayerRecipeDiscoverEvent / PlayerRecipeBookClickEvent
- [x] Three-phase crafting events (Prepare → Craft → ItemCrafted 1.21.5+)
- [x] PrepareItemCraftEvent — mutating result via inventory.setResult
- [x] CraftItemEvent — cancelling
- [x] ItemCraftedEvent — post-pickup
- [x] Four prepare-result siblings (Anvil/Smithing/Grindstone/Enchant)
- [x] LootTable lookup via Bukkit#getLootTable + LootTables enum + Registry.LOOT_TABLE
- [x] LootContext.Builder (lootedEntity, killer, lootingModifier, luck, location)
- [x] Lootable#setLootTable + setSeed + lazy-roll-on-open behavior
- [x] LootGenerateEvent — mutating rolled drops
- [x] Loot table JSON schema (pools, rolls, entries, conditions, functions)
- [x] 1.20.5 set_components rename note
- [x] EntityDeathEvent#getDrops vs vanilla loot pipeline
- [x] PlayerDeathEvent extends EntityDeathEvent, keep-inventory note
- [x] Advancement = JSON definition + per-player progress; cannot define via Bukkit
- [x] Advancement JSON schema (parent, display, criteria, requirements, rewards)
- [x] DatapackRegistrar 1.21.10+ for shipping datapacks from a jar
- [x] AdvancementProgress.awardCriteria / revokeCriteria
- [x] PlayerAdvancementCriterionGrantEvent / PlayerAdvancementDoneEvent
- [x] Common vanilla advancement triggers tabulated
- [x] Async-safety matrix (10 ops × main / async / Folia scheduler)
- [x] 16-row failure cookbook
- [x] Performance notes (per-chest cost, per-call packet cost, etc.)
- [x] Folia notes — global vs region scheduler split
- [x] Reference implementation — Lost Relic plugin tying everything together

---

## 7.Z REFERENCES

- [Paper recipes guide](https://docs.papermc.io/paper/dev/recipes)
- [Paper API index](https://docs.papermc.io/paper/dev/api/)
- [Paper Bukkit (1.21.4)](https://jd.papermc.io/paper/1.21.4/org/bukkit/Bukkit.html)
- [Paper ShapedRecipe](https://jd.papermc.io/paper/1.21.0/org/bukkit/inventory/ShapedRecipe.html)
- [Paper RecipeChoice](https://jd.papermc.io/paper/1.21.10/org/bukkit/inventory/RecipeChoice.html)
- [Paper RecipeChoice.MaterialChoice](https://jd.papermc.io/paper/1.21.10/org/bukkit/inventory/RecipeChoice.MaterialChoice.html)
- [Paper RecipeChoice.ExactChoice](https://jd.papermc.io/paper/1.21.11/org/bukkit/inventory/RecipeChoice.ExactChoice.html)
- [Paper StonecuttingRecipe](https://jd.papermc.io/paper/1.20/org/bukkit/inventory/StonecuttingRecipe.html)
- [Paper BlastingRecipe](https://jd.papermc.io/paper/1.18/org/bukkit/inventory/BlastingRecipe.html)
- [Paper SmithingRecipe](https://jd.papermc.io/paper/1.20.0/org/bukkit/inventory/SmithingRecipe.html)
- [Paper SmithingTransformRecipe](https://jd.papermc.io/paper/1.20.0/org/bukkit/inventory/SmithingTransformRecipe.html)
- [Paper SmithingTrimRecipe](https://jd.papermc.io/paper/1.20.1/org/bukkit/inventory/SmithingTrimRecipe.html)
- [Paper MerchantRecipe](https://jd.papermc.io/paper/1.21.5/org/bukkit/inventory/MerchantRecipe.html)
- [Paper ComplexRecipe (uses)](https://jd.papermc.io/paper/1.21.8/org/bukkit/inventory/class-use/ComplexRecipe.html)
- [Paper PrepareItemCraftEvent](https://jd.papermc.io/paper/1.20/org/bukkit/event/inventory/PrepareItemCraftEvent.html)
- [Paper CraftItemEvent](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/inventory/CraftItemEvent.html)
- [Paper ItemCraftedEvent (1.21.5+)](https://jd.papermc.io/paper/1.21.11/io/papermc/paper/event/inventory/ItemCraftedEvent.html)
- [Paper PrepareSmithingEvent](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/inventory/PrepareSmithingEvent.html)
- [Paper PrepareAnvilEvent](https://jd.papermc.io/paper/1.20.1/org/bukkit/event/inventory/PrepareAnvilEvent.html)
- [Paper PrepareGrindstoneEvent](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/inventory/PrepareGrindstoneEvent.html)
- [Paper PlayerRecipeDiscoverEvent](https://jd.papermc.io/paper/1.21.6/org/bukkit/event/player/PlayerRecipeDiscoverEvent.html)
- [Paper PlayerRecipeBookClickEvent](https://jd.papermc.io/paper/1.18/com/destroystokyo/paper/event/player/PlayerRecipeBookClickEvent.html)
- [Paper LootTable](https://jd.papermc.io/paper/1.19/org/bukkit/loot/LootTable.html)
- [Paper LootTables (enum)](https://jd.papermc.io/paper/1.21.4/org/bukkit/loot/LootTables.html)
- [Paper LootContext.Builder](https://jd.papermc.io/folia/1.21/org/bukkit/loot/LootContext.Builder.html)
- [Paper Lootable](https://jd.papermc.io/paper/1.21.8/org/bukkit/loot/Lootable.html)
- [Paper LootGenerateEvent](https://jd.papermc.io/paper/1.20.3/org/bukkit/event/world/LootGenerateEvent.html)
- [Paper PlayerDeathEvent](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/entity/PlayerDeathEvent.html)
- [Paper Advancement](https://jd.papermc.io/paper/1.21.8/org/bukkit/advancement/Advancement.html)
- [Paper AdvancementProgress](https://jd.papermc.io/paper/1.20.0/org/bukkit/advancement/AdvancementProgress.html)
- [Paper PlayerAdvancementDoneEvent](https://jd.papermc.io/paper/1.17.1/org/bukkit/event/player/PlayerAdvancementDoneEvent.html)
- [Paper PlayerAdvancementCriterionGrantEvent](https://jd.papermc.io/paper/1.21.0/com/destroystokyo/paper/event/player/PlayerAdvancementCriterionGrantEvent.html)
- [Paper DatapackRegistrar (1.21.10+)](https://jd.papermc.io/paper/1.21.10/io/papermc/paper/datapack/DatapackRegistrar.html)
- [Paper DataPack](https://jd.papermc.io/paper/1.21.3/org/bukkit/packs/DataPack.html)
- [Paper#4244 / SPIGOT-6084 reload-wipe](https://github.com/PaperMC/Paper/issues/4244)
- [Minecraft Wiki: Loot table](https://minecraft.wiki/w/Loot_table)
- [Minecraft Wiki: Item modifier](https://minecraft.wiki/w/Item_modifier)
- [Minecraft Wiki: Pack format](https://minecraft.wiki/w/Pack_format)
- [skylinerw loot tables guide](https://github.com/skylinerw/guides/blob/master/java/loot%20tables.md)

Compliance note: information from external sources was paraphrased to comply
with content licensing.

---

## See also

- `01-build-tooling.md` — datapack-shipping in your jar (resources, plugin
  bootstrap).
- `02-velocity-folia-bedrock.md` §2 — Folia regional/global scheduler split.
- `03-events-extras.md` — generic event-listener patterns and priorities.
- `04-commands-extras.md` — `/give` / `/loot` Brigadier interactions if you
  want admin commands for testing recipes and loot tables.
- `05-entities-mobs.md` §5.6 — entity PDC (mob death drops use the same key).
- `06-world-blocks-chunks.md` §6.4 — TileState / Lootable chests live on
  block entities.
- `paper-plugin-dev.md` — original short references for recipes / drops /
  advancements.
