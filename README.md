# fruits

Java container images for Minecraft servers on [Pelican](https://pelican.dev/) (and Pterodactyl), plus a Paper egg tuned for them.

Four JVM distributions are built for each of the two current LTS releases, so you can pick the runtime that fits your server. Each image runs the standard Pelican `entrypoint.sh`, which expands the panel's `STARTUP` variable, so the images can be used from any egg that expects a Java image.

## Images

Published to GitHub Container Registry, `linux/amd64` only. Rebuilt monthly and on every push to `java/`.

| Image | JDK | Base |
|-------|-----|------|
| `ghcr.io/morinoparty/fruits:java_25-temurin` | Eclipse Temurin 25 | `eclipse-temurin:25-jdk` |
| `ghcr.io/morinoparty/fruits:java_25-azul_zulu` | Azul Zulu 25 | `azul/zulu-openjdk:25` |
| `ghcr.io/morinoparty/fruits:java_25-graalvm_jdk` | GraalVM Community 25.0.2 | `ubuntu:22.04` + upstream tarball |
| `ghcr.io/morinoparty/fruits:java_25-open_j9` | IBM Semeru / Eclipse OpenJ9 25 | `ibm-semeru-runtimes:open-25-jdk` |
| `ghcr.io/morinoparty/fruits:java_21-temurin` | Eclipse Temurin 21 | `eclipse-temurin:21-jdk` |
| `ghcr.io/morinoparty/fruits:java_21-azul_zulu` | Azul Zulu 21 | `azul/zulu-openjdk:21` |
| `ghcr.io/morinoparty/fruits:java_21-graalvm_jdk` | GraalVM Community 21.0.2 | `ubuntu:22.04` + upstream tarball |
| `ghcr.io/morinoparty/fruits:java_21-open_j9` | IBM Semeru / Eclipse OpenJ9 21 | `ibm-semeru-runtimes:open-21-jdk` |

The Temurin, Zulu and Semeru images track their upstream tag and pick up patch releases on rebuild. The GraalVM images pin an exact tarball version, so bumping them means editing the `Dockerfile`.

## Paper egg

`eggs/paper/egg-fruits-paper.yaml` is a Pelican egg (`PLCN_v3`) for [Paper](https://papermc.io/), derived from the [pelican-eggs](https://github.com/pelican-eggs/minecraft) egg. The install script is unchanged — it resolves versions and builds through the PaperMC Fill v3 API — but the Docker images and the startup commands are specific to this repository.

The egg offers the Temurin and Azul Zulu images only — they measured fastest, see [Runtime comparison](#runtime-comparison). GraalVM and OpenJ9 are still built here and can be selected by editing a server's image, but they are not offered as choices.

### Install

In the panel: **Admin → Eggs → Import Egg → From URL**, and paste:

```
https://raw.githubusercontent.com/morinoparty/fruits/refs/heads/main/eggs/paper/egg-fruits-paper.yaml
```

Or download it and import the file:

```bash
curl -O https://raw.githubusercontent.com/morinoparty/fruits/refs/heads/main/eggs/paper/egg-fruits-paper.yaml
```

That URL is also the egg's `update_url`, so **Update Egg** in the panel pulls changes from this repository.

### Startup commands

The egg ships four startup commands. `Default` is the only one that is safe on every image; the others are tied to one Java release, and picking the wrong pair stops the JVM before Paper is reached.

| Startup command | Use with | Notes |
|-----------------|----------|-------|
| `Default` | Any image, Java 21 and 25 | [Aikar's flags](https://docs.papermc.io/paper/aikars-flags/) — G1GC, the only collector PaperMC recommends. Start here. |
| `Java 25 - G1GC (compact headers)` | Java 25 images | Adds `-XX:+UseCompactObjectHeaders` ([JEP 519](https://openjdk.org/jeps/519)), which shrinks object headers from 12 to 8 bytes. Minecraft's heap is dominated by small objects, so this is the cheapest heap win available. Java 21 refuses to start with this flag. |
| `Java 25 - Generational ZGC` | Java 25 images | Sub-millisecond pauses at the cost of throughput and footprint. Worth trying on large heaps (16GB+) where G1's mixed collections still show up as tick lag. |
| `Java 21 - Generational ZGC` | Java 21 images | Same, but Java 21 needs `-XX:+ZGenerational` to opt into generational mode. That flag was removed in Java 24, which is why this is a separate entry. |

### Server jar file

`SERVER_JARFILE` defaults to `paper-*.jar`. The installer saves the jar under Paper's own filename — `paper-1.21.11-132.jar`, the same thing you get downloading it by hand — and the `*` is expanded at startup by `entrypoint.sh`, which passes `STARTUP` through `eval echo` before `exec`.

The wildcard has one sharp edge, and the installer is what blunts it: if it matches more than one jar, the JVM does not complain. It runs the first match and treats the rest as program arguments, so `java -jar paper-1.21.10-130.jar paper-1.21.11-132.jar nogui` **silently starts the older build**. Since the install script also runs on every reinstall — and `MINECRAFT_VERSION=latest` means a reinstall usually brings a new filename — it renames any existing `paper-*.jar` to `.jar.old` before downloading, so exactly one file ever matches.

Set a literal name such as `server.jar` if you would rather have a fixed filename; the installer detects that there is no wildcard and saves under exactly that name, which is the pre-existing behaviour.

### JVM heap size

The heap is set explicitly, to the same value on both sides as Aikar's flags require (which is also what makes `-XX:+AlwaysPreTouch` meaningful):

```
-Xms{{JVM_HEAP_MB}}M -Xmx{{JVM_HEAP_MB}}M
```

`JVM_HEAP_MB` is a panel variable, **default 2048**. This is the heap *alone* — Metaspace, GC bookkeeping, thread stacks, direct/native buffers and any off-heap allocation your plugins do all sit on top of it, so the value must be meaningfully **lower** than the memory you allocated to the server.

How much lower depends on what Wings hands the container. It does not use the number shown in the panel as the cgroup limit; it multiplies the allocation to leave room for exactly that non-heap usage (`Docker.Overhead` defaults — a host admin can override them in `config.yml`):

| Allocation | Multiplier | Container limit |
|-----------|-----------|-----------------|
| ≤ 2048MB | 1.15 | allocation + 15% |
| ≤ 4096MB | 1.10 | allocation + 10% |
| > 4096MB | 1.05 | allocation + 5% |

That overhead alone is thinner than it looks — 15% of 2GB is 307MB, and Metaspace by itself can be 100–200MB. PaperMC's guidance is to leave 1000–1500MB outside the heap, so subtract that from the allocation rather than relying on the multiplier:

| Allocation | Container limit | Suggested `JVM_HEAP_MB` | Left for non-heap |
|-----------|-----------------|------------------------|-------------------|
| 2GB | 2355MB | 1024 | 1331MB |
| 4GB | 4506MB | 3072 | 1434MB |
| 8GB | 8602MB | 7168 | 1434MB |
| 16GB | 17203MB | 15360 | 1843MB |

The default of 2048 fits a server with roughly 3GB or more allocated; below that, follow the table.

Setting the heap larger than the container can hold does not degrade gracefully. `-Xms == -Xmx` with `-XX:+AlwaysPreTouch` means the JVM commits and touches the entire heap during startup, so the kernel kills it before Paper prints anything: running the `Default` command with `JVM_HEAP_MB=2048` in a 1178MB container (a 1GB allocation) exits **137 with no Java-side error at all**. The same command in the 2355MB container starts normally.

#### Why not `-XX:MaxRAMPercentage`

It is a percentage of the container limit, which makes it wrong in both directions here. On a 2GB server, `MaxRAMPercentage=85` measured out to a 2002MB heap inside the 2355MB container — it spends the headroom Wings just reserved. And a server allocated **0 (unlimited)** memory has no container limit at all, so the JVM reads the percentage against the host's RAM instead; on the 28944MB test host that produced a **24608MB** heap. The same startup command with `-Xmx2048M` stayed at 2048MB in both cases.

## Notes

- **Minecraft 26.1 and newer require Java 25.** On a Java 21 image, Paper refuses to start with `Minecraft 26.1 and newer requires running the server with Java 25 or above`. The Java 21 images remain useful for 1.21.x and older.
- **`WARNING: A terminally deprecated method in sun.misc.Unsafe has been called`** appears on every Java 25 image. It comes from Paper and its dependencies, not from the egg, and it is harmless. It could be hidden with `--sun-misc-unsafe-memory-access=allow`, but the egg deliberately does not — the warning is the upstream signal that something needs porting before a future JDK removes the method.
- **The `-XX:+UnlockExperimentalVMOptions` in Aikar's flags is required, not decoration.** `G1NewSizePercent` and `G1MaxNewSizePercent` are still experimental options on both Java 21 and 25; without the unlock flag the JVM exits before Paper is reached.
- **OpenJ9 is not offered in the egg.** Paperclip's bzip2 patch step fails intermittently on OpenJ9 with `java.io.IOException: block overrun` — 3 of 6 clean first starts failed here on Java 25. This is an open upstream bug ([PaperMC/Paperclip#68](https://github.com/PaperMC/Paperclip/issues/68)), not something the egg can work around. OpenJ9 also has no G1GC and *silently ignores* unknown `-XX:` options, so Aikar's flags would do nothing on it; it needs `-Xgcpolicy:gencon -Xtune:virtualized` instead.

## Verification

Every startup command in the egg was checked against the images it is documented for: each flag individually (rejecting anything that printed `Unrecognized`, `Ignoring option` or a deprecation notice), then the combined string, then a real Paper boot to `Done (...)! For help, type "help"` — Paper 1.21.11 on the Java 21 images, Paper 26.2 on the Java 25 images — and finally through `entrypoint.sh` with `STARTUP` set exactly as the panel would, to confirm `{{SERVER_JARFILE}}` and `{{JVM_HEAP_MB}}` expand. The install script was run against the real API for both the wildcard and the literal-name case, twice over, to confirm the `.old` rename leaves a single match — then booted from that directory to `Done (11.556s)!` on `paper-*.jar`. The heap numbers above come from `-XX:+PrintFlagsFinal` through that same path, run both in a memory-limited container and in an unlimited one.

### Runtime comparison

Which JVM to offer was measured rather than guessed. Same Paper build (1.21.11), same seed, same flags, generating a fresh world at `view-distance=16` in a 3GB container; median of 3 runs of Paper's own `Done (...)` timing, on a 16-core host.

| Runtime | Java 21 | Java 25 |
|---------|---------|---------|
| Eclipse Temurin | 15.5s | 15.1s |
| Azul Zulu | 15.2s | 14.8s |
| GraalVM Community | 16.5s | 16.4s |
| OpenJ9 (gencon) | 18.7s | 18.0s |

Temurin and Zulu are the fastest and are indistinguishable from each other — their individual runs overlap, so the 0.3s between the medians is noise, not a ranking. GraalVM is 7–9% behind and OpenJ9 18–20%, both well outside that noise. That is why the egg offers Temurin and Zulu only.

Read this for what it is: a short, chunk-generation-dominated workload measured before any JIT has fully warmed up. It says nothing about steady-state tick throughput or GC pause behaviour over a long session — if you have a reason to prefer GraalVM, the image is still built and you can point a server at it.

## License

Written in 2025-2026 by Morinoparty developer team. No Rights Reserved.

To the extent possible under law, morinoparty has waived all copyright and related or neighboring rights to fruits. This work is published from: Japan.

You should have received a copy of the CC0 Public Domain Dedication along with this software. If not, see http://creativecommons.org/publicdomain/zero/1.0/.

This CC0 dedication applies to the source code only. Non-code assets (images, icons, logos, and other media) are **not** covered and remain under their respective rights unless stated otherwise.
