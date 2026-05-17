---
name: paper-plugin-dev-commands-extras
description: >
  Expansion to paper-plugin-dev.md covering Paper command development end-to-end:
  raw Brigadier (literals, arguments, redirects, forks, requires, custom argument types,
  suggestion providers, command exception types), Paper's Commands API + LifecycleEventManager
  registration, BasicCommand interface, the deprecation of CommandExecutor / TabCompleter /
  PluginCommand / commands.yml / setExecutor, command frameworks (Cloud v2, Lamp v4,
  ACF status), CommandSourceStack vs CommandSender, ArgumentResolver pattern,
  AsyncTabCompleteEvent + AsyncPlayerSendCommandsEvent, datapack-visible commands via
  PluginBootstrap, programmatic command dispatch, error handling patterns, and a
  comprehensive command-failure cookbook.
---

# 4. COMMANDS — DEEP DIVE

This document is a comprehensive companion to §4 / §5 of `paper-plugin-dev.md`. The base
file shows a `JavaPlugin#getCommand` + `CommandExecutor` example, references "Brigadier",
and mentions Cloud / Lamp / ACF as alternatives. Every one of those topics has either been
deprecated, replaced, or substantially expanded since 1.20.6. This file walks through the
modern Paper command surface from the wire format up.

All snippets target **Paper 1.21.4+** (Mojang-mapped runtime). Where 1.21.6 / 1.21.7 add
new surfaces they are flagged inline. Authoritative sources are linked at end of section,
plus a [References](#4z-references) appendix.

---

## 4.0 MENTAL MODEL — THE FIVE LAYERS

```
   ┌────────────────────────────────────────────────────────────┐
   │  Client (typing /foo bar...)                               │
   │   - validates literals, parses argument types client-side  │
   │   - shows colored argument hints + suggestions             │
   └──────────────────┬─────────────────────────────────────────┘
                      │ Brigadier "command tree" + suggestion packets
                      ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Brigadier dispatcher (Mojang's library)                   │
   │   - LiteralCommandNode / ArgumentCommandNode               │
   │   - .requires(...), .executes(...), .redirect(...)         │
   │   - throws CommandSyntaxException on parse failure         │
   └──────────────────┬─────────────────────────────────────────┘
                      │ wrapped by
                      ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Paper Commands API (io.papermc.paper.command.brigadier)   │
   │   - Commands.literal / .argument                            │
   │   - CommandSourceStack (sender + location + context)        │
   │   - ArgumentTypes.player(), .entity(), .resource(...)       │
   │   - LifecycleEvents.COMMANDS for registration               │
   └──────────────────┬─────────────────────────────────────────┘
                      │
                      ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Frameworks (optional)                                     │
   │   - Cloud v2 (Incendo/cloud)        — builders + annotations│
   │   - Lamp v4   (Revxrsal/Lamp)       — annotations + Brigadier
   │   - ACF (aikar/commands)            — annotations, legacy, mostly maintenance only
   └──────────────────┬─────────────────────────────────────────┘
                      │
                      ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Bukkit legacy (CommandExecutor / TabCompleter / commands.yml)│
   │   - DEPRECATED for removal as of 1.20.6                     │
   │   - still works at runtime, but no Brigadier integration    │
   └────────────────────────────────────────────────────────────┘
```

When someone says "command" in modern Paper docs, they mean the **Paper Commands API**
layer. Frameworks are optional, the Bukkit layer is legacy, and raw Brigadier is what
everything else is built on. Pick the lowest-friction layer that meets your needs.

---

## 4.1 WHAT IS DEPRECATED (and why your old plugins still work)

In Paper 1.20.6 the entire `org.bukkit.command.*` tree was marked deprecated for removal.
This includes:

| Deprecated | Replacement |
|---|---|
| `JavaPlugin#getCommand(String)` + `setExecutor` | `LifecycleEvents.COMMANDS` + `Commands#register` |
| `CommandExecutor` interface | `Command` (Brigadier) or `BasicCommand` |
| `TabCompleter` interface | Brigadier `SuggestionProvider` or `BasicCommand#suggest` |
| `PluginCommand` class | Direct registration via `Commands` |
| `commands` block in `plugin.yml` | Programmatic registration only |
| `Bukkit#dispatchCommand` | Still works, but use Brigadier dispatcher when possible |
| `BukkitCommand` (extending) | `Command` in `io.papermc.paper.command.brigadier` |

> **Why your existing plugin still loads:** Paper has not removed these classes — they
> are bridged to the Brigadier dispatcher under the hood. New plugins should not use them
> because (1) they don't get colored argument types, (2) they miss out on the redirect /
> fork tree features, and (3) they will eventually disappear.

The legacy bridge means: if you register `CommandExecutor`, Paper auto-creates a single
literal node in Brigadier with a greedy-string argument and routes the `String[] args` to
your executor. This works but has zero client-side parsing intelligence.

References: [Bukkit→Brigadier comparison](https://docs.papermc.io/paper/dev/command-api/misc/comparison-bukkit-brigadier),
[Paper Command class deprecation note](https://jd.papermc.io/paper/1.21.11/org/bukkit/command/Command.html).

---

## 4.2 PAPER COMMANDS API — THE NEW STANDARD

Paper's `io.papermc.paper.command.brigadier.Commands` is the recommended way to register
commands as of 1.20.6. Everything below assumes Paper API ≥ 1.20.6.

### 4.2.1 Hello-world command

```java
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;
import com.mojang.brigadier.Command;
import net.kyori.adventure.text.Component;

public final class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        this.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            final Commands commands = event.registrar();

            commands.register(
                Commands.literal("hello")
                    .executes(ctx -> {
                        ctx.getSource().getSender().sendMessage(Component.text("Hello!"));
                        return Command.SINGLE_SUCCESS;
                    })
                    .build(),
                "Greet the sender",          // bukkit help description (optional)
                List.of("hi", "hey")         // aliases (optional)
            );
        });
    }
}
```

Things to notice:

- No `plugin.yml` `commands:` block. Registration is 100% programmatic.
- The handler runs *every* time the COMMANDS lifecycle event fires (server start, datapack
  reload, `/reload confirm`). You don't manage that yourself.
- `ctx.getSource()` is a `CommandSourceStack`, **not** a `CommandSender`. See §4.3.
- `Command.SINGLE_SUCCESS` is `1`. Brigadier interprets non-zero as "success" and 0/throw
  as "failure". The number flows up through `redirect` and `fork` nodes.

References: [Paper command registration docs](https://docs.papermc.io/paper/dev/command-api/basics/registration/),
[LifecycleEvents.COMMANDS](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/plugin/lifecycle/event/types/LifecycleEvents.html).

### 4.2.2 Command with arguments

```java
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.command.brigadier.argument.ArgumentTypes;
import io.papermc.paper.command.brigadier.argument.resolvers.PlayerProfileListResolver;
import com.mojang.brigadier.arguments.IntegerArgumentType;

commands.register(
    Commands.literal("heal")
        .requires(src -> src.getSender().hasPermission("myplugin.heal"))
        .then(Commands.argument("targets", ArgumentTypes.playerProfiles())
            .then(Commands.argument("amount", IntegerArgumentType.integer(1, 20))
                .executes(ctx -> {
                    final var resolver = ctx.getArgument("targets", PlayerProfileListResolver.class);
                    final int amount = IntegerArgumentType.getInteger(ctx, "amount");
                    final var profiles = resolver.resolve(ctx.getSource());

                    int healed = 0;
                    for (var profile : profiles) {
                        var p = Bukkit.getPlayer(profile.getId());
                        if (p == null) continue;
                        p.setHealth(Math.min(p.getHealth() + amount, p.getAttribute(Attribute.MAX_HEALTH).getValue()));
                        healed++;
                    }
                    ctx.getSource().getSender().sendMessage(Component.text("Healed " + healed + " players"));
                    return healed;          // returned value ↑ to redirector (if any)
                })))
        .build()
);
```

Brigadier's argument types are *resolvers*, not values. This is intentional: argument
resolution may need access to the source (e.g. `@s` resolves differently for console vs a
player), so resolution is deferred until `executes(...)` calls `resolve(source)`.

### 4.2.3 Subcommands (literal trees)

```java
commands.register(
    Commands.literal("party")
        .then(Commands.literal("create")
            .then(Commands.argument("name", StringArgumentType.word())
                .executes(MyPlugin::onPartyCreate)))
        .then(Commands.literal("invite")
            .then(Commands.argument("target", ArgumentTypes.player())
                .executes(MyPlugin::onPartyInvite)))
        .then(Commands.literal("disband")
            .executes(MyPlugin::onPartyDisband))
        .build()
);
```

Each `.then(literal("..."))` becomes a child node. The client sees them as autocomplete
options. Reuse subtrees by extracting the builder to a helper:

```java
private static LiteralArgumentBuilder<CommandSourceStack> partyTargetSubcommand(String name) {
    return Commands.literal(name)
        .then(Commands.argument("target", ArgumentTypes.player())
            .executes(ctx -> {
                // shared body for /party invite, /party kick, /party promote
                ...
            }));
}
```

### 4.2.4 The `requires(...)` predicate

```java
.requires(src -> src.getSender().hasPermission("myplugin.admin"))
```

`requires(...)` is checked **before** the client even sees the node. If it returns false,
Brigadier strips the entire subtree from the command tree sent to the client. The user
cannot tab-complete `/myadmincmd` if they lack permission — it literally doesn't exist
to them.

This is *not* a security boundary on its own (a malicious client could craft the packet),
but Paper double-checks `requires` server-side before executing. Use it for both UX and
defense in depth.

For multi-permission gates:

```java
.requires(src -> {
    var sender = src.getSender();
    return sender.hasPermission("myplugin.use") && sender.isOp();
})
```

### 4.2.5 Common `ArgumentTypes`

| Argument type | Returns (resolver type) | Notes |
|---|---|---|
| `ArgumentTypes.player()` | `PlayerSelectorArgumentResolver` | One online player or selector resolving to one |
| `ArgumentTypes.players()` | `PlayerSelectorArgumentResolver` | Multiple players (selectors allowed) |
| `ArgumentTypes.entity()` | `EntitySelectorArgumentResolver` | Exactly one entity |
| `ArgumentTypes.entities()` | `EntitySelectorArgumentResolver` | Multiple entities |
| `ArgumentTypes.playerProfiles()` | `PlayerProfileListResolver` | Online or offline (last-known UUID) |
| `ArgumentTypes.world()` | `World` | World by namespaced key |
| `ArgumentTypes.blockPosition()` | `BlockPositionResolver` | Supports `~`, `^`, integer coords |
| `ArgumentTypes.finePosition()` | `FinePositionResolver` | Decimal coords |
| `ArgumentTypes.resource(RegistryKey)` | T from registry | Items, blocks, biomes, etc. |
| `ArgumentTypes.resourceKey(RegistryKey)` | `TypedKey<T>` | Lazy reference (doesn't have to exist yet) |
| `ArgumentTypes.namespacedKey()` | `NamespacedKey` | Generic resource location |
| `ArgumentTypes.signedMessage()` | `SignedMessageResolver` | Chat-signed message (1.19.1+) |
| `ArgumentTypes.component()` | `Component` | MiniMessage / JSON-style chat |
| `ArgumentTypes.namedColor()` | `NamedTextColor` | Adventure named color |
| `ArgumentTypes.gameMode()` | `GameMode` | survival/creative/etc |
| Brigadier built-ins (`StringArgumentType`, `IntegerArgumentType`, `BoolArgumentType`, ...) | primitives | From `com.mojang.brigadier.arguments` |

`ArgumentTypes.entity()` and `players()` automatically require `minecraft.command.selector`
on the sender to use selectors like `@a`. Add a `.requires(...)` clause if you don't want
the rest of the subtree visible without that permission. See
[Paper docs — entities and players](https://docs.papermc.io/paper/dev/command-api/arguments/entity-player).

### 4.2.6 Resolving arguments

```java
final var resolver = ctx.getArgument("targets", PlayerSelectorArgumentResolver.class);
final List<Player> players = resolver.resolve(ctx.getSource());

// For single-selector:
final Player target = ctx.getArgument("target", PlayerSelectorArgumentResolver.class)
    .resolve(ctx.getSource())
    .getFirst();
```

Always call `.resolve(...)` exactly once; the result already represents the snapshot at
command-execution time. Caching the resolver across ticks is not safe — it captures
parse-time state.

### 4.2.7 Aliases & description

```java
commands.register(
    Commands.literal("teleport")
        .executes(...)
        .build(),
    "Teleport to another player",   // shown in /help
    List.of("tp", "tpa")
);
```

The description appears in `/help` as the Bukkit help-topic synopsis. Aliases are
registered as separate root literals that share the same tree.

If you need namespace control (avoid `/myplugin:teleport` colliding with `/teleport`):

```java
commands.register(
    PluginMeta meta,                 // pulled from event.registrar()'s plugin
    Commands.literal("teleport").build(),
    "...",
    List.of()
);
```

Without specifying meta, the namespace is derived from the registering plugin name
(lower-cased). The `<plugin>:<command>` form is always available even if your literal name
collides with another plugin's command.

---

## 4.3 `CommandSourceStack` vs `CommandSender`

Brigadier's source is **richer** than Bukkit's `CommandSender`:

```java
public interface CommandSourceStack {
    CommandSender getSender();                   // legacy Bukkit sender
    Entity getExecutor();                        // entity running the command (e.g. /execute as)
    Location getLocation();                      // position + rotation (matters for ^ ^ ^ args)
}
```

Why this matters:

- For `/execute as @e[type=zombie] run mycommand` the `executor` is the zombie, but
  `getSender()` is still whoever ran `/execute`. Permissions check against `getSender()`,
  but world position uses `getLocation()`.
- For console, `executor` is `null` and `getLocation()` is `world.getSpawnLocation()`.
- For Folia, `getLocation()` tells you which region this command is currently scheduled to
  run on — relevant when you teleport or schedule region tasks (see file `02`).

Use `getSender()` for permissions and messaging, `getLocation()` for relative-coord args,
and `getExecutor()` only when you actually need entity-level context.

References: [CommandSourceStack Javadoc](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/command/brigadier/CommandSourceStack.html).

---

## 4.4 BASIC COMMAND — THE SHORT-FORM ALTERNATIVE

For commands that need none of Brigadier's typed arguments, Paper offers `BasicCommand`,
a near-1:1 replacement for `CommandExecutor` + `TabCompleter`:

```java
import io.papermc.paper.command.brigadier.BasicCommand;
import io.papermc.paper.command.brigadier.CommandSourceStack;

public final class GcCommand implements BasicCommand {
    @Override
    public void execute(CommandSourceStack source, String[] args) {
        long before = Runtime.getRuntime().freeMemory();
        System.gc();
        long after  = Runtime.getRuntime().freeMemory();
        source.getSender().sendMessage(Component.text(
            "GC freed " + ((after - before) / 1_048_576) + " MB"));
    }

    @Override
    public Collection<String> suggest(CommandSourceStack source, String[] args) {
        return args.length == 0 ? List.of("now") : List.of();
    }

    @Override
    public boolean canUse(CommandSender sender) {
        return sender.hasPermission("myplugin.gc");
    }
}
```

Registration (still through Lifecycle):

```java
this.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register("gc", "Force a Java GC", List.of(), new GcCommand());
});
```

When to choose `BasicCommand`:
- You don't need typed arguments (just space-separated strings).
- You're porting a 50-line `CommandExecutor` and don't want to learn Brigadier.
- The command takes 0 or 1 arguments.

When to **not** choose it:
- You want client-side parsing intelligence (typed args, colored hints, redirect/fork).
- You'll have multi-level subcommands.

References: [Paper basic command docs](https://docs.papermc.io/paper/dev/command-api/misc/basic-command/).

---

## 4.5 REGISTERING IN `PluginBootstrap` (datapack-visible commands)

Commands registered in `onEnable` are not visible to datapacks at the time datapacks
are loaded. To make them visible (so a datapack `function` can call your command), register
in the **bootstrap phase** instead:

```java
// paper-plugin.yml
name: MyPlugin
main: com.example.MyPlugin
bootstrapper: com.example.MyPluginBootstrap
api-version: '1.21'

// MyPluginBootstrap.java
public final class MyPluginBootstrap implements PluginBootstrap {
    @Override
    public void bootstrap(BootstrapContext context) {
        context.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            event.registrar().register(
                Commands.literal("datapackvisible")
                    .executes(ctx -> {
                        ctx.getSource().getSender().sendMessage(Component.text("hi"));
                        return Command.SINGLE_SUCCESS;
                    })
                    .build()
            );
        });
    }
}
```

Now a datapack `function` can call `/datapackvisible` and Brigadier resolves it.
Documentation: [LifecycleEventManager — bootstrapper](https://docs.papermc.io/paper/dev/lifecycle/),
[Commands#register](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/command/brigadier/Commands.html).

> **Caveat:** anything registered in bootstrap runs *very* early — `Bukkit.getServer()`
> is null. Use only for command tree definition; do not load configs or open DBs.

---

## 4.6 CUSTOM ARGUMENT TYPES

When the built-in `ArgumentTypes` don't cover what you need, write a `CustomArgumentType`.
There are two flavors:

### 4.6.1 `CustomArgumentType.Converted<T, N>` — wraps a vanilla argument

This is the *common* case: take an existing vanilla parser (so the client still gets the
correct argument-type packet) and convert its parse result into your custom type.

```java
public final class WarpArgument implements CustomArgumentType.Converted<Warp, String> {
    private final WarpManager manager;

    public WarpArgument(WarpManager manager) { this.manager = manager; }

    @Override
    public Warp convert(String nativeType) throws CommandSyntaxException {
        Warp warp = manager.byName(nativeType);
        if (warp == null) {
            throw new SimpleCommandExceptionType(
                MessageComponentSerializer.message().serialize(
                    Component.text("Unknown warp: " + nativeType, NamedTextColor.RED)))
                .create();
        }
        return warp;
    }

    @Override
    public ArgumentType<String> getNativeType() {
        return StringArgumentType.word();        // client treats this as a word argument
    }

    @Override
    public <S> CompletableFuture<Suggestions> listSuggestions(
            CommandContext<S> ctx, SuggestionsBuilder builder) {
        manager.allNames().stream()
            .filter(n -> n.toLowerCase().startsWith(builder.getRemainingLowerCase()))
            .forEach(builder::suggest);
        return builder.buildFuture();
    }
}
```

Usage:

```java
.then(Commands.argument("warp", new WarpArgument(warpManager))
    .executes(ctx -> {
        Warp warp = ctx.getArgument("warp", Warp.class);
        ctx.getSource().getSender().teleport(warp.location());
        return 1;
    }))
```

The client sees this as a plain word argument (good autocomplete), but server-side it
resolves directly to a `Warp` instance.

### 4.6.2 Pure `CustomArgumentType` — no vanilla wrapping

Possible but rarely needed; the client falls back to a generic string parse. Only use if
your data model has no vanilla equivalent and you don't care about colored hints. See
[CustomArgumentType Javadoc](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/command/brigadier/argument/CustomArgumentType.html).

### 4.6.3 Throwing parse errors with rich messages

```java
private static final DynamicCommandExceptionType UNKNOWN_WARP =
    new DynamicCommandExceptionType(name ->
        MessageComponentSerializer.message().serialize(
            Component.text("Unknown warp: " + name, NamedTextColor.RED)));

public Warp convert(String nativeType) throws CommandSyntaxException {
    Warp w = manager.byName(nativeType);
    if (w == null) throw UNKNOWN_WARP.create(nativeType);
    return w;
}
```

`SimpleCommandExceptionType` for static messages, `DynamicCommandExceptionType` for
1-arg, `Dynamic2CommandExceptionType` for 2 args, etc. The client gets the red error
underline and message inline. See [Paper custom-args docs](https://docs.papermc.io/paper/dev/command-api/basics/custom-arguments/).

---

## 4.7 SUGGESTION PROVIDERS

`.suggests(...)` on a `RequiredArgumentBuilder` provides a `SuggestionProvider`:

```java
.then(Commands.argument("kit", StringArgumentType.word())
    .suggests((ctx, builder) -> {
        for (String kit : kitManager.knownKits()) {
            if (kit.toLowerCase().startsWith(builder.getRemainingLowerCase())) {
                builder.suggest(kit, MessageComponentSerializer.message().serialize(
                    Component.text(kitManager.description(kit), NamedTextColor.GRAY)));
            }
        }
        return builder.buildFuture();
    })
    .executes(ctx -> {
        String kit = StringArgumentType.getString(ctx, "kit");
        ...
    }))
```

The `MessageComponentSerializer` form lets you attach a tooltip Component shown next to
each suggestion in the client UI.

### 4.7.1 Async suggestions

Suggestions can be expensive (DB lookups, Mojang API calls). Return a real future:

```java
.suggests((ctx, builder) -> CompletableFuture.supplyAsync(() -> {
    for (String name : myAsyncSearch(builder.getRemainingLowerCase())) {
        builder.suggest(name);
    }
    return builder.build();
}, asyncExecutor))
```

Brigadier waits on the future before sending the suggestions packet; the client shows a
spinner. Don't block the main thread doing the search.

Paper additionally fires `AsyncTabCompleteEvent` *before* the suggestion provider runs,
which lets other plugins inject suggestions. See [AsyncTabCompleteEvent Javadoc](https://jd.papermc.io/paper/1.21.4/com/destroystokyo/paper/event/server/AsyncTabCompleteEvent.html).

---

## 4.8 REDIRECTS AND FORKS

Two of Brigadier's most underused features.

### 4.8.1 `redirect(...)` — alias one node to another

`/foo` and `/bar` should run the same code? Don't duplicate the subtree:

```java
LiteralCommandNode<CommandSourceStack> root = Commands.literal("foo")
    .then(Commands.argument("amount", IntegerArgumentType.integer())
        .executes(ctx -> { /* ... */ return 1; }))
    .build();

commands.register(root);
commands.register(
    Commands.literal("bar")
        .redirect(root)              // /bar 5 dispatches /foo 5
        .build()
);
```

Redirection happens at parse time — when the client tab-completes `/bar `, it sees the
same children as `/foo`.

### 4.8.2 `redirect(...)` with a modifier — vanilla `/execute`-style transforms

```java
.redirect(rootNode, ctx -> {
    // transform the source before re-dispatching
    var newSource = ctx.getSource().withLocation(player.getLocation());
    return List.of(newSource);   // can return multiple sources for fork-style
})
```

This is exactly how vanilla's `/execute as @e[type=...] at @s run ...` works — a chain of
redirected modifiers, each producing one or more transformed sources, that finally
re-enter the dispatcher at the redirected node.

### 4.8.3 `fork(...)` — explicit multi-source dispatch

```java
.fork(rootNode, ctx -> {
    return Bukkit.getOnlinePlayers().stream()
        .map(p -> ctx.getSource().withSender(p))
        .toList();
})
```

`fork` is `redirect` but the modifier is allowed to return many sources, and Brigadier
runs the redirected node once per source. Use for "broadcast to all" or "for each entity"
semantics where you don't want to write a loop yourself.

The return value of `executes` propagates upward (sum of children for fork). This is how
`/execute if entity @e[type=...]` builds its truthy count.

References: [CommandDispatcher source](https://github.com/Mojang/brigadier/blob/master/src/main/java/com/mojang/brigadier/CommandDispatcher.java),
[sciwhiz12 Brigadier explanation gist](https://gist.github.com/sciwhiz12/8b258d493c764d2cf009e121bdc654d3) (good visualization of the tree).

---

## 4.9 ERROR HANDLING

Inside `executes(...)` you may either:

1. **Throw** `CommandSyntaxException` to show a red error in chat and abort.
2. **Return 0** to indicate failure with no error message.
3. **Return ≥ 1** for success (typically `Command.SINGLE_SUCCESS = 1`).

Throwing for argument-resolution failures:

```java
.executes(ctx -> {
    Player target = ctx.getArgument("target", PlayerSelectorArgumentResolver.class)
        .resolve(ctx.getSource()).getFirst();

    if (!target.isOnline()) {
        throw new SimpleCommandExceptionType(
            MessageComponentSerializer.message().serialize(
                Component.text("Player must be online", NamedTextColor.RED)))
            .create();
    }
    // ...
})
```

For runtime business-logic failures, Adventure messages are usually the right answer:

```java
if (player.getHealth() >= 20) {
    ctx.getSource().getSender().sendMessage(
        Component.text("Already at full health", NamedTextColor.YELLOW));
    return 0;
}
```

The difference: thrown exceptions interrupt redirect/fork chains; returning 0 lets the
chain continue but the failing leaf contributes 0 to the result counter.

---

## 4.10 PROGRAMMATIC DISPATCH

To run a command as another sender or from server code:

```java
// Modern: through Brigadier dispatcher (gets argument parsing, suggestions, etc.)
Bukkit.getServer().getCommandMap()                         // legacy bridge
    .dispatch(sender, "give @s minecraft:diamond 64");

// Or, explicit through Paper:
Bukkit.getServer().dispatchCommand(sender, "give @s diamond 64");
```

Both go through the same Brigadier dispatcher. `dispatchCommand` is *not* deprecated —
just `CommandExecutor` and friends are.

To run as console:

```java
Bukkit.dispatchCommand(Bukkit.getConsoleSender(), "say hello");
```

To run with elevated permissions, use `PermissionAttachment` on a temporary sender, or
use `Server#getCommandMap()` to access the dispatcher directly and pass a custom source.

> **Don't** parse + replay the command string yourself — Brigadier's parser handles
> selectors, NBT args, and quoting in ways you don't want to reimplement.

---

## 4.11 EVENTS RELATED TO COMMANDS

| Event | Fires when | Async-safe? |
|---|---|---|
| `AsyncPlayerSendCommandsEvent` | Server is about to send command tree to a client (login, world change, perms rebuild) | Yes — modify the tree |
| `AsyncTabCompleteEvent` | Player presses TAB. Lets you inject/suppress suggestions | Yes |
| `AsyncPlayerSendSuggestionsEvent` | Right before suggestion packet sent. Final filter | Yes |
| `PlayerCommandPreprocessEvent` | Player sent `/cmd` text (chat-style hook). Can cancel | No — sync |
| `ServerCommandEvent` | Console / RCON sent a command | No |
| `UnknownCommandEvent` | Command not found, lets you customize the "unknown command" message | No |

Hooking the command tree to inject runtime-computed nodes:

```java
@EventHandler
public void onSendCommands(AsyncPlayerSendCommandsEvent<CommandSourceStack> event) {
    if (!event.hasFiredAsync()) return;          // only modify the async pass
    if (!event.getPlayer().hasPermission("myplugin.dynamic")) return;

    var node = event.getCommandNode();
    // Build a new literal child and add it
    var newChild = Commands.literal("dynamic-" + System.currentTimeMillis() % 100)
        .executes(ctx -> { /* ... */ return 1; })
        .build();
    node.addChild(newChild);
}
```

Use sparingly — adding nodes per player creates per-player command trees. Prefer
permissions-based `requires(...)` filtering when possible.

---

## 4.12 COMMAND FRAMEWORKS

If raw Brigadier is too verbose, frameworks generate the tree from annotations.

### 4.12.1 Cloud v2 (Incendo) — RECOMMENDED

The most actively maintained framework with first-class Paper support. v2 (released mid
2024) is rewritten on top of Paper's Brigadier API — your annotated commands become
real Brigadier nodes with full client-side intelligence.

```kotlin
dependencies {
    implementation("org.incendo:cloud-paper:2.0.0-beta.10")
    implementation("org.incendo:cloud-annotations:2.0.0")
    implementation("org.incendo:cloud-minecraft-extras:2.0.0-beta.10")
}
```

Builder style:

```java
PaperCommandManager<CommandSender> mgr = PaperCommandManager.createNative(
    this, ExecutionCoordinator.simpleCoordinator()
);
// Brigadier integration:
mgr.registerBrigadier();
mgr.brigadierManager().setNativeNumberSuggestions(false);

mgr.command(mgr.commandBuilder("heal")
    .permission("myplugin.heal")
    .required("target", PlayerParser.playerParser())
    .required("amount", IntegerParser.integerParser(1, 20))
    .handler(ctx -> {
        Player target = ctx.get("target");
        int amount  = ctx.get("amount");
        target.setHealth(Math.min(20, target.getHealth() + amount));
        ctx.sender().sendMessage(Component.text("Healed " + target.getName()));
    }));
```

Annotation style:

```java
public final class HealCommand {
    @Command("heal <target> <amount>")
    @Permission("myplugin.heal")
    public void heal(CommandSender sender, @Argument("target") Player target,
                     @Argument("amount") @Range(min="1", max="20") int amount) {
        target.setHealth(Math.min(20, target.getHealth() + amount));
        sender.sendMessage(Component.text("Healed " + target.getName()));
    }
}

// Register:
AnnotationParser<CommandSender> parser =
    new AnnotationParser<>(mgr, CommandSender.class);
parser.parse(new HealCommand());
```

Both styles compile to the same Brigadier tree. References:
[Cloud v2 Paper docs](https://cloud.incendo.org/minecraft/paper/),
[Cloud GitHub](https://github.com/Incendo/cloud).

### 4.12.2 Lamp v4 (Revxrsal)

A clean annotation-driven framework with native Brigadier integration on Paper. Smaller
than Cloud, focused on annotations, fewer abstractions. Approaching v4.0 stable as of mid
2025.

```kotlin
dependencies {
    implementation("io.github.revxrsal:lamp.common:4.0.0-rc.16")
    implementation("io.github.revxrsal:lamp.bukkit:4.0.0-rc.16")
    implementation("io.github.revxrsal:lamp.brigadier:4.0.0-rc.16")
    implementation("io.github.revxrsal:lamp.paper:4.0.0-beta.19")
}
```

```java
public final class HealCommand {
    @Command({"heal", "h"})
    @CommandPermission("myplugin.heal")
    public void onHeal(BukkitCommandActor actor, Player target,
                       @Range(min=1, max=20) int amount) {
        target.setHealth(Math.min(20, target.getHealth() + amount));
        actor.reply("Healed " + target.getName());
    }
}

// Register:
Lamp<BukkitCommandActor> lamp = BukkitLamp.builder(this).build();
lamp.register(new HealCommand());
```

References: [Lamp GitHub](https://github.com/Revxrsal/Lamp),
[Lamp Bukkit/Paper docs](https://foxhut.gitbook.io/lamp-docs/platforms/bukkit-spigot-paper).

### 4.12.3 ACF (aikar/commands)

Status as of late 2025: aikar/commands is in maintenance mode. The original maintainer
(Aikar) is no longer actively shipping releases. Issues remain open ([aikar/commands issues](https://github.com/aikar/commands/issues)),
forks like `BlockeySports/AnnotationCommandFramework` carry small fixes. ACF still works
on modern Paper via the legacy bridge but does **not** integrate with Brigadier — your
clients see plain greedy-string args. Recommendation:

- **Existing ACF code:** keep it; migration is rarely worth the churn.
- **New code:** use Cloud v2 or Lamp v4 instead.

---

## 4.13 PERMISSIONS — INLINE VS plugin.yml

Both forms still work:

```yaml
# plugin.yml (legacy but still parsed)
permissions:
  myplugin.heal:
    description: Heal command
    default: op
  myplugin.heal.others:
    description: Heal other players
    default: op
```

```java
// programmatic registration (Paper)
Bukkit.getPluginManager().addPermission(new Permission(
    "myplugin.heal",
    "Heal command",
    PermissionDefault.OP
));
```

Either way, check via `sender.hasPermission("myplugin.heal")` in `requires(...)`.
LuckPerms / GroupManager / Vault all read the plugin's registered permissions for their
GUIs — declaring permissions properly makes admin lives easier.

For per-world permission overrides and group inheritance, see file `02` Velocity/Folia
content + section on LuckPerms API.

---

## 4.14 COMMON COMMAND-DEV FAILURES

| Symptom | Cause | Fix |
|---|---|---|
| Tab completion not appearing | `requires(...)` returns false at the wrong level | Move `requires` to the deepest node, not the leaf |
| `Unknown or incomplete command` from console | Command registered in `onEnable` but not fired before console runs it | Register in bootstrap, or wait for `ServerLoadEvent` |
| Argument resolves wrong player on `@s` | Used `getSender()` for location instead of `getLocation()` | Use `ctx.getSource().getLocation()` for position-aware logic |
| Aliases work but `/myplugin:cmd` doesn't | Forgot to pass `PluginMeta` | Use 3-arg `register(meta, node, ...)` overload |
| `commands.yml` aliases not tab-completing | Known Paper issue [#9214](https://github.com/PaperMC/Paper/issues/9214) | Re-register the alias programmatically |
| Suggestions appear but trigger lag | Doing a sync DB query in `.suggests(...)` | Move query to `CompletableFuture.supplyAsync(...)` |
| Custom argument doesn't show in client UI | `getNativeType()` returns null or wrong type | Return a real `ArgumentType` from `getNativeType()` |
| `CommandSyntaxException` not displayed in red | Threw `RuntimeException` instead | Use `SimpleCommandExceptionType.create()` |
| `Bukkit.dispatchCommand` returns false silently | Permission node missing | Sender must have the perm — or use `getConsoleSender()` |
| `/reload` breaks commands | Old CommandExecutor pattern; lifecycle handles this for new code | Migrate to LifecycleEvents.COMMANDS |
| Commands disappear after datapack reload | Registered in onEnable, not bootstrap | Move to bootstrap if datapack-visibility needed |
| `@a` selector throws on entity arg | Entity argument is single-target | Use `entities()` plural or `ArgumentTypes.players()` |
| Args resolve but values null | Resolver was kept across executes calls | Always call `.resolve(...)` inside the current `executes(...)` |

---

## 4.14.1 FOLIA THREAD-AFFINITY FOR COMMAND BODIES

On Folia, `executes(...)` runs on whichever thread the dispatcher is invoked from —
usually the **global region** for console / RCON, and the **player's region** for
in-game `/cmd` execution. This means:

```java
.executes(ctx -> {
    // OK — running on player's region
    if (ctx.getSource().getSender() instanceof Player p) {
        p.sendMessage(...);
    }

    // BAD on Folia — target may be in a different region
    Player target = ctx.getArgument("target", PlayerSelectorArgumentResolver.class)
        .resolve(ctx.getSource()).getFirst();
    target.teleport(loc);                      // throws on Folia if cross-region

    return 1;
})
```

Folia-safe pattern: schedule the heavy work onto the *target's* region:

```java
target.getScheduler().run(plugin, task -> target.teleport(loc), null);
return Command.SINGLE_SUCCESS;
```

See `02-velocity-folia-bedrock.md` §2 for the full Folia scheduler matrix.

---

## 4.14.2 NMS-BACKED ARGUMENT TYPES (when you need vanilla parsers verbatim)

Paper exposes wrappers around vanilla argument parsers via the `ArgumentTypes` factory
methods in `io.papermc.paper.command.brigadier.argument`. If you need a parser that
isn't yet exposed (e.g. `BlockStateArgument`, `ItemArgument` in some older Paper builds),
you can drop down to NMS through `paperweight-userdev`:

```java
import net.minecraft.commands.arguments.blocks.BlockStateArgument;
import net.minecraft.commands.arguments.blocks.BlockInput;

// Inside your registrar (paperweight-userdev required):
ArgumentType<BlockInput> raw = BlockStateArgument.block(buildContext);
.then(Commands.argument("block", raw)
    .executes(ctx -> {
        BlockInput block = ctx.getArgument("block", BlockInput.class);
        // ...
    }))
```

The downside is you lose forward-compatibility — these classes are obfuscation-mapped to
Mojang names, and Paper API may add a wrapper next release that you'd then prefer to use.
Always check `ArgumentTypes` first; only drop to NMS if the wrapper genuinely doesn't
exist in your target Paper version.

References: [ArgumentTypes Javadoc (1.21)](https://jd.papermc.io/paper/1.21.0/io/papermc/paper/command/brigadier/argument/ArgumentTypes.html).

---

## 4.14.3 `commands:` BLOCK IN plugin.yml — DEAD ON ARRIVAL FOR `paper-plugin.yml`

The classic Bukkit `plugin.yml`:

```yaml
commands:
  heal:
    description: Heal command
    usage: /heal [player] [amount]
    aliases: [h, hp]
    permission: myplugin.heal
```

…is *parsed* by Paper from `plugin.yml` for backwards compatibility, but the new
`paper-plugin.yml` format **does not support a `commands:` block at all**. Paper plugins
are expected to register everything through the Lifecycle API. If you migrate to
`paper-plugin.yml` and forget to migrate command registration, your commands silently
disappear at runtime. See [paper-plugin.yml format docs](https://papermc-paper.mintlify.app/plugins/lifecycle).

---

## 4.15 PERFORMANCE NOTES

- Building the command tree at startup costs almost nothing — do not lazy-load.
- Each player join sends the full *visible* (post-`requires`) command tree as a single
  packet. Trees with thousands of literals (typical of large warp/kit plugins) are
  expensive to send. Use **dynamic suggestions** for large lists, not literal nodes.
- `AsyncPlayerSendCommandsEvent` mutators run for every player every login. Keep them
  cheap or short-circuit early.
- Brigadier's parser is single-threaded by design (the dispatcher is shared). Do not call
  the dispatcher from async tasks; use `Bukkit.getScheduler().runTask(...)` to hop back
  to main first.

---

## 4.16 REFERENCE IMPLEMENTATION

A complete plugin showing all the patterns above:

```java
package com.example.commandsdemo;

import com.mojang.brigadier.Command;
import com.mojang.brigadier.arguments.IntegerArgumentType;
import com.mojang.brigadier.exceptions.CommandSyntaxException;
import com.mojang.brigadier.exceptions.SimpleCommandExceptionType;
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.command.brigadier.CommandSourceStack;
import io.papermc.paper.command.brigadier.MessageComponentSerializer;
import io.papermc.paper.command.brigadier.argument.ArgumentTypes;
import io.papermc.paper.command.brigadier.argument.resolvers.selector.PlayerSelectorArgumentResolver;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import org.bukkit.entity.Player;
import org.bukkit.plugin.java.JavaPlugin;

import java.util.List;

public final class CommandsDemoPlugin extends JavaPlugin {

    private static final SimpleCommandExceptionType ERR_OFFLINE =
        new SimpleCommandExceptionType(
            MessageComponentSerializer.message().serialize(
                Component.text("Target must be online", NamedTextColor.RED)));

    @Override
    public void onEnable() {
        this.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            Commands cmds = event.registrar();

            // /heal-self
            cmds.register(
                Commands.literal("heal-self")
                    .requires(src -> src.getSender() instanceof Player
                                  && src.getSender().hasPermission("demo.heal.self"))
                    .executes(this::healSelf)
                    .build(),
                "Heal yourself to full health",
                List.of("hs")
            );

            // /heal-other <player> [amount]
            var healOtherRoot = Commands.literal("heal-other")
                .requires(src -> src.getSender().hasPermission("demo.heal.other"))
                .then(Commands.argument("target", ArgumentTypes.player())
                    .executes(ctx -> healOther(ctx, 20))
                    .then(Commands.argument("amount", IntegerArgumentType.integer(1, 20))
                        .executes(ctx -> healOther(ctx, IntegerArgumentType.getInteger(ctx, "amount")))))
                .build();
            cmds.register(healOtherRoot, "Heal another player", List.of("ho"));

            // /h <target>  — alias via redirect
            cmds.register(
                Commands.literal("h")
                    .requires(src -> src.getSender().hasPermission("demo.heal.other"))
                    .redirect(healOtherRoot)
                    .build()
            );
        });
    }

    private int healSelf(com.mojang.brigadier.context.CommandContext<CommandSourceStack> ctx) {
        if (!(ctx.getSource().getSender() instanceof Player p)) return 0;
        p.setHealth(Math.min(20.0, p.getHealth() + 20));
        p.sendMessage(Component.text("Fully healed", NamedTextColor.GREEN));
        return Command.SINGLE_SUCCESS;
    }

    private int healOther(com.mojang.brigadier.context.CommandContext<CommandSourceStack> ctx,
                          int amount) throws CommandSyntaxException {
        Player target = ctx.getArgument("target", PlayerSelectorArgumentResolver.class)
            .resolve(ctx.getSource())
            .getFirst();
        if (!target.isOnline()) throw ERR_OFFLINE.create();
        target.setHealth(Math.min(20.0, target.getHealth() + amount));
        ctx.getSource().getSender().sendMessage(
            Component.text("Healed " + target.getName() + " by " + amount, NamedTextColor.GREEN));
        return amount;
    }
}
```

`paper-plugin.yml`:

```yaml
name: CommandsDemo
main: com.example.commandsdemo.CommandsDemoPlugin
api-version: '1.21'
authors: ['You']
permissions:
  demo.heal.self:
    default: op
  demo.heal.other:
    default: op
```

Notice:
- No `commands:` block — registration is programmatic.
- `requires(...)` checks both type (Player vs Console) and permission.
- `redirect(...)` aliases `/h` to `/heal-other`'s subtree without duplication.
- Argument-resolution failures throw a typed exception; runtime "already healed" cases
  return 0 + send a yellow message instead.

---

## 4.X SELF-REVIEW CHECKLIST

- [x] Five-layer mental model diagram included
- [x] Deprecation table for legacy CommandExecutor / TabCompleter / commands.yml
- [x] Hello-world example using LifecycleEvents.COMMANDS
- [x] Subcommands / literal trees explained
- [x] All common ArgumentTypes tabulated
- [x] CommandSourceStack vs CommandSender distinction
- [x] BasicCommand interface with full example
- [x] PluginBootstrap registration for datapack-visible commands
- [x] CustomArgumentType.Converted with full Warp example
- [x] CommandSyntaxException variants (Simple/Dynamic/Dynamic2)
- [x] Suggestion providers (sync + async)
- [x] redirect() and fork() with use cases
- [x] Programmatic dispatch via Bukkit.dispatchCommand
- [x] Command-related events (AsyncPlayerSendCommands, AsyncTabComplete, etc.)
- [x] Cloud v2 builder + annotation styles
- [x] Lamp v4 example with current artifact coords
- [x] ACF status (maintenance mode) called out clearly
- [x] Permissions: plugin.yml vs programmatic
- [x] 13-row failure cookbook
- [x] Performance notes (tree size, send-on-join cost, async safety)
- [x] Complete copy-paste reference plugin
- [x] References to all official sources

---

## 4.Z REFERENCES

- [Paper command-API introduction](https://docs.papermc.io/paper/dev/commands)
- [Paper command registration](https://docs.papermc.io/paper/dev/command-api/basics/registration/)
- [Arguments and literals](https://docs.papermc.io/paper/dev/command-api/basics/arguments-and-literals)
- [Requirements (`.requires`)](https://docs.papermc.io/paper/dev/command-api/basics/requirements/)
- [Custom arguments](https://docs.papermc.io/paper/dev/command-api/basics/custom-arguments/)
- [Entities and players args](https://docs.papermc.io/paper/dev/command-api/arguments/entity-player)
- [Basic commands](https://docs.papermc.io/paper/dev/command-api/misc/basic-command/)
- [Bukkit→Brigadier comparison](https://docs.papermc.io/paper/dev/command-api/misc/comparison-bukkit-brigadier)
- [Lifecycle API](https://docs.papermc.io/paper/dev/lifecycle/)
- [Commands Javadoc (1.21.4)](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/command/brigadier/Commands.html)
- [CustomArgumentType Javadoc](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/command/brigadier/argument/CustomArgumentType.html)
- [LifecycleEvents.COMMANDS Javadoc](https://jd.papermc.io/paper/1.21.4/io/papermc/paper/plugin/lifecycle/event/types/LifecycleEvents.html)
- [AsyncPlayerSendCommandsEvent](https://jd.papermc.io/paper/1.21.4/com/destroystokyo/paper/event/brigadier/AsyncPlayerSendCommandsEvent.html)
- [AsyncPlayerSendSuggestionsEvent](https://jd.papermc.io/paper/1.21.4/com/destroystokyo/paper/event/brigadier/AsyncPlayerSendSuggestionsEvent.html)
- [Mojang Brigadier (CommandDispatcher source)](https://github.com/Mojang/brigadier/blob/master/src/main/java/com/mojang/brigadier/CommandDispatcher.java)
- [Cloud v2 — Paper docs](https://cloud.incendo.org/minecraft/paper/)
- [Cloud GitHub](https://github.com/Incendo/cloud)
- [Lamp GitHub](https://github.com/Revxrsal/Lamp)
- [Lamp Bukkit/Paper docs](https://foxhut.gitbook.io/lamp-docs/platforms/bukkit-spigot-paper)
- [aikar/commands (ACF)](https://github.com/aikar/commands)
- [sciwhiz12 — Brigadier explanation gist](https://gist.github.com/sciwhiz12/8b258d493c764d2cf009e121bdc654d3)

Compliance note: information from external sources was paraphrased to comply with content
licensing.

---

## See also

- `01-build-tooling.md` §1.B — paperweight-userdev (needed if you want to wrap NMS-only
  argument types).
- `02-velocity-folia-bedrock.md` §2 — Velocity command API differences (`BrigadierCommand`,
  `SimpleCommand`, `RawCommand`) and Folia thread-affinity caveats for command bodies.
- `03-events-extras.md` — `AsyncTabCompleteEvent` covered there from the listener side.
- `paper-plugin-dev.md` §4–5 — original short command reference.
