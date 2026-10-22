# コーディングプラクティス

## 実装手順

1. **影響範囲の確認**
   - 変更するイメージ・egg・起動コマンドが、どの Java バージョンとどのディストリビューションで使われるかを先に確認する

2. **小さく検証してから組み合わせる**
   - JVM フラグは 1 つずつ確認してから、起動コマンド全体を確認する

## プラクティス

- 小さく始めて段階的に拡張
- 過度な抽象化を避ける
- 推測ではなく実測を重視 (フラグの効果やランタイムの比較は、実際にコンテナで動かして確認する)
- 複雑さに応じてアプローチを調整

## コードスタイル

- 常に既存コードの設計や記法を参考にしてください。
- 書籍「リーダブルコード」のようなベストプラクティスを常に適用してください。
- Dockerfile や egg のインストールスクリプトには、意図・背景などのコメントを入れてください。既存のコメントに合わせて英語で書きます。

## Dockerfile

- 各イメージは `java/<Javaのメジャーバージョン>-<ディストリビューション>/Dockerfile` に置きます (例: `java/25-temurin/Dockerfile`)。
- ビルドコンテキストは `java/` です (`COPY ./../entrypoint.sh /entrypoint.sh` はこれを前提にしています)。
- 既存の Dockerfile と同じく、`container` ユーザーを作成して `USER container` / `WORKDIR /home/container` で動かし、`CMD [ "/bin/bash", "/entrypoint.sh" ]` で起動してください。
- 必要なパッケージ (`lsof curl ca-certificates openssl git tar sqlite3 fontconfig libfreetype6 tzdata iproute2 libstdc++6`) は全イメージで揃えてください。
- `LABEL org.opencontainers.image.source="https://github.com/morinoparty/fruits"` を付けてください (現状 `java/21-graalvm_jdk/Dockerfile` だけは旧リポジトリの `https://github.com/Nlkomaru/yolks` のままです)。
- イメージを追加・削除したら、`.github/workflows/build.yaml` の `matrix.tag` も合わせて更新してください。

## egg (YAML)

- egg は `PLCN_v3` 形式です。先頭の `_comment` / `meta.update_url` などのヘッダーは既存のまま維持してください。
- インストールスクリプトは `ghcr.io/pelican-eggs/installers:alpine` 上の `ash` で動きます。bash 専用の構文を使わないでください。
- 起動コマンドは `java/entrypoint.sh` が `{{VAR}}` を `${VAR}` に変換して `eval echo` で展開します。ワイルドカードもここで展開されます。

## 確認コマンド

このリポジトリには package.json や Makefile は無く、ビルドは CI (`.github/workflows/build.yaml`) の Docker ビルドです。
コードを書いた後は、変更した内容に応じて以下を実行して確認してください。

- Dockerfile を変更した場合: CI と同じコンテキストでビルドする
  ```bash
  docker build --platform linux/amd64 -f java/<tag>/Dockerfile java
  ```
  (`<tag>` は `25-temurin` などのディレクトリ名。イメージを GHCR に push してはいけません)
- `java/entrypoint.sh` を変更した場合: `bash -n java/entrypoint.sh` で構文を確認する
- egg を変更した場合: YAML として読めることを確認する
  ```bash
  python3 -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]" eggs/paper/egg-fruits-paper.yaml eggs/velocity/egg-fruits-velocity.yaml
  ```
- 起動コマンドや JVM フラグを変更した場合は、README の Verification 節と同じ手順 (フラグ単体 → 結合した文字列 → 実際の起動 → `entrypoint.sh` 経由で `STARTUP` を展開) で、対象のすべてのイメージで確認してください。
  `Unrecognized` / `Ignoring option` / 非推奨の警告が出るフラグは採用しないでください。
