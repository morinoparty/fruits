## ディレクトリ配置規則

- `java/<Javaのメジャーバージョン>-<ディストリビューション>/Dockerfile` に各イメージの Dockerfile を配置します。
  ディストリビューション名は既存の `temurin` / `azul_zulu` / `graalvm_jdk` / `open_j9` に合わせます (例: `java/21-azul_zulu/Dockerfile`)。
  このディレクトリ名がそのまま CI の `matrix.tag` とイメージタグ (`java_<ディレクトリ名>`) になります。
- `java/entrypoint.sh` は全イメージ共通のエントリーポイントです。イメージごとに複製しないでください。
- `eggs/<ソフトウェア名>/egg-fruits-<ソフトウェア名>.yaml` に egg を配置します (例: `eggs/velocity/egg-fruits-velocity.yaml`)。
- `.github/workflows/` に GitHub Actions のワークフローを配置します。
- ドキュメントはルートの `README.md` に書きます。
