---
name: paper-plugin-dev-build-tooling
description: >
  Expansion to paper-plugin-dev.md covering: Gradle Groovy DSL, paperweight-patcher (forks),
  static analysis (Spotless, Checkstyle, ErrorProne, NullAway), Lombok, Kotlin/Scala plugin support,
  jar-in-jar, multi-module API split, reproducible builds, and additional toolchain hardening.
---

# 1. BUILD & TOOLING — DEEP DIVE

This section expands `paper-plugin-dev.md` §1. The original covers Gradle KTS, Maven, Shadow,
run-paper, paperweight-userdev, and Java 21 toolchain. Here we cover everything else.

---

## 1.A GRADLE GROOVY DSL (Legacy but still common)

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

group = 'com.example'
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
    minecraftVersion('1.21.4')
}
```

**KTS vs Groovy quick rule:**
- New projects → KTS (better IDE completion, type-safe).
- Existing Groovy projects → don't migrate just because; both are first-class.

---

## 1.B PAPERWEIGHT-PATCHER (Building a Paper Fork)

Use this when you maintain a Paper *fork* (Purpur, Pufferfish, your own private fork) — NOT for
plugin development. For plugins use `paperweight.userdev` instead.

```kotlin
// settings.gradle.kts (root project)
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}
rootProject.name = "myfork"

// build.gradle.kts (root project)
plugins {
    id("io.papermc.paperweight.patcher") version "2.0.0-beta.19"
}

paperweight {
    upstreams.register("paper") {
        ref = providers.gradleProperty("paperRef") // commit SHA from paperRef in gradle.properties
        patchTasks.register("server") {
            upstreamPath = "paper-server"
            patchesDir = file("patches/server")
            outputDir  = file("myfork-server")
        }
        patchTasks.register("api") {
            upstreamPath = "paper-api"
            patchesDir = file("patches/api")
            outputDir  = file("myfork-api")
        }
    }
}
```

**Layout of a Paper fork:**
```
myfork/
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties           # paperRef=<paper commit sha>
├── patches/
│   ├── api/                    # *.patch files applied to paper-api
│   └── server/                 # *.patch files applied to paper-server
├── myfork-api/                 # generated source tree (gitignored)
└── myfork-server/              # generated source tree (gitignored)
```

**Common tasks:**
```bash
./gradlew applyPatches          # apply patches → editable source tree
./gradlew rebuildPatches        # diff editable tree → write back to patches/
./gradlew createMojmapPaperclipJar  # build runnable server jar
```

**When to fork instead of plugin:** very deep changes (custom dimensions, vanilla physics,
networking layer rewrites, removing core mechanics). Almost everything else should be a plugin
or a Mixin/access-widener-style modification.

---

## 1.C STATIC ANALYSIS

### 1.C.1 Spotless (formatting + license headers)

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
}

// Run on every build:
tasks.named("check") { dependsOn("spotlessCheck") }
// Auto-fix locally: ./gradlew spotlessApply
```

### 1.C.2 Checkstyle

```kotlin
plugins { checkstyle }

checkstyle {
    toolVersion = "10.18.2"
    configFile = rootProject.file("config/checkstyle/checkstyle.xml")
    isIgnoreFailures = false
    maxWarnings = 0
}

tasks.withType<Checkstyle>().configureEach {
    reports {
        xml.required.set(false)
        html.required.set(true)
    }
}
```

`config/checkstyle/checkstyle.xml` minimum:
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
    </module>
    <module name="FileTabCharacter"><property name="eachLine" value="true"/></module>
</module>
```

### 1.C.3 ErrorProne + NullAway

```kotlin
plugins {
    id("net.ltgt.errorprone") version "4.1.0"
    id("net.ltgt.nullaway")   version "2.1.0"
}

dependencies {
    errorprone("com.google.errorprone:error_prone_core:2.36.0")
    errorprone("com.uber.nullaway:nullaway:0.12.1")

    // Annotations available on the runtime classpath (or compileOnly):
    compileOnly("org.jspecify:jspecify:1.0.0")           // recommended modern null-spec
    // OR: compileOnly("org.jetbrains:annotations:26.0.1")
}

tasks.withType<JavaCompile>().configureEach {
    options.errorprone {
        disableWarningsInGeneratedCode.set(true)
        // Tune Error Prone checks:
        error("UnusedVariable", "DefaultCharset", "MissingOverride")
        // NullAway config:
        nullaway {
            severity.set(net.ltgt.gradle.errorprone.CheckSeverity.ERROR)
            annotatedPackages.add("com.example.myplugin")
            // Treat Bukkit/Paper as un-annotated (they have @Nullable inconsistently)
            unannotatedSubPackages.add("org.bukkit")
            handleTestAssertionLibraries.set(true)
        }
    }
}
```

**Why NullAway for plugins:** Bukkit's API has many nullable returns (`Bukkit.getPlayer(name)`,
`world.getBlockAt(...).getRelative(...)`-chains). A `@Nullable`-aware compiler catches the entire
class of `NullPointerException` in events that are otherwise hit only at runtime under specific
edge cases.

### 1.C.4 PMD

```kotlin
plugins { pmd }
pmd {
    toolVersion = "7.7.0"
    ruleSets = listOf()
    ruleSetFiles = files("config/pmd/ruleset.xml")
}
```

### 1.C.5 SpotBugs (bug pattern detector)

```kotlin
plugins { id("com.github.spotbugs") version "6.0.26" }
spotbugs {
    toolVersion.set("4.8.6")
    effort.set(com.github.spotbugs.snom.Effort.MAX)
    reportLevel.set(com.github.spotbugs.snom.Confidence.LOW)
}
```

---

## 1.D ANNOTATION PROCESSORS / CODEGEN

### 1.D.1 Lombok (controversial — read warnings below)

```kotlin
dependencies {
    compileOnly("org.projectlombok:lombok:1.18.36")
    annotationProcessor("org.projectlombok:lombok:1.18.36")
    testCompileOnly("org.projectlombok:lombok:1.18.36")
    testAnnotationProcessor("org.projectlombok:lombok:1.18.36")
}
```

```java
@Getter @Setter @ToString @EqualsAndHashCode
@Builder @AllArgsConstructor @NoArgsConstructor
public class PlayerData {
    private UUID uuid;
    private int level;
    private long coins;
}

// Or terse:
@Value      // immutable + final fields + getters
public class Home {
    String name;
    Location location;
}
```

**Warnings:**
- Java 21 records cover most Lombok use cases (`@Value`, `@Data` for DTOs).
- `@Builder` with mutable state is an anti-pattern for plugin data — prefer records + factory methods.
- `@SneakyThrows` hides checked exceptions — avoid in event handlers; let Bukkit log them properly.
- Lombok requires every contributor's IDE to have the plugin installed.

**Recommendation:** prefer records / sealed types / pattern matching. Use Lombok only where
records are infeasible (e.g. JPA entities, mutable fluent builders for huge config objects).

### 1.D.2 Immutables / AutoValue

```kotlin
dependencies {
    compileOnly("org.immutables:value:2.10.1:annotations")
    annotationProcessor("org.immutables:value:2.10.1")
}
```

Generates immutable + builder + with-er classes from interfaces. Records have largely replaced
this pattern, but Immutables still wins for very large structs with lots of optional fields.

### 1.D.3 MapStruct (DTO mapping)

```kotlin
dependencies {
    implementation("org.mapstruct:mapstruct:1.6.3")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.6.3")
}
```

Useful when your plugin has a config-DTO ↔ runtime-model layer.

---

## 1.E KOTLIN PLUGIN SUPPORT

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    id("com.gradleup.shadow") version "8.3.5"
}

dependencies {
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    implementation(kotlin("stdlib"))             // Will be shaded into the plugin JAR
    // Coroutines for async work:
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1")
}

kotlin {
    jvmToolchain(21)
}

tasks.shadowJar {
    archiveClassifier.set("")
    // Relocate Kotlin stdlib if multiple Kotlin plugins coexist on the server:
    relocate("kotlin", "com.example.myplugin.libs.kotlin")
    relocate("kotlinx", "com.example.myplugin.libs.kotlinx")
    minimize {
        exclude(dependency("org.jetbrains.kotlin:.*"))
    }
}
```

**Kotlin-specific gotchas:**
- **Companion objects** in your main class can confuse Bukkit's reflection-based loader; keep
  the entry point a plain `class MyPlugin : JavaPlugin()`.
- **Top-level extensions** (`fun Player.heal()`) compile to static methods in `*Kt.class` files.
  Other Kotlin plugins on the server can collide if you don't relocate stdlib — always relocate.
- **`object` singletons** are fine but be careful: Paper plugins with classloader isolation will
  create a separate singleton per plugin instance.
- **Coroutine schedulers:** consider [MCCoroutine](https://github.com/Shynixn/MCCoroutine) which
  bridges Bukkit's main thread to a coroutine dispatcher.

```kotlin
// MCCoroutine example:
class MyPlugin : SuspendingJavaPlugin() {
    override suspend fun onEnableAsync() {
        // suspend function as plugin entry — runs on Bukkit main dispatcher
        val data = withContext(Dispatchers.IO) { loadFromDb() }
        Bukkit.broadcast(Component.text("Loaded ${data.size} entries"))
    }
}
```

---

## 1.F SCALA / GROOVY PLUGINS

Possible but rare. `scala` and `groovy` Gradle plugins work; relocate the runtime stdlib in
shadow exactly as with Kotlin. Performance and tooling are inferior to Java/Kotlin for plugin work.

---

## 1.G JAR-IN-JAR / MULTI-RELEASE JARs

### 1.G.1 paper-plugin libraries (preferred — no JIJ needed)

Use Paper's library loader (see §2 of the original) to declare runtime Maven dependencies that
the server will download and isolate. This avoids fat-JAR bloat and dependency conflicts.

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
            "central", "default", MavenLibraryResolver.MAVEN_CENTRAL_DEFAULT_MIRROR).build());
        resolver.addDependency(new Dependency(
            new DefaultArtifact("com.zaxxer:HikariCP:5.1.0"), null));
        resolver.addDependency(new Dependency(
            new DefaultArtifact("com.github.ben-manes.caffeine:caffeine:3.1.8"), null));
        classpathBuilder.addLibrary(resolver);
    }
}
```

### 1.G.2 Bukkit plugin.yml libraries (also no JIJ needed)

```yaml
libraries:
  - com.zaxxer:HikariCP:5.1.0
  - org.jetbrains.kotlin:kotlin-stdlib:2.1.0
```

This is even simpler — Paper resolves these from Maven Central at startup.

### 1.G.3 Multi-release JAR (rare — only if shipping cross-Java code)

```kotlin
tasks.jar {
    manifest {
        attributes["Multi-Release"] = "true"
    }
}

sourceSets {
    create("java21") {
        java {
            srcDir("src/main/java21")
        }
    }
}

tasks.named<JavaCompile>("compileJava21Java") {
    options.release.set(21)
}

tasks.jar {
    into("META-INF/versions/21") {
        from(sourceSets["java21"].output)
    }
}
```

For Paper plugins this is almost never needed — Paper requires Java 21+ on 1.21.x anyway.

---

## 1.H MULTI-MODULE: API + IMPL SPLIT

Big plugins often expose a thin API JAR for other plugins to depend on.

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
java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(21))
}
repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}
dependencies {
    "compileOnly"("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
}

// api/build.gradle.kts
plugins { id("myplugin.common") `java-library` `maven-publish` }

publishing {
    publications {
        create<MavenPublication>("api") { from(components["java"]) }
    }
}

// plugin/build.gradle.kts
plugins { id("myplugin.common") id("com.gradleup.shadow") version "8.3.5" }
dependencies {
    implementation(project(":api"))
    implementation("com.zaxxer:HikariCP:5.1.0")
}
tasks.shadowJar { archiveClassifier.set("") }
```

**Result:** other developers depend on `com.example:myplugin-api:1.0.0` (a tiny JAR with just
interfaces and records), while the runtime plugin shades implementation deps.

---

## 1.I REPRODUCIBLE BUILDS

```kotlin
tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false
    isReproducibleFileOrder = true
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

Combined with locked dependency versions (`./gradlew dependencies --write-locks`) you get
byte-identical JARs across machines, which makes hash-based plugin signing and CI caching reliable.

---

## 1.J DEPENDENCY LOCKING

```kotlin
dependencyLocking {
    lockAllConfigurations()
}
```

Then:
```bash
./gradlew dependencies --write-locks
git add gradle.lockfile
```

Critical for plugins that ship via paper-plugin libraries — pinning prevents transitive resolution
drift between dev machines and the server resolver.

---

## 1.K VERSION CATALOGS

```toml
# gradle/libs.versions.toml
[versions]
paper      = "1.21.4-R0.1-SNAPSHOT"
shadow     = "8.3.5"
runPaper   = "2.3.1"
hikari     = "5.1.0"
caffeine   = "3.1.8"
adventure  = "4.17.0"

[libraries]
paper-api  = { group = "io.papermc.paper", name = "paper-api", version.ref = "paper" }
hikari     = { group = "com.zaxxer", name = "HikariCP", version.ref = "hikari" }
caffeine   = { group = "com.github.ben-manes.caffeine", name = "caffeine", version.ref = "caffeine" }

[plugins]
shadow     = { id = "com.gradleup.shadow", version.ref = "shadow" }
runPaper   = { id = "xyz.jpenilla.run-paper", version.ref = "runPaper" }
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
}
```

**Why bother:** single source of truth for versions across multi-module builds; IDE
autocompletes them; Renovate/Dependabot understands them.

---

## 1.L IDE TOOLCHAIN HARDENING

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

[*.{yml,yaml,json}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

`gradle.properties` (perf tuning):
```properties
org.gradle.jvmargs=-Xmx4G -XX:+UseG1GC -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
```

---

## 1.M COMMON BUILD PITFALLS

| Symptom | Cause | Fix |
|---|---|---|
| `Plugin already initialized` | Two shaded plugins relocate to the same package | Relocate libs into per-plugin namespace |
| Hot-reload breaks | run-paper retains old classloader | Restart the Gradle daemon, or use `./gradlew --stop` |
| `NoClassDefFoundError` only on server | Dep is `compileOnly` instead of `implementation` | Move to `implementation` or use library loader |
| Server crashes on startup with `UnsupportedClassVersionError` | Mixed Java versions in toolchain | Set `java.toolchain.languageVersion = 21` and rebuild |
| Resource pack `plugin.yml` shows `${version}` | `processResources` filtering not enabled | Add `expand("version" to project.version)` |
| Shadow JAR is huge (>20MB) | Bundling `paper-api` accidentally | Verify dep is `compileOnly` |
| `IllegalAccessError` for sun.* / jdk.internal.* | Accessing JDK internals on Java 21 | Use `--add-opens` in `gradle.properties`: `org.gradle.jvmargs=... --add-opens=java.base/java.lang=ALL-UNNAMED` |

---

## 1.N TASK GRAPH BEST PRACTICES

```kotlin
// Always run check before assembling the published artifact:
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
```

---

## 1.O REFERENCES

- Paper project setup: https://docs.papermc.io/paper/dev/project-setup
- paperweight plugins: https://github.com/PaperMC/paperweight
- Shadow plugin (gradleup): https://gradleup.com/shadow/
- ErrorProne Gradle plugin: https://github.com/tbroyer/gradle-errorprone-plugin
- NullAway: https://github.com/uber/NullAway
- Spotless: https://github.com/diffplug/spotless
