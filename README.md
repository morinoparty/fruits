# fruits

Java container images for Minecraft servers on [Pelican](https://pelican.dev/) (and Pterodactyl), plus Paper and Velocity eggs tuned for them. Each image runs the standard Pelican `entrypoint.sh`, expanding the panel's `STARTUP` variable, so any egg expecting a Java image can use them.

## Images

Published to GitHub Container Registry, `linux/amd64` only. Rebuilt monthly and on every push to `java/`.

| Image | JDK | Base |
|-------|-----|------|
| `ghcr.io/morinoparty/fruits:java_25-temurin` | Eclipse Temurin 25.0.5.1+2 | `ubuntu:26.04` + Adoptium tarball |
| `ghcr.io/morinoparty/fruits:java_25-azul_zulu` | Azul Zulu 25.38.201 (JDK 25.0.5.1) | `ubuntu:26.04` + Azul tarball |
| `ghcr.io/morinoparty/fruits:java_25-graalvm_jdk` | GraalVM Community 25.0.5.0.3 | `ubuntu:26.04` + upstream tarball |
| `ghcr.io/morinoparty/fruits:java_27-temurin` | Eclipse Temurin 27.0.1.1+4 | `ubuntu:26.04` + Adoptium tarball |
| `ghcr.io/morinoparty/fruits:java_27-azul_zulu` | Azul Zulu 27.30.201 (JDK 27.0.1.1) | `ubuntu:26.04` + Azul tarball |

Every image installs its JDK from the vendor tarball, pinned by URL and SHA-256 in its `Dockerfile`, so a JDK update means editing that file. The tarballs are used because, when [CVE-2026-114938](https://app.opencve.io/cve/CVE-2026-114938) (Java4Shell) was fixed in 25.0.5.1 and 27.0.1.1, the `eclipse-temurin` and `azul/zulu-openjdk` Docker images had not caught up yet. Monthly rebuilds still pick up Ubuntu package updates.

The Temurin and Zulu images include the CVE-2026-114938 fix. The GraalVM image does **not**: no GraalVM Community build with the fix had been released, so it stays on the newest available build until one is.

Java 27 is a non-LTS release supported until Java 28 ships in March 2027. Java 25 is the LTS line.

## Paper egg

`eggs/paper/egg-fruits-paper.yaml` is a Pelican egg for [Paper](https://papermc.io/), derived from the [pelican-eggs](https://github.com/pelican-eggs/minecraft) egg. It offers the Temurin and Azul Zulu images for Java 25 and Java 27; GraalVM can still be used by editing a server's image.

### Import

In the panel: **Admin → Eggs → Import Egg → From URL**:

```
https://raw.githubusercontent.com/morinoparty/fruits/refs/heads/main/eggs/paper/egg-fruits-paper.yaml
```

That URL is also the egg's `update_url`, so **Update Egg** in the panel pulls changes from this repository.

### Startup commands

| Startup command | Notes |
|-----------------|-------|
| `Default` | [Aikar's flags](https://docs.papermc.io/paper/aikars-flags/) (G1GC). |
| `Generational ZGC` | Sub-millisecond pauses at the cost of throughput; worth trying on large heaps (16GB+). Also enables compact object headers ([JEP 519](https://openjdk.org/jeps/519)). |
| `Java 27 - G1GC` | Aikar's flags for Java 27 images: `-XX:InitiatingHeapOccupancyPercent` is deprecated in 27 in favour of `-XX:G1IHOP`, which Java 25 does not recognise. |

`Default` and `Generational ZGC` start on both Java 25 and Java 27; `Java 27 - G1GC` starts on Java 27 only. On a Java 27 image, `Default` prints a one-line deprecation warning for `InitiatingHeapOccupancyPercent`, which `Java 27 - G1GC` avoids. `-XX:+ParallelRefProcEnabled` was dropped from Aikar's flags: it is already the default on Java 25 and is removed (ignored with a warning) on Java 27.

Compact object headers ([JEP 519](https://openjdk.org/jeps/519)) shrink object headers from 12 to 8 bytes. They are the default on Java 27, so there is no separate G1GC command with `-XX:+UseCompactObjectHeaders`: choose a Java 27 image to get them.

The `sun.misc.Unsafe` warning on Java 25 is harmless and comes from Paper and its dependencies.

### Variables

- `SERVER_JARFILE`: Defaults to `paper-*.jar`. The installer renames old jars to `.jar.old` so the wildcard matches one file. A literal name such as `server.jar` also works.
- `JVM_HEAP_MB`: Default `2048`. This is the heap alone (`-Xms` and `-Xmx`), so it must be lower than the server's allocation to leave room for non-heap usage. Setting too large a heap makes the container exit 137 with no Java-side error. The default fits a server with roughly 3GB or more allocated; below that, follow the table.

| Allocation | Suggested `JVM_HEAP_MB` |
|------------|-------------------------|
| 2GB | 1024 |
| 4GB | 3072 |
| 8GB | 7168 |
| 16GB | 15360 |

## Velocity egg

`eggs/velocity/egg-fruits-velocity.yaml` is a Pelican egg for the [Velocity](https://papermc.io/software/velocity) proxy, modelled on the [pelican-eggs](https://github.com/pelican-eggs/minecraft) Velocity egg. It offers the Temurin and Azul Zulu images for Java 25 and Java 27; GraalVM can still be used by editing a server's image.

### Import

In the panel: **Admin → Eggs → Import Egg → From URL**:

```
https://raw.githubusercontent.com/morinoparty/fruits/refs/heads/main/eggs/velocity/egg-fruits-velocity.yaml
```

That URL is also the egg's `update_url`, so **Update Egg** in the panel pulls changes from this repository.

### Startup commands

| Startup command | Notes |
|-----------------|-------|
| `Default` | Flags from Velocity's [tuning guide](https://docs.papermc.io/velocity/tuning/) (G1GC). |

`Default` runs on the Java 25 and Java 27 images without warnings. Compact object headers are the default on Java 27, so choose a Java 27 image to get them. `-XX:+ParallelRefProcEnabled` was dropped because it is the default on Java 25 and removed on Java 27.

### Variables

- `SERVER_JARFILE`: Defaults to `velocity-*.jar`. The installer renames old jars to `.jar.old` so the wildcard matches one file; a literal name such as `server.jar` also works.
- `JVM_HEAP_MB`: Default `512`. It sets the heap alone, so it must be lower than the server's allocation; 512 fits a 1GB allocation, and 1024 on a 2GB allocation is plenty for most networks.
- `VELOCITY_VERSION`: Selects which version to install.

| Value | Installs |
|-------|----------|
| `latest` | Newest release |
| `snapshot` | Newest version of any kind, `-SNAPSHOT` included |
| Exact version, e.g. `4.2.1-SNAPSHOT` | That version; unlisted versions fall back to `latest` |

### Configuration

When no `velocity.toml` exists, the installer extracts the default config from the downloaded jar so the panel can set the port, and an existing config is never overwritten.

## Runtime comparison

| Runtime | Java 25 |
|---------|---------|
| Eclipse Temurin | 15.1s |
| Azul Zulu | 14.8s |
| GraalVM Community | 16.4s |

Measured on Paper 1.21.11 with the same seed and flags, generating a fresh world at `view-distance=16` in a 3GB container (median of 3 runs of Paper's `Done (...)` timing on a 16-core host). Temurin and Zulu are indistinguishable, and this workload says nothing about steady-state tick throughput or GC pauses.

## Verification

Each startup command was checked flag by flag, then as a combined string, then with a real boot through `entrypoint.sh` with `STARTUP` set as the panel would. Boots reached `Done (...)` on Paper 26.2 and Velocity 4.2.0 on the Java 25 images. After the move to tarball-installed JDKs, every startup command was booted again on the Temurin and Zulu images for Java 25 and Java 27 (`Java 27 - G1GC` on Java 27 only), reaching `Done (...)` on Paper 26.3 and Velocity 4.3.1, and each image's `java -version` was checked against the CVE-2026-114938 fixed versions (25.0.5.1 / 27.0.1.1). The install scripts were run against the real API, including a reinstall.

## License

Written in 2025-2026 by Morinoparty developer team. No Rights Reserved.

To the extent possible under law, morinoparty has waived all copyright and related or neighboring rights to fruits. This work is published from: Japan.

You should have received a copy of the CC0 Public Domain Dedication along with this software. If not, see http://creativecommons.org/publicdomain/zero/1.0/.

This CC0 dedication applies to the source code only. Non-code assets (images, icons, logos, and other media) are **not** covered and remain under their respective rights unless stated otherwise.
