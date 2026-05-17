---
name: paper-plugin-dev-velocity-folia-bedrock
description: >
  Comprehensive expansion to paper-plugin-dev.md covering the entire proxy + multithreading +
  cross-platform stack for Paper-adjacent ecosystems: Velocity proxy plugin development from
  scratch (DI, events, Brigadier with redirects + custom args, modern forwarding, plugin
  messaging, server-list ping, RCON), Folia regionised multithreading deep-dive (4 schedulers,
  threading invariants, ScheduledTask lifecycle, chunk loading concerns, Paper+Folia compat
  shim, concurrency primitives), and Bedrock cross-play (Floodgate API + UUID format, Geyser
  API, Cumulus forms in full — Simple/Modal/Custom with response handling, custom-item / custom-
  block mappings, skin upload pipeline, Java fallbacks, anti-cheat / proxy gotchas).
---

# 2. VELOCITY · FOLIA · BEDROCK — DEEP DIVE (v2)

The base `paper-plugin-dev.md` references Folia briefly (§9.2) and does not cover Velocity
or Bedrock cross-play at all. This file is a complete reference for building plugins that
target the proxy layer (Velocity), the threaded server fork (Folia), and the cross-platform
audience (Bedrock via GeyserMC + Floodgate).

All Javadoc references and behavioural claims were checked against authoritative sources at
write time. Links are inline and consolidated in the [References](#2z-references) appendix.

---

## 2.0 WHEN TO USE WHICH

Quick triage:

| Goal | Where to put the plugin |
|---|---|
| Cross-network /msg, /tell, /find, /hub | Velocity (proxy) |
| Per-backend gameplay logic (skyblock, pvp arena) | Paper backend |
| Anti-bot, login throttle, IP filtering | Velocity preferred |
| Tab list, server-list ping (MOTD) | Velocity (one source of truth) |
| Performance scaling on a busy survival world | Folia |
| Bedrock parity for an existing Paper plugin | Floodgate hook on the backend |
| Form-driven UI (RPG menus) for Bedrock players | Cumulus on the backend |
| Both Paper and Folia from one JAR | Compat shim (§2.B.7) |

You can mix all three — a single network can run Velocity + Folia backends + Bedrock
cross-play. Plugin authors target each layer independently.

---

## 2.A VELOCITY PROXY PLUGIN DEVELOPMENT

Velocity is PaperMC's modern Minecraft proxy ([PaperMC/Velocity](https://github.com/PaperMC/Velocity)).
It runs in front of Paper backend servers and routes players between them. Plugins running on
Velocity see **every** player on the network, route them, intercept chat, register cross-network
commands, and modify the proxy ↔ backend protocol stream.

### 2.A.1 Project setup

```kotlin
// build.gradle.kts
plugins {
    java
    id("com.gradleup.shadow") version "8.3.5"
}

repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    compileOnly("com.velocitypowered:velocity-api:3.4.0-SNAPSHOT")
    annotationProcessor("com.velocitypowered:velocity-api:3.4.0-SNAPSHOT")
}

java { toolchain.languageVersion.set(JavaLanguageVersion.of(21)) }

tasks.shadowJar { archiveClassifier.set("") }
tasks.build { dependsOn("shadowJar") }
```

> **No `plugin.yml`.** Velocity uses an annotation processor that generates
> `velocity-plugin.json` from `@Plugin` on your main class. Always include the
> `annotationProcessor` line.

Project skeleton:

```
src/main/java/com/example/myproxy/
├── MyProxy.java               # Main class with @Plugin
├── command/                   # BrigadierCommand subclasses
├── listener/                  # @Subscribe handlers
├── messaging/                 # PluginMessage handlers
└── config/                    # Configuration loading
src/main/resources/
└── (no plugin.yml — generated)
```

### 2.A.2 Main class

```java
package com.example.myproxy;

import com.google.inject.Inject;
import com.velocitypowered.api.event.Subscribe;
import com.velocitypowered.api.event.proxy.ProxyInitializeEvent;
import com.velocitypowered.api.event.proxy.ProxyShutdownEvent;
import com.velocitypowered.api.plugin.Dependency;
import com.velocitypowered.api.plugin.Plugin;
import com.velocitypowered.api.plugin.annotation.DataDirectory;
import com.velocitypowered.api.proxy.ProxyServer;
import org.slf4j.Logger;

import java.nio.file.Path;

@Plugin(
    id = "myproxy",
    name = "MyProxy",
    version = "1.0.0",
    description = "Example Velocity plugin",
    authors = { "AuthorName" },
    dependencies = {
        @Dependency(id = "luckperms"),                       // Hard dep
        @Dependency(id = "papiproxybridge", optional = true) // Soft dep
    }
)
public class MyProxy {

    private final ProxyServer proxy;
    private final Logger logger;
    private final Path dataDir;

    @Inject
    public MyProxy(ProxyServer proxy, Logger logger, @DataDirectory Path dataDir) {
        this.proxy = proxy;
        this.logger = logger;
        this.dataDir = dataDir;
    }

    @Subscribe
    public void onInit(ProxyInitializeEvent event) {
        logger.info("MyProxy initialized");
        proxy.getEventManager().register(this, new ConnectionListener(this));
        registerCommands();
    }

    @Subscribe
    public void onShutdown(ProxyShutdownEvent event) {
        logger.info("MyProxy shutting down");
    }

    public ProxyServer proxy()  { return proxy; }
    public Logger      logger() { return logger; }
    public Path        dataDir(){ return dataDir; }

    private void registerCommands() { /* §2.A.6 */ }
}
```

**Key differences from Bukkit/Paper plugins:**
- **Guice DI**: services like `ProxyServer`, `Logger`, `@DataDirectory Path` are
  constructor-injected. There's no `JavaPlugin#getConfig()`; load config yourself from
  `dataDir`.
- Events use **`@Subscribe`**, not `@EventHandler`.
- No `plugin.yml` — metadata is in `@Plugin`.
- Logger is **SLF4J**, not JUL.
- **No main thread** — all API methods are thread-safe unless the Javadoc says
  otherwise. Don't assume concurrent listener invocations are serialized.

### 2.A.3 Event reference (verified against velocity-api 3.4.0)

```java
// Connection lifecycle
PreLoginEvent             // Before authentication. Disconnect bots/banned IPs here.
LoginEvent                // After auth, before backend connect. Set Velocity tab list here.
PostLoginEvent            // Player has joined the proxy.
ServerPreConnectEvent     // Player about to connect to a backend. Modify target server here.
ServerConnectedEvent      // Player switched to a backend.
DisconnectEvent           // Player left. event.getLoginStatus() tells you why.
GameProfileRequestEvent   // Modify the username/UUID before login finishes.

// Chat / commands
PlayerChatEvent                          // Player typed in chat (no backend yet routed).
CommandExecuteEvent                      // Any command, intercept BEFORE routing.
PlayerAvailableCommandsEvent             // Customize tab-complete tree visible to a player.

// Network / data
PluginMessageEvent                       // Plugin messaging (BungeeCord channel etc.).
ProxyPingEvent                           // Server list ping (MOTD customization).
ResourcePackResponseEvent                // Pack accept/decline.
KickedFromServerEvent                    // Backend kicked the player. Choose: reconnect or pass through.
ServerResourcePackSendEvent              // Backend asks proxy to send a resource pack.

// Internal
ProxyInitializeEvent / ProxyShutdownEvent / ProxyReloadEvent
ListenerBoundEvent / ListenerCloseEvent
```

**Async event handlers (long-running work):**

```java
@Subscribe(order = PostOrder.NORMAL, async = true)
public EventTask onPreLogin(PreLoginEvent event) {
    return EventTask.async(() -> {
        if (banDatabase.isBanned(event.getUsername())) {
            event.setResult(PreLoginComponentResult.denied(
                Component.text("You are banned.", NamedTextColor.RED)));
        }
    });
}
```

`EventTask.async()` is Velocity's analogue of returning a `CompletableFuture` — the proxy
won't proceed past the event until the task completes. Use this for DB / HTTP lookups in
the login pipeline.

### 2.A.4 Routing players between backends

```java
@Subscribe
public void onCommand(CommandExecuteEvent event) {
    if (!(event.getCommandSource() instanceof Player player)) return;
    if (!event.getCommand().equalsIgnoreCase("lobby")) return;

    proxy.getServer("lobby").ifPresent(server -> {
        player.createConnectionRequest(server)
            .connect()
            .thenAccept(result -> {
                if (!result.isSuccessful()) {
                    player.sendMessage(Component.text("Lobby is offline.", NamedTextColor.RED));
                }
            });
    });
    event.setResult(CommandExecuteEvent.CommandResult.handled()); // Don't route to backend
}
```

`createConnectionRequest(server)` returns a future; `.fireAndForget()` skips waiting for
the result if you don't care about the outcome.

### 2.A.5 Cross-network broadcast / messaging

```java
// Send a message to every player on every backend:
public void broadcast(Component component) {
    proxy.getAllPlayers().forEach(p -> p.sendMessage(component));
}

// Find a player anywhere on the network:
Optional<Player> player = proxy.getPlayer("Notch");

// List players on a specific backend:
proxy.getServer("survival").ifPresent(rs ->
    rs.getPlayersConnected().forEach(p -> p.sendMessage(Component.text("Hi"))));
```

For high-volume cross-server messaging (chat plugins, party plugins), don't iterate
players in tight loops — use plugin messaging (§2.A.8) or Redis pub/sub (§2.A.9) instead.

### 2.A.6 Brigadier on Velocity (with redirects + custom args)

Velocity exposes Brigadier via [`BrigadierCommand`](https://docs.papermc.io/velocity/dev/command-api/).

```java
import com.mojang.brigadier.Command;
import com.mojang.brigadier.arguments.StringArgumentType;
import com.mojang.brigadier.builder.LiteralArgumentBuilder;
import com.mojang.brigadier.builder.RequiredArgumentBuilder;
import com.mojang.brigadier.tree.LiteralCommandNode;
import com.velocitypowered.api.command.BrigadierCommand;
import com.velocitypowered.api.command.CommandSource;
import com.velocitypowered.api.proxy.Player;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;

private void registerCommands() {
    LiteralCommandNode<CommandSource> hub = LiteralArgumentBuilder.<CommandSource>literal("hub")
        .requires(src -> src.hasPermission("myproxy.hub"))
        .executes(ctx -> {
            if (ctx.getSource() instanceof Player player) {
                proxy.getServer("hub").ifPresent(rs ->
                    player.createConnectionRequest(rs).fireAndForget());
            }
            return Command.SINGLE_SUCCESS;
        })
        .build();

    proxy.getCommandManager().register(
        proxy.getCommandManager().metaBuilder("hub")
            .aliases("lobby", "leave")
            .plugin(this)
            .build(),
        new BrigadierCommand(hub)
    );

    // Command with arg + redirect (a /go alias that forwards to the same node):
    LiteralCommandNode<CommandSource> server = LiteralArgumentBuilder.<CommandSource>literal("server")
        .then(RequiredArgumentBuilder.<CommandSource, String>argument("name", StringArgumentType.word())
            .suggests((ctx, builder) -> {
                proxy.getAllServers().forEach(rs -> builder.suggest(rs.getServerInfo().getName()));
                return builder.buildFuture();
            })
            .executes(ctx -> {
                String name = StringArgumentType.getString(ctx, "name");
                Player player = (Player) ctx.getSource();
                proxy.getServer(name).ifPresentOrElse(
                    rs -> player.createConnectionRequest(rs).fireAndForget(),
                    () -> player.sendMessage(Component.text("Unknown server: " + name, NamedTextColor.RED))
                );
                return Command.SINGLE_SUCCESS;
            })
        )
        .build();

    LiteralCommandNode<CommandSource> goAlias = LiteralArgumentBuilder.<CommandSource>literal("go")
        .redirect(server)   // /go <name> behaves identically to /server <name>
        .build();

    proxy.getCommandManager().register(
        proxy.getCommandManager().metaBuilder("server").plugin(this).build(),
        new BrigadierCommand(server)
    );
    proxy.getCommandManager().register(
        proxy.getCommandManager().metaBuilder("go").plugin(this).build(),
        new BrigadierCommand(goAlias)
    );
}
```

**Brigadier redirect tip:** prefer `.redirect(node)` over re-declaring the entire subtree.
You get help-text consistency, single-source-of-truth permissions, and the client-side tab
completion mirrors automatically.

**Forwarding to backend:** if you don't `setResult(handled())`, the command falls through to
the player's current backend server. Useful for `/say`, `/me`, etc. that should remain
backend-local.

### 2.A.7 Modern Forwarding (Velocity ↔ Paper)

Per [PaperMC docs on player-information-forwarding](https://docs.papermc.io/velocity/player-information-forwarding):

**On Velocity (`velocity.toml`):**
```toml
player-info-forwarding-mode = "modern"
forwarding-secret-file = "forwarding.secret"

# Optional but strongly recommended on a public proxy:
prevent-client-proxy-connections = false
ping-passthrough = "DISABLED"
enable-player-address-logging = true
```

**On each Paper backend (`config/paper-global.yml`):**
```yaml
proxies:
  velocity:
    enabled: true
    online-mode: true               # MUST be true with modern forwarding
    secret: '<contents of forwarding.secret>'
```

**On each Paper backend (`server.properties`):**
```properties
online-mode=false
```

**Why these settings:** modern forwarding embeds a signed token in the login packet that
the backend verifies using the shared secret. The backend trusts the proxy's
authentication, so `online-mode=false` on the backend is **safe** — but only because
`online-mode=true` is set inside `paper-global.yml` (which enables signature
verification). Misconfiguration here is the #1 cause of "I can join with any name"
cracked-account exploits on networks that *intended* to be online.

**Firewall rule:** the backend ports must be reachable only from the proxy's IP. Use
`iptables` / your host's firewall to enforce this — Velocity itself can't.

### 2.A.8 Plugin Messaging vs Velocity Channels

The legacy BungeeCord `Connect`/`PlayerCount`/`MessageRaw` channel still works for cross-
version compatibility. For new plugins use a custom Velocity-specific channel and let
Velocity treat it as opaque bytes:

```java
// On Paper backend - register & send
public void onEnable() {
    getServer().getMessenger().registerOutgoingPluginChannel(this, "myproxy:command");
}
public void send(Player player, String payload) {
    ByteArrayDataOutput out = ByteStreams.newDataOutput();
    out.writeUTF(payload);
    player.sendPluginMessage(this, "myproxy:command", out.toByteArray());
}

// On Velocity - register the channel at startup so backends can see it:
@Subscribe
public void onInit(ProxyInitializeEvent event) {
    proxy.getChannelRegistrar().register(
        MinecraftChannelIdentifier.from("myproxy:command"));
}

// On Velocity - receive
@Subscribe
public void onMessage(PluginMessageEvent event) {
    if (!event.getIdentifier().getId().equals("myproxy:command")) return;
    if (!(event.getSource() instanceof ServerConnection conn)) return;
    ByteArrayDataInput in = ByteStreams.newDataInput(event.getData());
    String payload = in.readUTF();
    // ...
    event.setResult(PluginMessageEvent.ForwardResult.handled()); // Don't forward to other backends
}
```

**Limitation:** plugin messaging requires at least one online player whose connection
carries the message. For player-independent messaging (e.g. server-startup announcements),
use the proxy itself as the message bus, or layer a Redis bus on top.

### 2.A.9 Cross-server messaging via Redis (when plugin messaging isn't enough)

For high-volume real-time messaging, party systems, friends online, network-wide chat:
plugin messaging is too constrained. Add Redis pub/sub on top.

Pattern (works for Paper backends and Velocity proxy alike):

```java
// On every node (proxy + backends), one shared Redis client:
private final JedisPool pool = new JedisPool("redis.example.com", 6379);

// Subscriber (each node):
new Thread(() -> {
    try (Jedis j = pool.getResource()) {
        j.subscribe(new JedisPubSub() {
            @Override public void onMessage(String channel, String message) {
                // dispatch to local players
                handle(channel, message);
            }
        }, "network:chat", "network:friends");
    }
}, "redis-sub").start();

// Publisher (any node):
public void broadcastChat(String json) {
    try (Jedis j = pool.getResource()) {
        j.publish("network:chat", json);
    }
}
```

**Why Redis:**
- Sub-millisecond latency between nodes
- Survives single-node restarts (other nodes keep talking)
- Decouples backends from the proxy (backends can talk directly)

**Don't use for:** stateful data — Redis pub/sub doesn't persist. Use Redis Streams or
add a database for that. See [Redis pub/sub patterns](https://redis.io/docs/manual/pubsub/)
for production tuning.

### 2.A.10 Tab-list / Server-list ping (MOTD)

```java
@Subscribe
public void onPing(ProxyPingEvent event) {
    ServerPing.Builder builder = event.getPing().asBuilder();
    builder.description(MiniMessage.miniMessage().deserialize(
        "<gradient:gold:red>My Network</gradient><newline><gray>Survival - Skyblock - Creative"));
    builder.maximumPlayers(1000);
    builder.onlinePlayers(proxy.getPlayerCount());
    builder.samplePlayers(
        new ServerPing.SamplePlayer("§a* Use /servers", UUID.randomUUID()),
        new ServerPing.SamplePlayer("§7play.example.net", UUID.randomUUID())
    );
    Path icon = dataDir.resolve("server-icon.png");
    if (Files.exists(icon)) {
        try { builder.favicon(Favicon.create(icon)); }
        catch (IOException e) { logger.warn("Failed to load favicon", e); }
    }
    event.setPing(builder.build());
}
```

The favicon must be a **64×64 PNG**. Cache the `Favicon` instance — converting on every
ping is expensive at scale.

### 2.A.11 Velocity-side scheduler

```java
proxy.getScheduler()
    .buildTask(this, () -> logger.info("Tick"))
    .delay(5, TimeUnit.SECONDS)
    .repeat(30, TimeUnit.SECONDS)
    .schedule();

// Cancel:
ScheduledTask task = proxy.getScheduler().buildTask(this, () -> {}).schedule();
task.cancel();
```

Velocity is fully async — no main thread, no Folia-style regions. Just don't block the
**network I/O threads** (Netty event loops). Heavy work → schedule on Velocity's executor.

### 2.A.12 LuckPerms on Velocity

```kotlin
// build.gradle.kts
dependencies {
    compileOnly("net.luckperms:api:5.4")
}
```

```java
import net.luckperms.api.LuckPerms;
import net.luckperms.api.LuckPermsProvider;
import net.luckperms.api.model.user.User;

@Subscribe
public void onPostLogin(PostLoginEvent event) {
    LuckPerms api = LuckPermsProvider.get();
    User user = api.getUserManager().getUser(event.getPlayer().getUniqueId());
    if (user != null) {
        String prefix = user.getCachedData().getMetaData().getPrefix();
        // ...
    }
}
```

LuckPerms ships proxy-aware: setting a permission on Velocity propagates to every backend
that shares the database/messaging layer. This is the ecosystem standard for cross-network
permissions.

### 2.A.13 Common cross-network plugins (reference for hooks)

| Plugin | Purpose |
|---|---|
| LuckPerms (Velocity) | Network-wide permissions |
| ViaVersion / ProtocolizeAPI | Multi-version protocol support |
| PAPIProxyBridge | PlaceholderAPI on the proxy |
| LibreLogin | Network-wide login/auth |
| Velocitab | Custom tab list |
| Geyser-Velocity / Floodgate-Velocity | Bedrock cross-play |
| Plan / SimpleCloud | Analytics / orchestration |
| OskarsMC/message | Cross-server private messaging |

### 2.A.14 Velocity gotchas

- **`proxy.getAllPlayers()` returns a snapshot** — players can join/leave during iteration.
  Tolerate that.
- **Don't block in event handlers** — use `EventTask.async()` for DB/HTTP work.
- **`@Subscribe` order**: `PostOrder.FIRST/EARLY/NORMAL/LATE/LAST`. There's no
  `MONITOR` like Bukkit; use `LAST` for read-only logging.
- **Velocity command signing** ([Issue #369](https://github.com/PaperMC/Velocity/issues/369))
  used to forward unhandled commands to backends with stripped args. Resolved in 3.4+ but
  test cross-server commands explicitly.
- **Plugin classloaders are isolated** like Paper plugins — depend on the LuckPerms API,
  don't call internals.

---

## 2.B FOLIA — REGIONISED MULTITHREADING

[Folia](https://github.com/PaperMC/Folia) is PaperMC's experimental fork that splits a
single world into independent **regions** (groups of nearby chunks), each ticked by its own
thread. There is **no main thread**; thousands of region threads tick in parallel.

### 2.B.1 What this changes for plugin authors

| Concept | Paper | Folia |
|---|---|---|
| Main thread | Yes — `Bukkit.isPrimaryThread()` | No — concept does not exist |
| `BukkitScheduler` | Works | Throws `UnsupportedOperationException` |
| Calling Bukkit API | OK from main thread | OK only from the **owning region thread** of the relevant entity/location |
| `Player#teleport` | Sync | Async (returns `CompletableFuture`) |
| Async events | Same | Same |
| `EntityMoveEvent` etc. | Fire on main | Fire on the region thread that owns the entity |
| Iterating online players | Snapshot via copy | Snapshot via copy |
| Server shutdown | One thread | All region threads, in parallel |

### 2.B.2 Required plugin metadata

```yaml
# plugin.yml or paper-plugin.yml
folia-supported: true
```

**Without this flag, Folia refuses to load the plugin.** This is intentional — most legacy
plugins are unsafe on Folia and would silently corrupt state.

### 2.B.3 The four schedulers

Per [Folia scheduler Javadoc](https://jd.papermc.io/folia/):

```java
import io.papermc.paper.threadedregions.scheduler.*;

// 1) Global region scheduler — for things not tied to any entity/location
//    (server-wide tasks, world iteration, gamemode-wide events)
GlobalRegionScheduler global = Bukkit.getGlobalRegionScheduler();
global.run(plugin, task -> { /* runs on the global tick thread */ });
global.runDelayed(plugin, task -> { /* ... */ }, 100L);
ScheduledTask repeat = global.runAtFixedRate(plugin, task -> { /* ... */ }, 0L, 20L);
repeat.cancel();

// 2) Region scheduler — for tasks at a specific Location/chunk
//    Use this for block/world manipulation
RegionScheduler region = Bukkit.getRegionScheduler();
region.run(plugin, world, chunkX, chunkZ, task -> {
    Block block = world.getBlockAt(chunkX << 4, 64, chunkZ << 4);
    block.setType(Material.DIAMOND_BLOCK);
});

// 3) Entity scheduler — for tasks involving a specific entity
//    The scheduler handles entity teleports between regions transparently
EntityScheduler ent = entity.getScheduler();
ent.run(plugin, task -> {
    if (entity.isValid()) entity.setVelocity(new Vector(0, 1, 0));
}, () -> {
    // Retired callback: entity got removed before task ran
    plugin.getLogger().info("Entity gone");
});

// 4) Async scheduler — for non-game tasks (DB, HTTP, file I/O)
AsyncScheduler async = Bukkit.getAsyncScheduler();
async.runNow(plugin, task -> {
    String result = httpClient.get("https://api.example.com");
    // Hop back to a region for game effects:
    entity.getScheduler().run(plugin, t -> entity.sendMessage(result), null);
});
async.runDelayed(plugin, task -> { }, 5, TimeUnit.SECONDS);
async.runAtFixedRate(plugin, task -> { }, 0, 1, TimeUnit.MINUTES);
```

**`ScheduledTask`** is the handle returned by every scheduler call. Save it if you need
to cancel; otherwise let it run.

### 2.B.4 Threading invariants on Folia

```
+=============================================================+
|  REGION THREAD (the thread currently ticking entity X):     |
|   - All API calls about X (teleport, damage, inventory)     |
|   - All API calls about blocks/chunks inside X's region     |
|   - CANNOT touch entities or blocks in other regions        |
+=============================================================+
|  GLOBAL THREAD:                                             |
|   - World list, weather, time, gamerule, scoreboard manager |
|   - Player join/quit handling                               |
|   - Cannot touch a specific entity's state directly         |
+=============================================================+
|  ASYNC THREAD:                                              |
|   - DB / HTTP / file I/O                                    |
|   - CANNOT touch any entity or block                        |
+=============================================================+
```

> **Verification rule:** if your code calls a Bukkit method that mutates state, ask: *what
> region owns that state, and am I currently ticking on that region?* If you can't answer,
> hop schedulers.

### 2.B.5 Common Bukkit → Folia migration patterns

```java
// BEFORE (Paper):
Bukkit.getScheduler().runTaskLater(plugin, () -> player.teleport(loc), 20L);

// AFTER (Folia):
player.getScheduler().runDelayed(plugin, t -> player.teleportAsync(loc), null, 20L);

// BEFORE (Paper):
new BukkitRunnable() {
    @Override public void run() { /* update HUD */ }
}.runTaskTimer(plugin, 0L, 20L);

// AFTER (Folia, per-player):
for (Player p : Bukkit.getOnlinePlayers()) {
    p.getScheduler().runAtFixedRate(plugin, t -> updateHud(p), null, 1L, 20L);
}

// BEFORE (Paper, iterating all entities in a world):
for (Entity e : world.getEntities()) e.remove();

// AFTER (Folia — must run on the region that owns each entity):
for (Entity e : world.getEntities()) {
    e.getScheduler().run(plugin, t -> e.remove(), null);
}

// BEFORE (Paper - shared ConcurrentHashMap is unnecessary):
Map<UUID, Long> cooldowns = new HashMap<>();

// AFTER (Folia - any cross-region read MUST be thread-safe):
Map<UUID, Long> cooldowns = new ConcurrentHashMap<>();
```

### 2.B.6 Teleporting between regions

Folia teleport is **always** async because the destination region might not even be loaded:

```java
player.teleportAsync(otherWorldLoc).thenAccept(success -> {
    if (success) {
        // We are now back on the destination region thread for the player:
        player.getScheduler().run(plugin, t ->
            player.sendMessage(Component.text("Teleported")), null);
    }
});
```

Don't depend on the player's `Location` immediately after the call — the actual move
happens later. If you cached the player's region thread, that cache is now stale.

### 2.B.7 Folia compatibility shim (supporting both Paper + Folia)

Per [Supporting Paper and Folia](https://docs.papermc.io/paper/dev/folia-support/):

```java
public final class Schedulers {
    private static final boolean FOLIA = detectFolia();

    private static boolean detectFolia() {
        try {
            Class.forName("io.papermc.paper.threadedregions.RegionizedServer");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    public static void runForEntity(Plugin plugin, Entity entity, Runnable task) {
        if (FOLIA) entity.getScheduler().run(plugin, t -> task.run(), null);
        else        Bukkit.getScheduler().runTask(plugin, task);
    }

    public static void runAtLocation(Plugin plugin, Location loc, Runnable task) {
        if (FOLIA) Bukkit.getRegionScheduler().run(plugin, loc, t -> task.run());
        else       Bukkit.getScheduler().runTask(plugin, task);
    }

    public static void runGlobal(Plugin plugin, Runnable task) {
        if (FOLIA) Bukkit.getGlobalRegionScheduler().run(plugin, t -> task.run());
        else       Bukkit.getScheduler().runTask(plugin, task);
    }

    public static void runAsync(Plugin plugin, Runnable task) {
        if (FOLIA) Bukkit.getAsyncScheduler().runNow(plugin, t -> task.run());
        else       Bukkit.getScheduler().runTaskAsynchronously(plugin, task);
    }

    public static void runLater(Plugin plugin, Entity entity, Runnable task, long ticks) {
        if (FOLIA) entity.getScheduler().runDelayed(plugin, t -> task.run(), null, ticks);
        else        Bukkit.getScheduler().runTaskLater(plugin, task, ticks);
    }

    public static boolean isFolia() { return FOLIA; }
}
```

Use this wrapper everywhere instead of `Bukkit.getScheduler()` directly. One JAR runs on
both. The original code calls Bukkit's API; the shim picks the right scheduler.

### 2.B.8 Folia chunk loading

Folia's chunk loader is region-aware. Plugins that force-load chunks must use Paper's
chunk-ticket API rather than `chunk.load()` synchronously:

```java
// Add a chunk ticket (keeps chunk loaded, region-thread safe):
world.addPluginChunkTicket(chunkX, chunkZ, plugin);
// Remove it:
world.removePluginChunkTicket(chunkX, chunkZ, plugin);
```

Async chunk loading on Folia:

```java
world.getChunkAtAsync(x, z).thenAccept(chunk -> {
    // Now back on the region thread that owns the chunk
    Bukkit.getRegionScheduler().run(plugin, world, x, z, task -> {
        // Mutate chunk safely
    });
});
```

Don't iterate `world.getLoadedChunks()` looking for things — the result is a snapshot and
chunks may be ticked by different threads. Use the region scheduler at known coordinates.

**Chunk events on Folia:** `ChunkLoadEvent` and `ChunkUnloadEvent` fire on the **region
thread that owns the loading/unloading chunk**, not on a global thread. Listeners can
safely access the chunk's blocks/entities without hopping. But don't iterate *other*
chunks from inside that handler — they may belong to a different region.

### 2.B.9 Concurrency primitives that work on Folia

Pick the right tool. Many plugins lazily reach for `synchronized` blocks; that scales
poorly when 32 region threads are all contending on one lock.

| Need | Tool |
|---|---|
| Per-player cache (UUID → state) | `ConcurrentHashMap<UUID, T>` |
| Per-region count / aggregate | `LongAdder` per region key |
| Producer/consumer queue | `ConcurrentLinkedQueue` (lock-free) |
| Read-mostly structure | `CopyOnWriteArrayList` (small) or `AtomicReference<ImmutableList<T>>` |
| Snapshot-then-mutate | `volatile` reference + immutable copies |
| Counter | `AtomicLong` / `LongAdder` (LongAdder for high contention) |

Avoid:
- `Collections.synchronizedXxx` — single global lock, defeats Folia's parallelism
- `synchronized` on `Bukkit.class`, world, entity — never fully safe across regions
- Field accesses without `volatile` when read by multiple threads

### 2.B.10 Folia's official "do / don't" list

**Do:**
- Use `teleportAsync` everywhere.
- Hop into the right scheduler before touching state.
- Use thread-safe collections for shared state.
- Treat `Bukkit.getOnlinePlayers()` as a snapshot — players may join/leave mid-iteration.
- Defensively check `entity.isValid()` and `player.isOnline()` inside scheduled callbacks.

**Don't:**
- Don't synchronize on world/entity instances expecting a global lock.
- Don't iterate one world's entities to apply state to entities in *another* region.
- Don't rely on `Bukkit.isPrimaryThread()` — it always returns false.
- Don't write tick-counter loops — use `runAtFixedRate`.
- Don't share mutable state between regions without explicit locking.

### 2.B.11 Performance & limitations

Per [Folia FAQ](https://docs.papermc.io/folia/faq/):

- Folia shines on **player-spread servers** (skyblock, faction, large survival).
- Folia loses on **player-clustered servers** (a single PvP arena = one thread doing all the work).
- Recommended hardware: at least 16 **cores** (not threads).
- Some plugins (WorldEdit, dynmap, etc.) need Folia-specific forks.
- Folia can run **with** the Paper API but expects more careful code — the performance gain
  is proportional to player spread.

### 2.B.12 Debugging a Folia crash

Region-thread crashes leave a stack trace pointing at "RegionizedTaskQueueWorker" or
similar. Look for:

1. **Cross-region access** — typically `e.getLocation().getWorld().getEntities()` from a
   different entity's region. Fix: hop to the right scheduler.
2. **Mutating state from async** — you called Bukkit API from `runAsync`. Fix: hop back.
3. **Stale reference** — entity removed between scheduling and execution. Fix: check
   `entity.isValid()` first thing in the lambda.
4. **`UnsupportedOperationException` from `BukkitScheduler`** — legacy scheduler call
   slipped through. Fix: replace with `Schedulers.*` (§2.B.7).

---

## 2.C BEDROCK CROSS-PLAY (GeyserMC + Floodgate)

Bedrock players cannot join a Java server natively. **Geyser** is a translator (Java
protocol ↔ Bedrock protocol) and **Floodgate** is the auth bridge — it gives Bedrock
players a Java-style account on your server without a Mojang license.

### 2.C.1 Setup recap (server-side)

| Plugin | Place on | Purpose |
|---|---|---|
| Geyser-Spigot or Geyser-Velocity | proxy or backend | Protocol translation |
| Floodgate | proxy or backend | Bedrock auth |

You generally install both on the proxy (Velocity) when running a network, or both on each
Paper backend when running standalone. Mixed setups (Geyser on proxy + Floodgate on
backend) are valid but require correct UUID forwarding.

### 2.C.2 Floodgate UUID format

Bedrock players don't have Java UUIDs. Floodgate generates **Floodgate UUIDs** with a
fixed-prefix pattern, derived from the Bedrock XUID. Per [Floodgate FAQ](https://wiki.geysermc.org/floodgate/faq/):

```
Format:  00000000-0000-0000-XXXX-XXXXXXXXXXXX
                              └────── 16 hex chars from XUID ──┘
```

Examples:

```
00000000-0000-0000-0009-01f64f65c7c3   # a real Bedrock player UUID
00000000-0000-0000-0009-01ee9e50efff   # another Bedrock UUID
```

The first 16 hex chars are always zeros, which is how you visually identify a Bedrock
player without calling the API. **Permission systems and databases must accept this format
as a valid `UUID`** — don't validate against a Mojang-style version-4 random UUID layout.

```java
public static boolean looksLikeFloodgate(UUID uuid) {
    return (uuid.getMostSignificantBits() & 0xFFFFFFFFFFFFFF00L) == 0L;
}
```

### 2.C.3 Detecting Bedrock players (Floodgate API)

```kotlin
// build.gradle.kts
repositories {
    maven("https://repo.opencollab.dev/main/")
}
dependencies {
    compileOnly("org.geysermc.floodgate:api:2.2.4-SNAPSHOT")
    compileOnly("org.geysermc.cumulus:Cumulus:1.5")
    compileOnly("org.geysermc.geyser:api:2.4.2")
}
```

```yaml
# plugin.yml or paper-plugin.yml
softdepend: [floodgate, Geyser-Spigot]
```

```java
import org.geysermc.floodgate.api.FloodgateApi;
import org.geysermc.floodgate.api.player.FloodgatePlayer;

public boolean isBedrock(Player player) {
    return FloodgateApi.getInstance().isFloodgatePlayer(player.getUniqueId());
}

FloodgatePlayer fp = FloodgateApi.getInstance().getPlayer(player.getUniqueId());
if (fp != null) {
    String xuid             = fp.getXuid();
    String javaUsername     = fp.getJavaUsername();
    String bedrockUsername  = fp.getUsername();
    DeviceOs device         = fp.getDeviceOs();          // ANDROID, IOS, WINDOWS, NX (Switch), XBOX, PS4, etc.
    InputMode input         = fp.getInputMode();          // KEYBOARD_MOUSE, TOUCH, GAMEPAD, MOTION_CONTROLLER
    UiProfile ui            = fp.getUiProfile();          // CLASSIC or POCKET
    String langCode         = fp.getLanguageCode();       // e.g. "en_US"
    boolean linked          = fp.isLinked();              // Linked to a Java account?
}
```

**Bedrock username prefix:** Floodgate prefixes Bedrock usernames with `.` by default
(configurable in `config.yml`). So `Steve` from Bedrock joins as `.Steve` to avoid
collisions. Always treat `Player#getName()` as opaque; never assume a fixed format.

**Linked accounts:** if a Bedrock player has linked their Bedrock account to a Java
account (via Floodgate's `/linkaccount` flow), they join with their **Java UUID**, not
the Floodgate UUID. `fp.isLinked()` is `true`, and `fp.getJavaUuid()` returns the linked
Java UUID. This means a Bedrock player can have a normal Mojang skin and Java permissions
inherit correctly. Always check `isLinked()` before assuming the UUID format.

### 2.C.4 Geyser API (alternative / supplemental)

```java
import org.geysermc.geyser.api.GeyserApi;
import org.geysermc.geyser.api.connection.GeyserConnection;

GeyserConnection conn = GeyserApi.api().connectionByUuid(player.getUniqueId());
if (conn != null) {
    String version       = conn.bedrockVersion();
    int protocolVersion  = conn.protocolVersion();
}
```

Use **Floodgate API** when you only need to detect/identify Bedrock players. Use **Geyser
API** when you need protocol-level info, custom items/blocks, or want to render Bedrock-only
UI (forms, see §2.C.5).

### 2.C.5 Cumulus Forms (native Bedrock UI)

[Cumulus](https://github.com/GeyserMC/Cumulus) is the form library shared by Geyser and
Floodgate. It lets you display Bedrock-native modal popups, form selectors, and structured
input forms. These render on Bedrock as proper mobile-style UIs, not as fake chat windows.

```java
import org.geysermc.cumulus.form.SimpleForm;
import org.geysermc.cumulus.form.ModalForm;
import org.geysermc.cumulus.form.CustomForm;
import org.geysermc.cumulus.component.*;
import org.geysermc.cumulus.util.FormImage;
import org.geysermc.floodgate.api.FloodgateApi;

// SimpleForm — a vertical list of buttons
SimpleForm form = SimpleForm.builder()
    .title("Server Menu")
    .content("Choose a destination:")
    .button("Survival", FormImage.Type.URL, "https://example.com/icons/survival.png")
    .button("Skyblock")
    .button("Creative")
    .button("Cancel")
    .validResultHandler(response -> {
        int clicked = response.clickedButtonId();
        switch (clicked) {
            case 0 -> teleportToServer(player, "survival");
            case 1 -> teleportToServer(player, "skyblock");
            case 2 -> teleportToServer(player, "creative");
        }
    })
    .closedOrInvalidResultHandler(() -> player.sendMessage(Component.text("Form closed")))
    .build();

FloodgateApi.getInstance().sendForm(player.getUniqueId(), form);

// ModalForm — yes/no confirmation
ModalForm confirm = ModalForm.builder()
    .title("Are you sure?")
    .content("This will delete your home.")
    .button1("Confirm")
    .button2("Cancel")
    .validResultHandler(r -> {
        if (r.clickedFirst()) deleteHome(player);
    })
    .build();

// CustomForm — mixed inputs
CustomForm customForm = CustomForm.builder()
    .title("Settings")
    .label("Configure your preferences:")
    .toggle("Enable particles", true)
    .slider("Render distance", 4, 32, 1, 12)
    .stepSlider("Difficulty", 0, "Easy", "Normal", "Hard")
    .input("Display name", "Your name", player.getName())
    .dropdown("Language", 0, "English", "Spanish", "Japanese")
    .validResultHandler(response -> {
        boolean particles = response.asToggle(1);
        int rd            = (int) response.asSlider(2);
        int diffIndex     = response.asStepSlider(3);
        String name       = response.asInput(4);
        int langIndex     = response.asDropdown(5);
        savePrefs(player, particles, rd, diffIndex, name, langIndex);
    })
    .invalidResultHandler((response, error) -> {
        // Triggered if the form data is malformed (rare; usually a Geyser bug)
        plugin.getLogger().warning("Bedrock form returned invalid data: " + error);
    })
    .closedResultHandler(() -> {
        // Player clicked the X / pressed back
        player.sendMessage(Component.text("Settings cancelled"));
    })
    .build();

FloodgateApi.getInstance().sendForm(player.getUniqueId(), customForm);
```

**Sending forms:** `sendForm` returns `boolean` (sent successfully?) on Floodgate; check
the return value if you need to retry. Some Bedrock clients (older Switch/PS4 builds)
silently drop forms — always include a chat-fallback timeout:

```java
boolean sent = FloodgateApi.getInstance().sendForm(player.getUniqueId(), form);
if (!sent) {
    player.sendMessage(Component.text("Couldn't open menu — type /menu in chat."));
}
```

**Form result handlers:**
- `validResultHandler` — form submitted with valid data
- `closedResultHandler` — player cancelled
- `invalidResultHandler` — form data couldn't be parsed (rare)
- `closedOrInvalidResultHandler` — convenience for "anything other than valid"

Form indices in `CustomForm` are **0-based** but the *first slot is whatever you added
first* — labels count as a slot. Use named local variables to avoid index drift.

### 2.C.6 Java fallback for cross-platform UIs

Always provide a chest-GUI / chat fallback so Java players can use the same feature:

```java
public void openMenu(Player player) {
    if (FloodgateApi.getInstance().isFloodgatePlayer(player.getUniqueId())) {
        sendBedrockForm(player);
    } else {
        new MenuGUI().open(player);   // Chest GUI for Java players
    }
}
```

Better: build a single abstraction that picks the right renderer based on the player. See
the reference implementation pattern in `12-reference-implementation.md` (planned).

### 2.C.7 Custom items and custom blocks (Geyser mappings)

For plugins that ship custom items via resource pack + CMD, Bedrock players see the wrong
texture by default. Geyser has a [custom items mapping system](https://wiki.geysermc.org/geyser/custom-items/):

**Option A: JSON mappings** — drop a file in `Geyser/custom_mappings/`:

```json
{
  "format_version": "1",
  "items": {
    "minecraft:diamond": [
      {
        "name": "lightsaber",
        "display_name": "Lightsaber",
        "icon": "lightsaber",
        "custom_model_data": 1001,
        "allow_offhand": true
      }
    ]
  }
}
```

**Option B: Geyser API extension** — a Geyser extension that registers items
programmatically:

```java
import org.geysermc.geyser.api.event.lifecycle.GeyserDefineCustomItemsEvent;
import org.geysermc.geyser.api.item.custom.CustomItemData;
import org.geysermc.geyser.api.item.custom.CustomItemOptions;

public class MyExtension implements Extension {
    @Subscribe
    public void onItems(GeyserDefineCustomItemsEvent event) {
        event.register("minecraft:diamond",
            CustomItemData.builder()
                .name("lightsaber")
                .displayName("Lightsaber")
                .icon("lightsaber")
                .customItemOptions(CustomItemOptions.builder()
                    .customModelData(1001)
                    .build())
                .build());
    }
}
```

For [custom blocks](https://wiki.geysermc.org/geyser/custom-blocks/) the same pattern
applies — drop a JSON in `custom_mappings/` or use the Geyser extension API.

**Texture pipeline:** Geyser does **not** convert Java resource packs to Bedrock packs.
You must ship a separate Bedrock pack and configure it via Geyser's
`config.yml` `remote.allow-resource-pack-loading: true`.

### 2.C.8 Skin upload pipeline

Per [Floodgate skin features](https://wiki.geysermc.org/floodgate/features/):

- Bedrock skins use a different format than Java's MineSkin signed-textures.
- Floodgate's "Global API" service converts Bedrock skins → Java skin textures and uploads
  them to Mojang servers so they appear correctly to Java players.
- The conversion queue can lag during peak hours; players sometimes appear as Steve/Alex
  for several minutes after first join.

In code, you can fetch the converted skin via the [Global API](https://wiki.geysermc.org/geyser/global-api/):

```
GET https://api.geysermc.org/v2/skin/{xuid}
→ { "texture_id": "...", "value": "...", "signature": "..." }
```

The `value` and `signature` are MineSkin-formatted and can be applied to a player head's
`PROFILE` Data Component or to Floodgate's player skin via the API. Use this for player-
head displays of Bedrock players in shops, leaderboards, etc.

### 2.C.9 Bedrock-specific gotchas

- **Texture / model differences:** Bedrock auto-converts most Java models but with quality
  loss. Test custom items on a real Bedrock client, not just Geyser-Standalone.
- **Item ID gaps:** some Java items don't exist on Bedrock (specific banner patterns, etc.).
  Geyser substitutes them.
- **Inventory layouts:** chest GUIs work fine; anvil/loom/grindstone GUIs sometimes glitch.
- **No client-side mods:** Bedrock players can't run client mods. Anti-cheat false positives
  for "modded clients" must whitelist Floodgate UUIDs (see §2.C.2).
- **Movement subtleties:** Bedrock physics differ subtly. Don't write reach-checks that fail
  Bedrock players based on tight Java bounding boxes.
- **Chat encoding:** the client/server character set differs; emoji and unusual whitespace
  can be dropped.
- **Resource pack acceptance UX:** Bedrock shows a different prompt than Java. Test the
  refusal/exit flow on both.

### 2.C.10 Permissions on Bedrock players

LuckPerms, Vault, and most permission plugins handle Floodgate UUIDs transparently — they
just look like any other UUID. To whitelist a Bedrock player by UUID, use their Floodgate
UUID (start with the 16-zero prefix), not their raw XUID.

`fwhitelist` is Floodgate's own command; pure Java whitelist plugins also work as long as
they accept the UUID format.

### 2.C.11 Useful event hooks

```java
// Floodgate's own events:
@EventHandler
public void onSkinUpdate(SkinApplyEvent event) {
    // Fired when Floodgate finishes converting & uploading a Bedrock skin
}

// Bedrock-aware Bukkit events:
@EventHandler
public void onJoin(PlayerJoinEvent event) {
    if (FloodgateApi.getInstance().isFloodgatePlayer(event.getPlayer().getUniqueId())) {
        event.getPlayer().sendMessage(Component.text("Welcome, Bedrock player!"));
    }
}
```

For Velocity-side Bedrock detection, the `floodgate-velocity` plugin exposes the same
`FloodgateApi` from the proxy. Use proxy-side detection for tab-list / motd customization
that should reflect device type.

### 2.C.12 Production checklist

- [ ] Floodgate forwarding mode matches Velocity mode (modern / proxy / etc.)
- [ ] Backend `paper-global.yml` has correct `forwarding-secret`
- [ ] Geyser is configured with the correct `remote.address` and `remote.port`
- [ ] Custom items/blocks have JSON mappings or a Geyser extension
- [ ] Bedrock skin pipeline tested (joined a Bedrock account, saw skin on Java)
- [ ] Permission plugin accepts Floodgate UUID format
- [ ] Anti-cheat whitelist includes Floodgate UUIDs
- [ ] Java fallback exists for every Bedrock-only UI
- [ ] LuckLink or equivalent installed for Geyser permission registration on the proxy
  (fixes a known Velocity quirk — see [LuckLink](https://github.com/onebeastchris/LuckLink))

---

## 2.D SELF-REVIEW CHECKLIST

- [x] Velocity project setup with annotation processor
- [x] Velocity main class with Guice DI
- [x] Velocity event reference (verified against velocity-api 3.4.0)
- [x] `EventTask.async` pattern documented
- [x] Routing players between backends
- [x] Brigadier with **redirect** + custom **suggester**
- [x] Modern forwarding configuration on Velocity + Paper sides
- [x] Plugin messaging (BungeeCord-channel + custom Velocity channel)
- [x] **Cross-server messaging via Redis** (when plugin messaging isn't enough)
- [x] ProxyPing / MOTD / favicon
- [x] Velocity scheduler basics
- [x] **LuckPerms on Velocity** integration
- [x] Velocity gotchas list
- [x] Folia threading model invariants
- [x] All 4 Folia schedulers with `ScheduledTask` lifecycle
- [x] Bukkit → Folia migration patterns
- [x] **Folia chunk tickets** for force-loaded chunks
- [x] **Folia ChunkLoadEvent thread semantics**
- [x] **Concurrency primitives table** (what scales / what doesn't)
- [x] Paper+Folia compat shim (`Schedulers` utility class)
- [x] Folia performance / hardware guidance
- [x] **Folia crash debugging checklist**
- [x] Floodgate **UUID format** with prefix detection
- [x] Floodgate API: detection + player metadata
- [x] **Linked-account behaviour** (Bedrock players who linked Java accounts)
- [x] Geyser API for protocol-level data
- [x] **All 3 Cumulus form types** with valid/closed/invalid handlers
- [x] **`sendForm` return value + chat fallback** for older clients
- [x] Java fallback pattern
- [x] **Custom items / custom blocks** (JSON + extension API)
- [x] **Bedrock skin upload pipeline** + Global API endpoint
- [x] Bedrock gotchas (textures, layouts, anti-cheat, etc.)
- [x] Permission plugin compatibility note
- [x] Production checklist for cross-play deployment

---

## 2.Z REFERENCES

- [Velocity API basics](https://docs.papermc.io/velocity/dev/api-basics/)
- [Velocity Javadoc 3.4.0](https://jd.papermc.io/velocity/3.4.0/)
- [Velocity Command API](https://docs.papermc.io/velocity/dev/command-api/)
- [Velocity modern forwarding](https://docs.papermc.io/velocity/player-information-forwarding)
- [Velocity Brigadier example by 4drian3d](https://gist.github.com/4drian3d/8b8db7bd01d2c7b4f8a7e4744620afbc)
- [PaperMC/Velocity GitHub](https://github.com/PaperMC/Velocity)
- [Folia README](https://github.com/PaperMC/Folia)
- [Folia compatibility guide](https://docs.papermc.io/paper/dev/folia-support/)
- [Folia Javadoc — schedulers](https://jd.papermc.io/folia/)
- [Folia FAQ](https://docs.papermc.io/folia/faq/)
- [Folia region logic](https://docs.papermc.io/folia/reference/region-logic/)
- [Floodgate API wiki](https://wiki.geysermc.org/floodgate/api/)
- [Floodgate features (skin upload)](https://wiki.geysermc.org/floodgate/features/)
- [Floodgate FAQ (UUID format)](https://wiki.geysermc.org/floodgate/faq/)
- [Cumulus forms](https://wiki.geysermc.org/geyser/forms/)
- [GeyserMC/Cumulus GitHub](https://github.com/GeyserMC/Cumulus)
- [Geyser API](https://wiki.geysermc.org/geyser/api/)
- [Geyser custom items](https://wiki.geysermc.org/geyser/custom-items/)
- [Geyser custom blocks](https://wiki.geysermc.org/geyser/custom-blocks/)
- [Geyser Global API (skins, UUIDs)](https://wiki.geysermc.org/geyser/global-api/)
- [LuckLink (Geyser permission registration on Velocity)](https://github.com/onebeastchris/LuckLink)
- [LuckPerms](https://luckperms.net/)

---

## See also

- `01-build-tooling.md` — Velocity-specific build setup (annotation processor), run-velocity
  task plugin.
- `paper-plugin-dev.md` §9 — original BukkitScheduler reference (replaced by Folia
  schedulers above).
- `paper-plugin-dev.md` §14.3 — original BungeeCord plugin messaging (still relevant).
