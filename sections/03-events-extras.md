---
name: paper-plugin-dev-events-extras
description: >
  Comprehensive expansion to paper-plugin-dev.md §3 covering every event family the base file
  glosses over: Vehicle (boat/minecart) events, Hanging (item frame/painting) events, Raid
  events, Weather/Lightning events, Paper-only events catalog (Arm swing, Arrow body, Bed-fail
  enter, Async chat, Async preprocess, Player click unknown entity, etc.), AsyncTabComplete,
  AsyncPlayerSendCommands, dynamic event registration via EventExecutor lambdas, manual
  HandlerList unregistration, custom event design with proper getHandlerList semantics,
  priority/cancellation deep dive, the event-loop performance model, exception isolation, and
  patterns for one-shot listeners, conditional listeners, and event-await coroutines.
---

# 3. EVENT SYSTEM — DEEP DIVE / EXTRAS (v2)

`paper-plugin-dev.md` §3 covers basic listener registration, EventPriority, and the most
common Player/Block/Entity/Inventory events. This file fills in everything else: the entire
Vehicle / Hanging / Raid / Weather event families, all the Paper-only events worth knowing,
async tab completion, dynamic registration via EventExecutor lambdas, manual unregistration,
custom event design (with the `getHandlerList` static gotcha that breaks subclass-event
plugins), and performance/threading details about how the event loop actually executes.

Verified against [Paper 1.21.x event packages](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/package-summary.html)
and [Paper-specific event package](https://jd.papermc.io/paper/1.21.11/io/papermc/paper/event/player/package-summary.html).

---

## 3.0 EVENT-LOOP MENTAL MODEL

Before discussing specific events, understand how Bukkit's event system actually executes:

1. Plugin calls `getServer().getPluginManager().registerEvents(listener, plugin)`.
2. The `PluginManager` reflects each `@EventHandler` method on `listener` and creates a
   `RegisteredListener` for each `(eventClass, priority, executor, listener, plugin,
   ignoreCancelled)` tuple.
3. Each `RegisteredListener` is added to a per-event `HandlerList`.
4. When `Bukkit.getPluginManager().callEvent(event)` runs:
    - The current event's `HandlerList` is "baked" (sorted by priority, flattened to an
      array — Paper's `HandlerList` is an [array-backed structure for cache locality](https://papermc-paper.mintlify.app/concepts/events)).
    - The system iterates the array in priority order: `LOWEST → LOW → NORMAL → HIGH →
      HIGHEST → MONITOR`.
    - For each listener: if the event is cancellable AND already cancelled AND the
      listener has `ignoreCancelled = true`, it's skipped. Otherwise the executor is
      invoked.
    - Exceptions from any single handler are caught and logged but **don't stop the
      iteration** — other listeners still run.
5. `callEvent` returns. The caller checks `event.isCancelled()` etc.

**Key implications:**
- Listener invocation is **synchronous and serial** within a single `callEvent` call.
- "Async events" (e.g. `AsyncChatEvent`) are simply called from a thread that isn't the
  main thread; the dispatch logic above is identical.
- A handler that throws does **not** prevent other handlers from running, but the world
  state is whatever that handler partially mutated. Idempotent handlers tolerate this;
  non-idempotent handlers can corrupt state.
- Adding/removing handlers during dispatch is safe (the array is a snapshot).

---

## 3.A VEHICLE EVENTS

The base file mentions `EntityDamageByEntityEvent` for combat but skips vehicle-specific
events. They live in [`org.bukkit.event.vehicle`](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/vehicle/package-summary.html).

```java
import org.bukkit.event.vehicle.*;

// Player gets in / out
@EventHandler public void onEnter(VehicleEnterEvent e) {
    Entity passenger = e.getEntered();          // The Entity (often a Player) entering
    Vehicle vehicle  = e.getVehicle();          // Boat, Minecart, AbstractHorse, etc.
    if (passenger instanceof Player p && vehicle instanceof Boat) {
        p.sendMessage(Component.text("Anchors aweigh!"));
    }
}

@EventHandler public void onExit(VehicleExitEvent e) { /* ... */ }

// Vehicle is broken (player attacks, explosion, etc.)
@EventHandler public void onDestroy(VehicleDestroyEvent e) {
    if (e.getAttacker() instanceof Player p && !p.hasPermission("vehicles.break")) {
        e.setCancelled(true);
    }
}

// Movement (very high frequency — gate carefully)
@EventHandler public void onMove(VehicleMoveEvent e) {
    if (e.getFrom().getBlockX() == e.getTo().getBlockX()
        && e.getFrom().getBlockZ() == e.getTo().getBlockZ()) return;
    // crossed a block boundary — safe to do real work
}

// Other vehicle events:
VehicleBlockCollisionEvent     // hit a block
VehicleEntityCollisionEvent    // hit an entity (cancellable for selective collisions)
VehicleCreateEvent             // spawned
VehicleUpdateEvent             // ticked  ← extremely high frequency, avoid
VehicleDamageEvent             // took damage but not yet destroyed
```

**Vehicle event gotchas:**
- `VehicleEnterEvent`/`ExitEvent` fire for **any** entity entering/exiting, not just
  players. Always type-check.
- `VehicleMoveEvent` fires every tick the vehicle moves — gate with block-boundary
  comparison just like `PlayerMoveEvent`.
- `VehicleUpdateEvent` is "ticked" — multiple times per second per vehicle. Don't
  register a listener unless you genuinely need per-tick callbacks; prefer
  `runTaskTimer` instead.
- For horses/donkeys/llamas, `VehicleEnterEvent` fires on mount. To detect dismount in
  combat, prefer `EntityDismountEvent` (Spigot/Paper) for richer reasons.

---

## 3.B HANGING EVENTS (Item Frames, Paintings, Glow Item Frames)

[`org.bukkit.event.hanging`](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/hanging/package-summary.html)
covers item frames, glow item frames, and paintings.

```java
import org.bukkit.event.hanging.*;

@EventHandler public void onPlace(HangingPlaceEvent e) {
    Hanging hanging = e.getEntity();          // ItemFrame, GlowItemFrame, Painting
    Player player   = e.getPlayer();          // null if placed by non-player (rare)
    Block surface   = e.getBlock();           // wall block it's hanging on
    BlockFace face  = e.getBlockFace();
    EquipmentSlot hand = e.getHand();          // hand used (1.17.1+)
    ItemStack item  = e.getItemStack();        // the placed item (1.21+)

    if (player != null && !player.hasPermission("art.place")) {
        e.setCancelled(true);
    }
}

@EventHandler public void onBreak(HangingBreakEvent e) {
    HangingBreakEvent.RemoveCause cause = e.getCause();
    // ENTITY, EXPLOSION, OBSTRUCTION, PHYSICS, DEFAULT
}

@EventHandler public void onBreakBy(HangingBreakByEntityEvent e) {
    Entity remover = e.getRemover();
    if (remover instanceof Player p) {
        // protect spawn-area frames
        if (isInProtectedRegion(e.getEntity().getLocation())) e.setCancelled(true);
    }
}
```

**Hanging gotchas:**
- `HangingBreakEvent` fires for ALL break causes; `HangingBreakByEntityEvent` fires
  *additionally* when an entity is the cause. Both fire for entity-caused breaks; cancel
  in the more specific one.
- An item frame breaking when its supporting block breaks fires `HangingBreakEvent` with
  `Cause.PHYSICS`. To prevent this, you need to also listen to `BlockBreakEvent` and
  cancel that.
- For interactions (rotating an item frame), use `PlayerInteractEntityEvent` against
  `ItemFrame`, not a hanging event.
- `Hanging` is the supertype for all three — useful when writing code that doesn't care
  which kind. Check `instanceof ItemFrame`/`Painting`/`GlowItemFrame` if you do.

---

## 3.C RAID EVENTS (1.14+)

A pillager raid in a village. Per the [Bukkit raid event package](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/raid/package-summary.html):

```java
import org.bukkit.event.raid.*;

// A player with Bad Omen entered a village; raid begins
@EventHandler public void onTrigger(RaidTriggerEvent e) {
    Raid raid     = e.getRaid();
    Player player = e.getPlayer();
    Location at   = raid.getLocation();
    plugin.getLogger().info(player.getName() + " triggered a raid at " + at);
}

// Wave starts
@EventHandler public void onWaveSpawn(RaidSpawnWaveEvent e) {
    Raid raid     = e.getRaid();
    int wave      = raid.getWavesSpawned();
    int badOmen   = raid.getBadOmenLevel();
    List<Raider> raiders = e.getRaiders();
    Raider patron        = e.getPatrolLeader();   // captain of the wave
    raiders.forEach(r -> r.setCustomName("Wave " + wave + " Raider"));
}

// Raid ends successfully (heroes-of-the-village applied)
@EventHandler public void onFinish(RaidFinishEvent e) {
    List<Player> heroes = e.getWinners();
    heroes.forEach(p -> p.sendMessage(Component.text("You defended the village!")));
}

// Raid stopped (manually, peace mode, expired)
@EventHandler public void onStop(RaidStopEvent e) {
    RaidStopEvent.Reason reason = e.getReason();
    // PEACE, TIMEOUT, FINISHED, NOT_IN_VILLAGE, UNSPAWNABLE
}
```

**Raid gotchas:**
- Raids only trigger in **villages with at least one bed and a villager**. If your custom
  village has no villagers, `RaidTriggerEvent` won't fire.
- `RaidSpawnWaveEvent` fires *per wave*, not per raider. Iterate `getRaiders()` to act
  on individuals.
- Custom raids are not creatable via this API — only natural Bad-Omen-triggered raids.

---

## 3.D WEATHER & LIGHTNING EVENTS

[`org.bukkit.event.weather`](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/weather/package-summary.html):

```java
import org.bukkit.event.weather.*;

// Rain on/off
@EventHandler public void onWeather(WeatherChangeEvent e) {
    World world = e.getWorld();
    boolean toRain = e.toWeatherState();   // true = starting to rain
    if (toRain && world.getName().equals("dry_world")) e.setCancelled(true);
}

// Thunder on/off
@EventHandler public void onThunder(ThunderChangeEvent e) {
    boolean toThunder = e.toThunderState();
    ThunderChangeEvent.Cause cause = e.getCause();
    // NATURAL, COMMAND, SLEEP, PLUGIN, etc. (1.20+)
}

// Lightning bolt strikes (cancellable on Paper)
@EventHandler public void onLightning(LightningStrikeEvent e) {
    LightningStrike bolt = e.getLightning();
    LightningStrikeEvent.Cause cause = e.getCause();
    // COMMAND, CUSTOM, ENCHANTMENT, TRAP, TRIDENT, WEATHER, UNKNOWN

    if (cause == LightningStrikeEvent.Cause.WEATHER
        && e.getWorld().getName().equals("safe_world")) {
        e.setCancelled(true);
    }
}
```

**Weather gotchas:**
- `WeatherChangeEvent` fires when the weather *boolean* flips. The world's `setStorm()`
  call from another plugin fires this event, so listeners can collide.
- `LightningStrikeEvent.Cause.TRIDENT` (1.16+) — useful to disable Channeling enchantment
  on safe worlds.
- Lightning damage is a separate `EntityDamageEvent` with cause `LIGHTNING`. Cancelling
  the lightning event stops the bolt but **separately** the damage event still fires for
  any affected entities.
- `LightningStrikeEvent.Cause.CUSTOM` indicates `World#strikeLightning(loc, true)` —
  filter on this if you care about plugin-spawned vs. natural bolts.

---

## 3.E PAPER-ONLY EVENTS CATALOG

Most live in [`io.papermc.paper.event`](https://jd.papermc.io/paper/1.21.11/io/papermc/paper/event/player/package-summary.html)
and [`com.destroystokyo.paper.event`](https://jd.papermc.io/paper/1.21.4/com/destroystokyo/paper/event/server/package-summary.html).
The base file mentions a few; this is the comprehensive working set:

### 3.E.1 Player events (Paper)

```java
import io.papermc.paper.event.player.*;
import com.destroystokyo.paper.event.player.*;

PlayerArmSwingEvent              // Swing left/right arm — slot lets you tell which
PlayerArmorChangeEvent            // Armor slot changed (CHESTPLATE, etc.)
PlayerJumpEvent                   // Player jumped (Paper)
PlayerStartSpectatingEntityEvent  // /spectate or right-click in spectator mode
PlayerStopSpectatingEntityEvent
PlayerHandshakeEvent              // Modify handshake (proxy compat)
PlayerInitialSpawnEvent           // Spawn point at first join
PlayerLoadedWorldEvent            // World fully sent to client (NEW since 1.20.5)
PlayerLocaleChangeEvent           // Player changed language in client
PlayerPickItemEvent               // Middle-click pick block (creative)
PlayerPostRespawnEvent            // After respawn, world / inventory restored
PlayerStopUsingItemEvent          // Released right-click on bow/crossbow/shield
PlayerInsertLecternBookEvent       // Inserted a book into a lectern (1.21+)
PlayerClickAtEntityEvent           // Right-click any entity (incl. unknown server-side)
PlayerBucketEntityEvent            // Scoop axolotl/fish into bucket
PlayerLecternPageChangeEvent
PlayerInventorySlotChangeEvent     // ANY inventory slot of a player changed
PlayerOpenSignEvent                // Opened a sign for editing
PlayerSetSpawnEvent                // /spawnpoint or natural respawn point change
PlayerSignCommandPreprocessEvent   // 1.21+ slash-command on sign
PlayerStonecutterRecipeSelectEvent
PlayerTrackEntityEvent / PlayerUntrackEntityEvent  // Server tracking radius changes
```

### 3.E.2 Server events (Paper)

```java
ServerLoadEvent                   // Server fully started or reloaded
ServerExceptionEvent              // Internal server exception fired (great for Sentry)
ServerTickStartEvent / ServerTickEndEvent  // Per-tick hooks (perf-critical, rarely needed)
PaperServerListPingEvent          // Customise SLP per IP
WhitelistToggleEvent / WhitelistStateUpdateEvent
GS4QueryEvent                     // Query protocol response
AsyncPlayerSendCommandsEvent      // Modify command tree sent to a specific player (Paper)
```

### 3.E.3 Entity / Block events (Paper)

```java
EntityMoveEvent                    // Like PlayerMoveEvent but for any entity (PERF: very heavy)
EntityZapEvent                     // Lightning converts entity (zombie villager etc.)
EntityKnockbackByEntityEvent
EntityRemoveFromWorldEvent
PreCreatureSpawnEvent              // BEFORE spawn (cancel without packet) — much cheaper than CreatureSpawnEvent
PreSpawnerSpawnEvent
SkeletonHorseTrapEvent
EntityToggleSitEvent               // Wolf/cat/parrot sitting toggled
ExperienceOrbMergeEvent
PlayerNaturallySpawnCreaturesEvent // Modify natural mob caps per player
ProjectileCollideEvent             // BEFORE collision result determined (Paper extra)

BeaconActivatedEvent / BeaconDeactivatedEvent / BeaconEffectEvent
PlayerChunkLoadEvent / PlayerChunkUnloadEvent  // Per-player chunk send/forget
```

### 3.E.4 The "pre-" pattern

Several Paper events are **pre-** variants of Bukkit events. Prefer them when available
because they fire **before** the expensive operation rather than after, and they accept
a `setSpawn(false)` / `setShouldAbortVanilla(true)` cancel-equivalent that is cheaper:

| Bukkit | Paper "pre" |
|---|---|
| `CreatureSpawnEvent` | `PreCreatureSpawnEvent` (don't even spawn the entity) |
| `SpawnerSpawnEvent` | `PreSpawnerSpawnEvent` |
| `ProjectileHitEvent` | `ProjectileCollideEvent` (decide before damage) |

Cancellation cost: cancelling `CreatureSpawnEvent` still constructs the entity, runs
mob initialisation, then deletes it. Cancelling the `Pre` variant skips all that. On
busy farms this is the difference between 10 ms/tick and 100 ms/tick.

---

## 3.F ASYNC TAB COMPLETION

Per [`AsyncTabCompleteEvent` Javadoc](https://jd.papermc.io/paper/1.20.4/com/destroystokyo/paper/event/server/AsyncTabCompleteEvent.html):

```java
import com.destroystokyo.paper.event.server.AsyncTabCompleteEvent;

@EventHandler
public void onAsyncTab(AsyncTabCompleteEvent event) {
    if (!event.getBuffer().startsWith("/findplayer ")) return;

    // We can do DB lookups, HTTP, etc. here — it's already on an async thread.
    String prefix = event.getBuffer().substring("/findplayer ".length());
    List<String> matches = playerDatabase.searchByPrefix(prefix);

    // Add suggestions WITH tooltips:
    matches.forEach(name -> event.getCompletions().add(
        new AsyncTabCompleteEvent.Completion(name,
            Component.text("Click to fill", NamedTextColor.GRAY))));

    event.setHandled(true);   // skip the synchronous TabCompleteEvent
}
```

**Why it matters:**
- The synchronous `TabCompleteEvent` runs on the main thread. Heavy lookup → main-thread
  lag → TPS drops as players type.
- `AsyncTabCompleteEvent` fires on a Netty I/O thread. You **can** do DB queries here.
- Setting `event.setHandled(true)` tells Paper not to fire `TabCompleteEvent` afterwards,
  saving a main-thread bounce.

**Brigadier note:** Brigadier commands handle their own suggestions through
`SuggestionProvider`. `AsyncTabCompleteEvent` only fires for legacy Bukkit-style commands
and for arguments past the command literal in Brigadier trees that don't define their own
suggester.

---

## 3.G ASYNC PLAYER SEND COMMANDS

Modify the command tree the client sees, per-player. Useful for permission-aware tab
completion: only show `/ban` to staff:

```java
import com.destroystokyo.paper.event.brigadier.AsyncPlayerSendCommandsEvent;

@EventHandler
public void onSend(AsyncPlayerSendCommandsEvent<?> event) {
    if (!event.isAsynchronous() && event.hasFiredAsync()) return;  // dedupe

    Player player = event.getPlayer();
    var root = event.getCommandNode();   // RootCommandNode<?>

    if (!player.hasPermission("server.staff")) {
        root.removeChildByName("ban");
        root.removeChildByName("kick");
    }
}
```

Without this, `/ban<TAB>` would still show the command in the tab list (without
permission, the player just gets "Unknown command" on execute) — leaking the existence of
admin commands. With this, the client has no idea those commands exist.

---

## 3.H DYNAMIC LISTENER REGISTRATION (EventExecutor lambdas)

`registerEvents` requires a `Listener` class with `@EventHandler` methods. For cases
where a class is overkill — one-shot listeners, ReactiveX, runtime-built rules — use
`PluginManager#registerEvent(...)` with an `EventExecutor` lambda:

```java
import org.bukkit.event.*;
import org.bukkit.plugin.EventExecutor;
import org.bukkit.plugin.PluginManager;

PluginManager pm = Bukkit.getPluginManager();
Listener owner   = new Listener() {};   // empty marker — required as a "key"

EventExecutor exec = (listener, event) -> {
    PlayerJoinEvent e = (PlayerJoinEvent) event;
    e.getPlayer().sendMessage(Component.text("Hi from a lambda"));
};

pm.registerEvent(
    PlayerJoinEvent.class,
    owner,
    EventPriority.NORMAL,
    exec,
    plugin,
    /* ignoreCancelled */ false
);
```

**Use cases:**
- **One-shot listeners:** see §3.J.
- **Runtime-generated rules:** "if config says X, listen to event Y".
- **Reactive frameworks:** RxJava / Project Reactor adapters that map Bukkit events to
  Observables.

**Performance:** the lambda path is identical to the reflective `@EventHandler` path
once registered — Bukkit's reflection-based registration is one-time at registration. No
ongoing overhead.

---

## 3.I MANUAL UNREGISTRATION (HandlerList)

A common need: temporarily disable a listener (e.g. during boss fights), or unregister
on plugin disable for cleanliness. Per the [HandlerList Javadoc](https://jd.papermc.io/paper/org/bukkit/event/HandlerList.html):

```java
// Unregister ALL handlers from a single Listener instance:
HandlerList.unregisterAll(listenerInstance);

// Unregister ALL handlers belonging to a Plugin (Bukkit auto-does this on disable, so
// only call manually if you want to detach early):
HandlerList.unregisterAll(plugin);

// Unregister just one event type from a listener:
PlayerJoinEvent.getHandlerList().unregister(listenerInstance);

// Unregister all handlers from a specific event type globally (rarely useful):
HandlerList.unregisterAll();   // takes nothing — nukes everything
```

**`getHandlerList()` is a static method on the event class** — so accessing the handler
list for a custom event requires that the event class declared one (see §3.K).

---

## 3.J PATTERNS

### 3.J.1 One-shot listener

```java
public static <E extends Event> void waitForOnce(
        PluginManager pm,
        Plugin plugin,
        Class<E> eventType,
        EventPriority priority,
        Predicate<E> filter,
        Consumer<E> action
) {
    Listener key = new Listener() {};
    EventExecutor exec = (l, ev) -> {
        @SuppressWarnings("unchecked")
        E e = (E) ev;
        if (filter.test(e)) {
            action.accept(e);
            HandlerList.unregisterAll(l);   // self-disposal
        }
    };
    pm.registerEvent(eventType, key, priority, exec, plugin, false);
}

// Usage: wait for this player's next chat message
waitForOnce(pm, plugin, AsyncChatEvent.class, EventPriority.NORMAL,
    e -> e.getPlayer().equals(target),
    e -> handleAnswer(target, e.signedMessage().message())
);
```

### 3.J.2 Conditional listener with timeout

```java
public static <E extends Event> CompletableFuture<E> awaitWithTimeout(
        PluginManager pm, Plugin plugin,
        Class<E> eventType,
        Predicate<E> filter,
        Duration timeout
) {
    CompletableFuture<E> future = new CompletableFuture<>();
    Listener key = new Listener() {};
    EventExecutor exec = (l, ev) -> {
        @SuppressWarnings("unchecked") E e = (E) ev;
        if (filter.test(e)) {
            HandlerList.unregisterAll(l);
            future.complete(e);
        }
    };
    pm.registerEvent(eventType, key, EventPriority.NORMAL, exec, plugin, false);

    Bukkit.getScheduler().runTaskLater(plugin, () -> {
        if (!future.isDone()) {
            HandlerList.unregisterAll(key);
            future.completeExceptionally(new TimeoutException("Event timed out"));
        }
    }, timeout.toSeconds() * 20L);

    return future;
}
```

Use this for chat-prompt patterns, "press F to interact" countdowns, etc.

### 3.J.3 Exception-safe wrapper

Bukkit catches and logs handler exceptions but continues to other handlers. If you want
loud failures during development without nuking the server, wrap the executor:

```java
public static EventExecutor strictHandler(EventExecutor inner, Plugin plugin) {
    return (l, ev) -> {
        try { inner.execute(l, ev); }
        catch (Throwable t) {
            plugin.getLogger().log(Level.SEVERE,
                "Handler crashed for " + ev.getEventName() + " in " + l.getClass(), t);
            // Optional: rethrow if you want Paper's default crash logging too:
            throw new RuntimeException("Handler failed", t);
        }
    };
}
```

In production, **do not** call `Bukkit.shutdown()` from a handler — it kills paying
players' sessions. Surface exceptions via `ServerExceptionEvent` (§3.N) and forward to
your monitoring instead.

### 3.J.4 Rate-limited handler

Hot events (`PlayerMoveEvent`, `BlockPhysicsEvent`) sometimes need to do moderately
expensive work (region lookups, anti-cheat math) that's too slow per-event. Bucket the
work per-player with a simple per-key cooldown:

```java
public final class Throttle<K> {
    private final long cooldownNanos;
    private final ConcurrentHashMap<K, Long> last = new ConcurrentHashMap<>();

    public Throttle(Duration cooldown) {
        this.cooldownNanos = cooldown.toNanos();
    }

    /** Returns true if the action should proceed; false if it's still cooling down. */
    public boolean tryAcquire(K key) {
        long now = System.nanoTime();
        Long prev = last.get(key);
        if (prev != null && (now - prev) < cooldownNanos) return false;
        last.put(key, now);
        return true;
    }
}

private final Throttle<UUID> regionCheckThrottle = new Throttle<>(Duration.ofMillis(250));

@EventHandler(ignoreCancelled = true)
public void onMove(PlayerMoveEvent e) {
    if (!regionCheckThrottle.tryAcquire(e.getPlayer().getUniqueId())) return;
    // expensive region lookup — runs at most 4x/sec per player
    if (regionManager.isForbidden(e.getTo(), e.getPlayer())) e.setCancelled(true);
}
```

`ConcurrentHashMap` makes this safe under Folia (§2.B). For larger keyspaces, swap in
[Caffeine](https://github.com/ben-manes/caffeine) with an expire-after-access policy.

---

## 3.K CUSTOM EVENTS — DESIGN PROPERLY

The base file shows a one-class custom event. Two common mistakes break that:

### 3.K.1 The `getHandlerList()` static gotcha

**Problem:** if you put `getHandlerList()` on a base/abstract event class and subclass it,
Bukkit will reflect the *base* class's `getHandlerList()` for all subclass listeners,
meaning every subclass listener fires for every subclass event. See [TPGamesNL's gist](https://gist.github.com/TPGamesNL/ee79fc4c348721925e2b4cfafb1a9ea0).

**Wrong:**

```java
public abstract class MyBaseEvent extends Event {
    private static final HandlerList HANDLERS = new HandlerList();
    @Override public HandlerList getHandlers() { return HANDLERS; }
    public static HandlerList getHandlerList() { return HANDLERS; }   // shared - bad
}
public class MySpecificEvent extends MyBaseEvent { /* no static */ }
```

A listener registered for `MySpecificEvent` will also fire for *every other subclass* of
`MyBaseEvent` because the dispatcher reflects the base's static `getHandlerList()`.

**Right:** every concrete event class declares its own:

```java
public abstract class MyBaseEvent extends Event {
    /* no static handler list, or only protected helpers */
}

public class MySpecificEvent extends MyBaseEvent {
    private static final HandlerList HANDLERS = new HandlerList();
    @Override public HandlerList getHandlers() { return HANDLERS; }
    public static HandlerList getHandlerList() { return HANDLERS; }
}

public class MyOtherEvent extends MyBaseEvent {
    private static final HandlerList HANDLERS = new HandlerList();
    @Override public HandlerList getHandlers() { return HANDLERS; }
    public static HandlerList getHandlerList() { return HANDLERS; }
}
```

### 3.K.2 Async events

If you fire your event from a non-main thread, mark it as async:

```java
public class MyAsyncEvent extends Event {
    public MyAsyncEvent() {
        super(true);   // isAsync = true
    }
    /* ... */
}
```

This makes Bukkit reject `@EventHandler` methods that try to call sync API after the
fact (it throws `IllegalPluginAccessException`). It's a guardrail, not magic — your
handlers still need to themselves dispatch back to the main thread for Bukkit calls.

### 3.K.3 Cancellable boilerplate

```java
public class PlayerLevelUpEvent extends Event implements Cancellable {
    private static final HandlerList HANDLERS = new HandlerList();
    private final Player player;
    private final int oldLevel, newLevel;
    private boolean cancelled;

    public PlayerLevelUpEvent(Player player, int oldLevel, int newLevel) {
        this.player = player;
        this.oldLevel = oldLevel;
        this.newLevel = newLevel;
    }

    public Player getPlayer()  { return player; }
    public int getOldLevel()   { return oldLevel; }
    public int getNewLevel()   { return newLevel; }

    @Override public boolean isCancelled() { return cancelled; }
    @Override public void setCancelled(boolean c) { this.cancelled = c; }

    @Override public HandlerList getHandlers() { return HANDLERS; }
    public static HandlerList getHandlerList() { return HANDLERS; }
}

// Firing:
PlayerLevelUpEvent ev = new PlayerLevelUpEvent(player, 9, 10);
Bukkit.getPluginManager().callEvent(ev);
if (ev.isCancelled()) return;
applyLevelUp(player);
```

---

## 3.L PRIORITY & CANCELLATION DEEP DIVE

The base file lists `LOWEST → ... → MONITOR`. Here's what each is for:

| Priority | Use it for |
|---|---|
| `LOWEST` | Anti-cheat, security. Run first; deny early. |
| `LOW` | Game-mechanic adjustments (region protection, claim plugins). |
| `NORMAL` | Default. The main behaviour your plugin is here to add. |
| `HIGH` | Overrides of other plugins' decisions. |
| `HIGHEST` | Very last word — almost always followed by another plugin doing the same. |
| `MONITOR` | **Read-only** observation. Don't cancel or modify here. |

**Cancellation rules:**
- Once any handler calls `setCancelled(true)`, the event is *marked* cancelled but
  iteration continues.
- A higher-priority handler can call `setCancelled(false)` to revive.
- Handlers with `ignoreCancelled = true` are skipped if the event is currently cancelled
  when they are reached. They DO see the cancellation if it happened earlier.
- `MONITOR` handlers fire regardless of cancellation. **Never modify state in MONITOR.**
  This is a hard convention: Paper checks `isCancelled()` semantics consistently and
  some events even log warnings if a MONITOR handler mutates them.

**Decision logic example:**

```java
// Anti-cheat plugin: deny first, get out of the way.
@EventHandler(priority = EventPriority.LOWEST)
public void onMove(PlayerMoveEvent e) {
    if (isHacking(e.getPlayer())) e.setCancelled(true);
}

// Region protection: run after AC, before main mechanics.
@EventHandler(priority = EventPriority.LOW, ignoreCancelled = true)
public void onMove2(PlayerMoveEvent e) {
    if (enteringForbidden(e.getTo(), e.getPlayer())) e.setCancelled(true);
}

// Logger: monitor only.
@EventHandler(priority = EventPriority.MONITOR)
public void onMoveLog(PlayerMoveEvent e) {
    if (e.isCancelled()) statsCancelled.increment();
    else                  statsAllowed.increment();
}
```

---

## 3.M EVENT-LOOP PERFORMANCE

`PlayerMoveEvent` fires every tick, per moving player. On a 200-player server, that's
4000 events/sec. Three rules:

### Rule 1: Block-boundary gating

```java
@EventHandler(ignoreCancelled = true)
public void onMove(PlayerMoveEvent e) {
    Location from = e.getFrom();
    Location to   = e.getTo();
    if (from.getBlockX() == to.getBlockX()
        && from.getBlockY() == to.getBlockY()
        && from.getBlockZ() == to.getBlockZ()) return;
    // only runs ~1-3x per second per moving player
}
```

### Rule 2: Don't allocate inside hot loops

```java
// BAD: allocates a Location every event:
Location target = e.getTo().clone().add(0, 1, 0);

// GOOD: reuse a Vector, or just compute fields:
double tx = e.getTo().getX();
double ty = e.getTo().getY() + 1;
double tz = e.getTo().getZ();
```

### Rule 3: Don't I/O on hot events

If you must persist a player's position to a database, do it on a 5-second timer, not
on every move event. Cache writes; flush in a scheduled task.

### Rule 4: Avoid `EntityMoveEvent` unless you need it

`EntityMoveEvent` (Paper) fires for every moving entity in every chunk. On a server with
2000 entities it's catastrophic — multiple thousand events/tick. Prefer:
- `EntityToggleSitEvent` for sitting state
- A scheduler that polls only the entities you care about
- `PlayerChunkLoadEvent` to scan once when chunks come in

---

## 3.N EXCEPTION ISOLATION

Paper logs handler exceptions but doesn't unregister the handler. A buggy handler will
crash on every event call until you reload. To detect this in dev:

```java
// Paper-only event:
@EventHandler
public void onException(ServerExceptionEvent e) {
    plugin.getLogger().severe("Server exception: " + e.getException());
    // Forward to Sentry / your monitoring:
    sentry.captureException(e.getException().getCause());
}
```

Pair with the `strictHandler` wrapper in §3.J.3 during development.

---

## 3.N.1 EXPLOSION EVENT FAMILY

A common confusion: which explosion event fires when?

| Source | Event |
|---|---|
| Creeper, TNT, Wither head, Ghast fireball, Crystal | `EntityExplodeEvent` |
| Bed in Nether/End, Respawn anchor in Overworld, Wind charge | `BlockExplodeEvent` |

```java
@EventHandler public void onEntityBoom(EntityExplodeEvent e) {
    // e.getEntity() = the exploding entity (TNTPrimed, Creeper, etc.)
    // e.blockList() = mutable list of broken blocks — clear() to keep terrain intact
    if (isProtected(e.getLocation())) e.blockList().clear();
}

@EventHandler public void onBlockBoom(BlockExplodeEvent e) {
    // e.getBlock() = the source block (BED, RESPAWN_ANCHOR)
    // Same blockList() pattern.
}
```

Both events are cancellable (cancel = no explosion at all) and let you filter the block
list (cancel = "explosion happens but no terrain damage"). For damage to entities
caught in the blast, listen to `EntityDamageEvent` with cause `BLOCK_EXPLOSION` /
`ENTITY_EXPLOSION` separately.

---

## 3.O EVENT-PERFORMANCE CHEAT SHEET

| Event | Frequency | Performance care |
|---|---|---|
| `PlayerMoveEvent` | every tick per moving player | Always block-gate |
| `EntityMoveEvent` | every tick per moving entity | Avoid; poll instead |
| `PlayerChangedWorldEvent` | rare (teleport) | Free |
| `BlockPhysicsEvent` | every redstone tick / sand fall | Filter by Material early |
| `BlockFromToEvent` | water/lava flow tick | Filter by Material early |
| `AsyncChatEvent` | per chat | Free unless you do slow work |
| `EntityDamageEvent` | per hit | Free; cache victim/attacker outside lambda |
| `VehicleUpdateEvent` | every tick per vehicle | Avoid |
| `InventoryClickEvent` | per click | Use raw slot to skip non-our-GUI fast |
| `CreatureSpawnEvent` | per natural spawn | Prefer `PreCreatureSpawnEvent` |
| `PlayerInteractEvent` | per right/left click | Filter by `Action` early |
| `ChunkLoadEvent` | per chunk load | Free; prefer to cache than poll |

---

## 3.P SELF-REVIEW CHECKLIST

- [x] Vehicle event family (Enter/Exit/Damage/Destroy/Move/Update/Collision)
- [x] Hanging event family (Place / Break / BreakByEntity) with cause enum
- [x] Raid event family (Trigger / Wave / Finish / Stop)
- [x] Weather + Lightning events with cause enum
- [x] Paper player-events catalog (~25 events)
- [x] Paper server-events catalog (Tick / Exception / SLP / Whitelist / Query)
- [x] Paper entity-events catalog (Move / Zap / PreSpawn / etc.)
- [x] **The "Pre-" pattern explanation** (cheaper cancel)
- [x] AsyncTabCompleteEvent — full example with rich Completion
- [x] AsyncPlayerSendCommandsEvent — per-player command tree
- [x] **Dynamic registration via `PluginManager.registerEvent` + EventExecutor**
- [x] **Manual `HandlerList.unregister*` API** documented
- [x] One-shot listener pattern
- [x] Conditional listener with timeout (`CompletableFuture` event-await)
- [x] Exception-safe `strictHandler` wrapper
- [x] **Rate-limited handler pattern** (per-key throttle for hot events)
- [x] **Custom event design** with `getHandlerList()` subclass gotcha
- [x] Async custom events (`super(true)`)
- [x] **Priority + cancellation behaviour table** with worked example
- [x] **Event-loop performance rules** (block-gate, no allocation, no I/O)
- [x] **Event-performance cheat sheet** for hot/cold events
- [x] Exception isolation via `ServerExceptionEvent`
- [x] **Explosion event family** (`EntityExplodeEvent` vs `BlockExplodeEvent`)
- [x] Event-loop dispatch model documented (HandlerList baking)

---

## 3.Z REFERENCES

- [PaperMC Docs — Event listeners](https://docs.papermc.io/paper/dev/event-listeners/)
- [Paper API event tree](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/package-summary.html)
- [Paper player-event package](https://jd.papermc.io/paper/1.21.11/io/papermc/paper/event/player/package-summary.html)
- [Paper destroystokyo events (server)](https://jd.papermc.io/paper/1.21.4/com/destroystokyo/paper/event/server/package-summary.html)
- [Vehicle events package](https://jd.papermc.io/paper/1.21.4/org/bukkit/event/vehicle/package-summary.html)
- [Hanging events package](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/hanging/package-summary.html)
- [Raid events package (Spigot Javadoc)](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/event/raid/package-summary.html)
- [Weather events package](https://jd.papermc.io/paper/1.21.5/org/bukkit/event/weather/package-summary.html)
- [AsyncTabCompleteEvent](https://jd.papermc.io/paper/1.20.4/com/destroystokyo/paper/event/server/AsyncTabCompleteEvent.html)
- [HandlerList](https://jd.papermc.io/paper/org/bukkit/event/HandlerList.html)
- [Paper event system architecture](https://papermc-paper.mintlify.app/concepts/events)
- [TPGamesNL — base events `getHandlerList()` gotcha](https://gist.github.com/TPGamesNL/ee79fc4c348721925e2b4cfafb1a9ea0)
- [SilverCory — registerable / unregisterable listeners gist](https://gist.github.com/SilverCory/83c50a2143a6e294db096f4cf1183d51)
- [aadnk — exception-handler wrapper for every listener](https://gist.github.com/aadnk/5430459)
- [sasuked — event-await pattern](https://gist.github.com/sasuked/9f1b512cb9381ec7b12e2ab215bb367a)

---

## See also

- `paper-plugin-dev.md` §3 — original event basics (priority, registration, common events)
- `04-commands-extras.md` — Brigadier suggestion providers (related to AsyncTabComplete)
- `02-velocity-folia-bedrock.md` §2.B — Folia event-thread semantics (events fire on
  region threads)
