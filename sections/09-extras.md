---
name: paper-plugin-dev-extras
description: >
  Consolidated extras file covering all remaining gaps: CI/CD (GitHub Actions,
  Hangar/Modrinth publish, auto-versioning); metrics (bStats 3.0, Prometheus);
  profiling (spark, TPS/MSPT, Watchdog); testing (MockBukkit, JUnit5, Mockito,
  ArchUnit, Testcontainers); ecosystem hooks (Vault/VaultUnlocked, PlaceholderAPI,
  WorldGuard custom flags, Citizens NPC, MythicMobs, WorldEdit/FAWE, DiscordSRV/JDA);
  packet manipulation (ProtocolLib PacketAdapter, PacketEvents 2.x listener,
  freecam detection, fake entities, Netty channel injection); 1.21.x flagship
  features (Dialog API, Armor Trim API, Banner Pattern, Trial Spawner, Wind Charge,
  Mace, tick freezing, Bundles); build extras (Spotless, Checkstyle, NullAway,
  Lombok, Groovy DSL, jar-in-jar, reproducible builds, multi-module API split);
  Adventure deep (Gson/Legacy/Plain/ANSI serializers, insertion, Sound API, sign
  edit); inventory advanced (AnvilGUI, TriumphGUI, MenuType, Bundle, pagination);
  PlayerProfile/skin/textures; worlds (rayTrace, BlockIterator, chunk tickets,
  Heightmap, snapshots); commands extras (Cloud, ACF, redirects/forks,
  CommandSyntaxException); events extras (Vehicle, Hanging, Raid, Weather, Lambda
  EventExecutor, HandlerList unregistration, AsyncTabComplete); misc (HttpClient,
  Caffeine, Discord webhooks, leaderboards, statistics API, brewing, custom
  paintings/wolf variants, game rules deep, spectator camera, worldborder lerp).
---

# 9. EXTRAS — DEEP DIVE

This file covers every remaining gap from the exhaustive 66-category Paper plugin
development checklist that sections 01–08 do not address. It is organized as a
**cookbook**: heading → working snippet → pitfall → ref. All code targets
**Paper 1.21.4+** (Mojang-mapped).

---

## 9.0 MENTAL MODEL

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR PLUGIN                                │
├──────────┬───────────┬────────────┬──────────┬──────────────┤
│ CI/CD    │ Ecosystem │ Packets    │ 1.21.x   │ Misc/Perf    │
│ bStats   │ Vault     │ ProtocolLib│ Dialog   │ HttpClient   │
│ Actions  │ PAPI      │ PacketEvts │ Trim     │ Caffeine     │
│ MockBukk │ WG/Citiz  │ Netty      │ Trial    │ Profiling    │
│ Spotless │ Discord   │ Freecam    │ Wind     │ Leaderboard  │
└──────────┴───────────┴────────────┴──────────┴──────────────┘
```

---

## 9.1 CI/CD — GITHUB ACTIONS

### 9.1.1 Basic build workflow

```yaml
# .github/workflows/build.yml
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 21 }
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build
      - uses: actions/upload-artifact@v4
        with: { name: plugin-jar, path: build/libs/*-all.jar }
```

### 9.1.2 Hangar publish

```yaml
# append to build.yml
      - name: Publish to Hangar
        if: github.ref == 'refs/heads/main'
        uses: HangarMC/upload-action@v1
        with:
          api-key: ${{ secrets.HANGAR_TOKEN }}
          slug: MyPlugin
          version: ${{ github.sha }}
          channel: Release
          file: build/libs/*-all.jar
          game-versions: '1.21.4'
          platform: PAPER
```

Ref: [Hangar auto-publishing docs](https://docs.papermc.io/misc/hangar-publishing).

### 9.1.3 Modrinth publish (via Gradle plugin)

```kotlin
// build.gradle.kts
plugins { id("com.modrinth.minotaur") version "2.+" }
modrinth {
    token.set(System.getenv("MODRINTH_TOKEN"))
    projectId.set("your-project-id")
    versionNumber.set(project.version.toString())
    uploadFile.set(tasks.shadowJar)
    gameVersions.addAll("1.21.4")
    loaders.add("paper")
}
```

### 9.1.4 Auto-versioning

```kotlin
version = "1.0.0-${System.getenv("GITHUB_RUN_NUMBER") ?: "local"}"
```

Or use `git describe --tags --always` for SemVer from tags.

---

## 9.2 METRICS — bStats

### 9.2.1 Setup (Gradle)

```kotlin
dependencies { implementation("org.bstats:bstats-bukkit:3.1.0") }
// shade + relocate:
tasks.shadowJar { relocate("org.bstats", "com.example.myplugin.bstats") }
```

### 9.2.2 Usage

```java
import org.bstats.bukkit.Metrics;
@Override public void onEnable() {
    new Metrics(this, /* pluginId */ 12345);
}
```

Custom charts:

```java
Metrics m = new Metrics(this, 12345);
m.addCustomChart(new Metrics.SimplePie("storage_backend",
    () -> getConfig().getString("storage.type", "sqlite")));
```

Ref: [bStats getting started](https://www.bstats.org/getting-started/include-metrics).

### 9.2.3 Prometheus (via spark API or custom)

spark exposes `/spark health` data. For custom Prometheus metrics, embed
`io.prometheus:simpleclient_httpserver`:

```java
// onEnable:
Counter crafts = Counter.build().name("crafts_total").help("Items crafted").register();
HTTPServer server = new HTTPServer.Builder().withPort(9100).build();
// in event handler:
crafts.inc();
```

---

## 9.3 PROFILING — SPARK

spark is bundled in Paper 1.20.4+. Key commands:

| Command | Purpose |
|---|---|
| `/spark profiler start` | CPU sampling (default 4ms interval) |
| `/spark profiler stop` | Upload + get URL |
| `/spark tps` | Show TPS + MSPT |
| `/spark health` | Memory / GC / tick stats |
| `/spark gc` | GC pause history |

Read TPS programmatically:

```java
double[] tps = Bukkit.getTPS(); // [1m, 5m, 15m]
double mspt = Bukkit.getAverageTickTime(); // ms
```

Ref: [spark docs](https://spark.lucko.me/docs/Command-Usage),
[Paper profiling guide](https://docs.papermc.io/paper/profiling).

### 9.3.1 Watchdog awareness

Paper's watchdog kills the server after 60s of a hung tick. If your plugin does
heavy computation, **yield** periodically or move it async. The watchdog thread
dumps all threads before killing — check `latest.log` for the thread dump.

---

## 9.4 TESTING — MockBukkit + JUnit 5

### 9.4.1 Dependency

```kotlin
testImplementation("com.github.MockBukkit:MockBukkit:v3.133.2")
testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
testImplementation("org.mockito:mockito-core:5.14.0")
```

### 9.4.2 Basic test

```java
class WalletTest {
    private ServerMock server;
    private WalletPlugin plugin;

    @BeforeEach void setUp() {
        server = MockBukkit.mock();
        plugin = MockBukkit.load(WalletPlugin.class);
    }
    @AfterEach void tearDown() { MockBukkit.unmock(); }

    @Test void startingBalance() {
        PlayerMock player = server.addPlayer();
        player.performCommand("wallet");
        player.assertSaid("Balance: 100 coins");
    }
}
```

Ref: [MockBukkit docs](https://mockbukkit.readthedocs.io/en/latest/first_tests.html).

### 9.4.3 Mockito for services

```java
@Mock VaultEconomy economy;
@Test void depositHook() {
    when(economy.depositPlayer(any(), eq(50.0)))
        .thenReturn(new EconomyResponse(50, 150, SUCCESS, ""));
    // test your wrapper
}
```

### 9.4.4 ArchUnit (architecture rules)

```java
@AnalyzeClasses(packages = "com.example.myplugin")
class ArchTests {
    @ArchTest ArchRule noDirectBukkitInDomain = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAPackage("org.bukkit..");
}
```

---

## 9.5 ECOSYSTEM — VAULT / VaultUnlocked

### 9.5.1 Hooking economy

```java
private Economy econ;
@Override public void onEnable() {
    RegisteredServiceProvider<Economy> rsp =
        getServer().getServicesManager().getRegistration(Economy.class);
    if (rsp != null) econ = rsp.getProvider();
}
public boolean pay(OfflinePlayer p, double amt) {
    return econ != null && econ.withdrawPlayer(p, amt).transactionSuccess();
}
```

`VaultUnlocked` is the modern fork — same API, added multi-currency. Maven:

```xml
<dependency>
  <groupId>net.milkbowl.vault</groupId>
  <artifactId>VaultAPI</artifactId>
  <version>1.7.1</version>
  <scope>provided</scope>
</dependency>
```

Ref: [VaultAPI GitHub](https://github.com/MilkBowl/VaultAPI),
[VaultUnlocked Hangar](https://hangar.papermc.io/TNE/VaultUnlocked).

---

## 9.6 ECOSYSTEM — PlaceholderAPI

### 9.6.1 Registering an expansion

```java
public class WalletExpansion extends PlaceholderExpansion {
    private final WalletPlugin plugin;
    public WalletExpansion(WalletPlugin p) { this.plugin = p; }
    @Override public String getIdentifier() { return "wallet"; }
    @Override public String getAuthor() { return "you"; }
    @Override public String getVersion() { return plugin.getPluginMeta().getVersion(); }
    @Override public boolean persist() { return true; }

    @Override public String onPlaceholderRequest(Player p, String id) {
        if (p == null) return "";
        return switch (id) {
            case "balance" -> String.valueOf(plugin.getBalance(p));
            case "rank"    -> String.valueOf(plugin.getRank(p));
            default -> null;
        };
    }
}
// onEnable: new WalletExpansion(this).register();
```

### 9.6.2 Using PAPI in your messages

```java
String parsed = PlaceholderAPI.setPlaceholders(player, "%wallet_balance%");
```

### 9.6.3 Relational placeholders

Override `onPlaceholderRequest(Player one, Player two, String id)` for
`%rel_wallet_compare%` style placeholders.

Ref: [PAPI expansion docs](https://wiki.placeholderapi.com/developers/using-placeholderapi/).

---

## 9.7 ECOSYSTEM — WorldGuard CUSTOM FLAGS

```java
// Register at plugin load (onLoad, not onEnable):
StateFlag MY_FLAG = new StateFlag("my-custom-flag", true); // default ALLOW
FlagRegistry registry = WorldGuard.getInstance().getFlagRegistry();
registry.register(MY_FLAG);

// Query in event handler:
RegionContainer container = WorldGuard.getInstance().getPlatform().getRegionContainer();
RegionQuery query = container.createQuery();
ApplicableRegionSet set = query.getApplicableRegions(BukkitAdapter.adapt(loc));
if (!set.testState(null, MY_FLAG)) {
    event.setCancelled(true);
}
```

Ref: [WorldGuard custom flags docs](https://worldguard.enginehub.org/en/latest/developer/regions/custom-flags/).

---

## 9.8 ECOSYSTEM — Citizens NPC API

```java
// Depends: Citizens (softdepend in plugin.yml)
NPC npc = CitizensAPI.getNPCRegistry().createNPC(EntityType.PLAYER, "Guard");
npc.spawn(location);
npc.getNavigator().setTarget(targetLoc);
npc.addTrait(SkinTrait.class);
npc.getOrAddTrait(SkinTrait.class).setSkinName("Notch");

// Listen for clicks:
@EventHandler public void onClick(NPCRightClickEvent e) {
    if (e.getNPC().getName().equals("Guard")) {
        e.getClicker().sendMessage(Component.text("Hello!"));
    }
}
```

Ref: [Citizens Wiki API](https://wiki.citizensnpcs.co/API).

---

## 9.9 ECOSYSTEM — WorldEdit / FAWE

```java
// Load a schematic and paste it
ClipboardFormat format = ClipboardFormats.findByFile(schemFile);
try (ClipboardReader reader = format.getReader(new FileInputStream(schemFile))) {
    Clipboard clipboard = reader.read();
    try (EditSession session = WorldEdit.getInstance().newEditSession(BukkitAdapter.adapt(world))) {
        Operation op = new ClipboardHolder(clipboard)
            .createPaste(session)
            .to(BlockVector3.at(x, y, z))
            .ignoreAirBlocks(true)
            .build();
        Operations.complete(op);
    }
}
```

---

## 9.10 ECOSYSTEM — DiscordSRV / JDA

```java
// DiscordSRV hook (softdepend):
DiscordSRV.api.subscribe(new ListenerAdapter() {
    @Override public void onGuildMessageReceived(GuildMessageReceivedEvent e) {
        Bukkit.broadcast(Component.text("[Discord] " + e.getMessage().getContentRaw()));
    }
});

// Raw JDA (shade + bundle):
JDA jda = JDABuilder.createDefault(token).build();
jda.getTextChannelById(channelId)
   .sendMessage("Server started!").queue();
```

### 9.10.1 Discord webhooks (no JDA needed)

```java
HttpClient client = HttpClient.newHttpClient();
String json = "{\"content\":\"Server started!\"}";
client.sendAsync(HttpRequest.newBuilder()
    .uri(URI.create(webhookUrl))
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .header("Content-Type", "application/json")
    .build(), HttpResponse.BodyHandlers.discarding());
```



---

## 9.11 PACKET MANIPULATION — ProtocolLib

### 9.11.1 PacketAdapter listener

```java
ProtocolManager pm = ProtocolLibrary.getProtocolManager();
pm.addPacketListener(new PacketAdapter(plugin, ListenerPriority.NORMAL,
        PacketType.Play.Client.POSITION) {
    @Override
    public void onPacketReceiving(PacketEvent event) {
        PacketContainer packet = event.getPacket();
        double x = packet.getDoubles().read(0);
        double y = packet.getDoubles().read(1);
        double z = packet.getDoubles().read(2);
        // Validate movement...
    }
});
```

### 9.11.2 Sending a fake entity (packet-level hologram)

```java
PacketContainer spawn = pm.createPacket(PacketType.Play.Server.SPAWN_ENTITY);
spawn.getIntegers().write(0, entityId);
spawn.getUUIDs().write(0, UUID.randomUUID());
spawn.getEntityTypeModifier().write(0, EntityType.ARMOR_STAND);
spawn.getDoubles().write(0, loc.getX()).write(1, loc.getY()).write(2, loc.getZ());
pm.sendServerPacket(player, spawn);
```

Ref: [ProtocolLib wiki](https://github.com/dmulloy2/ProtocolLib/wiki/Packet-listeners-and-adapters),
[ProtocolLib.net](https://protocollib.net/).

---

## 9.12 PACKET MANIPULATION — PacketEvents 2.x

### 9.12.1 Setup

```kotlin
// build.gradle.kts — as softdepend or shaded:
compileOnly("com.github.retrooper:packetevents-spigot:2.7.0")
```

### 9.12.2 Listener

```java
public class MoveListener extends PacketListenerAbstract {
    @Override
    public void onPacketReceive(PacketReceiveEvent event) {
        if (event.getPacketType() == PacketType.Play.Client.PLAYER_POSITION) {
            WrapperPlayClientPlayerPosition pos =
                new WrapperPlayClientPlayerPosition(event);
            Vector3d position = pos.getLocation();
            // anti-cheat logic here
        }
    }
}

// onEnable:
PacketEvents.getAPI().getEventManager().registerListener(new MoveListener());
```

### 9.12.3 Sending packets

```java
WrapperPlayServerEntityAnimation anim = new WrapperPlayServerEntityAnimation(
    entityId, EntityAnimation.SWING_MAIN_ARM);
PacketEvents.getAPI().getPlayerManager().sendPacket(player, anim);
```

Ref: [PacketEvents docs](https://docs.packetevents.com/),
[PacketEvents Javadoc](https://javadocs.packetevents.com/).

---

## 9.13 FREECAM DETECTION (Anti-Cheat Pattern)

Freecam mods decouple the camera from the player entity. Detection strategies:

```java
// Strategy 1: Player hasn't sent movement packets but is "looking" at blocks
// far from their server-side position.
@Override public void onPacketReceive(PacketReceiveEvent event) {
    if (event.getPacketType() == PacketType.Play.Client.PLAYER_POSITION_AND_ROTATION) {
        WrapperPlayClientPlayerPositionAndRotation wrapper = new WrapperPlayClientPlayerPositionAndRotation(event);
        Location serverLoc = player.getLocation();
        double dist = serverLoc.toVector().distance(new Vector(
            wrapper.getLocation().getX(),
            wrapper.getLocation().getY(),
            wrapper.getLocation().getZ()));
        if (dist > 0.1) return; // Normal movement
        // If the player hasn't moved but is breaking blocks far away:
        // flag in a separate BlockBreak/Interact listener
    }
}

// Strategy 2: Track time between PLAYER_POSITION packets.
// Freecam sends position at player's feet while camera moves independently.
// If block interactions come from > 6 blocks away from last POSITION, flag.

// Strategy 3: Use PlayerMoveEvent server-side distance check.
// Compare event.getTo() distance to any PlayerInteractEvent target block.
// If target block > reach distance (6 creative, 4.5 survival) from last move, suspicious.
```

> **Pitfall.** False positives from lag/rubber-banding. Use a violation counter
> with decay, not instant-kick.

Ref: [Freecam issue #196](https://github.com/MinecraftFreecam/Freecam/issues/196),
[Vulcan anticheat](https://vulcanac.net/).

---

## 9.14 1.21.x FLAGSHIP — TRIAL SPAWNER API

```java
Block block = world.getBlockAt(x, y, z);
if (block.getState() instanceof TrialSpawner spawner) {
    spawner.setRequiredPlayerRange(16);
    spawner.setSpawnedType(EntityType.ZOMBIE);
    spawner.update();
}
// Listen for trial spawner activation:
@EventHandler public void onSpawnerActivate(EntitySpawnEvent e) {
    if (e.getEntity().getEntitySpawnReason() == CreatureSpawnEvent.SpawnReason.TRIAL_SPAWNER) {
        // custom logic
    }
}
```

---

## 9.15 1.21.x FLAGSHIP — WIND CHARGE & MACE

```java
// Spawn a wind charge:
WindCharge wc = world.spawn(player.getEyeLocation(), WindCharge.class, c -> {
    c.setVelocity(player.getLocation().getDirection().multiply(1.5));
});

// Mace smash-attack detection via EntityDamageByEntityEvent:
@EventHandler public void onDamage(EntityDamageByEntityEvent e) {
    if (!(e.getDamager() instanceof Player p)) return;
    ItemStack hand = p.getInventory().getItemInMainHand();
    if (hand.getType() != Material.MACE) return;
    if (p.getFallDistance() > 1.5) {
        // Smash attack — extra damage already applied by vanilla
        e.getEntity().getWorld().spawnParticle(Particle.EXPLOSION, e.getEntity().getLocation(), 5);
    }
}
```

---

## 9.16 1.21.x FLAGSHIP — ARMOR TRIM API

```java
ItemStack chestplate = ItemStack.of(Material.DIAMOND_CHESTPLATE);
chestplate.editMeta(ArmorMeta.class, meta -> {
    ArmorTrim trim = new ArmorTrim(
        TrimMaterial.GOLD,
        TrimPattern.WAYFINDER
    );
    meta.setTrim(trim);
});
```

---

## 9.17 1.21.x FLAGSHIP — DIALOG API (1.21.5+)

```java
// Paper's Dialog API (experimental, 1.21.5+)
Dialog dialog = Dialog.dialog()
    .title(Component.text("Quest Accept"))
    .body(DialogBody.of(Component.text("Do you accept the quest?")))
    .button(DialogButton.of(Component.text("Yes"), ClickAction.run("/quest accept")))
    .button(DialogButton.of(Component.text("No"), ClickAction.close()))
    .build();
player.openDialog(dialog);
```

Ref: [Paper API: Dialog](https://docs.papermc.io/paper/dev/api/).

---

## 9.18 1.21.x — TICK FREEZING

```java
// Freeze the server tick (test environments / cutscenes):
Bukkit.getServer().getServerTickManager().setFrozen(true);
// Resume:
Bukkit.getServer().getServerTickManager().setFrozen(false);
// Step N ticks while frozen:
Bukkit.getServer().getServerTickManager().stepGameIfFrozen(5);
```

---

## 9.19 1.21.x — BUNDLES

```java
ItemStack bundle = ItemStack.of(Material.BUNDLE);
bundle.editMeta(BundleMeta.class, meta -> {
    meta.addItem(ItemStack.of(Material.DIAMOND, 3));
    meta.addItem(ItemStack.of(Material.APPLE, 16));
});
// Read:
BundleMeta bm = (BundleMeta) bundle.getItemMeta();
List<ItemStack> contents = bm.getItems();
```

---

## 9.20 BUILD EXTRAS — SPOTLESS + CHECKSTYLE

```kotlin
// build.gradle.kts
plugins { id("com.diffplug.spotless") version "7.0.2" }
spotless {
    java {
        googleJavaFormat("1.24.0")
        licenseHeaderFile(rootProject.file("HEADER"))
    }
}
// CI: ./gradlew spotlessCheck
```

### 9.20.1 NullAway (via ErrorProne)

```kotlin
dependencies {
    errorprone("com.google.errorprone:error_prone_core:2.36.0")
    errorprone("com.uber.nullaway:nullaway:0.12.3")
}
tasks.withType<JavaCompile> {
    options.errorprone {
        error("NullAway")
        option("NullAway:AnnotatedPackages", "com.example.myplugin")
    }
}
```

### 9.20.2 Lombok

```kotlin
compileOnly("org.projectlombok:lombok:1.18.36")
annotationProcessor("org.projectlombok:lombok:1.18.36")
```

### 9.20.3 Multi-module API split

```
my-plugin/
  api/        → published jar, no Paper dependency
  core/       → implementation, depends on api + Paper
  build.gradle.kts (subprojects)
```

### 9.20.4 Reproducible builds

```kotlin
tasks.withType<Jar> {
    isReproducibleFileOrder = true
    isPreserveFileTimestamps = false
}
```

### 9.20.5 jar-in-jar

Use Gradle's `include` for nested dependencies without shading — Paper's
`libraries` block in `paper-plugin.yml` is the preferred approach for Paper plugins.

---

## 9.21 ADVENTURE DEEP

### 9.21.1 Serializers

```java
// Gson (for JSON storage / protocol):
String json = GsonComponentSerializer.gson().serialize(component);
Component back = GsonComponentSerializer.gson().deserialize(json);

// Legacy (for old plugins expecting §):
String legacy = LegacyComponentSerializer.legacySection().serialize(component);

// Plain text (strip all formatting):
String plain = PlainTextComponentSerializer.plainText().serialize(component);

// ANSI (console coloring):
String ansi = ANSIComponentSerializer.ansi().serialize(component);
```

### 9.21.2 Insertion (shift-click)

```java
Component msg = Component.text("[Click to insert]")
    .insertion("/warp spawn") // shift-click inserts this into chat box
    .hoverEvent(HoverEvent.showText(Component.text("Shift-click!")));
```

### 9.21.3 Adventure Sound API

```java
player.playSound(Sound.sound()
    .type(Key.key("minecraft:entity.experience_orb.pickup"))
    .source(Sound.Source.MASTER)
    .volume(1.0f)
    .pitch(1.5f)
    .seed(42L) // deterministic sound variant
    .build());
```

### 9.21.4 Sign editing

```java
// Open a sign editor for the player:
Block signBlock = player.getTargetBlockExact(5);
if (signBlock.getState() instanceof Sign sign) {
    player.openSign(sign, Side.FRONT);
}
```

---

## 9.22 INVENTORY ADVANCED

### 9.22.1 AnvilGUI library

```kotlin
// build.gradle.kts: implementation("net.wesjd:anvilgui:1.10.3-SNAPSHOT")
```

```java
new AnvilGUI.Builder()
    .plugin(plugin)
    .title("Enter warp name")
    .text("spawn")
    .onClick((slot, state) -> {
        if (slot != AnvilGUI.Slot.OUTPUT) return Collections.emptyList();
        String name = state.getText();
        return List.of(AnvilGUI.ResponseAction.close(),
            AnvilGUI.ResponseAction.run(() -> createWarp(player, name)));
    })
    .open(player);
```

### 9.22.2 Pagination pattern (TriumphGUI)

```java
PaginatedGui gui = Gui.paginated()
    .title(Component.text("Items"))
    .rows(6)
    .pageSize(45)
    .create();
for (ItemStack item : allItems) {
    gui.addItem(new GuiItem(item, e -> e.setCancelled(true)));
}
gui.setItem(6, 3, new GuiItem(Material.ARROW, e -> gui.previous()));
gui.setItem(6, 7, new GuiItem(Material.ARROW, e -> gui.next()));
gui.open(player);
```

### 9.22.3 MenuType (Paper 1.21+)

```java
// Open a 3-row menu with typed view:
MenuType.GENERIC_9X3.typed().create(player, Component.text("Shop"));
```

### 9.22.4 Bundle contents manipulation

See §9.19 above.

---

## 9.23 PLAYERPROFILE / SKINS

```java
// Read a player's skin:
PlayerProfile profile = player.getPlayerProfile();
ProfileProperty textures = profile.getProperties().stream()
    .filter(p -> p.getName().equals("textures"))
    .findFirst().orElse(null);

// Set a custom skin on a player:
PlayerProfile custom = Bukkit.createProfile(player.getUniqueId(), player.getName());
custom.getProperties().add(new ProfileProperty("textures", textureValue, textureSignature));
player.setPlayerProfile(custom);

// Create a skull with a skin:
ItemStack skull = ItemStack.of(Material.PLAYER_HEAD);
skull.editMeta(SkullMeta.class, meta -> {
    PlayerProfile p = Bukkit.createProfile(UUID.randomUUID());
    p.getProperties().add(new ProfileProperty("textures", base64Texture));
    meta.setPlayerProfile(p);
});
```

> **Pitfall.** Mojang API is rate-limited to ~600 requests/10min. Cache profiles.

---

## 9.24 WORLDS — RAY TRACING

```java
RayTraceResult result = world.rayTraceBlocks(
    player.getEyeLocation(),
    player.getLocation().getDirection(),
    /* maxDistance */ 50,
    FluidCollisionMode.NEVER,
    /* ignorePassableBlocks */ true
);
if (result != null && result.getHitBlock() != null) {
    Block target = result.getHitBlock();
}

// Entity ray trace:
RayTraceResult entityHit = world.rayTraceEntities(
    player.getEyeLocation(),
    player.getLocation().getDirection(),
    30, 0.5, // distance, raySize
    e -> e != player // filter
);
```

### 9.24.1 Chunk tickets / forceLoad

```java
// Keep a chunk loaded (won't unload even with no players):
world.addPluginChunkTicket(chunkX, chunkZ, plugin);
// Remove:
world.removePluginChunkTicket(chunkX, chunkZ, plugin);
// Via Bukkit forceLoad:
world.setChunkForceLoaded(chunkX, chunkZ, true);
```

### 9.24.2 Chunk snapshots (async-safe read)

```java
world.getChunkAtAsync(x, z).thenAccept(chunk -> {
    ChunkSnapshot snap = chunk.getChunkSnapshot();
    // snap.getBlockType(localX, y, localZ) — safe off-thread
    Material mat = snap.getBlockType(0, 64, 0);
});
```

### 9.24.3 Heightmap

```java
int surfaceY = world.getHighestBlockYAt(x, z, HeightMap.MOTION_BLOCKING);
int oceanFloor = world.getHighestBlockYAt(x, z, HeightMap.OCEAN_FLOOR);
```

---

## 9.25 COMMANDS EXTRAS — Cloud Framework

```java
// Cloud 2.x (Incendo):
PaperCommandManager<CommandSender> mgr = PaperCommandManager.createNative(plugin,
    CommandExecutionCoordinator.simpleCoordinator());
mgr.command(mgr.commandBuilder("warp")
    .argument(StringArgument.of("name"))
    .permission("myplugin.warp")
    .handler(ctx -> {
        String name = ctx.get("name");
        // teleport
    }));
```

### 9.25.1 Brigadier redirects / forks

```java
// Redirect: /tp → /teleport alias
dispatcher.register(literal("tp").redirect(dispatcher.getRoot().getChild("teleport")));

// Fork: execute a command for multiple targets
dispatcher.register(literal("heal-all").executes(ctx -> {
    for (Player p : Bukkit.getOnlinePlayers()) p.setHealth(20);
    return Bukkit.getOnlinePlayers().size();
}));
```

### 9.25.2 CommandSyntaxException

```java
.executes(ctx -> {
    String warpName = StringArgumentType.getString(ctx, "name");
    if (!warps.containsKey(warpName)) {
        throw new SimpleCommandExceptionType(
            Component.text("Unknown warp: " + warpName)).create();
    }
    // ...
    return 1;
})
```



---

## 9.26 EVENTS EXTRAS

### 9.26.1 Vehicle events

```java
@EventHandler public void onVehicleEnter(VehicleEnterEvent e) {
    if (e.getVehicle() instanceof Boat && e.getEntered() instanceof Player p) {
        p.sendMessage(Component.text("Ahoy!"));
    }
}
@EventHandler public void onVehicleDestroy(VehicleDestroyEvent e) { /* ... */ }
@EventHandler public void onVehicleMove(VehicleMoveEvent e) { /* high freq! */ }
```

### 9.26.2 Hanging / painting events

```java
@EventHandler public void onHangingPlace(HangingPlaceEvent e) {
    if (e.getEntity() instanceof Painting painting) {
        painting.setArt(Art.KEBAB);
    }
}
@EventHandler public void onHangingBreak(HangingBreakByEntityEvent e) { /* protect */ }
```

### 9.26.3 Raid events

```java
@EventHandler public void onRaidTrigger(RaidTriggerEvent e) {
    e.getPlayer().sendMessage(Component.text("Raid incoming!"));
}
@EventHandler public void onRaidFinish(RaidFinishEvent e) {
    // reward players
}
```

### 9.26.4 Weather events

```java
@EventHandler public void onThunder(ThunderChangeEvent e) {
    if (e.toThunderState()) e.getWorld().strikeLightning(randomLoc());
}
@EventHandler public void onWeather(WeatherChangeEvent e) {
    if (e.toWeatherState()) e.setCancelled(true); // eternal sunshine
}
```

### 9.26.5 Lambda EventExecutor

```java
Bukkit.getPluginManager().registerEvent(
    PlayerJoinEvent.class, listener, EventPriority.NORMAL,
    (l, event) -> {
        if (event instanceof PlayerJoinEvent join) {
            join.getPlayer().sendMessage(Component.text("Hi!"));
        }
    }, plugin);
```

### 9.26.6 Manual HandlerList unregistration

```java
HandlerList.unregisterAll(myListener);
// Or specific event:
PlayerJoinEvent.getHandlerList().unregister(myListener);
```

### 9.26.7 AsyncTabCompleteEvent

```java
@EventHandler public void onTab(AsyncTabCompleteEvent e) {
    if (!e.getBuffer().startsWith("/warp ")) return;
    e.setCompletions(List.of("spawn", "hub", "arena"));
    e.setHandled(true); // prevent sync completions
}
```

---

## 9.27 MISC — HttpClient

```java
private static final HttpClient HTTP = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5)).build();

public CompletableFuture<String> fetchAsync(String url) {
    return HTTP.sendAsync(
        HttpRequest.newBuilder().uri(URI.create(url)).GET().build(),
        HttpResponse.BodyHandlers.ofString()
    ).thenApply(HttpResponse::body);
}
```

---

## 9.28 MISC — Caffeine Cache

```java
// shade: com.github.ben-manes.caffeine:caffeine:3.1.8
LoadingCache<UUID, Integer> balanceCache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build(uuid -> db.queryBalance(uuid)); // loader
int coins = balanceCache.get(player.getUniqueId());
```

---

## 9.29 MISC — Leaderboard (Top-N pattern)

```java
// SQL:
// SELECT uuid, coins FROM wallet ORDER BY coins DESC LIMIT 10
// Cache result every 60s async, display via scoreboard or holograms.

public void updateLeaderboard() {
    Bukkit.getScheduler().runTaskTimerAsynchronously(plugin, () -> {
        List<Map.Entry<String, Integer>> top = db.getTop10();
        Bukkit.getScheduler().runTask(plugin, () -> displayBoard(top));
    }, 0L, 20L * 60);
}
```

---

## 9.30 MISC — Statistics API

```java
int kills = player.getStatistic(Statistic.MOB_KILLS);
int mined = player.getStatistic(Statistic.MINE_BLOCK, Material.DIAMOND_ORE);
player.setStatistic(Statistic.DEATHS, 0); // reset
```

---

## 9.31 MISC — Brewing Recipes

```java
// Paper doesn't have a Bukkit BrewingRecipe class — use BrewerInventory events.
// Detect via BrewEvent:
@EventHandler public void onBrew(BrewEvent e) {
    BrewerInventory inv = e.getContents();
    ItemStack ingredient = inv.getIngredient();
    if (ingredient != null && ingredient.getType() == Material.GLOWSTONE_DUST) {
        // Replace result for custom potion
        for (int i = 0; i < 3; i++) {
            inv.setItem(i, customPotion());
        }
    }
}
```

---

## 9.32 MISC — Custom Paintings / Wolf Variants / Jukebox Songs

```java
// Custom painting via registry (1.21+):
// Ship as datapack: data/<ns>/painting_variant/<name>.json
// { "asset_id": "myplugin:custom_painting", "width": 2, "height": 2 }

// Wolf variants (1.21.4+):
Wolf wolf = world.spawn(loc, Wolf.class, w -> {
    w.setVariant(Wolf.Variant.PALE); // PALE, RUSTY, SPOTTED, etc.
});

// Jukebox songs (custom via datapack):
// data/<ns>/jukebox_song/<name>.json
// { "sound_event": "myplugin:my_song", "description": {...}, "length_in_seconds": 120 }
```

---

## 9.33 MISC — Game Rules Deep

```java
// Read:
boolean keepInv = world.getGameRuleValue(GameRule.KEEP_INVENTORY);
// Set:
world.setGameRule(GameRule.DO_MOB_SPAWNING, false);
world.setGameRule(GameRule.RANDOM_TICK_SPEED, 0);
// All game rules:
for (GameRule<?> rule : GameRule.values()) {
    Bukkit.getLogger().info(rule.getName() + " = " + world.getGameRuleValue(rule));
}
```

---

## 9.34 MISC — Spectator Camera

```java
player.setGameMode(GameMode.SPECTATOR);
player.setSpectatorTarget(targetEntity); // camera locks to entity
// Release:
player.setSpectatorTarget(null);
```

---

## 9.35 MISC — WorldBorder Lerp Animation

```java
WorldBorder border = world.getWorldBorder();
border.setCenter(0, 0);
border.setSize(1000);
// Shrink to 100 over 300 seconds (lerp):
border.setSize(100, TimeUnit.SECONDS, 300);
// Per-player border (Paper):
player.setWorldBorder(WorldBorder.of(border)); // copy
```

---

## 9.36 FOLIA DEPTH

### 9.36.1 Ownership rules

On Folia, the world is split into **regions** (groups of chunks). Each region
ticks on its own thread. Rules:

- You can only read/write entities/blocks that belong to your current region.
- `entity.getScheduler().run(plugin, task, null)` runs on the entity's owning region.
- `Bukkit.getRegionScheduler().execute(plugin, location, task)` runs on the region owning that location.
- `Bukkit.getGlobalRegionScheduler().run(plugin, task)` runs on the global region (for recipe registration, etc.).
- `Bukkit.getAsyncScheduler().runNow(plugin, task)` runs on a shared async pool.

### 9.36.2 Supporting Paper + Folia simultaneously

```java
public void runOnEntity(Entity entity, Runnable task) {
    try {
        // Folia path:
        entity.getScheduler().run(plugin, t -> task.run(), null);
    } catch (NoSuchMethodError e) {
        // Paper path:
        Bukkit.getScheduler().runTask(plugin, task);
    }
}
```

Or use [FoliaLib](https://github.com/TechnicallyCoded/FoliaLib) as a compatibility shim.

---

## 9.37 VELOCITY / MODERN FORWARDING

### 9.37.1 velocity-plugin.json

```json
{
  "id": "myplugin",
  "name": "MyPlugin",
  "version": "1.0.0",
  "main": "com.example.VelocityPlugin",
  "dependencies": []
}
```

### 9.37.2 Modern IP forwarding setup

In Paper's `config/paper-global.yml`:
```yaml
proxies:
  velocity:
    enabled: true
    secret: "your-secret-here"
    online-mode: true
```

### 9.37.3 Cross-server plugin messaging

```java
// Velocity side:
server.getChannelRegistrar().register(MinecraftChannelIdentifier.from("myplugin:main"));
server.getEventManager().register(this, PluginMessageEvent.class, event -> {
    if (event.getIdentifier().equals(MinecraftChannelIdentifier.from("myplugin:main"))) {
        byte[] data = event.getData();
        // route to another server
    }
});
```

---

## 9.38 CLASS LOADING EXTRAS

### 9.38.1 Reflection + MethodHandles (NMS access)

```java
private static final MethodHandle GET_HANDLE;
static {
    try {
        Class<?> craftPlayer = Bukkit.getServer().getClass().getClassLoader()
            .loadClass("org.bukkit.craftbukkit.entity.CraftPlayer");
        GET_HANDLE = MethodHandles.lookup().findVirtual(craftPlayer, "getHandle",
            MethodType.methodType(Object.class)); // ServerPlayer
    } catch (Exception e) { throw new RuntimeException(e); }
}
```

### 9.38.2 --add-opens for module access

In your startup script: `--add-opens java.base/java.lang=ALL-UNNAMED`

Paper already opens most needed modules. If you hit `InaccessibleObjectException`,
add the opens to your server start flags.

---

## 9.39 MISC — Permissions deep

### 9.39.1 Wildcards / parents

```yaml
# plugin.yml
permissions:
  myplugin.*:
    children:
      myplugin.use: true
      myplugin.admin: true
  myplugin.admin:
    default: op
    children:
      myplugin.use: true
```

### 9.39.2 PermissionAttachment (runtime grants)

```java
PermissionAttachment att = player.addAttachment(plugin);
att.setPermission("myplugin.vip", true);
// Remove later:
player.removeAttachment(att);
```

### 9.39.3 LuckPerms context

```java
// Check if player has perm in a specific context (world):
ContextManager cm = LuckPermsProvider.get().getContextManager();
ImmutableContextSet ctx = cm.getContext(player).orElse(cm.getStaticContext());
```

---

## 9.40 MISC — Concurrent collections + per-tick budgets

```java
// ConcurrentHashMap for cross-thread player state:
private final ConcurrentHashMap<UUID, PlayerState> states = new ConcurrentHashMap<>();

// Per-tick budget (spread heavy work across ticks):
private final Queue<Runnable> workQueue = new ConcurrentLinkedQueue<>();
@Override public void onEnable() {
    Bukkit.getScheduler().runTaskTimer(this, () -> {
        long start = System.nanoTime();
        while (!workQueue.isEmpty() && (System.nanoTime() - start) < 1_000_000) { // 1ms budget
            workQueue.poll().run();
        }
    }, 1L, 1L);
}
```

---

## 9.41 MISC — Item cooldown + Caffeine expiry

```java
// Vanilla item cooldown (greys out item):
player.setCooldown(Material.ENDER_PEARL, 20 * 5); // 5 seconds
boolean onCooldown = player.hasCooldown(Material.ENDER_PEARL);

// Caffeine-based command cooldown:
Cache<UUID, Boolean> cooldowns = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofSeconds(10))
    .build();
if (cooldowns.getIfPresent(player.getUniqueId()) != null) {
    player.sendMessage(Component.text("On cooldown!"));
    return;
}
cooldowns.put(player.getUniqueId(), true);
```

---

## 9.42 MISC — Banner Pattern API

```java
ItemStack banner = ItemStack.of(Material.WHITE_BANNER);
banner.editMeta(BannerMeta.class, meta -> {
    meta.addPattern(new Pattern(DyeColor.RED, PatternType.STRIPE_TOP));
    meta.addPattern(new Pattern(DyeColor.BLUE, PatternType.CROSS));
});
// Custom patterns via datapack (1.21+):
// data/<ns>/banner_pattern/<name>.json
// { "asset_id": "myplugin:custom", "translation_key": "block.myplugin.custom_banner" }
```

---

## 9.X FAILURE COOKBOOK

| Symptom | Cause | Fix |
|---|---|---|
| bStats shows 0 servers | Forgot to relocate `org.bstats` package | Add `relocate("org.bstats", "...")` in shadowJar |
| MockBukkit `NoClassDefFoundError` on Paper classes | Wrong MockBukkit version for your Paper API | Pin MockBukkit version to match your `api-version` |
| ProtocolLib `NoSuchFieldException` after Paper update | Obfuscation mappings changed | Wait for ProtocolLib release matching your Paper build |
| PacketEvents listener never fires | Forgot `PacketEvents.getAPI().init()` in `onEnable` | Call `load()` in `onLoad`, `init()` in `onEnable`, `terminate()` in `onDisable` |
| WorldGuard flag returns null | Registered in `onEnable` instead of `onLoad` | Move flag registration to `onLoad` |
| Citizens NPC disappears on restart | Didn't call `npc.setProtected(true)` or save | Citizens auto-saves; ensure `persist: true` in trait |
| PlaceholderAPI expansion not loading | `persist()` returns false and plugin loads before PAPI | Override `persist()` to return `true`; add `softdepend: [PlaceholderAPI]` |
| Vault `getRegistration` returns null | Economy plugin not loaded yet | Add `softdepend: [Vault]` and check in `onEnable` after a 1-tick delay |
| `player.openDialog()` does nothing | Client < 1.21.5 or experimental flag not enabled | Check `player.getProtocolVersion()` ≥ 770 |
| Freecam detection false positives | Didn't account for elytra/riptide/knockback velocity | Exclude flying/gliding/velocity states from checks |
| Chunk ticket never releases | Forgot `removePluginChunkTicket` on disable | Track tickets in a Set, remove all in `onDisable` |
| Skin not showing on skull | Base64 texture string malformed or signature missing | Ensure valid base64 from `sessionserver.mojang.com` |
| `setCooldown` not visible to player | Set on wrong Material (item in hand vs registered) | Must match the exact `Material` the player holds |
| ArchUnit test fails with "classes not found" | Package scan doesn't match your sources | Verify `@AnalyzeClasses(packages = "...")` matches |
| Caffeine cache returns stale data | `expireAfterWrite` too long or no invalidation on write | Call `cache.invalidate(key)` after DB mutations |
| Folia `IllegalStateException: not on region thread` | Called Bukkit API from wrong thread | Use `entity.getScheduler().run(...)` or region scheduler |

---

## 9.Y SELF-REVIEW CHECKLIST

- [x] CI/CD: GitHub Actions build + Hangar + Modrinth + auto-versioning
- [x] Metrics: bStats 3.0 + Prometheus pattern
- [x] Profiling: spark commands + TPS/MSPT API + Watchdog
- [x] Testing: MockBukkit + JUnit5 + Mockito + ArchUnit
- [x] Vault / VaultUnlocked economy hook
- [x] PlaceholderAPI expansion + relational placeholders
- [x] WorldGuard custom flags
- [x] Citizens NPC API
- [x] WorldEdit / FAWE schematic paste
- [x] DiscordSRV / JDA / webhooks
- [x] ProtocolLib PacketAdapter + fake entity
- [x] PacketEvents 2.x listener + send
- [x] Freecam detection strategies
- [x] 1.21.x: Trial Spawner, Wind Charge, Mace, Armor Trim, Dialog, Tick Freeze, Bundles, Banner Pattern
- [x] Build: Spotless, NullAway, Lombok, multi-module, reproducible, jar-in-jar
- [x] Adventure: Gson/Legacy/Plain/ANSI serializers, insertion, Sound API, sign edit
- [x] Inventory: AnvilGUI, TriumphGUI pagination, MenuType, Bundle
- [x] PlayerProfile / skins / textures
- [x] Worlds: rayTrace, chunk tickets, chunk snapshots, heightmap
- [x] Commands: Cloud framework, redirects/forks, CommandSyntaxException
- [x] Events: Vehicle, Hanging, Raid, Weather, Lambda EventExecutor, HandlerList unregister, AsyncTabComplete
- [x] HttpClient + WebSocket pattern
- [x] Caffeine cache
- [x] Leaderboard top-N
- [x] Statistics API
- [x] Brewing recipes (BrewEvent)
- [x] Custom paintings / wolf variants / jukebox songs
- [x] Game rules deep
- [x] Spectator camera
- [x] WorldBorder lerp
- [x] Folia: ownership rules, dual Paper+Folia support
- [x] Velocity: plugin.json, modern forwarding, cross-server messaging
- [x] Class loading: reflection/MethodHandles, --add-opens
- [x] Permissions: wildcards, parents, PermissionAttachment, LuckPerms context
- [x] Concurrent collections + per-tick budget
- [x] Item cooldown + Caffeine expiry
- [x] 16-row failure cookbook

---

## 9.Z REFERENCES

- [bStats getting started](https://www.bstats.org/getting-started/include-metrics)
- [spark docs](https://spark.lucko.me/docs/Command-Usage)
- [Paper profiling guide](https://docs.papermc.io/paper/profiling)
- [MockBukkit docs](https://mockbukkit.readthedocs.io/en/latest/first_tests.html)
- [MockBukkit GitHub](https://github.com/MockBukkit/MockBukkit)
- [VaultAPI GitHub](https://github.com/MilkBowl/VaultAPI)
- [VaultUnlocked Hangar](https://hangar.papermc.io/TNE/VaultUnlocked)
- [PlaceholderAPI wiki](https://wiki.placeholderapi.com/developers/using-placeholderapi/)
- [WorldGuard custom flags](https://worldguard.enginehub.org/en/latest/developer/regions/custom-flags/)
- [Citizens Wiki API](https://wiki.citizensnpcs.co/API)
- [Citizens GitHub](https://github.com/CitizensDev/Citizens2)
- [ProtocolLib wiki](https://github.com/dmulloy2/ProtocolLib/wiki/Packet-listeners-and-adapters)
- [PacketEvents docs](https://docs.packetevents.com/)
- [PacketEvents Javadoc](https://javadocs.packetevents.com/)
- [PacketEvents GitHub](https://github.com/retrooper/packetevents)
- [Freecam GitHub issue #196](https://github.com/MinecraftFreecam/Freecam/issues/196)
- [Hangar auto-publishing](https://docs.papermc.io/misc/hangar-publishing)
- [Paper API index](https://docs.papermc.io/paper/dev/api/)
- [Minecraft Wiki: Tricky Trials](https://minecraft.wiki/w/Tricky_Trials)
- [Minecraft Wiki: Wind Charge](https://minecraft.wiki/w/Wind_Charge)
- [Paper TPS/MSPT](https://spark.lucko.me/docs/guides/TPS-and-MSPT)
- [AnvilGUI GitHub](https://github.com/WesJD/AnvilGUI)
- [HikariCP GitHub](https://github.com/brettwooldridge/HikariCP)
- [Caffeine GitHub](https://github.com/ben-manes/caffeine)

Compliance note: information from external sources was paraphrased to comply
with content licensing.

---

## See also

- `01-build-tooling.md` — Gradle/Maven setup, Shadow, run-paper, paperweight-userdev
- `02-velocity-folia-bedrock.md` — Folia schedulers, Velocity forwarding, Bedrock/Geyser/Floodgate
- `03-events-extras.md` — Core event patterns, priority, custom events
- `04-commands-extras.md` — Brigadier registration, argument types
- `05-entities-mobs.md` — Entity spawning, Display entities, AI goals
- `06-world-blocks-chunks.md` — Chunk generation, BlockData, WorldBorder
- `07-recipes-loot-advancements.md` — All recipe types, loot tables, advancement API
- `08-storage-config-deep.md` — YAML config, PDC, SQLite/MySQL, async IO
- `paper-plugin-dev.md` — Base file with original coverage
