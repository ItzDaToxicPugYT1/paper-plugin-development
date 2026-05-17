---
name: paper-plugin-dev-build-tooling
description: >
  Expansion to paper-plugin-dev.md covering Paper plugin build & tooling end-to-end:
  Gradle (KTS + Groovy), Maven, paperweight-userdev, paperweight-patcher, run-paper, Shadow,
  static analysis (Spotless, Checkstyle, PMD, SpotBugs, Error Prone, NullAway), annotation
  processors (Lombok, Immutables, MapStruct, AutoService), polyglot (Kotlin + MCCoroutine,
  Scala, Groovy), JIJ vs Paper library loader, multi-release JARs, multi-module API/impl
  splits, Gradle composite builds, version catalogs, dependency locking, reproducible builds,
  remote build cache, IDE setup (IntelliJ, VSCode), JDK runtime flags (--add-opens), Renovate
  / Dependabot for Gradle, plugin-template repos, and a comprehensive build-failure cookbook.
---

# 1. BUILD & TOOLING — DEEP DIVE (v2)

This document is a comprehensive companion to §1 of `paper-plugin-dev.md`. The base file
covers Gradle KTS, Maven, Shadow, run-paper, paperweight-userdev, and the Java 21 toolchain
at a sketch level. This file expands every one of those areas with verified configuration,
explains the underlying tools (so you can debug them), and adds every adjacent topic a
production plugin author needs.

All version pins below were verified against official sources at write time. Treat them as
floors — newer versions are usually drop-in compatible. Authoritative sources are linked at
the end of each subsection or in the [References](#1z-references) appendix.

---

## 1.0 MENTAL MODEL OF THE TOOLCHAIN

Before any config: understand what each layer does. This avoids 90% of "weird Gradle errors".

```
                        ┌─────────────────────────────┐
                        │  Your IDE (IntelliJ/VSCode) │
                        └──────────────┬──────────────┘
                                       │ syncs with
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │  Gradle wrapper (gradlew)  →  Gradle daemon       │
        │   - reads settings.gradle.kts                     │
        │   - reads build.gradle.kts                        │
        │   - resolves plugin classpath from                │
        │     plugins.gradle.org & PaperMC repo             │
        └──────────────┬───────────────────────────────────┘
                       │ applies
        ┌──────────────▼─────────────────────────────────────┐
        │  Plugins                                           │
        │   ├── java                                         │
        │   ├── com.gradleup.shadow      (fat jar / relocate)│
        │   ├── xyz.jpenilla.run-paper   (test server)       │
        │   ├── io.papermc.paperweight.userdev (NMS access)  │
        │   ├── com.diffplug.spotless    (format / license)  │
        │   ├── checkstyle / pmd / spotbugs                  │
        │   ├── net.ltgt.errorprone & net.ltgt.nullaway      │
        │   └── version catalogs / dependency locking        │
        └──────────────┬─────────────────────────────────────┘
                       │ produces
        ┌──────────────▼─────────────────────────────────────┐
        │  build/libs/MyPlugin-1.0.0.jar                     │
        │   - Java 21 bytecode                               │
        │   - shaded libs in com.example.libs.*              │
        │   - plugin.yml with token-expanded version         │
        │   - META-INF/MANIFEST.MF                           │
        └────────────────────────────────────────────────────┘
```

Knowing **which layer fails** when you see an error tells you where to look:
- "Could not resolve" → repository / network
- "Plugin already initialized" → Shadow relocation collision
- "Cannot find symbol" only at runtime → `compileOnly` vs `implementation` mismatch
- "Unsupported class file major version" → JDK / toolchain mismatch

---

## 1.0.1 BUILD SYSTEM CHOICE — GRADLE VS MAVEN VS OTHERS

| System | Recommended for | Notes |
|---|---|---|
| **Gradle KTS** | New Paper plugin projects | First-class plugin support, fastest iteration, all Paper tooling targets it |
| **Gradle Groovy** | Existing legacy projects | Same capability as KTS, slightly worse IDE support |
| **Maven** | Teams forced to use Maven (corp policy, existing pipelines) | Works fine but `paperweight-userdev` is **Gradle only** — Maven users must use [paper-nms-maven-plugin](https://github.com/Alvinn8/paper-nms-maven-plugin) for NMS |
| **Bazel / Buck / Pants** | Polyglot mono-repos | No Paper integration exists. Possible to wire `java_binary` rules manually but no community support — out of scope for this guide |
| **sbt** | Scala-only shops | Possible; relocation handled via sbt-assembly. Niche, no template repos exist |

If you're starting fresh and have any choice: **Gradle KTS**.

---

## 1.A GRADLE GROOVY DSL (Legacy — but still 30%+ of repos)

If you join an existing Bukkit/Spigot project from before ~2022, it will be Groovy. Don't migrate
unless you have a reason; both DSLs are first-class citizens in Gradle.

```groovy
// settings.gradle
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven { url = 'https://repo.papermc.io/repository/maven-public/' }
    }
}
rootProject.name = 'MyPlugin'

// build.gradle
plugins {
    id 'java'
    id 'com.gradleup.shadow'           version '8.3.5'
    id 'xyz.jpenilla.run-paper'        version '2.3.1'
    // id 'io.papermc.paperweight.userdev' version '2.0.0-beta.19'
}

group   = 'com.example'
version = '1.0.0'

repositories {
    mavenCentral()
    maven { url = 'https://repo.papermc.io/repository/maven-public/' }
}

dependencies {
    compileOnly 'io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

processResources {
    filteringCharset = 'UTF-8'
    filesMatching('plugin.yml') {
        expand version: project.version, name: project.name
    }
}

runServer {
    minecraftVersion '1.21.4'
}

shadowJar {
    archiveClassifier.set('')
}
build.dependsOn shadowJar
```

**KTS vs Groovy decision matrix:**

| Factor | KTS | Groovy |
|---|---|---|
| IDE autocomplete | ★★★★★ | ★★ |
| First-time config time | slower | faster |
| Type errors at parse time | yes | no |
| Eval performance | slower (compile to JVM) | faster |
| Existing plugin examples online | growing | abundant |
| Recommended for new projects | yes | only if team knows Groovy |

**Migration path:** rename `build.gradle` → `build.gradle.kts` and `settings.gradle` →
`settings.gradle.kts`, then run `./gradlew help`. The error messages walk you through the
syntactic differences (parens, equals-vs-set, string templating).

---

## 1.A.1 MAVEN — Full equivalent

A complete Maven `pom.xml` mirroring the recommended Gradle setup:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>myplugin</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <repositories>
        <repository>
            <id>papermc</id>
            <url>https://repo.papermc.io/repository/maven-public/</url>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>io.papermc.paper</groupId>
            <artifactId>paper-api</artifactId>
            <version>1.21.4-R0.1-SNAPSHOT</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>
    </dependencies>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>      <!-- enables ${project.version} expansion in plugin.yml -->
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <release>21</release>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <createDependencyReducedPom>false</createDependencyReducedPom>
                            <relocations>
                                <relocation>
                                    <pattern>com.zaxxer.hikari</pattern>
                                    <shadedPattern>com.example.myplugin.libs.hikari</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <!-- Static analysis equivalents -->
            <plugin>
                <groupId>com.diffplug.spotless</groupId>
                <artifactId>spotless-maven-plugin</artifactId>
                <version>2.43.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-checkstyle-plugin</artifactId>
                <version>3.5.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

**Maven equivalents of Gradle plugins:**

| Gradle | Maven |
|---|---|
| `com.gradleup.shadow` | `maven-shade-plugin` |
| `xyz.jpenilla.run-paper` | No equivalent — manual server setup |
| `io.papermc.paperweight.userdev` | [paper-nms-maven-plugin](https://github.com/Alvinn8/paper-nms-maven-plugin) |
| `com.diffplug.spotless` | `spotless-maven-plugin` |
| `checkstyle` | `maven-checkstyle-plugin` |
| `net.ltgt.errorprone` | `maven-compiler-plugin` `<annotationProcessorPaths>` |
| Gradle dependency locking | Maven `dependency:tree` + manual review |
| Gradle version catalogs | `<dependencyManagement>` |

**Why teams pick Maven anyway:**
- Existing build infrastructure (corp Nexus / Artifactory)
- Team unfamiliarity with Gradle
- Simpler conceptual model (XML, fixed lifecycle)

**Why Gradle is preferred for Paper plugins:**
- `paperweight-userdev` is Gradle-only
- `run-paper` is Gradle-only
- Most Paper community templates are Gradle
- Faster builds (incremental compilation, daemon, configuration cache)

---

## 1.B PAPERWEIGHT-USERDEV (NMS access for plugins)

This is the *plugin-side* paperweight. It downloads the Mojang-mapped Paper server jar so
your plugin can reference `net.minecraft.*` and `org.bukkit.craftbukkit.*` classes during
compile, then on the server those classes work because (since 1.20.5) Paper ships
Mojang-mapped at runtime. See [paperweight-userdev docs](https://docs.papermc.io/paper/dev/userdev/).

### 1.B.1 Minimal config

```kotlin
// build.gradle.kts
plugins {
    java
    id("io.papermc.paperweight.userdev") version "2.0.0-beta.19"
    id("xyz.jpenilla.run-paper") version "2.3.1"
}

dependencies {
    paperweight.paperDevBundle("1.21.4-R0.1-SNAPSHOT")
}

java { toolchain.languageVersion.set(JavaLanguageVersion.of(21)) }
```

You no longer add `compileOnly("io.papermc.paper:paper-api:...")` — `paperDevBundle` brings
in both the API and the server internals.

### 1.B.2 What it actually does

1. Downloads the Paper "dev bundle" (a tarball containing the Mojang-mapped server jar,
   sources, the Paper API jar, and reobf mappings).
2. Extracts it into `~/.gradle/caches/paperweight/`.
3. Adds the mapped server jar to your `compileOnly` classpath.
4. **Pre-1.20.5 only:** registers a `reobfJar` task that re-obfuscates your compiled output
   from Mojang names to Spigot's mappings before publishing. **From 1.20.5 onward this step
   is no longer needed** — Paper itself runs Mojang-mapped at runtime, so your compiled jar
   already matches what the server expects.

### 1.B.3 Reobf vs. Mojmap runtime — the historical context

Before 1.20.5 Paper kept Spigot's obfuscation at runtime so plugins compiled against
"Spigot-mapped" code worked. Plugins that wanted readable NMS names had to compile against
Mojang names and then `reobfJar` translated those names back to Spigot's at build time.

After 1.20.5 Paper switched to running Mojang-mapped. Plugins now match server bytecode
1:1, so:

- **`./gradlew build`** produces a plugin jar with Mojang names — works on Paper 1.20.5+.
- **`./gradlew reobfJar`** is unnecessary on modern Paper. Some legacy plugins still call
  it; that produces a Spigot-mapped jar that runs on Spigot but **not** on modern Paper.

### 1.B.4 Custom Paper version (e.g. testing a snapshot)

```kotlin
dependencies {
    paperweight.paperDevBundle("1.21.5-R0.1-SNAPSHOT")
}
```

Or pin a specific build number from PaperMC repo:

```kotlin
dependencies {
    paperweight.devBundle("io.papermc.paper", "1.21.4-R0.1-20241224.123456-789")
}
```

### 1.B.5 NMS code style guidance

```java
// In a class with paperweight active:
import net.minecraft.server.level.ServerPlayer;
import net.minecraft.network.protocol.game.ClientboundSetTitleTextPacket;
import org.bukkit.craftbukkit.entity.CraftPlayer;       // No version suffix on 1.20.5+
import net.kyori.adventure.text.Component;
import io.papermc.paper.adventure.PaperAdventure;       // Adventure -> NMS bridge

public final class PacketUtil {
    public static void sendNmsTitle(Player player, Component title) {
        ServerPlayer nms = ((CraftPlayer) player).getHandle();
        var nmsComponent = PaperAdventure.asVanilla(title);
        nms.connection.send(new ClientboundSetTitleTextPacket(nmsComponent));
    }
}
```

> **Cardinal NMS rule:** never call NMS from async threads. The internal types are not
> thread-safe and the server explodes silently in production.

### 1.B.6 Avoiding NMS where possible

Almost every NMS use case in 2025 has a Paper-API replacement. Audit before reaching for it:

| You want | Paper API alternative |
|---|---|
| Send a packet | `Player#sendPlayerListHeaderAndFooter`, BossBar API, MapView |
| Spawn fake entity | Display Entities (1.19.4+) |
| Modify entity AI | `mob.getPathfinder()`, `mob.setTarget(...)` |
| Read raw NBT | PDC + Data Component API |
| Custom enchantments | Enchantment registry events (1.21+) |
| Resource pack prompt | `ResourcePackRequest` |

Use NMS when you genuinely need a feature absent from the API (e.g. custom mob brains,
packet interception). Otherwise the API is faster to write, doesn't require paperweight,
and survives Minecraft updates without recompilation.

---

## 1.C PAPERWEIGHT-PATCHER (Building a Paper Fork)

For Paper *forks* (Purpur-style projects), not plugins. If you're writing a plugin you can
skip this section. See [PaperMC/paperweight](https://github.com/PaperMC/paperweight).

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}
rootProject.name = "myfork"

// build.gradle.kts
plugins {
    id("io.papermc.paperweight.patcher") version "2.0.0-beta.19"
}

paperweight {
    upstreams.register("paper") {
        ref = providers.gradleProperty("paperRef") // commit SHA in gradle.properties
        patchTasks.register("server") {
            upstreamPath = "paper-server"
            patchesDir   = file("patches/server")
            outputDir    = file("myfork-server")
        }
        patchTasks.register("api") {
            upstreamPath = "paper-api"
            patchesDir   = file("patches/api")
            outputDir    = file("myfork-api")
        }
    }
}
```

**Layout:**
```
myfork/
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties           # paperRef=<commit sha>
├── patches/
│   ├── api/                    # *.patch files applied to paper-api
│   └── server/                 # *.patch files applied to paper-server
├── myfork-api/                 # generated source tree (gitignored)
└── myfork-server/              # generated source tree (gitignored)
```

**Common tasks:**
```bash
./gradlew applyPatches               # apply patches → editable source tree
./gradlew rebuildPatches             # diff editable tree → write back to patches/
./gradlew createMojmapPaperclipJar   # build runnable Paperclip jar (Mojang-mapped runtime)
./gradlew createReobfPaperclipJar    # legacy Spigot-mapped runtime
```

**When to fork instead of plugin:**
- Custom dimensions or world data formats
- Networking-layer rewrites (compression, encryption alternatives)
- Removing core gameplay (e.g. permanently disabling Nether)
- Mixin-style rewrites of vanilla physics

Almost everything else should be a plugin or a Mixin/access-widener mod.

---

## 1.D RUN-PAPER (Test Server in CI / Local)

[xyz.jpenilla.run-paper](https://github.com/jpenilla/run-task) launches a real Paper server
with your plugin pre-installed and configured. Iteration loop: `./gradlew runServer` →
the JAR is built, copied into `build/run/plugins/`, and Paper boots.

### 1.D.1 Standard config

```kotlin
plugins {
    id("xyz.jpenilla.run-paper") version "2.3.1"
}

tasks.runServer {
    minecraftVersion("1.21.4")              // pulls latest Paper build for that MC version

    // Pass JVM args to the test server (mirror what your prod runs):
    jvmArgs("-Xmx4G", "-Xms2G",
            "-XX:+UseG1GC",
            "-XX:+UnlockExperimentalVMOptions",
            "-XX:+ParallelRefProcEnabled",
            "-XX:MaxGCPauseMillis=200")

    // Pass system properties (e.g. enable Aikar's flags in development):
    systemProperty("paper.disableChannelLimit", "true")
    systemProperty("com.mojang.eula.agree", "true")  // never use this in prod
}
```

### 1.D.2 Multiple test versions in one project

```kotlin
val runServerLatest by tasks.registering(xyz.jpenilla.runpaper.task.RunServer::class) {
    minecraftVersion("1.21.4")
    pluginJars(tasks.shadowJar.flatMap { it.archiveFile })
    runDirectory.set(layout.buildDirectory.dir("run-1.21.4"))
}

val runServerLegacy by tasks.registering(xyz.jpenilla.runpaper.task.RunServer::class) {
    minecraftVersion("1.20.6")
    pluginJars(tasks.shadowJar.flatMap { it.archiveFile })
    runDirectory.set(layout.buildDirectory.dir("run-1.20.6"))
}
```

This is invaluable when verifying that a plugin still works across the API breaks at
1.20.5 (Mojmap runtime) and 1.21.4 (hardfork).

### 1.D.3 Running Velocity / Folia variants

The same author also publishes [run-velocity](https://github.com/jpenilla/run-task) and
[run-paper](https://github.com/jpenilla/run-task) for Folia-fork test runs:

```kotlin
plugins {
    id("xyz.jpenilla.run-velocity") version "2.3.1"
}

tasks.runVelocity {
    velocityVersion("3.4.0-SNAPSHOT")
}
```

### 1.D.4 Hot-swap during runServer

Paper does not officially support hot reload, but you can use **DCEVM + HotswapAgent** to
hot-swap classes during a `runServer` debug session. Sketch:

```kotlin
tasks.runServer {
    jvmArgs("-XX:+AllowEnhancedClassRedefinition",     // requires JBR / DCEVM JDK
            "-javaagent:hotswap-agent.jar")
}
```

This is brittle and version-sensitive; most teams instead lean on a fast `./gradlew build &&
./gradlew runServer` cycle.

---

## 1.E SHADOW (Fat JARs and Relocation)

[com.gradleup.shadow](https://gradleup.com/shadow/) is the maintained successor to
`com.github.johnrengelman.shadow`.

### 1.E.1 Why Shadow

A plugin jar must contain its dependencies because the Bukkit classloader does **not**
share Maven dependencies between plugins (each plugin gets its own classloader, except for
plugins explicitly listed under `loadbefore`/`depend`). If two plugins each ship the same
library at incompatible versions, you get `LinkageError`. Relocation (renaming the dep's
package) avoids that collision.

### 1.E.2 Standard config

```kotlin
plugins {
    id("com.gradleup.shadow") version "8.3.5"
}

dependencies {
    implementation("com.zaxxer:HikariCP:5.1.0")
    implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
}

tasks.shadowJar {
    archiveClassifier.set("")     // produce MyPlugin-1.0.0.jar, not MyPlugin-1.0.0-all.jar

    // Relocate every shaded dep so it can't collide with another plugin:
    relocate("com.zaxxer.hikari",          "com.example.myplugin.libs.hikari")
    relocate("com.github.benmanes.caffeine","com.example.myplugin.libs.caffeine")
    relocate("org.jetbrains",              "com.example.myplugin.libs.jetbrains")
    relocate("kotlin",                     "com.example.myplugin.libs.kotlin")

    // Strip useless metadata to shrink jar:
    minimize {
        exclude(dependency("com.zaxxer:HikariCP:.*"))   // keep all of HikariCP
    }

    mergeServiceFiles()           // critical for SLF4J / META-INF/services merging
}
tasks.build { dependsOn(tasks.shadowJar) }
tasks.jar { enabled = false }       // skip plain jar, ship only shaded
```

### 1.E.3 What to relocate (heuristic)

Always relocate libraries that are **commonly bundled** by other plugins (you don't know
who else loads them):
- Kotlin stdlib (`kotlin`, `kotlinx`)
- Adventure (only if you bundle your own copy — Paper bundles it, so don't)
- Jackson, Gson
- Caffeine
- HikariCP
- jOOQ
- Configurate
- Apache Commons (any version)

Do **not** relocate:
- The Bukkit/Paper API (it's `compileOnly` and never enters your jar)
- Adventure (Paper provides it; relocating breaks event types)
- SLF4J/Log4j (Paper provides them)

### 1.E.4 Library loader as an alternative

Paper's library loader (`paper-plugin.yml` `loader:` or `plugin.yml` `libraries:`) downloads
deps at server startup into an isolated classloader. **No Shadow needed.** Trade-off:

| Shadow | Library loader |
|---|---|
| Plugin jar 5–30 MB | Plugin jar small |
| Works on Spigot / Bukkit / forks | Paper-only |
| Works offline | Server needs internet on first start |
| You ship the exact lib version | Server can pin or share |
| Relocation required to avoid collisions | Auto-isolated by classloader |

Most modern Paper plugins use library loader for big deps and Shadow only for tiny utility
jars or things that need to be relocated.

### 1.E.5 Common Shadow gotchas

```text
LinkageError: loader constraint violation
  → Two plugins shaded the same lib at the same package. Relocate.

NoClassDefFoundError: org/jetbrains/annotations/NotNull
  → A compileOnly annotation lib leaked at compile time. Use compileOnlyApi or shade it.

ServiceLoader can't find provider
  → Forgot tasks.shadowJar { mergeServiceFiles() }.

Plugin jar suddenly 80 MB
  → A transitive `implementation` pulled in something huge. Use `./gradlew dependencies`
    to inspect; switch big deps to compileOnly + library loader.
```

---

## 1.F STATIC ANALYSIS

A serious plugin runs CI with at minimum: Spotless + Checkstyle (or PMD/SpotBugs) + Error
Prone + NullAway. Each tool catches a different bug class.

### 1.F.1 Spotless (formatting + license headers)

Sources: [diffplug/spotless](https://github.com/diffplug/spotless).

```kotlin
plugins {
    id("com.diffplug.spotless") version "6.25.0"
}

spotless {
    java {
        target("src/**/*.java")
        palantirJavaFormat("2.50.0")           // or googleJavaFormat() / eclipse()
        removeUnusedImports()
        importOrder("java", "javax", "org", "com", "io", "net")
        endWithNewline()
        trimTrailingWhitespace()
        licenseHeaderFile(file("HEADER.txt"))
    }
    kotlin {
        target("src/**/*.kt")
        ktlint("1.3.1")
    }
    format("yaml") {
        target("**/*.yml", "**/*.yaml")
        prettier()
    }
    format("misc") {
        target("*.md", "*.gitignore", "*.editorconfig")
        trimTrailingWhitespace()
        endWithNewline()
    }
}

tasks.named("check") { dependsOn("spotlessCheck") }
// Auto-fix locally: ./gradlew spotlessApply
```

`HEADER.txt`:

```
/*
 * Copyright (c) $YEAR YourTeam
 * Licensed under the MIT License.
 */
```

### 1.F.2 Checkstyle

```kotlin
plugins { checkstyle }

checkstyle {
    toolVersion    = "10.18.2"
    configFile     = rootProject.file("config/checkstyle/checkstyle.xml")
    isIgnoreFailures = false
    maxWarnings    = 0
}

tasks.withType<Checkstyle>().configureEach {
    reports {
        xml.required.set(false)
        html.required.set(true)
    }
}
```

Minimum `config/checkstyle/checkstyle.xml`:

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
    <property name="charset" value="UTF-8"/>
    <module name="TreeWalker">
        <module name="UnusedImports"/>
        <module name="RedundantImport"/>
        <module name="MissingOverride"/>
        <module name="EmptyBlock"/>
        <module name="OneStatementPerLine"/>
        <module name="MultipleVariableDeclarations"/>
        <module name="EqualsHashCode"/>
        <module name="StringLiteralEquality"/>
        <module name="FallThrough"/>
        <module name="MissingSwitchDefault"/>
    </module>
    <module name="FileTabCharacter"><property name="eachLine" value="true"/></module>
    <module name="LineLength">
        <property name="max" value="120"/>
        <property name="ignorePattern" value="^\s*\*.*https?://"/>
    </module>
</module>
```

### 1.F.3 PMD

```kotlin
plugins { pmd }

pmd {
    toolVersion = "7.7.0"
    ruleSetFiles = files("config/pmd/ruleset.xml")
    isIgnoreFailures = false
}
```

PMD overlaps with Checkstyle but catches more semantic issues (dead code, suspicious
control flow). Pick one of {Checkstyle, PMD} — running both is overkill.

### 1.F.4 SpotBugs

```kotlin
plugins { id("com.github.spotbugs") version "6.0.26" }

spotbugs {
    toolVersion.set("4.8.6")
    effort.set(com.github.spotbugs.snom.Effort.MAX)
    reportLevel.set(com.github.spotbugs.snom.Confidence.LOW)
    excludeFilter.set(file("config/spotbugs/exclude.xml"))
}
```

SpotBugs runs on bytecode, so it catches runtime bugs (null deref, infinite recursion,
unsafe locking). Highly recommended for plugin code that touches threads.

### 1.F.5 Error Prone + NullAway

Sources: [tbroyer/gradle-errorprone-plugin](https://github.com/tbroyer/gradle-errorprone-plugin),
[uber/NullAway](https://github.com/uber/NullAway), [tbroyer/gradle-nullaway-plugin](https://github.com/tbroyer/gradle-nullaway-plugin).

```kotlin
plugins {
    id("net.ltgt.errorprone") version "4.1.0"
    id("net.ltgt.nullaway")   version "2.1.0"
}

dependencies {
    errorprone("com.google.errorprone:error_prone_core:2.36.0")
    errorprone("com.uber.nullaway:nullaway:0.12.1")

    // Either of these (pick one — don't mix):
    compileOnly("org.jspecify:jspecify:1.0.0")          // modern, recommended
    // compileOnly("org.jetbrains:annotations:26.0.1")  // legacy, more familiar
}

tasks.withType<JavaCompile>().configureEach {
    options.errorprone {
        disableWarningsInGeneratedCode.set(true)
        error("UnusedVariable", "DefaultCharset", "MissingOverride", "EqualsHashCode")
        nullaway {
            severity.set(net.ltgt.gradle.errorprone.CheckSeverity.ERROR)
            annotatedPackages.add("com.example.myplugin")
            unannotatedSubPackages.add("org.bukkit")            // Bukkit's null annotations are inconsistent
            unannotatedSubPackages.add("org.spigotmc")
            unannotatedSubPackages.add("io.papermc")
            handleTestAssertionLibraries.set(true)
            checkOptionalEmptiness.set(true)
        }
    }
}
```

**Why NullAway is worth it for plugins:** Bukkit / Paper APIs that frequently return
`null`:
- `Bukkit.getPlayer(String)` (player offline)
- `world.getBlockAt(...).getRelative(...)` (chain breaks at world border)
- `event.getEntity()` for some events (cleared mid-tick)
- `ItemStack#getItemMeta()` (null on AIR)
- `Location#getWorld()` (deserialized from disk before world load)

NullAway forces you to handle each one *or* explicitly opt out at the call site.

### 1.F.6 Putting it all together

```kotlin
tasks.named("check") {
    dependsOn(
        "spotlessCheck",
        "checkstyleMain",
        "spotbugsMain",
        // Error Prone runs as part of compileJava, not a separate task
    )
}
```

Run `./gradlew check` in CI; fail the build on any deviation.

---

## 1.G ANNOTATION PROCESSORS / CODEGEN

### 1.G.1 Lombok (controversial)

```kotlin
dependencies {
    compileOnly("org.projectlombok:lombok:1.18.36")
    annotationProcessor("org.projectlombok:lombok:1.18.36")
    testCompileOnly("org.projectlombok:lombok:1.18.36")
    testAnnotationProcessor("org.projectlombok:lombok:1.18.36")
}
```

Java 21 records cover the common cases (`@Value`, `@Data` for DTOs). Lombok still helps for
mutable fluent builders or JPA entities. Avoid `@SneakyThrows` — it hides checked exceptions
and breaks Bukkit's logging. **Recommendation: prefer records + sealed types.**

### 1.G.2 Immutables / AutoValue

```kotlin
dependencies {
    compileOnly("org.immutables:value:2.10.1:annotations")
    annotationProcessor("org.immutables:value:2.10.1")
}
```

Records mostly replaced this. Still useful for very large structs with optional fields.

### 1.G.3 MapStruct (DTO mapping)

```kotlin
dependencies {
    implementation("org.mapstruct:mapstruct:1.6.3")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.6.3")
}
```

Useful when you have a config-DTO ↔ runtime-model layer and don't want hand-written
copy logic.

### 1.G.4 Google AutoService (META-INF/services generation)

```kotlin
dependencies {
    compileOnly("com.google.auto.service:auto-service-annotations:1.1.1")
    annotationProcessor("com.google.auto.service:auto-service:1.1.1")
}
```

```java
@AutoService(MyPluginAPI.class)
public class MyPluginAPIImpl implements MyPluginAPI { ... }
```

Generates `META-INF/services/com.example.MyPluginAPI` so `ServiceLoader` finds your impl.
Nice when you ship multiple plugins that should auto-discover each other.

---

## 1.H KOTLIN PLUGIN SUPPORT

Kotlin works on Paper. The two pitfalls: stdlib relocation and main-class inheritance.

### 1.H.1 Standard config

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    id("com.gradleup.shadow") version "8.3.5"
}

dependencies {
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    implementation(kotlin("stdlib"))
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1")
}

kotlin {
    jvmToolchain(21)
}

tasks.shadowJar {
    archiveClassifier.set("")
    relocate("kotlin",  "com.example.myplugin.libs.kotlin")
    relocate("kotlinx", "com.example.myplugin.libs.kotlinx")
    minimize {
        exclude(dependency("org.jetbrains.kotlin:.*"))
    }
}
```

### 1.H.2 Common pitfalls

- **Companion objects in main class:** Bukkit's reflection-based loader can't always
  instantiate plugins with companions in the main class. Keep `class MyPlugin : JavaPlugin()`
  pure; put statics in a separate `object`.
- **Top-level extensions** (`fun Player.heal()`) compile to static methods in `*Kt.class`.
  Multiple plugins shipping different stdlibs collide unless you relocate.
- **`object` singletons under Paper plugin classloader isolation:** each plugin instance
  gets its own singleton — don't rely on cross-plugin singleton sharing.

### 1.H.3 MCCoroutine — bridging Bukkit main thread to Kotlin coroutines

[Shynixn/MCCoroutine](https://github.com/Shynixn/MCCoroutine) provides a Bukkit dispatcher.

```kotlin
class MyPlugin : SuspendingJavaPlugin() {
    override suspend fun onEnableAsync() {
        // suspend on the Bukkit main dispatcher
        val data = withContext(Dispatchers.IO) { loadFromDb() }
        Bukkit.broadcast(Component.text("Loaded ${data.size} entries"))
    }
}

// Listener with suspend handlers:
class MyListener : SuspendingListener {
    @EventHandler
    suspend fun onJoin(event: PlayerJoinEvent) {
        val profile = withContext(Dispatchers.IO) { fetchProfile(event.player.uniqueId) }
        event.player.sendMessage(Component.text("Welcome, ${profile.title}"))
    }
}
```

For Folia, use the Folia variant of MCCoroutine, which routes coroutines to the appropriate
region scheduler.

### 1.H.4 Adventure-Kotlin DSL (optional)

```kotlin
implementation("net.kyori:adventure-extra-kotlin:4.17.0")
```

```kotlin
val msg = text {
    content("Welcome ")
    color(NamedTextColor.GREEN)
    append(player.displayName())
    decoration(TextDecoration.BOLD, true)
}
```

---

## 1.I SCALA / GROOVY PLUGINS

Possible but rare.

```kotlin
plugins {
    scala
    id("com.gradleup.shadow") version "8.3.5"
}
dependencies {
    implementation("org.scala-lang:scala3-library_3:3.5.2")
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
}
tasks.shadowJar {
    relocate("scala", "com.example.myplugin.libs.scala")
}
```

Performance and tooling are inferior to Java/Kotlin for plugin work. Choose only if your
team is heavily Scala-fluent.

---

## 1.J JAR-IN-JAR / MULTI-RELEASE JARs / LIBRARY LOADERS

Three different ways to ship dependencies. Choose by trade-off:

### 1.J.1 paper-plugin libraries (preferred for Paper-only)

```yaml
# paper-plugin.yml
name: MyPlugin
main: com.example.MyPlugin
api-version: '1.21.4'
loader: com.example.MyLoader
```

```java
public class MyLoader implements PluginLoader {
    @Override public void classloader(PluginClasspathBuilder classpathBuilder) {
        var resolver = new MavenLibraryResolver();
        resolver.addRepository(new RemoteRepository.Builder(
            "central", "default",
            MavenLibraryResolver.MAVEN_CENTRAL_DEFAULT_MIRROR).build());
        resolver.addDependency(new Dependency(
            new DefaultArtifact("com.zaxxer:HikariCP:5.1.0"), null));
        resolver.addDependency(new Dependency(
            new DefaultArtifact("com.github.ben-manes.caffeine:caffeine:3.1.8"), null));
        classpathBuilder.addLibrary(resolver);
    }
}
```

### 1.J.2 plugin.yml libraries (Bukkit-style)

```yaml
libraries:
  - com.zaxxer:HikariCP:5.1.0
  - org.jetbrains.kotlin:kotlin-stdlib:2.1.0
```

Paper resolves these from Maven Central at startup. Smaller jars, automatic isolation,
no Shadow gymnastics. Trade-off: your server needs internet on first launch.

### 1.J.3 Multi-release JAR (rare for plugins)

```kotlin
sourceSets {
    create("java21") {
        java { srcDir("src/main/java21") }
    }
}
tasks.named<JavaCompile>("compileJava21Java") { options.release.set(21) }
tasks.jar {
    manifest { attributes["Multi-Release"] = "true" }
    into("META-INF/versions/21") { from(sourceSets["java21"].output) }
}
```

Paper requires Java 21+ on 1.21.x, so multi-release usually isn't needed. Useful if you
ship a library JAR consumed by both modern and legacy code.

---

## 1.K MULTI-MODULE: API + IMPL SPLIT

When your plugin exposes an API for other plugins:

```
myplugin/
├── settings.gradle.kts
├── build.gradle.kts             # convention plugin / common config
├── api/
│   └── build.gradle.kts
└── plugin/
    └── build.gradle.kts
```

```kotlin
// settings.gradle.kts
rootProject.name = "MyPlugin"
include("api", "plugin")

// buildSrc/src/main/kotlin/myplugin.common.gradle.kts (convention plugin)
plugins { java }
java { toolchain.languageVersion.set(JavaLanguageVersion.of(21)) }
repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}
dependencies {
    "compileOnly"("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
}

// api/build.gradle.kts
plugins {
    id("myplugin.common")
    `java-library`
    `maven-publish`
}
publishing {
    publications {
        create<MavenPublication>("api") { from(components["java"]) }
    }
}

// plugin/build.gradle.kts
plugins {
    id("myplugin.common")
    id("com.gradleup.shadow") version "8.3.5"
}
dependencies {
    implementation(project(":api"))
    implementation("com.zaxxer:HikariCP:5.1.0")
}
tasks.shadowJar {
    archiveClassifier.set("")
    relocate("com.zaxxer.hikari", "com.example.myplugin.libs.hikari")
}
```

Other developers depend on a thin `com.example:myplugin-api:1.0.0` JAR while the runtime
plugin shades implementation deps. This is how LuckPerms, WorldGuard, and most major
plugins ship.

---

## 1.L COMPOSITE BUILDS

[Composite builds](https://docs.gradle.org/current/userguide/composite_builds.html) let
you `includeBuild("../mylib")` so changes to `mylib` are picked up without publishing to
Maven Local.

```kotlin
// settings.gradle.kts
includeBuild("../my-shared-lib")

// build.gradle.kts
dependencies {
    implementation("com.example:my-shared-lib")  // resolved from the included build
}
```

When to use:
- You maintain a shared utility library across multiple plugins.
- You want to debug into the lib's source from your plugin without publishing.
- You're refactoring the lib API and want both projects to compile together.

---

## 1.M VERSION CATALOGS + DEPENDABOT/RENOVATE

```toml
# gradle/libs.versions.toml — verified format per Gradle 8.5+ docs
[versions]
paper      = "1.21.4-R0.1-SNAPSHOT"
shadow     = "8.3.5"
runPaper   = "2.3.1"
hikari     = "5.1.0"
caffeine   = "3.1.8"
adventure  = "4.17.0"
jackson    = "2.18.2"
kotlin     = "2.1.0"
errorprone = "2.36.0"
nullaway   = "0.12.1"

[libraries]
paper-api     = { group = "io.papermc.paper", name = "paper-api", version.ref = "paper" }
hikari        = { group = "com.zaxxer", name = "HikariCP", version.ref = "hikari" }
caffeine      = { group = "com.github.ben-manes.caffeine", name = "caffeine", version.ref = "caffeine" }
jackson-core  = { group = "com.fasterxml.jackson.core", name = "jackson-databind", version.ref = "jackson" }

[bundles]
jackson = ["jackson-core"]            # use bundle when 3+ libs always go together

[plugins]
shadow     = { id = "com.gradleup.shadow",         version.ref = "shadow" }
runPaper   = { id = "xyz.jpenilla.run-paper",      version.ref = "runPaper" }
kotlin     = { id = "org.jetbrains.kotlin.jvm",    version.ref = "kotlin" }
```

```kotlin
// build.gradle.kts
plugins {
    java
    alias(libs.plugins.shadow)
    alias(libs.plugins.runPaper)
}
dependencies {
    compileOnly(libs.paper.api)
    implementation(libs.hikari)
    implementation(libs.caffeine)
    implementation(libs.bundles.jackson)
}
```

### 1.M.1 Renovate config

`.github/renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":dependencyDashboard"],
  "schedule": ["before 4am on Monday"],
  "packageRules": [
    {
      "matchManagers": ["gradle-version-catalog"],
      "rangeStrategy": "pin"
    },
    {
      "matchPackageNames": ["io.papermc.paper:paper-api"],
      "groupName": "Paper API",
      "schedule": ["* * 1-7 * 1"]
    }
  ]
}
```

Renovate fully understands Gradle version catalogs ([renovatebot/renovate](https://docs.renovatebot.com/modules/manager/gradle/)).

### 1.M.2 Dependabot config

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule: { interval: "weekly" }
    groups:
      paper:
        patterns: ["io.papermc.*"]
      kotlin:
        patterns: ["org.jetbrains.kotlin*", "org.jetbrains.kotlinx:*"]
    open-pull-requests-limit: 5
```

GitHub's Dependabot supports version catalogs as of late 2024.

---

## 1.N DEPENDENCY LOCKING + REPRODUCIBLE BUILDS

```kotlin
// build.gradle.kts
dependencyLocking {
    lockAllConfigurations()
}

tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false
    isReproducibleFileOrder  = true
}

tasks.jar {
    manifest {
        attributes(
            "Implementation-Title"   to project.name,
            "Implementation-Version" to project.version,
            "Built-By"               to "ci",
            "Built-Jdk"              to "21"
        )
    }
}
```

Then:

```bash
./gradlew dependencies --write-locks   # creates gradle.lockfile
git add gradle.lockfile
```

Combined with locked deps you get **byte-identical jars** across machines. Critical for:
- Hash-based plugin signing
- CI cache hits
- Comparing CI vs local builds during incident response

---

## 1.O REMOTE BUILD CACHE & BUILD SCANS

```kotlin
// settings.gradle.kts
plugins {
    id("com.gradle.develocity") version "3.18.2"      // adds --scan support
}

develocity {
    buildScan {
        termsOfUseUrl.set("https://gradle.com/help/legal-terms-of-use")
        termsOfUseAgree.set("yes")
        publishing.onlyIf { System.getenv("CI") == "true" }
    }
}

buildCache {
    local {
        isEnabled = true
        directory = File(rootDir, ".gradle/build-cache")
    }
    remote<HttpBuildCache> {
        url = uri("https://cache.example.com/")
        isPush = System.getenv("CI") == "true"   // only CI writes
        credentials {
            username = providers.gradleProperty("buildCacheUser").orNull
            password = providers.gradleProperty("buildCacheKey").orNull
        }
    }
}
```

After any build, run with `./gradlew build --scan` and you get a permanent shareable URL
showing every task, every dependency resolved, every test, and where time was spent. This
is the single best Gradle debugging tool — share scans in bug reports.

For team plugins with long compile times (NMS-heavy projects compile 30s+), a remote cache
turns CI runs from 5 minutes into 30 seconds for unchanged tasks. See
[Develocity Build Cache docs](https://docs.gradle.com/develocity/2026.1/administration/build-acceleration/build-cache/).

---

## 1.O.1 GRADLE WRAPPER PINNING

Always commit a Gradle wrapper. It pins the exact Gradle version per repo so CI and dev
environments match.

```bash
./gradlew wrapper --gradle-version 8.10.2 --distribution-type all
```

`gradle/wrapper/gradle-wrapper.properties`:

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.10.2-all.zip
distributionSha256Sum=31c55713e40233a8303827ceb42ca48a47267a0ad4bab9177123121e71524c26
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

The `distributionSha256Sum` line is critical — it lets Gradle verify the downloaded
distribution wasn't tampered with. Get the hash from
[gradle.org/release-checksums](https://gradle.org/release-checksums/).

---

## 1.O.2 PUBLISHING TO MAVEN CENTRAL / GITHUB PACKAGES

For the API artifact in a multi-module split (§1.K), you'll publish to a Maven repo.

### Maven Central (Sonatype OSSRH)

```kotlin
plugins {
    `maven-publish`
    signing
}

publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["java"])

            pom {
                name.set("MyPlugin API")
                description.set("Public API for MyPlugin")
                url.set("https://github.com/example/myplugin")
                licenses {
                    license {
                        name.set("MIT")
                        url.set("https://opensource.org/licenses/MIT")
                    }
                }
                developers {
                    developer { id.set("you"); name.set("Your Name") }
                }
                scm {
                    connection.set("scm:git:git://github.com/example/myplugin.git")
                    url.set("https://github.com/example/myplugin")
                }
            }
        }
    }
    repositories {
        maven {
            name = "OSSRH"
            url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            credentials {
                username = providers.gradleProperty("ossrhUser").orNull
                password = providers.gradleProperty("ossrhPassword").orNull
            }
        }
    }
}

signing {
    val key      = providers.environmentVariable("SIGNING_KEY").orNull
    val password = providers.environmentVariable("SIGNING_PASSWORD").orNull
    if (key != null && password != null) {
        useInMemoryPgpKeys(key, password)
        sign(publishing.publications["maven"])
    }
}
```

### GitHub Packages (easier, no PGP)

```kotlin
publishing.repositories {
    maven {
        name = "GitHubPackages"
        url = uri("https://maven.pkg.github.com/example/myplugin")
        credentials {
            username = System.getenv("GITHUB_ACTOR")
            password = System.getenv("GITHUB_TOKEN")
        }
    }
}
```

### Hangar (PaperMC's plugin distribution)

Hangar publishing is a separate concern from API artifact publishing. Covered in file
`10-cicd-metrics-profiling-testing.md`.

---

## 1.P JDK RUNTIME FLAGS

Java 21's strong encapsulation breaks naive reflection. If you reflect into JDK internals
(rare in plugins, common in libraries), you must add `--add-opens` either via the server
launch script or via `gradle.properties`:

```properties
# gradle.properties (for build / test JVM only)
org.gradle.jvmargs=-Xmx4G -XX:+UseG1GC -Dfile.encoding=UTF-8 \
  --add-opens=java.base/java.lang=ALL-UNNAMED \
  --add-opens=java.base/java.lang.reflect=ALL-UNNAMED \
  --add-opens=java.base/java.util=ALL-UNNAMED \
  --add-opens=java.base/java.io=ALL-UNNAMED
```

For the server runtime (production), edit your start script:

```bash
java -Xmx8G \
  --add-opens=java.base/java.lang=ALL-UNNAMED \
  --add-opens=java.base/java.lang.reflect=ALL-UNNAMED \
  -jar paper.jar nogui
```

**When you actually need this:**
- ProtocolLib at certain versions
- ASM / bytecode rewriting libs
- Classfile transformers
- Some serialization libs touching `Unsafe`

If your plugin doesn't reflect into JDK internals, you don't need any flags.

---

## 1.Q IDE SETUP

### 1.Q.1 IntelliJ IDEA

1. Install the [Minecraft Development plugin](https://github.com/minecraft-dev/MinecraftDev).
   Adds project templates, plugin.yml inspections, NMS class navigation.
2. **File → Project Structure → SDK** → set Project SDK to **JDK 21** (Temurin, Adoptium,
   or JBR all work).
3. **Settings → Build, Execution, Deployment → Build Tools → Gradle** → set Gradle JVM to
   the same JDK 21.
4. **Settings → Editor → Code Style → Java → Imports** → match your Spotless / Checkstyle
   config so format-on-save doesn't fight CI.
5. **Run/Debug Configurations →** `Gradle` task `runServer` for one-click test server.

For NMS work, IntelliJ's "Decompile to Java" feature lets you read `net.minecraft.*`
sources directly even if paperweight didn't decompile them.

### 1.Q.2 VSCode

1. Install **Extension Pack for Java** + **Gradle for Java** (Microsoft).
2. `.vscode/settings.json`:

```json
{
  "java.configuration.runtimes": [
    { "name": "JavaSE-21", "path": "/path/to/jdk21", "default": true }
  ],
  "java.compile.nullAnalysis.mode": "automatic",
  "java.format.settings.url": "config/eclipse-format.xml",
  "gradle.javaDebug.cleanOutput": true
}
```

3. `.vscode/launch.json` for debugging the test server:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Attach to Paper",
      "request": "attach",
      "hostName": "localhost",
      "port": 5005
    }
  ]
}
```

Then run `./gradlew runServer -Dorg.gradle.jvmargs="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"`
and attach.

---

## 1.R `.editorconfig` & `gradle.properties`

`.editorconfig` (place at repo root):

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.{yml,yaml,json,toml}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

`gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4G -XX:+UseG1GC -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
# Configuration cache breaks some plugins; disable per-task with:
#   tasks.foo { notCompatibleWithConfigurationCache("reason") }
```

---

## 1.S TASK GRAPH BEST PRACTICES

```kotlin
// Always run linters before assembling:
tasks.named("build") {
    dependsOn("spotlessCheck", "checkstyleMain", "test")
}

// Skip flaky integration tests in normal builds:
tasks.test { useJUnitPlatform { excludeTags("integration") } }
tasks.register<Test>("integrationTest") {
    useJUnitPlatform { includeTags("integration") }
    shouldRunAfter("test")
}
tasks.named("check") { dependsOn("integrationTest") }

// Don't ship the unshaded jar:
tasks.jar { enabled = false }

// Always rebuild plugin.yml after version bumps:
tasks.processResources {
    inputs.property("version", project.version)
    filesMatching("plugin.yml") {
        expand("version" to project.version)
    }
}
```

---

## 1.T PLUGIN-TEMPLATE REPOS (Reference Implementations)

Solid starting points if you want a production-grade build out of the box:

- [CrimsonWarpedcraft/plugin-template](https://github.com/CrimsonWarpedcraft/plugin-template)
  — Maven, Spotless, Checkstyle, run-paper, GitHub Actions, releases via JReleaser.
- [jpenilla/run-task](https://github.com/jpenilla/run-task) — example tasks for Paper /
  Velocity / Folia.
- Paper's own [paperweight-test-plugin](https://github.com/PaperMC/paperweight) — minimal
  paperweight-userdev sample that the team uses to verify the toolchain.

Don't fork blindly; read what they include and adopt the pieces you need.

---

## 1.U COMMON BUILD FAILURES — DEEP TROUBLESHOOTING TABLE

| Symptom | Layer | Cause | Fix |
|---|---|---|---|
| `Could not resolve plugin io.papermc...` | settings.gradle.kts | Missing PaperMC repo in `pluginManagement` | Add `maven("https://repo.papermc.io/repository/maven-public/")` to `pluginManagement.repositories` |
| `Could not resolve io.papermc.paper:paper-api` | dependencies | Same repo missing in `repositories {}` | Add it to project repositories too |
| `Plugin already initialized` at server start | runtime | Two shaded plugins at same package | Add `relocate(...)` for the lib that matches |
| `NoClassDefFoundError` only on server | dependencies | Dep is `compileOnly` but should be shaded | Move to `implementation` or use library loader |
| `UnsupportedClassVersionError` on server | toolchain | JDK mismatch (compiled 21, server runs 17) | Match `java.toolchain` to server JDK |
| `plugin.yml` shows `${version}` literal | processResources | Token expansion not configured | Add `expand("version" to project.version)` |
| Shadow JAR is huge (>20 MB) | dependencies | Bundling `paper-api` accidentally | Verify `compileOnly`, run `./gradlew dependencies` |
| `IllegalAccessError` for `sun.*` | runtime | JDK encapsulation | Add `--add-opens` flags on the server JVM |
| `No service of type Plugin found` | gradle | Composite build path wrong | `includeBuild` path must point at `settings.gradle.kts` parent |
| `reobfJar` not found | task graph | Pre-1.20.5 paperweight installed | Upgrade to paperweight 2.x; reobfJar no longer auto-registered |
| Plugin works locally, fails in CI | env | JDK / locale mismatch | Pin JDK in `actions/setup-java`, set `LANG=en_US.UTF-8` |
| Configuration cache fails | gradle | Plugin uses `Project` at execution time | Disable per-task: `notCompatibleWithConfigurationCache("...")` |
| Daemon eats RAM | gradle | Too many parallel modules | Lower `org.gradle.workers.max` in `gradle.properties` |
| `Plugin is not compatible with this version` | runtime | `api-version` set higher than server | Drop `api-version` in plugin.yml or upgrade server |
| `Uberjar` contains JAR-in-JAR weirdness | shadow | Forgot `mergeServiceFiles()` | Add to `shadowJar { ... }` |
| Hot-reload changes don't apply | runtime | Paper deprecated `/reload` | Restart the server; don't rely on `/reload` |

---

## 1.V WHAT CHANGED IN 1.21.x BUILD-WISE

- **1.20.5** — Paper switched to Mojang-mapped runtime. `reobfJar` no longer required for
  plugins. paperweight 2.x is required for new projects.
- **1.21** (June 2024) — paperweight 2.0-beta line begins. Old paperweight 1.x configs
  fail to resolve new dev bundles.
- **1.21.4** (Dec 2024) — Paper hardforked from Spigot. Spigot patches now live inside
  Paper. Plugins compiled for Paper 1.21.4 may not work on Spigot 1.21.4 anymore.
- **1.21.5** (Mar 2025) — Protocol 770. No build-system breakage but bumped paper-api
  version.
- **1.21.7** — Dialog API enters API surface. Plugins targeting <1.21.7 must softdepend or
  feature-detect.

---

## 1.W MINIMAL REFERENCE BUILD

A complete, copy-paste-ready production build for a Paper 1.21.x plugin with all the
recommended hardening:

```kotlin
// settings.gradle.kts
rootProject.name = "MyPlugin"

pluginManagement {
    repositories {
        gradlePluginPortal()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}

// build.gradle.kts
plugins {
    java
    id("com.gradleup.shadow")            version "8.3.5"
    id("xyz.jpenilla.run-paper")         version "2.3.1"
    id("com.diffplug.spotless")          version "6.25.0"
    id("net.ltgt.errorprone")            version "4.1.0"
    id("net.ltgt.nullaway")              version "2.1.0"
    checkstyle
}

group   = "com.example"
version = "1.0.0"

dependencies {
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    implementation("com.zaxxer:HikariCP:5.1.0")

    errorprone("com.google.errorprone:error_prone_core:2.36.0")
    errorprone("com.uber.nullaway:nullaway:0.12.1")
    compileOnly("org.jspecify:jspecify:1.0.0")
}

java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(21))
    withSourcesJar()
}

checkstyle {
    toolVersion = "10.18.2"
    configFile  = rootProject.file("config/checkstyle/checkstyle.xml")
    maxWarnings = 0
}

spotless {
    java {
        palantirJavaFormat("2.50.0")
        removeUnusedImports()
        importOrder("java", "javax", "org", "com", "io", "net")
        endWithNewline()
        trimTrailingWhitespace()
    }
}

tasks.withType<JavaCompile>().configureEach {
    options.errorprone {
        nullaway {
            severity.set(net.ltgt.gradle.errorprone.CheckSeverity.ERROR)
            annotatedPackages.add("com.example.myplugin")
            unannotatedSubPackages.addAll("org.bukkit", "io.papermc")
        }
    }
}

tasks.processResources {
    filteringCharset = "UTF-8"
    filesMatching("plugin.yml") {
        expand("version" to project.version)
    }
}

tasks.shadowJar {
    archiveClassifier.set("")
    relocate("com.zaxxer.hikari", "com.example.myplugin.libs.hikari")
    mergeServiceFiles()
    minimize()
}

tasks.runServer {
    minecraftVersion("1.21.4")
    jvmArgs("-Xmx4G", "-XX:+UseG1GC")
}

tasks.jar { enabled = false }
tasks.build { dependsOn("shadowJar") }
tasks.named("check") { dependsOn("spotlessCheck", "checkstyleMain") }

tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false
    isReproducibleFileOrder  = true
}
```

---

## 1.X SELF-REVIEW CHECKLIST

Before considering this section complete, verify:

- [x] Build-system choice matrix (Gradle / Maven / Bazel / sbt) included with rationale
- [x] **Maven full equivalent** with `pom.xml` and Gradle-to-Maven plugin map
- [x] All plugin coordinates verified against [plugins.gradle.org](https://plugins.gradle.org)
- [x] Mojang-mapped runtime change at 1.20.5 documented
- [x] Hardfork change at 1.21.4 documented
- [x] paperweight reobf vs Mojmap distinction explained
- [x] Library loader vs Shadow trade-off explained
- [x] All 4 static-analysis tools configured with working examples
- [x] NullAway example handles Bukkit's inconsistent annotations
- [x] Kotlin section addresses companion-object loader pitfall
- [x] Folia variant of MCCoroutine called out
- [x] Multi-module API/impl example references real-world precedent
- [x] Composite builds explained with use case
- [x] Version catalog includes bundles + plugins
- [x] Renovate AND Dependabot configs both shown
- [x] Reproducible builds + dependency locking covered
- [x] Remote build cache covered (Develocity)
- [x] **Build scans (`--scan`) covered**
- [x] **Gradle wrapper pinning with SHA verification**
- [x] **Maven Central / GitHub Packages publishing covered**
- [x] **PGP signing config for releases**
- [x] `--add-opens` matrix included (when needed, when not)
- [x] IntelliJ + VSCode setup both included
- [x] `.editorconfig` + `gradle.properties` shown
- [x] Plugin-template references included
- [x] 16-row failure cookbook
- [x] What-changed-in-1.21.x build-wise summary
- [x] Complete reference build at the end

---

## 1.Z REFERENCES

- [PaperMC Docs — Project setup](https://docs.papermc.io/paper/dev/project-setup)
- [PaperMC Docs — paperweight-userdev](https://docs.papermc.io/paper/dev/userdev/)
- [PaperMC Docs — Minecraft internals](https://docs.papermc.io/paper/dev/internals)
- [PaperMC Docs — Debugging your plugin](https://docs.papermc.io/paper/dev/debugging)
- [PaperMC/paperweight on GitHub](https://github.com/PaperMC/paperweight)
- [Paper 1.21.4 Javadoc](https://jd.papermc.io/paper/1.21.4/)
- [The future of Paper — hardfork announcement](https://papermc.io/news/the-future-of-paper-hard-fork)
- [Shadow Gradle Plugin docs](https://gradleup.com/shadow/)
- [run-task plugins by jpenilla](https://github.com/jpenilla/run-task)
- [Error Prone Gradle plugin (tbroyer)](https://github.com/tbroyer/gradle-errorprone-plugin)
- [NullAway](https://github.com/uber/NullAway) and [its Gradle plugin](https://github.com/tbroyer/gradle-nullaway-plugin)
- [Spotless](https://github.com/diffplug/spotless)
- [Gradle composite builds](https://docs.gradle.org/current/userguide/composite_builds.html)
- [Gradle version catalogs](https://docs.gradle.org/current/userguide/version_catalogs.html)
- [Renovate Gradle support](https://docs.renovatebot.com/modules/manager/gradle/)
- [Develocity Build Cache docs](https://docs.gradle.com/develocity/2026.1/administration/build-acceleration/build-cache/)
- [JetBrains — Run/Debug configuration: Plugin](https://www.jetbrains.com/help/idea/run-debug-configuration-plugin.html)
- [Minecraft Development IntelliJ plugin](https://github.com/minecraft-dev/MinecraftDev)
- [CrimsonWarpedcraft/plugin-template](https://github.com/CrimsonWarpedcraft/plugin-template)
- [MCCoroutine](https://github.com/Shynixn/MCCoroutine)

---

## See also

- `02-velocity-folia-bedrock.md` — Folia-specific build flags (`folia-supported: true`),
  Velocity build setup.
- `paper-plugin-dev.md` §1 — original short build setup.
- `paper-plugin-dev.md` §36 — version / protocol matrix.
