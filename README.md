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

### JVM heap percentage

Rather than a hardcoded `-Xms`/`-Xmx`, the startup commands size the heap from the container limit:

```
-XX:InitialRAMPercentage={{JVM_RAM_PERCENT}} -XX:MaxRAMPercentage={{JVM_RAM_PERCENT}}
```

Setting both to the same value keeps Aikar's `Xms == Xmx` behaviour (and makes `-XX:+AlwaysPreTouch` meaningful) while tracking whatever memory the panel allocates, so resizing a server needs no startup edit.

`JVM_RAM_PERCENT` defaults to **85**. The remaining 15% is not spare — it is Metaspace, GC bookkeeping, thread stacks, direct/native buffers and any off-heap allocation your plugins do. Exceeding the container limit gets the process OOM-killed by the kernel, which is far less diagnosable than an `OutOfMemoryError`. PaperMC's own guidance is to leave 1000–1500MB of headroom, which works out to roughly 81% at 8GB. Lower the value if you run many plugins or a large view distance.

## Notes

- **Minecraft 26.1 and newer require Java 25.** On a Java 21 image, Paper refuses to start with `Minecraft 26.1 and newer requires running the server with Java 25 or above`. The Java 21 images remain useful for 1.21.x and older.
- **`WARNING: A terminally deprecated method in sun.misc.Unsafe has been called`** appears on every Java 25 image. It comes from Paper and its dependencies, not from the egg, and it is harmless. It could be hidden with `--sun-misc-unsafe-memory-access=allow`, but the egg deliberately does not — the warning is the upstream signal that something needs porting before a future JDK removes the method.
- **The `-XX:+UnlockExperimentalVMOptions` in Aikar's flags is required, not decoration.** `G1NewSizePercent` and `G1MaxNewSizePercent` are still experimental options on both Java 21 and 25; without the unlock flag the JVM exits before Paper is reached.
- **OpenJ9 is not offered in the egg.** Paperclip's bzip2 patch step fails intermittently on OpenJ9 with `java.io.IOException: block overrun` — 3 of 6 clean first starts failed here on Java 25. This is an open upstream bug ([PaperMC/Paperclip#68](https://github.com/PaperMC/Paperclip/issues/68)), not something the egg can work around. OpenJ9 also has no G1GC and *silently ignores* unknown `-XX:` options, so Aikar's flags would do nothing on it; it needs `-Xgcpolicy:gencon -Xtune:virtualized` instead.

## Verification

Every startup command in the egg was checked against the images it is documented for: each flag individually (rejecting anything that printed `Unrecognized`, `Ignoring option` or a deprecation notice), then the combined string, then a real Paper boot to `Done (...)! For help, type "help"` — Paper 1.21.11 on the Java 21 images, Paper 26.2 on the Java 25 images — and finally through `entrypoint.sh` with `STARTUP` set exactly as the panel would, to confirm `{{SERVER_JARFILE}}` and `{{JVM_RAM_PERCENT}}` expand.

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
