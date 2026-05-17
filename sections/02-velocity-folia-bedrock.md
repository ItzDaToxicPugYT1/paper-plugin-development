---
name: paper-plugin-dev-velocity-folia-bedrock
description: >
  Expansion to paper-plugin-dev.md covering: Velocity proxy plugin development, Folia regionised
  multithreading deep-dive, supporting Paper+Folia simultaneously, and Bedrock cross-play
  via Geyser/Floodgate (Cumulus forms, BedrockPlayer detection, modern forwarding).
---

# 2. VELOCITY · FOLIA · BEDROCK — DEEP DIVE

The original `paper-plugin-dev.md` only briefly references Folia (§9.2) and does not cover
Velocity or Bedrock cross-play at all. This section is a full reference for building plugins
that target the proxy layer (Velocity), the threaded server fork (Folia), and the cross-platform
audience (Bedrock via GeyserMC + Floodgate).

---

## 2.A VELOCITY PROXY PLUGIN DEVELOPMENT

Velocity is PaperMC's modern Minecraft proxy (replacement for BungeeCord/Waterfall). It runs in
front of Paper backend servers and routes players between them. Plugins running on Velocity see
**every** player on the network, can route them, intercept chat, register cross-network commands,
and modify the proxy <-> backend protocol stream.

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
```

> **No `plugin.yml`.** Velocity uses an annotation processor that generates
> `velocity-plugin.json` from `@Plugin` on your main class.

### 2.A.2 Main class

```java
package com.example.myproxy;

import com.google.inject.Inject;
import com.velocitypowered.api.event.Subscribe;
import com.velocitypowered.api.event.proxy.ProxyInitializeEvent;
import com.velocitypowered.api.event.proxy.ProxyShutdownEvent;
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
        // Hard dependency on LuckPerms (proxy-side):
        @com.velocitypowered.api.plugin.Dependency(id = "luckperms"),
        // Optional dependency:
        @com.velocitypowered.api.plugin.Dependency(id = "papiproxybridge", optional = true)
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
    }

    @Subscribe
    public void onShutdown(ProxyShutdownEvent event) {
        logger.info("MyProxy shutting down");
    }

    public ProxyServer proxy() { return proxy; }
}
```

**Key differences from Bukkit/Paper plugins:**
- Velocity uses **Guice DI**: services like `ProxyServer`, `Logger`, `@DataDirectory Path`
  are constructor-injected. There's no equivalent of `JavaPlugin#getConfig()`; you load config
  yourself from `dataDir`.
- Events are subscribed via `@Subscribe`, not `@EventHandler`.
- No `plugin.yml` - metadata is in `@Plugin`.
- Logger is **SLF4J**, not JUL.
- Plugins can be loaded asynchronously and concurrently - **no shared global state assumptions**.

### 2.A.3 Velocity event reference

```java
// Connection lifecycle
PreLoginEvent             // Before authentication. Disconnect bots/banned IPs here.
LoginEvent                // After auth, before backend connect. Set Velocity tab list here.
PostLoginEvent            // Player has joined the proxy.
ServerPreConnectEvent     // Player about to connect to a backend. Modify target server here.
ServerConnectedEvent      // Player switched to a backend.
DisconnectEvent           // Player left. event.getLoginStatus() tells you why.

// Chat / commands
PlayerChatEvent                          // Player typed in chat.
CommandExecuteEvent                      // Any command, intercept BEFORE routing.
PlayerAvailableCommandsEvent             // Customize tab-complete tree visible to a player.

// Network / data
PluginMessageEvent                       // Plugin messaging (BungeeCord channel etc.).
ProxyPingEvent                           // Server list ping (MOTD customization).
ResourcePackResponseEvent                // Pack accept/decline.
KickedFromServerEvent                    // Backend kicked the player. Choose: reconnect to lobby or pass through.

// Internal
ProxyInitializeEvent / ProxyShutdownEvent / ProxyReloadEvent
```

### 2.A.4 Routing players between backends

```java
@Subscribe
public void onCommand(CommandExecuteEvent event) {
    if (!(event.getCommandSource() instanceof Player player)) return;
    if (event.getCommand().equalsIgnoreCase("lobby")) {
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
}
```

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

### 2.A.6 Brigadier on Velocity

```java
@Subscribe
public void onInit(ProxyInitializeEvent event) {
    BrigadierCommand command = new BrigadierCommand(
        LiteralArgumentBuilder.<CommandSource>literal("hub")
            .requires(src -> src.hasPermission("myproxy.hub"))
            .executes(ctx -> {
                if (ctx.getSource() instanceof Player player) {
                    proxy.getServer("hub").ifPresent(rs ->
                        player.createConnectionRequest(rs).fireAndForget());
                }
                return Command.SINGLE_SUCCESS;
            })
            .build()
    );
    proxy.getCommandManager().register(
        proxy.getCommandManager().metaBuilder("hub").aliases("lobby").plugin(this).build(),
        command
    );
}
```

### 2.A.7 Modern Forwarding (Velocity <-> Paper)

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

**Why these settings:** modern forwarding embeds a signed token in the login packet that the
backend verifies using the shared secret. The backend trusts the proxy's authentication, so
`online-mode=false` on the backend is **safe** - but only because `online-mode=true` is set
inside `paper-global.yml` to enable signature verification. Misconfiguration here is the #1 cause
of "I can join with any name" cracked-account exploits on networks that *intended* to be online.

**Firewall rule:** the backend ports must be reachable only from the proxy's IP.

### 2.A.8 Plugin Messaging vs Velocity Channels

Old BungeeCord channel still works (cross-version compatibility) but Velocity recommends a
custom Velocity-specific channel for new plugins:

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

> **Register the channel** on Velocity at startup:
> ```java
> proxy.getChannelRegistrar().register(MinecraftChannelIdentifier.from("myproxy:command"));
> ```

### 2.A.9 Tab-list / Server-list ping

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
    // Custom favicon (64x64 PNG):
    Path icon = dataDir.resolve("server-icon.png");
    if (Files.exists(icon)) {
        builder.favicon(Favicon.create(icon));
    }
    event.setPing(builder.build());
}
```

### 2.A.10 Velocity-side scheduler

```java
proxy.getScheduler()
    .buildTask(this, () -> logger.info("Tick"))
    .delay(5, TimeUnit.SECONDS)
    .repeat(30, TimeUnit.SECONDS)
    .schedule();
```

> Velocity is **fully async** - there is no main thread. All API methods are thread-safe unless
> the Javadoc says otherwise.

### 2.A.11 Common cross-network plugins (reference for hooks)

| Plugin | Purpose |
|---|---|
| LuckPerms (Velocity) | Network-wide permissions |
| ProtocolizeAPI / ViaVersion (proxy) | Multi-version protocol support |
| PAPIProxyBridge | PlaceholderAPI on the proxy |
| LibreLogin | Network-wide login/auth |
| Velocitab | Custom tab list |
| Geyser-Velocity / Floodgate-Velocity | Bedrock cross-play |
| Plan / SimpleCloud | Analytics / orchestration |

---

## 2.B FOLIA - REGIONISED MULTITHREADING

Folia is PaperMC's experimental fork that splits a single world into independent **regions**
(groups of nearby chunks), each ticked by its own thread. There is **no main thread**; instead,
thousands of region threads tick in parallel.

### 2.B.1 What this changes for plugin authors

| Concept | Paper | Folia |
|---|---|---|
| Main thread | Yes - `Bukkit.isPrimaryThread()` | No - concept does not exist |
| `BukkitScheduler` | Works | Throws `UnsupportedOperationException` |
| Calling Bukkit API | OK from main thread | OK only from the **owning region thread** of the relevant entity/location |
| `Player#teleport` | Sync | Async (returns `CompletableFuture`) |
| Async events | Same | Same |
| `EntityMoveEvent` etc. | Fire on main | Fire on the region thread that owns the entity |

### 2.B.2 Required plugin metadata

```yaml
# plugin.yml or paper-plugin.yml
folia-supported: true
```

**Without this flag, Folia refuses to load the plugin.** This is intentional - most legacy
plugins are unsafe on Folia.

### 2.B.3 The four schedulers

```java
// 1) Global region scheduler - for things not tied to any entity/location
//    (server-wide tasks, world iteration, gamemode-wide events)
Bukkit.getGlobalRegionScheduler().run(plugin, task -> {
    // Runs on the global tick thread
});
Bukkit.getGlobalRegionScheduler().runDelayed(plugin, task -> { /* ... */ }, 100L);
Bukkit.getGlobalRegionScheduler().runAtFixedRate(plugin, task -> { /* ... */ }, 0L, 20L);

// 2) Region scheduler - for tasks at a specific Location/chunk
//    Use this for block/world manipulation
Bukkit.getRegionScheduler().run(plugin, world, chunkX, chunkZ, task -> {
    Block block = world.getBlockAt(chunkX << 4, 64, chunkZ << 4);
    block.setType(Material.DIAMOND_BLOCK);
});

// 3) Entity scheduler - for tasks involving a specific entity
//    The scheduler handles entity teleports between regions transparently
entity.getScheduler().run(plugin, task -> {
    if (entity.isValid()) entity.setVelocity(new Vector(0, 1, 0));
}, () -> {
    // Retired callback: entity got removed before task ran
    plugin.getLogger().info("Entity gone");
});

// 4) Async scheduler - for non-game tasks (DB, HTTP, file I/O)
Bukkit.getAsyncScheduler().runNow(plugin, task -> {
    String result = httpClient.get("https://api.example.com");
    // Hop back to a region for game effects:
    entity.getScheduler().run(plugin, t -> entity.sendMessage(result), null);
});
Bukkit.getAsyncScheduler().runDelayed(plugin, task -> { }, 5, TimeUnit.SECONDS);
Bukkit.getAsyncScheduler().runAtFixedRate(plugin, task -> { }, 0, 1, TimeUnit.MINUTES);
```

### 2.B.4 Threading rules on Folia

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

### 2.B.5 Common Bukkit -> Folia migration patterns

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

// AFTER (Folia - must run on the region that owns each entity):
for (Entity e : world.getEntities()) {
    e.getScheduler().run(plugin, t -> e.remove(), null);
}
```

### 2.B.6 Teleporting between regions

```java
// Folia teleport is always async. Don't depend on the player's location after the call:
player.teleportAsync(otherWorldLoc).thenAccept(success -> {
    if (success) {
        // We are now back on the destination region thread for the player:
        player.getScheduler().run(plugin, t ->
            player.sendMessage(Component.text("Teleported")), null);
    }
});
```

### 2.B.7 Folia compatibility shim (supporting both Paper + Folia)

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
}
```

> Use this wrapper everywhere instead of `Bukkit.getScheduler()` directly. Then your plugin
> ships one JAR that runs on Paper or Folia.

### 2.B.8 Folia's official "do / don't" list

**Do:**
- Use `teleportAsync` everywhere.
- Hop into the right scheduler before touching state.
- Use thread-safe collections for shared state (`ConcurrentHashMap`, `CopyOnWriteArrayList`).
- Treat `Bukkit.getOnlinePlayers()` as a snapshot - players may join/leave mid-iteration.
- Defensively check `entity.isValid()` and `player.isOnline()` inside scheduled callbacks.

**Don't:**
- Don't synchronize on world/entity instances expecting a global lock.
- Don't iterate one world's entities to apply state to entities in *another* region.
- Don't rely on `Bukkit.isPrimaryThread()` - it always returns false.
- Don't write tick-counter loops - use `runAtFixedRate`.

### 2.B.9 Performance & limitations

- Folia shines on player-spread servers (skyblock, faction-wide, large survival).
- Folia loses on player-clustered servers (a single PvP arena = one thread doing all the work).
- Some plugins (WorldEdit, dynmap, etc.) need Folia-specific forks.
- Folia can run **with** the Paper API but expects you to write more careful code - the
  performance gain is proportional to how spread-out your players are.

---

## 2.C BEDROCK CROSS-PLAY (GeyserMC + Floodgate)

Bedrock players cannot join a Java server natively. **Geyser** is a translator (Java protocol <->
Bedrock protocol) and **Floodgate** is the auth bridge (gives Bedrock players a Java-style
account on your server without a Mojang license).

### 2.C.1 Setup recap (server-side)

| Plugin | Place on | Purpose |
|---|---|---|
| Geyser-Spigot or Geyser-Velocity | proxy or backend | Protocol translation |
| Floodgate | proxy or backend | Bedrock auth |

You generally install both on the proxy (Velocity) when running a network, or both on each Paper
backend when running standalone.

### 2.C.2 Detecting Bedrock players

```java
// Floodgate API dependency (compileOnly):
//   org.geysermc.floodgate:api:2.2.4-SNAPSHOT
// repository:
//   https://repo.opencollab.dev/main/

import org.geysermc.floodgate.api.FloodgateApi;

public boolean isBedrock(Player player) {
    return FloodgateApi.getInstance().isFloodgatePlayer(player.getUniqueId());
}

// Read Bedrock-specific data:
FloodgatePlayer fp = FloodgateApi.getInstance().getPlayer(player.getUniqueId());
if (fp != null) {
    String xuid = fp.getXuid();
    String javaUsername = fp.getJavaUsername();
    String bedrockUsername = fp.getUsername();
    DeviceOs device = fp.getDeviceOs();          // ANDROID, IOS, WINDOWS, NX (Switch), XBOX, PS4, etc.
    InputMode input = fp.getInputMode();          // KEYBOARD_MOUSE, TOUCH, GAMEPAD, MOTION_CONTROLLER
    UiProfile ui = fp.getUiProfile();             // CLASSIC or POCKET
    String langCode = fp.getLanguageCode();       // e.g. "en_US"
    boolean linked = fp.isLinked();               // Linked to a Java account?
}
```

**Bedrock username prefix:** Floodgate prefixes Bedrock usernames with `.` by default (configurable
in `config.yml`). So `Steve` from Bedrock joins as `.Steve` to avoid collisions. Always treat
`Player#getName` as opaque; never assume a fixed format.

### 2.C.3 Geyser API (alternative)

```java
// dependency: org.geysermc.geyser:api:2.4.2
import org.geysermc.geyser.api.GeyserApi;
import org.geysermc.geyser.api.connection.GeyserConnection;

GeyserConnection conn = GeyserApi.api().connectionByUuid(player.getUniqueId());
if (conn != null) {
    // Same data as Floodgate, but with extra Bedrock-protocol details:
    String version = conn.bedrockVersion();
    int protocolVersion = conn.protocolVersion();
}
```

> Use **Floodgate API** when you only need to detect/identify Bedrock players. Use **Geyser API**
> when you need protocol-level info or want to render Bedrock-only UI (forms below).

### 2.C.4 Cumulus Forms (native Bedrock UI)

Cumulus is the form library shared by Geyser and Floodgate. It lets you display Bedrock-native
modal popups, form selectors, and structured input forms - these render on Bedrock as proper
mobile-style UIs, not as fake chat windows.

```java
import org.geysermc.cumulus.form.SimpleForm;
import org.geysermc.cumulus.form.ModalForm;
import org.geysermc.cumulus.form.CustomForm;
import org.geysermc.cumulus.component.*;

// Simple form (button list)
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

// Modal (yes/no confirmation)
ModalForm confirm = ModalForm.builder()
    .title("Are you sure?")
    .content("This will delete your home.")
    .button1("Confirm")
    .button2("Cancel")
    .validResultHandler(r -> {
        if (r.clickedFirst()) deleteHome(player);
    })
    .build();

// Custom form (mixed inputs)
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
        int rd = (int) response.asSlider(2);
        int diffIndex = response.asStepSlider(3);
        String name = response.asInput(4);
        int langIndex = response.asDropdown(5);
        savePrefs(player, particles, rd, diffIndex, name, langIndex);
    })
    .build();
```

### 2.C.5 Java fallback for cross-platform UIs

Always provide a chat-based / inventory-GUI fallback so Java players can use the same feature:

```java
public void openMenu(Player player) {
    if (FloodgateApi.getInstance().isFloodgatePlayer(player.getUniqueId())) {
        sendBedrockForm(player);
    } else {
        new MenuGUI().open(player);   // Chest GUI for Java players
    }
}
```

### 2.C.6 Bedrock-specific gotchas

- **Texture / model differences:** custom resource packs and CMD models look different on
  Bedrock. Geyser auto-converts most things but with quality loss.
- **Skin restrictions:** Bedrock skins use a different format. Floodgate provides `getSkin()`.
- **Item ID gaps:** some Java items don't exist on Bedrock (e.g. some banner patterns).
  Geyser substitutes them, but your plugin should test on a real Bedrock client.
- **Inventory slot count:** Bedrock often uses different inventory layouts; chest GUIs work fine
  but anvil/loom/grindstone GUIs sometimes glitch.
- **No client-side mods:** Bedrock players can't run client mods. Anti-cheat false positives
  for "modded clients" must whitelist Floodgate UUIDs.

### 2.C.7 Permissions on Bedrock players

Floodgate's UUIDs are deterministic and prefixed (start with the bytes `0x00000000-0000-0000-`
in raw form - Floodgate uses a "Floodgate UUID" namespace). LuckPerms, Vault and most
permission plugins handle them transparently. To whitelist Bedrock players, use their Floodgate
UUID, not their Bedrock XUID directly.

### 2.C.8 Useful event hooks

```java
// Floodgate fires its own events (proxy or server side):
@EventHandler
public void onSkinUpdate(SkinApplyEvent event) { /* ... */ }

// Bedrock player join detection (Bukkit event + check):
@EventHandler
public void onJoin(PlayerJoinEvent event) {
    if (FloodgateApi.getInstance().isFloodgatePlayer(event.getPlayer().getUniqueId())) {
        event.getPlayer().sendMessage(Component.text("Welcome, Bedrock player!"));
    }
}
```

### 2.C.9 Floodgate dependency declaration

```yaml
# plugin.yml or paper-plugin.yml
softdepend: [floodgate, Geyser-Spigot]
```

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

---

## 2.D REFERENCES

- Velocity API basics: https://docs.papermc.io/velocity/dev/api-basics/
- Velocity Javadoc: https://jd.papermc.io/velocity/3.4.0/
- Velocity modern forwarding: https://docs.papermc.io/velocity/player-information-forwarding
- Folia README: https://github.com/PaperMC/Folia
- Folia compatibility guide: https://docs.papermc.io/paper/dev/folia-support/
- Folia Javadoc (schedulers): https://jd.papermc.io/folia/
- Floodgate API wiki: https://wiki.geysermc.org/floodgate/api/
- Cumulus forms: https://wiki.geysermc.org/geyser/forms/
- Geyser API: https://geysermc.org/wiki/geyser/getting-started-with-the-api/
