## このリポジトリの概要
「fruits」という、Minecraft サーバー向けの Java コンテナイメージと、Pelican (および Pterodactyl) 用の egg をまとめたリポジトリです。
リポジトリ名は `fruits` です。アプリケーションのソースコードやパッケージマネージャー (package.json / Gradle など) はありません。

## 目的
morinoparty の Minecraft サーバーを [Pelican](https://pelican.dev/) パネルで動かすための実行環境を提供します。

- **Java コンテナイメージ** - 2 つの LTS (Java 21 / Java 25) それぞれについて、4 種類の JVM ディストリビューションのイメージを作ります。
  どのイメージも Pelican 標準の `entrypoint.sh` を実行し、パネルの `STARTUP` 変数を展開するので、Java イメージを前提とする任意の egg から利用できます。
- **Paper egg** (`eggs/paper/egg-fruits-paper.yaml`) - Paper 用の Pelican egg (`PLCN_v3`)。pelican-eggs の egg をベースに、イメージと起動コマンドをこのリポジトリ向けにしています。
- **Velocity egg** (`eggs/velocity/egg-fruits-velocity.yaml`) - Velocity プロキシ用の Pelican egg (`PLCN_v3`)。インストールスクリプトは Paper egg のものを元に書き直しています。

### イメージ一覧
GitHub Container Registry (GHCR) に `linux/amd64` のみで公開します。

| タグ | JDK | ベース |
|------|-----|--------|
| `ghcr.io/morinoparty/fruits:java_25-temurin` | Eclipse Temurin 25 | `eclipse-temurin:25-jdk` |
| `ghcr.io/morinoparty/fruits:java_25-azul_zulu` | Azul Zulu 25 | `azul/zulu-openjdk:25` |
| `ghcr.io/morinoparty/fruits:java_25-graalvm_jdk` | GraalVM Community 25.0.2 | `ubuntu:22.04` + 公式 tarball |
| `ghcr.io/morinoparty/fruits:java_25-open_j9` | IBM Semeru / Eclipse OpenJ9 25 | `ibm-semeru-runtimes:open-25-jdk` |
| `ghcr.io/morinoparty/fruits:java_21-temurin` | Eclipse Temurin 21 | `eclipse-temurin:21-jdk` |
| `ghcr.io/morinoparty/fruits:java_21-azul_zulu` | Azul Zulu 21 | `azul/zulu-openjdk:21` |
| `ghcr.io/morinoparty/fruits:java_21-graalvm_jdk` | GraalVM Community 21.0.2 | `ubuntu:22.04` + 公式 tarball |
| `ghcr.io/morinoparty/fruits:java_21-open_j9` | IBM Semeru / Eclipse OpenJ9 21 | `ibm-semeru-runtimes:open-21-jdk` |

- Temurin / Zulu / Semeru は上流のタグに追従するので、再ビルドでパッチリリースが入ります。
- GraalVM は tarball のバージョンを固定しているため、上げるときは `Dockerfile` を編集します。

### egg の前提知識
- egg が提供するイメージは Temurin と Azul Zulu のみです (README の Runtime comparison で実測して最速だったため)。GraalVM と OpenJ9 はビルドはしますが egg の選択肢には入れません。
- OpenJ9 を egg に入れない理由: Paperclip の bzip2 パッチ処理が OpenJ9 で断続的に `java.io.IOException: block overrun` で失敗する上流バグ (PaperMC/Paperclip#68) があり、また G1GC が無く未知の `-XX:` オプションを黙って無視するため Aikar's flags が効きません。
- Minecraft 26.1 以降と Velocity 4.x は Java 25 が必須です。Java 21 イメージは 1.21.x 以前 / Velocity 3.x 向けです。
- `-XX:+UseCompactObjectHeaders` は Java 25 専用で、Java 21 では起動しません。Java 21 の Generational ZGC には `-XX:+ZGenerational` が必要ですが、このフラグは Java 24 で削除されたため別の起動コマンドにしています。
- Aikar's flags の `-XX:+UnlockExperimentalVMOptions` は必須です (`G1NewSizePercent` などが Java 21 / 25 ともに experimental のため)。
- ヒープは `-Xms{{JVM_HEAP_MB}}M -Xmx{{JVM_HEAP_MB}}M` と固定 MB で指定します (Paper のデフォルト 2048、Velocity のデフォルト 512)。`-XX:MaxRAMPercentage` は、Wings がコンテナに上乗せした余裕を食いつぶし、メモリ無制限のサーバーではホストの RAM 基準になるため使いません。
- `SERVER_JARFILE` のデフォルトはワイルドカード (`paper-*.jar` / `velocity-*.jar`) です。複数の jar にマッチすると JVM は最初の 1 つを黙って起動するため、インストールスクリプトは既存の jar を `.jar.old` にリネームしてから新しい jar を保存します。
- Velocity egg の `VELOCITY_VERSION=latest` は、`-SNAPSHOT` を含まない最新リリースを指します (Fill API の先頭は SNAPSHOT のことがあり、channel でも区別できないため)。
- Velocity egg のインストーラーは、`velocity.toml` が無いときだけ jar 内の `default-velocity.toml` を展開します (パネルが `bind = ` 行を書き換えてポートを設定するため)。既存の設定は上書きしません。
- `WARNING: A terminally deprecated method in sun.misc.Unsafe has been called` は Java 25 で常に出ますが、上流への移行シグナルとして意図的に隠していません。

詳細な理由と実測値は `README.md` に書かれています。egg や起動コマンドを変更する前に必ず読んでください。

## 主な技術スタック
- Docker (Dockerfile によるイメージ定義)
- GitHub Actions (`docker/build-push-action` によるビルドと GHCR への push)
- GitHub Container Registry (イメージの公開先)
- Pelican / Pterodactyl (ゲームサーバーパネル。egg は `PLCN_v3` 形式の YAML)
- PaperMC Fill v3 API (egg のインストールスクリプトが Paper / Velocity のバージョンとビルドを解決する)
- Bash / ash (`java/entrypoint.sh` とインストールスクリプト。インストールコンテナは `ghcr.io/pelican-eggs/installers:alpine`)

## 構成
- `java/` - イメージのビルドコンテキスト。`<Javaのメジャーバージョン>-<ディストリビューション>/Dockerfile` と共通の `entrypoint.sh`
- `eggs/` - Pelican egg (`paper/`, `velocity/`)
- `.github/workflows/build.yaml` - イメージのビルドと push
- `README.md` - イメージ・egg の説明、検証方法、ランタイム比較
