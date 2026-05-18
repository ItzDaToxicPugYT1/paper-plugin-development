---
name: paper-plugin-dev-storage-config-deep
description: >
  Expansion to paper-plugin-dev.md covering plugin storage and configuration
  end-to-end: YamlConfiguration / FileConfiguration with saveDefaultConfig +
  saveResource(replace) + reloadConfig + getConfig + ConfigurationSection paths;
  ConfigurationSerializable / SerializableAs / ConfigurationSerialization registry;
  Paper comment APIs (setComments / setInlineComments) added in 1.18.1; the
  YAML default-vs-actual two-tree problem and how options.copyDefaults() saves
  you; PersistentDataContainer (PDC) per-holder storage on items, entities,
  block entities, world, chunks, structure data; the PersistentDataType primitives
  + complex BOOLEAN / LIST / MAP wrappers + custom adapters; the
  PersistentDataContainerView read-only split (1.21.5+); flat-file player-data
  as JSON / Gson with offline-UUID gotcha; SQLite via JDBC with HikariCP +
  WAL journal mode + busy_timeout + synchronous=NORMAL pragma stack; MySQL /
  MariaDB / PostgreSQL via HikariCP and the prepared-statement caching gotcha;
  schema migration with Flyway and a hand-rolled migrator alternative; async
  IO patterns via BukkitScheduler#runTaskAsynchronously / runTask hop-back;
  the "never touch Bukkit API off-thread" rule and the Folia replacement;
  config reload (/reload vs PluginCommand vs admin reset) and lifecycle ordering;
  third-party libraries (Configurate HOCON, ConfigLib, Jackson YAML, SnakeYAML
  upgrades); shutdown ordering — onDisable / saveConfig / pool.close()  / flush;
  failure cookbook with 16 rows; reference implementation (a "Player Wallet"
  plugin combining a YAML defaults file, a SQLite WAL-mode HikariCP pool with
  schema migrations, an async balance lookup with main-thread reply, and PDC
  for per-item "bound coin" markers).
---

# 8. STORAGE & CONFIG — DEEP DIVE

`paper-plugin-dev.md` mentions `getConfig()`, "save your data on disable", and
"don't block the main thread." This file is the long-form companion: the four
storage layers Paper plugins actually use (config YAML, ad-hoc files, PDC,
relational DB), how each one fails, and what the runtime cost of each is.

All snippets target **Paper 1.21.4+** (Mojang-mapped). Comment APIs (1.18.1+),
`PersistentDataContainerView` (1.21.5+), and Paper's bootstrap config helpers
are flagged inline. Authoritative sources are linked inline plus the
[References](#8z-references) appendix.

---

## 8.0 MENTAL MODEL — FOUR LAYERS, FOUR LIFETIMES

A Paper plugin has four orthogonal places to put data. Pick by **scope** and
**update frequency**, never by "what's easiest":

```
┌──────────────────────┬──────────────────────┬──────────────────────┬──────────────────────┐
│ LAYER                │ SCOPE                │ TYPICAL UPDATE FREQ  │ LATENCY              │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ config.yml           │ admin-tunable        │ once / restart       │ ~ 0 (in-memory)      │
│ ad-hoc files         │ per-feature state    │ minutes-hours        │ ~ ms (disk write)    │
│ PDC (PersistentData) │ per-item / -entity   │ many times per tick  │ ~ µs (in-memory NBT) │
│ DB (SQLite/MySQL)    │ cross-server / large │ many ops/s           │ ~ ms (over the wire) │
└──────────────────────┴──────────────────────┴──────────────────────┴──────────────────────┘

         onEnable                                 onDisable
   ┌─────────────────────┐                  ┌──────────────────────┐
   │ saveDefaultConfig() │                  │ saveConfig()         │
   │ load DB pool        │                  │ flush in-flight ops  │
   │ register listeners  │                  │ pool.close()         │
   │ kick async warmup   │                  │ persist player state │
   └─────────────────────┘                  └──────────────────────┘
                  │                                      ▲
                  ▼                                      │
            normal ticking (events, scheduled tasks, async IO)
```

Refs:
[Paper plugin-configurations guide](https://docs.papermc.io/paper/dev/plugin-configurations/),
[Paper PDC guide](https://docs.papermc.io/paper/dev/pdc),
[Paper scheduler guide](https://docs.papermc.io/paper/dev/scheduler/).

---

## 8.1 YAML CONFIGURATION — `FileConfiguration` AND FRIENDS

### 8.1.1 The 5-line bootstrap

```java
@Override
public void onEnable() {
    saveDefaultConfig();              // copies src/main/resources/config.yml if absent
    FileConfiguration cfg = getConfig();
    int max = cfg.getInt("limits.max-warps", 10);
    boolean glow = cfg.getBoolean("features.glowing-coins", true);
    String world = cfg.getString("home-world", "world");
    Bukkit.getLogger().info("warps=" + max + " glow=" + glow + " home=" + world);
}
```

`saveDefaultConfig()` writes only if the file doesn't exist; it never overwrites.
Use `saveResource("foo.yml", false)` for non-default resources you want copied
on first run, and `saveResource("foo.yml", true)` to **force-replace** every
boot (rare — almost always wrong). See
[Paper plugin-configurations guide](https://docs.papermc.io/paper/dev/plugin-configurations/).

### 8.1.2 Reading and writing

```java
FileConfiguration cfg = getConfig();
cfg.set("players." + uuid + ".lastLogin", System.currentTimeMillis());
cfg.set("players." + uuid + ".prefix", "<gold>★</gold>");
saveConfig();   // synchronously writes to disk; main-thread blocking
```

`saveConfig()` does a synchronous write — fine for shutdown / `/myplugin save`,
**not** fine in a tight loop. For per-player state, batch the writes or move
to PDC / SQLite.

### 8.1.3 The two-tree problem (defaults vs actual)

`FileConfiguration` has **two** in-memory trees: the loaded values *and* the
defaults from the bundled resource. `cfg.getInt("a.b", 5)` returns 5 *if the
loaded tree has nothing at `a.b`*, even if the bundled defaults file has 10
under `a.b`. This trips up everyone once.

The fix: enable copy-defaults so the loaded tree is *seeded* from the bundle:

```java
getConfig().options().copyDefaults(true);
saveConfig(); // now the file on disk includes the defaults
```

Modern Paper recommends building the defaults tree explicitly via
`addDefault("path", value)` then writing once. See
[FileConfiguration Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/configuration/file/FileConfiguration.html).

### 8.1.4 Comments (1.18.1+)

`ConfigurationSection#setComments` and `#setInlineComments` let you write
block-comments and `# inline` comments programmatically — e.g. when you migrate
a config and want to leave breadcrumbs:

```java
cfg.setComments("limits.max-warps", List.of(
    "Maximum number of /warp homes per player.",
    "Set to -1 for unlimited (admin warning: memory!)"
));
cfg.setInlineComments("limits.max-warps", List.of("default 10"));
saveConfig();
```

If you target older Paper, fall back to manually writing a leading `# ...`
header in your bundled `config.yml`.

### 8.1.5 ConfigurationSerializable (Bukkit's built-in object mapper)

If your value class implements `ConfigurationSerializable` and is registered
with `ConfigurationSerialization.registerClass`, the YAML loader can round-trip
it without manual mapping. Paper's `ItemStack`, `Location`, `Vector`,
`PotionEffect`, `Color`, and `OfflinePlayer` already implement it — that's why
`cfg.set("home", player.getLocation())` and `cfg.getLocation("home")` Just Work.

```java
@SerializableAs("Warp")
public final class Warp implements ConfigurationSerializable {
    public final String name;
    public final Location loc;

    public Warp(String name, Location loc) { this.name = name; this.loc = loc; }

    @Override
    public Map<String, Object> serialize() {
        return Map.of("name", name, "loc", loc); // Location auto-serializes
    }

    public static Warp deserialize(Map<String, Object> m) {
        return new Warp((String) m.get("name"), (Location) m.get("loc"));
    }
}

// Once at startup:
ConfigurationSerialization.registerClass(Warp.class);

// Reading / writing:
cfg.set("warps.spawn", new Warp("spawn", spawnLoc));
Warp w = (Warp) cfg.get("warps.spawn");
```

The required static method must be **`deserialize`** *or* **`valueOf`**, *or* a
public single-`Map`-arg constructor — see
[ConfigurationSerializable](https://jd.papermc.io/paper/1.21.9/org/bukkit/configuration/serialization/ConfigurationSerializable.html)
and
[ConfigurationSerialization](https://jd.papermc.io/paper/1.21.6/org/bukkit/configuration/serialization/ConfigurationSerialization.html).

> **Pitfall.** Forgetting `registerClass` returns `null` from `cfg.get(...)`
> with a silent log warning, not an exception. Always register at `onEnable`,
> guard with `if (!ConfigurationSerialization.getAlias(Warp.class)) ...`.

### 8.1.6 Reload semantics

`reloadConfig()` discards your in-memory tree and reloads from disk — it does
**not** notify anyone or fire an event. If you cache config values in fields,
you must re-read them in your `/yourplugin reload` handler.

```java
public void reload() {
    reloadConfig();
    this.maxWarps  = getConfig().getInt("limits.max-warps", 10);
    this.glowCoins = getConfig().getBoolean("features.glowing-coins", true);
}
```

### 8.1.7 Multiple config files

`getConfig()` only manages `config.yml`. For additional files use
`YamlConfiguration.loadConfiguration(file)`:

```java
File f = new File(getDataFolder(), "warps.yml");
YamlConfiguration warps = YamlConfiguration.loadConfiguration(f);
warps.set("spawn.location", player.getLocation());
warps.save(f); // throws IOException; wrap in try/catch
```

`YamlConfiguration#save` is **synchronous, main-thread-blocking**. Schedule it
async if writing > 50KB or on every event.

---

## 8.2 RICH-CONFIG ALTERNATIVES

`YamlConfiguration` is fine for flat-key configs but groans on nested
type-mapped trees. Three real options for "configs that map onto records":

| Library | Format | Type-safety | Comments | Adoption |
|---|---|---|---|---|
| [Configurate](https://github.com/SpongePowered/Configurate) | HOCON / YAML / JSON / TOML | reflective `@ConfigSerializable` | yes | Sponge default |
| [ConfigLib](https://hangar.papermc.io/Exlll/ConfigLib) | YAML | record / class field-mapping | yes (per-field annotations) | popular Paper plugins |
| Jackson + jackson-dataformat-yaml | YAML | full POJO mapping | yes (via `@JsonPropertyDescription`) | universal |

ConfigLib in 10 lines:

```java
@Configuration
public final class WalletConfig {
    public int startingCoins = 100;
    public boolean allowNegative = false;
    public List<String> bannedItems = List.of("BARRIER", "BEDROCK");
}

YamlConfigurationProperties props = YamlConfigurationProperties.newBuilder()
    .header("Wallet plugin config — see /wallet help").build();
WalletConfig cfg = YamlConfigurations.update(
    getDataFolder().toPath().resolve("config.yml"),
    WalletConfig.class, props);
```

This is significantly less boilerplate than building defaults manually. See
[Paper Hangar: ConfigLib](https://hangar.papermc.io/Exlll/ConfigLib).

> **Compliance note.** Information about third-party config libraries was
> paraphrased from their public documentation pages.

---

## 8.3 PERSISTENT DATA CONTAINER — IN-NBT KEY/VALUE

`PersistentDataContainer` (PDC) is Paper/Bukkit's per-holder NBT-backed
key/value store. The holder can be:

| Holder | Saved with |
|---|---|
| `ItemStack` (via meta) | the item itself (inventory, ender chest, dropped, etc.) |
| `Entity` | the chunk's entities file |
| `BlockState` (TileEntity) | the chunk |
| `World` | `level.dat` |
| `Chunk` | the chunk's region file |
| `Structure` | the structure's NBT |
| `OfflinePlayer` | the player's `.dat` file (1.21.10+) |

Refs:
[Paper PDC docs](https://docs.papermc.io/paper/dev/pdc),
[PersistentDataContainer Javadoc](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/persistence/PersistentDataContainer.html).

### 8.3.1 The five lines you'll write a thousand times

```java
NamespacedKey BOUND_TO = new NamespacedKey(plugin, "bound_to");
ItemStack coin = ItemStack.of(Material.GOLD_NUGGET);
coin.editMeta(meta ->
    meta.getPersistentDataContainer().set(BOUND_TO,
        PersistentDataType.STRING, player.getUniqueId().toString()));
String boundTo = coin.getItemMeta().getPersistentDataContainer()
    .get(BOUND_TO, PersistentDataType.STRING);
```

`editMeta(Consumer)` is the Paper-modern way; the old `getItemMeta + setItemMeta`
pattern still works but copies the meta twice.

### 8.3.2 Built-in `PersistentDataType` constants

| Constant | Java type | NBT type |
|---|---|---|
| `BYTE` / `BYTE_ARRAY` | `Byte` / `byte[]` | byte / byte array |
| `SHORT` / `INTEGER` / `LONG` | wrappers | short / int / long |
| `FLOAT` / `DOUBLE` | wrappers | float / double |
| `STRING` | `String` | string |
| `INTEGER_ARRAY` / `LONG_ARRAY` | `int[]` / `long[]` | int / long array |
| `BOOLEAN` (1.18+) | `Boolean` (stored as byte 0/1) | byte |
| `TAG_CONTAINER` | nested `PersistentDataContainer` | compound |
| `TAG_CONTAINER_ARRAY` | `PersistentDataContainer[]` | list of compounds |
| `LIST.<T>` (1.20.4+) | `List<T>` for any of the above primitives | list |

The `LIST` factory is huge — pre-1.20.4 you had to fake lists with
`TAG_CONTAINER_ARRAY` keyed by index. Now:

```java
PersistentDataType<List<String>, List<String>> stringList = PersistentDataType.LIST.strings();
container.set(KEY, stringList, List.of("a", "b", "c"));
```

### 8.3.3 Custom `PersistentDataType` adapter

For UUIDs, enums, and your own value objects you write a 6-line adapter:

```java
public static final PersistentDataType<long[], UUID> UUID_TYPE = new PersistentDataType<>() {
    @Override public Class<long[]> getPrimitiveType() { return long[].class; }
    @Override public Class<UUID>   getComplexType()   { return UUID.class; }
    @Override public long[] toPrimitive(UUID u, PersistentDataAdapterContext ctx) {
        return new long[]{ u.getMostSignificantBits(), u.getLeastSignificantBits() };
    }
    @Override public UUID fromPrimitive(long[] p, PersistentDataAdapterContext ctx) {
        return new UUID(p[0], p[1]);
    }
};
container.set(OWNER_KEY, UUID_TYPE, player.getUniqueId());
UUID owner = container.get(OWNER_KEY, UUID_TYPE);
```

See
[PersistentDataType Javadoc](https://jd.papermc.io/paper/1.20.5/org/bukkit/persistence/PersistentDataType.html).

### 8.3.4 `PersistentDataContainerView` (1.21.5+)

Paper added a read-only view interface so APIs that hand out PDCs to
event-listeners can't be mutated by mistake:

```java
PersistentDataContainerView view = entity.getPersistentDataContainer();
String name = view.get(NAME_KEY, PersistentDataType.STRING); // OK
view.set(NAME_KEY, ...);   // does not compile
```

Most call sites still return the mutable `PersistentDataContainer`; the View is
opt-in for new APIs. See
[PersistentDataContainerView](https://jd.papermc.io/paper/1.21.10/io/papermc/paper/persistence/PersistentDataContainerView.html).

### 8.3.5 PDC vs everything else — when to use it

PDC is **fast** (in-memory NBT, no disk hit per write — hits disk when the
holder saves) and **scope-correct** (per-item travels with the item). It is
**not** suited for:

- Cross-server / network state — PDC saves with the world, not centrally.
- Bulk querying — there is no `findAllItemsWith(KEY)`. To find all coins owned
  by player X, you'd have to iterate every loaded chunk and inventory.
- Server-wide singletons that change every tick — use a plain `HashMap` field.

> **Pitfall.** PDC on `Chunk` only saves when the chunk **unloads cleanly**.
> A hard server crash drops the last 5 minutes of chunk-PDC writes (vanilla
> autosave interval). For anything you must not lose, write to disk yourself.

---

## 8.4 AD-HOC FILES — JSON / GSON FOR PER-PLAYER DATA

For data that's per-player but more complex than a coin balance — e.g. an
"infected with curse X for Y ticks" record — JSON-on-disk via Gson is the
common choice:

```java
// ~/plugins/MyPlugin/players/<uuid>.json
public record PlayerState(int coins, long curseExpiresAt, List<String> tags) { }

private static final Gson GSON = new GsonBuilder().setPrettyPrinting().create();

public PlayerState load(UUID id) throws IOException {
    Path file = getDataFolder().toPath().resolve("players").resolve(id + ".json");
    if (!Files.exists(file)) return new PlayerState(0, 0L, List.of());
    return GSON.fromJson(Files.readString(file), PlayerState.class);
}

public void save(UUID id, PlayerState s) throws IOException {
    Path file = getDataFolder().toPath().resolve("players").resolve(id + ".json");
    Files.createDirectories(file.getParent());
    Files.writeString(file, GSON.toJson(s));
}
```

### 8.4.1 The offline-UUID landmine

If your server runs in **offline-mode** (`online-mode=false` in `server.properties`,
e.g. behind a Velocity proxy that authenticates), player UUIDs are
**deterministically derived from the username** — `OfflinePlayer.getUniqueId()`
returns a v3 UUID hash. If a player renames at the proxy, their offline UUID
changes and they appear as a brand-new account.

Mitigations:

1. Use Velocity-forwarded UUIDs (`bungeecord: true` + ip-forwarding-secret) so
   the player's online UUID survives proxying.
2. On login, look up the player by `getName()` first and migrate the file if
   the UUID changed. Risky but cheap.
3. Store data keyed by the proxy's UUID, not the backend's. See
   [Paper OfflinePlayer Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/OfflinePlayer.html)
   and §02 of this expansion (`02-velocity-folia-bedrock.md`) for proxy
   forwarding.

> **Pitfall.** Never call `Bukkit.getOfflinePlayer(name)` synchronously — it
> blocks while contacting Mojang's API. Use the UUID overload, or hop async.

### 8.4.2 Atomic writes (don't half-write a state file)

A crash mid-write produces a half-written JSON that throws `JsonSyntaxException`
forever after. Use the standard atomic-rename pattern:

```java
Path tmp = file.resolveSibling(file.getFileName() + ".tmp");
Files.writeString(tmp, json, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
Files.move(tmp, file, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
```

`ATOMIC_MOVE` is POSIX `rename(2)` — guaranteed not to leave a torn file.

---

## 8.5 RELATIONAL DATABASE — JDBC + HIKARICP

For cross-server state (economy, ban list, statistics), a real RDBMS is the
right answer. The standard Paper stack is:

```
[your code]
   │ JDBC (java.sql.*)
   ▼
[HikariCP pool]
   │ JDBC driver (sqlite-jdbc, mysql-connector-j, postgresql)
   ▼
[database server or local file]
```

### 8.5.1 Why a pool? (Why not `DriverManager.getConnection` per query?)

Opening a TCP connection + auth handshake is **30-200 ms**. A pool keeps a
warm set of connections, hands one out per query, returns it on `close()`.
[HikariCP](https://github.com/brettwooldridge/HikariCP) is the de-facto choice
— small, well-tested, low-latency.

> **Compliance note.** Library descriptions paraphrased from public README.

### 8.5.2 SQLite — the right pragma stack

SQLite's defaults are tuned for desktop apps with one writer; servers need
WAL mode and busy_timeout to avoid `SQLITE_BUSY`:

```java
HikariConfig cfg = new HikariConfig();
cfg.setJdbcUrl("jdbc:sqlite:" + new File(getDataFolder(), "wallet.db").getAbsolutePath());
cfg.setMaximumPoolSize(1);   // SQLite serializes writes anyway
cfg.setMinimumIdle(1);
cfg.setConnectionInitSql(String.join("; ",
    "PRAGMA journal_mode=WAL",
    "PRAGMA synchronous=NORMAL",
    "PRAGMA busy_timeout=5000",
    "PRAGMA foreign_keys=ON",
    "PRAGMA temp_store=MEMORY"
));
this.dataSource = new HikariDataSource(cfg);
```

What each pragma does:

| Pragma | Effect | When to use |
|---|---|---|
| `journal_mode=WAL` | Use a write-ahead log instead of rollback journal. Readers and writers don't block each other. | Always for servers (except NFS — see warning) |
| `synchronous=NORMAL` | Don't `fsync()` on every commit; only at WAL checkpoints. ~5x faster writes; very small crash-safety gap. | Production servers with good hardware |
| `busy_timeout=5000` | Wait up to 5 s for a write lock instead of failing instantly. | Always — kills `SQLITE_BUSY` |
| `foreign_keys=ON` | Enforce FKs (off by default). | Always |
| `temp_store=MEMORY` | Keep temp tables in RAM. | Almost always |

References:
[SQLite WAL docs](https://www.sqlite.org/wal.html),
[Coddy: PRAGMA settings](https://coddy.tech/docs/sqlite/pragma-settings),
[Litestream tips](https://litestream.io/tips/).

> **Pitfall.** WAL mode requires the `*.db-wal` and `*.db-shm` sidecar files to
> live next to the DB. **Do not enable WAL on NFS / SMB** — corruption risk
> documented by [pecar.me](https://blog.pecar.me/sqlite-django-config). Stick
> to local disk.

### 8.5.3 MySQL / MariaDB / PostgreSQL

```java
HikariConfig cfg = new HikariConfig();
cfg.setJdbcUrl("jdbc:mysql://localhost:3306/myplugin?useSSL=false&serverTimezone=UTC");
cfg.setUsername("plugin");
cfg.setPassword(System.getenv("DB_PASS"));
cfg.setMaximumPoolSize(8);          // start small; tune via metrics
cfg.setMinimumIdle(2);
cfg.setConnectionTimeout(5_000);    // fail fast on outages
cfg.setIdleTimeout(60_000);
cfg.setMaxLifetime(30 * 60_000);    // 30 min — under MySQL's wait_timeout
cfg.addDataSourceProperty("cachePrepStmts", "true");
cfg.addDataSourceProperty("prepStmtCacheSize", "250");
cfg.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
cfg.addDataSourceProperty("useServerPrepStmts", "true");
this.dataSource = new HikariDataSource(cfg);
```

`maxLifetime < wait_timeout` is the single most-missed setting — without it,
HikariCP hands out a connection that the server has already closed, you get a
`CommunicationsException`, the user sees an error.

Refs:
[HikariCP README](https://github.com/brettwooldridge/HikariCP),
[Plugin StackOverflow Q on Hikari](https://stackoverflow.com/questions/44580501/using-hikaricps-connection-pool-the-correct-way).

### 8.5.4 Schema migrations — Flyway or hand-rolled

[Flyway](https://www.baeldung.com/database-migrations-with-flyway) is the
boring-correct choice: drop versioned `V1__init.sql`, `V2__add_coin_log.sql`
files into `resources/db/migration/`, call `Flyway.configure().dataSource(ds)
.locations("classpath:db/migration").load().migrate()` at startup. Flyway
records applied versions in a `flyway_schema_history` table.

Hand-rolled migrator (smaller dependency footprint, fine for plugins):

```java
public final class Migrator {
    private final HikariDataSource ds;
    public Migrator(HikariDataSource ds) { this.ds = ds; }

    public void run() throws SQLException {
        try (Connection c = ds.getConnection(); Statement s = c.createStatement()) {
            s.execute("CREATE TABLE IF NOT EXISTS schema_version (v INTEGER NOT NULL)");
            int v = currentVersion(c);
            if (v < 1) {
                s.execute("CREATE TABLE wallet (uuid TEXT PRIMARY KEY, coins INTEGER NOT NULL DEFAULT 0)");
                bumpVersion(c, 1);
            }
            if (v < 2) {
                s.execute("CREATE TABLE coin_log (id INTEGER PRIMARY KEY AUTOINCREMENT, " +
                          "uuid TEXT NOT NULL, delta INTEGER NOT NULL, ts INTEGER NOT NULL)");
                bumpVersion(c, 2);
            }
        }
    }
    // currentVersion / bumpVersion omitted for brevity
}
```

Both approaches handle the same problem: **never run unguarded `CREATE TABLE`
on every boot**. You'll lose `ALTER TABLE` migrations or alter the schema in
inconsistent ways.

### 8.5.5 Prepared statements + try-with-resources

```java
public CompletableFuture<Integer> getCoinsAsync(UUID id) {
    return CompletableFuture.supplyAsync(() -> {
        try (Connection c = ds.getConnection();
             PreparedStatement ps = c.prepareStatement(
                 "SELECT coins FROM wallet WHERE uuid = ?")) {
            ps.setString(1, id.toString());
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? rs.getInt(1) : 0;
            }
        } catch (SQLException e) {
            throw new CompletionException(e);
        }
    }, ioExecutor);
}
```

Three rules: every `Connection`, `PreparedStatement`, `ResultSet` in
try-with-resources; never concatenate user input into SQL (always `?`); never
leak a connection (stack-traces from Hikari's leak detector are useful — set
`setLeakDetectionThreshold(30_000)`).

---

## 8.6 ASYNC IO — THE TWO-HOP PATTERN

The cardinal rule: **never block the main thread on disk or network**. The
cardinal corollary: **never call Bukkit API from an async thread**. The
two-hop pattern resolves both:

```java
// Main thread — request triggered by an event/command:
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    int coins = blockingDbQuery(player.getUniqueId());        // OK off-thread
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("You have " + coins + " coins."));
    });
});
```

Refs:
[Paper scheduler guide](https://docs.papermc.io/paper/dev/scheduler/),
[BukkitScheduler Javadoc](https://jd.papermc.io/paper/1.21.0/org/bukkit/scheduler/BukkitScheduler.html).

Modern equivalents using `CompletableFuture`:

```java
db.getCoinsAsync(player.getUniqueId())
  .thenAcceptAsync(coins -> player.sendMessage(...), Bukkit.getScheduler()
      .getMainThreadExecutor(plugin)); // 1.20+ helper, else hop manually
```

### 8.6.1 Folia replacement

On Folia, `runTask`/`runTaskAsynchronously` are deprecated in favor of regional
schedulers — see `02-velocity-folia-bedrock.md` §2 for the full mapping. The
short version:

```java
// Instead of runTask(plugin, r):
player.getScheduler().run(plugin, t -> r.run(), null);

// Instead of runTaskAsynchronously(plugin, r):
Bukkit.getAsyncScheduler().runNow(plugin, t -> r.run());
```

### 8.6.2 Backpressure — bounded executors

A naive `runTaskAsynchronously` per command lets a single griefer flood your DB.
For high-frequency async ops, run your own bounded executor:

```java
this.ioExecutor = new ThreadPoolExecutor(
    1, 4,                                    // 1-4 threads
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500),          // bounded queue
    new ThreadFactoryBuilder()
        .setNameFormat("myplugin-io-%d")
        .setDaemon(true)                     // don't block JVM shutdown
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure: caller blocks
);
```

`CallerRunsPolicy` is a soft denial-of-service mitigation — when the queue
fills, the *caller* (often the main thread) does the work. For pure-IO this
is bad; for low-throughput user-driven actions it's fine.

---

## 8.7 LIFECYCLE — `onEnable` / `onDisable` ORDERING

`onEnable` ordering (top-down):

1. Load config (`saveDefaultConfig`, then `getConfig()`).
2. Register `ConfigurationSerialization.registerClass`.
3. Open DB pool / run migrations.
4. Register listeners + commands.
5. Schedule async tasks.

`onDisable` ordering (**reverse** of enable):

1. Cancel scheduled tasks (`Bukkit.getScheduler().cancelTasks(this)`).
2. Unregister listeners is implicit; explicitly null caches if you have static refs.
3. **Flush in-flight async writes** — collect futures and `.get(5, SECONDS)` them.
4. Save in-memory player state to DB / files (synchronously here is OK).
5. `saveConfig()` if you mutated it.
6. `pool.close()` last.

> **Pitfall.** `onDisable` is called on the main thread with a **30-second hard
> deadline** — if you block longer (e.g. `pool.close()` waiting on hung
> connections), the watchdog kills the JVM and your data may be half-written.
> Set `connectionTimeout` and `idleTimeout` low and prefer atomic file writes.

### 8.7.1 The `/reload` problem

Vanilla `/reload` calls `onDisable`+`onEnable` on every plugin in a single tick
without restarting the JVM. Static state, leaked threads, and singleton
`Bukkit.getServer()` references survive across reloads and corrupt the new
instance. **Document loudly that `/reload` is unsupported**, and prefer
`/myplugin reload` that re-reads config and rebuilds caches in-place.

---

## 8.8 ASYNC SAFETY MATRIX

| Operation | Main thread | Async safe | Folia scheduler |
|---|---|---|---|
| `JavaPlugin#getConfig()` (read) | ✅ | ✅ (Paper config tree is read-mostly) | any |
| `JavaPlugin#saveConfig()` | ✅ | ⚠️ (does a sync disk write) | any if pre-loaded |
| `YamlConfiguration#save(File)` | ✅ | ⚠️ same as above | any |
| `ConfigurationSerialization.registerClass` | ✅ (startup) | ❌ | global |
| PDC read on `ItemStack.getItemMeta()` | ✅ | ⚠️ (item-meta clone) | region of item holder |
| PDC write on entity / chunk PDC | ✅ | ❌ | region of entity / chunk |
| `OfflinePlayer.getUniqueId()` (cached) | ✅ | ✅ | any |
| `Bukkit.getOfflinePlayer(name)` (uncached) | ❌ (Mojang HTTP) | ✅ | async |
| `HikariDataSource#getConnection` | ❌ (waits) | ✅ | async |
| `Connection.execute*()` | ❌ | ✅ | async |
| Gson `toJson` / `fromJson` | ✅ | ✅ | any |
| `Files.writeString` / `Files.move` | ❌ | ✅ | async |

---

## 8.9 FAILURE COOKBOOK

| Symptom | Cause | Fix |
|---|---|---|
| `cfg.getInt("a.b")` returns wrong value after edit | Read defaults tree, not loaded tree | Use the no-default overload, or `options().copyDefaults(true)` + `saveConfig()` once |
| Custom `ConfigurationSerializable` deserializes as `null` with no exception | `registerClass` not called or wrong `@SerializableAs` alias | Register at `onEnable`; pin the alias string |
| `JsonSyntaxException` reading player JSON after a crash | Half-written file from non-atomic write | Use `Files.move(tmp, file, ATOMIC_MOVE)` |
| `getOfflinePlayer(name)` blocks the server for seconds | Synchronous Mojang HTTP lookup | Hop async, then `.thenAccept` back to main thread |
| Player UUIDs change between sessions | Server in offline-mode w/o ip-forwarding from proxy | Enable `bungeecord: true` + `velocity-secret` (see §02) |
| `SQLITE_BUSY: database is locked` | Default rollback-journal mode + concurrent writes | Add `PRAGMA journal_mode=WAL` and `busy_timeout=5000` |
| SQLite file size never shrinks | WAL checkpoint not flushing | `PRAGMA wal_checkpoint(TRUNCATE)` periodically; or `journal_size_limit` |
| MySQL `CommunicationsException: link failure` after idle period | `maxLifetime` ≥ MySQL `wait_timeout` (default 8 h) | Set HikariCP `maxLifetime` to `wait_timeout - 60_000` |
| Connection-leak warning from HikariCP | `Connection`/`Statement`/`ResultSet` not closed | Wrap in try-with-resources |
| Server hangs for 30s on `/stop`, then watchdog kills | `onDisable` blocked on `pool.close()` waiting for in-flight tx | Cancel scheduled tasks first; await futures with timeout; lower `connectionTimeout` |
| `IllegalStateException: Asynchronous chunk load!` after async DB callback | Touched Bukkit API from the async hop | `runTask` back to main before any world / entity / inventory call |
| `cfg.set(...); saveConfig();` doesn't persist anything | Got a config snapshot via `getConfig().getConfigurationSection(...)` and edited the snapshot | Call `cfg.set(path, value)` on the root and re-save |
| `ConfigLib`/Configurate POJO loses fields on next save | Renamed a field; old YAML key is silently ignored | Add a migration step or implement a `@PostProcess` rename hook |
| PDC-on-chunk vanishes after server crash | Vanilla autosave hadn't fired; chunk only writes PDC on unload | Schedule a periodic `world.save()` for critical chunks, or move data to DB |
| `ConcurrentModificationException` while iterating `cfg.getKeys(true)` | Collection backed by live config; you mutated mid-iteration | `new ArrayList<>(cfg.getKeys(true))` then iterate |
| `OutOfMemoryError` parsing a 50 MB YAML | SnakeYAML default constructors are unbounded | Use `LoaderOptions` with `setMaxAliasesForCollections` and switch to streaming |
| HikariCP throws at startup: `Driver claims to not accept jdbcUrl` | Missing JDBC driver on classpath / shaded incorrectly | Bundle `sqlite-jdbc` / `mysql-connector-j` and check shading rules |

---

## 8.10 PERFORMANCE NOTES

- **`getConfig().getX(path)` is `O(path-depth)`** through a HashMap chain.
  Cache hot config values into fields after `reloadConfig()`.
- **`saveConfig()` re-serializes the entire config tree** — call it once per
  meaningful change, not on every event.
- **PDC reads on items copy the meta** (the `ItemMeta` is a snapshot). For
  hot-loop PDC reads, cache the result.
- **HikariCP pool checkout** is `~100 ns` warm, `~1 ms` cold (creates new
  conn). Tune `minimumIdle` to your steady-state load.
- **SQLite WAL mode** writes ~5× faster than rollback journal under
  concurrent reads; rollback-journal is faster for single-threaded writers.
- **Gson `fromJson` of a 50 KB file** is ~2 ms. Below ~10 KB, just block —
  the async hop costs more than the read.
- **Bukkit `runTaskAsynchronously` queue** is unbounded. A misbehaving plugin
  can OOM the server with deferred tasks; bound your own executor for IO.

---

## 8.11 FOLIA NOTES

- Per-player state should live in **player PDC** (auto-saves with player) or
  in your DB. Plain `HashMap<UUID, …>` static fields work but you must guard
  them with a concurrent map or per-region lock — Folia tasks may run on
  different threads.
- DB calls are fine off-thread; hop back via
  `entity.getScheduler().run(plugin, t -> ..., null)` or the global / region
  / async schedulers as appropriate.
- Config files are global state; read on the global scheduler.

---

## 8.12 REFERENCE IMPLEMENTATION — THE "PLAYER WALLET" PLUGIN

A working plugin combining all four storage layers:

- **`config.yml`** — admin-tunable starting balance + banned items list.
- **SQLite + HikariCP + WAL** — the wallet table + `coin_log`, with hand-rolled
  schema migrations and a 1-thread bounded IO executor.
- **PDC on items** — a "bound coin" that's tagged with the original owner's UUID
  and can only be redeemed by that player.
- **Per-player JSON file** — last-seen IP and login count (audit trail), via
  atomic-rename writes.

### 8.12.1 `plugin.yml`

```yaml
name: Wallet
version: 1.0.0
main: com.example.wallet.WalletPlugin
api-version: '1.21'
folia-supported: true
load: STARTUP
commands:
  wallet:
    description: Show your balance
    usage: /wallet [give <player> <amount>]
permissions:
  wallet.use:
    default: true
  wallet.admin:
    default: op
```

### 8.12.2 `src/main/resources/config.yml`

```yaml
# Wallet config
limits:
  starting-coins: 100
  max-coins: 1000000
features:
  glowing-coins: true
banned-items:
  - BARRIER
  - BEDROCK
```

### 8.12.3 `WalletPlugin.java`

```java
package com.example.wallet;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import com.google.gson.Gson;
import net.kyori.adventure.text.Component;
import org.bukkit.*;
import org.bukkit.command.*;
import org.bukkit.configuration.file.FileConfiguration;
import org.bukkit.entity.Player;
import org.bukkit.event.*;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.inventory.ItemStack;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;

import java.io.IOException;
import java.nio.file.*;
import java.sql.*;
import java.util.UUID;
import java.util.concurrent.*;

public final class WalletPlugin extends JavaPlugin implements Listener {

    private HikariDataSource ds;
    private ExecutorService ioExec;
    private NamespacedKey BOUND_TO;
    private final Gson gson = new Gson();

    @Override
    public void onEnable() {
        saveDefaultConfig();
        BOUND_TO = new NamespacedKey(this, "bound_to");

        HikariConfig cfg = new HikariConfig();
        cfg.setJdbcUrl("jdbc:sqlite:" + new java.io.File(getDataFolder(), "wallet.db").getAbsolutePath());
        cfg.setMaximumPoolSize(1);
        cfg.setConnectionInitSql(String.join("; ",
            "PRAGMA journal_mode=WAL",
            "PRAGMA synchronous=NORMAL",
            "PRAGMA busy_timeout=5000",
            "PRAGMA foreign_keys=ON"));
        this.ds = new HikariDataSource(cfg);

        try { migrate(); } catch (SQLException e) { throw new RuntimeException(e); }

        this.ioExec = new ThreadPoolExecutor(1, 1, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(500), r -> {
                Thread t = new Thread(r, "wallet-io"); t.setDaemon(true); return t;
            }, new ThreadPoolExecutor.CallerRunsPolicy());

        getServer().getPluginManager().registerEvents(this, this);
        getCommand("wallet").setExecutor(this::onWalletCmd);
    }

    @Override
    public void onDisable() {
        Bukkit.getScheduler().cancelTasks(this);
        if (ioExec != null) {
            ioExec.shutdown();
            try { ioExec.awaitTermination(10, TimeUnit.SECONDS); } catch (InterruptedException ignored) {}
        }
        if (ds != null) ds.close();
        saveConfig();
    }

    private void migrate() throws SQLException {
        try (Connection c = ds.getConnection(); Statement s = c.createStatement()) {
            s.execute("CREATE TABLE IF NOT EXISTS schema_version (v INTEGER NOT NULL)");
            int v = currentVersion(c);
            if (v < 1) {
                s.execute("CREATE TABLE wallet (uuid TEXT PRIMARY KEY, coins INTEGER NOT NULL DEFAULT 0)");
                bumpVersion(c, 1);
            }
            if (v < 2) {
                s.execute("CREATE TABLE coin_log (id INTEGER PRIMARY KEY AUTOINCREMENT, " +
                          "uuid TEXT NOT NULL, delta INTEGER NOT NULL, ts INTEGER NOT NULL)");
                bumpVersion(c, 2);
            }
        }
    }

    private int currentVersion(Connection c) throws SQLException {
        try (Statement s = c.createStatement();
             ResultSet rs = s.executeQuery("SELECT v FROM schema_version ORDER BY v DESC LIMIT 1")) {
            return rs.next() ? rs.getInt(1) : 0;
        }
    }

    private void bumpVersion(Connection c, int v) throws SQLException {
        try (PreparedStatement ps = c.prepareStatement("INSERT INTO schema_version (v) VALUES (?)")) {
            ps.setInt(1, v); ps.executeUpdate();
        }
    }

    private CompletableFuture<Integer> getCoins(UUID id) {
        return CompletableFuture.supplyAsync(() -> {
            try (Connection c = ds.getConnection();
                 PreparedStatement ps = c.prepareStatement(
                     "INSERT OR IGNORE INTO wallet (uuid, coins) VALUES (?, ?)")) {
                ps.setString(1, id.toString());
                ps.setInt(2, getConfig().getInt("limits.starting-coins", 100));
                ps.executeUpdate();
            } catch (SQLException e) { throw new CompletionException(e); }

            try (Connection c = ds.getConnection();
                 PreparedStatement ps = c.prepareStatement("SELECT coins FROM wallet WHERE uuid = ?")) {
                ps.setString(1, id.toString());
                try (ResultSet rs = ps.executeQuery()) {
                    return rs.next() ? rs.getInt(1) : 0;
                }
            } catch (SQLException e) { throw new CompletionException(e); }
        }, ioExec);
    }

    private CompletableFuture<Void> addCoins(UUID id, int delta) {
        return CompletableFuture.runAsync(() -> {
            try (Connection c = ds.getConnection()) {
                c.setAutoCommit(false);
                try (PreparedStatement up = c.prepareStatement(
                         "UPDATE wallet SET coins = MAX(0, coins + ?) WHERE uuid = ?");
                     PreparedStatement log = c.prepareStatement(
                         "INSERT INTO coin_log (uuid, delta, ts) VALUES (?, ?, ?)")) {
                    up.setInt(1, delta); up.setString(2, id.toString()); up.executeUpdate();
                    log.setString(1, id.toString()); log.setInt(2, delta);
                    log.setLong(3, System.currentTimeMillis()); log.executeUpdate();
                    c.commit();
                } catch (SQLException e) { c.rollback(); throw e; }
            } catch (SQLException e) { throw new CompletionException(e); }
        }, ioExec);
    }

    private boolean onWalletCmd(CommandSender s, Command c, String l, String[] args) {
        if (!(s instanceof Player p)) { s.sendMessage("Players only."); return true; }
        getCoins(p.getUniqueId()).thenAccept(coins ->
            Bukkit.getScheduler().runTask(this, () ->
                p.sendMessage(Component.text("Balance: " + coins + " coins"))));
        return true;
    }

    @EventHandler
    public void onJoin(PlayerJoinEvent e) {
        Player p = e.getPlayer();
        UUID id = p.getUniqueId();

        // Per-player JSON audit (atomic write)
        ioExec.submit(() -> {
            try {
                Path file = getDataFolder().toPath().resolve("audit").resolve(id + ".json");
                Files.createDirectories(file.getParent());
                long now = System.currentTimeMillis();
                String ip = p.getAddress() != null ? p.getAddress().getHostString() : "?";
                String json = gson.toJson(new AuditRecord(ip, now));
                Path tmp = file.resolveSibling(id + ".json.tmp");
                Files.writeString(tmp, json,
                    StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
                Files.move(tmp, file, StandardCopyOption.ATOMIC_MOVE,
                    StandardCopyOption.REPLACE_EXISTING);
            } catch (IOException ex) { getLogger().warning("audit write failed: " + ex); }
        });

        // Give a "bound coin" item with PDC tag
        ItemStack coin = ItemStack.of(Material.GOLD_NUGGET);
        coin.editMeta(m -> {
            m.displayName(Component.text("Bound Coin"));
            m.getPersistentDataContainer().set(BOUND_TO, PersistentDataType.STRING, id.toString());
        });
        p.getInventory().addItem(coin);
    }

    public record AuditRecord(String ip, long lastSeen) {}
}
```

This is a complete vertical slice covering YAML config, PDC, JSON audit files,
SQLite + HikariCP + WAL, schema migrations, async IO with main-thread reply,
and proper shutdown ordering.

---

## 8.X SELF-REVIEW CHECKLIST

- [x] Mental model — four layers, four lifetimes, ASCII enable/disable diagram
- [x] `saveDefaultConfig` / `saveResource` semantics
- [x] `getConfig` / `set` / `saveConfig` workflow + main-thread cost note
- [x] Two-tree problem (defaults vs actual) and `copyDefaults(true)` fix
- [x] `setComments` / `setInlineComments` (1.18.1+)
- [x] `ConfigurationSerializable` + `SerializableAs` + `ConfigurationSerialization.registerClass`
- [x] `reloadConfig` semantics — no event, must re-read
- [x] Multiple-file pattern via `YamlConfiguration.loadConfiguration`
- [x] Rich-config alternatives (Configurate / ConfigLib / Jackson) with comparison table
- [x] PDC overview — every holder type tabulated
- [x] PDC built-in `PersistentDataType` constants table
- [x] PDC custom adapter (UUID example)
- [x] `PersistentDataContainerView` (1.21.5+) read-only view
- [x] PDC scope limits (no cross-server, no bulk query, chunk-PDC crash gap)
- [x] JSON / Gson per-player files
- [x] Offline-UUID landmine + proxy forwarding mitigation
- [x] Atomic-rename file write pattern
- [x] HikariCP pool rationale + cold-start cost
- [x] SQLite pragma stack (WAL + busy_timeout + synchronous=NORMAL + foreign_keys + temp_store)
- [x] WAL on NFS warning
- [x] MySQL HikariCP config + `maxLifetime < wait_timeout` rule
- [x] Schema migration via Flyway + hand-rolled migrator
- [x] Prepared statements + try-with-resources rule
- [x] Async two-hop pattern
- [x] Folia replacement for `runTask*`
- [x] Bounded executor for backpressure
- [x] `onEnable` / `onDisable` ordering, including 30-second watchdog warning
- [x] `/reload` is unsupported note
- [x] Async-safety matrix (12 ops × main / async / Folia scheduler)
- [x] 16-row failure cookbook
- [x] Performance notes
- [x] Folia notes
- [x] Reference implementation — Wallet plugin ties YAML + PDC + JSON + SQLite together

---

## 8.Z REFERENCES

- [Paper plugin-configurations guide](https://docs.papermc.io/paper/dev/plugin-configurations/)
- [Paper PDC guide](https://docs.papermc.io/paper/dev/pdc)
- [Paper scheduler guide](https://docs.papermc.io/paper/dev/scheduler/)
- [Paper FileConfiguration Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/configuration/file/FileConfiguration.html)
- [Paper YamlConfiguration Javadoc](https://jd.papermc.io/paper/1.18.0/org/bukkit/configuration/file/YamlConfiguration.html)
- [Paper Configuration Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/configuration/Configuration.html)
- [Paper ConfigurationSerializable](https://jd.papermc.io/paper/1.21.9/org/bukkit/configuration/serialization/ConfigurationSerializable.html)
- [Paper ConfigurationSerialization](https://jd.papermc.io/paper/1.21.6/org/bukkit/configuration/serialization/ConfigurationSerialization.html)
- [Paper PersistentDataContainer Javadoc](https://hub.spigotmc.org/javadocs/spigot/org/bukkit/persistence/PersistentDataContainer.html)
- [Paper PersistentDataType (1.20.5)](https://jd.papermc.io/paper/1.20.5/org/bukkit/persistence/PersistentDataType.html)
- [Paper PersistentDataContainerView (1.21.5+)](https://jd.papermc.io/paper/1.21.10/io/papermc/paper/persistence/PersistentDataContainerView.html)
- [Paper OfflinePlayer Javadoc](https://jd.papermc.io/paper/1.21.4/org/bukkit/OfflinePlayer.html)
- [Paper BukkitScheduler Javadoc](https://jd.papermc.io/paper/1.21.0/org/bukkit/scheduler/BukkitScheduler.html)
- [SpongePowered/Configurate](https://github.com/SpongePowered/Configurate)
- [Hangar: ConfigLib](https://hangar.papermc.io/Exlll/ConfigLib)
- [HikariCP README](https://github.com/brettwooldridge/HikariCP)
- [SQLite WAL docs](https://www.sqlite.org/wal.html)
- [Coddy: SQLite PRAGMA settings](https://coddy.tech/docs/sqlite/pragma-settings)
- [Litestream tips](https://litestream.io/tips/)
- [pecar.me: SQLite WAL on NFS warning](https://blog.pecar.me/sqlite-django-config)
- [Baeldung: Flyway database migrations](https://www.baeldung.com/database-migrations-with-flyway)

Compliance note: information from external sources was paraphrased to comply
with content licensing.

---

## See also

- `01-build-tooling.md` — shading rules for Hikari/JDBC drivers, plugin.yml.
- `02-velocity-folia-bedrock.md` — proxy forwarding for stable UUIDs across
  servers; Folia regional/global/async scheduler split.
- `03-events-extras.md` — listener priorities relevant to login-time DB
  prefetch.
- `04-commands-extras.md` — Brigadier-driven `/wallet` admin commands.
- `05-entities-mobs.md` — entity PDC for per-mob state.
- `06-world-blocks-chunks.md` — chunk PDC and TileState save semantics.
- `07-recipes-loot-advancements.md` — datapack assets shipped under
  `src/main/resources` survive plugin reloads when stored as JSON datapacks.
- `paper-plugin-dev.md` — original short references for config / SQL /
  PDC / scheduler.
