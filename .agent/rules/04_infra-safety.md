# 公開物の扱い

このリポジトリの成果物 (イメージと egg) は、稼働中のサーバーから直接参照されます。

## イメージ
- `.github/workflows/build.yaml` は、`main` への `java/**` の push、毎月 1 日の schedule、`workflow_dispatch` で全イメージをビルドし、`ghcr.io/morinoparty/fruits:java_<tag>` に push します。
- ローカルで `docker push` したり、`gh workflow run` でワークフローを実行したりしないでください。公開はマージ後の CI に任せます。
- 既存のタグ名 (`java_25-temurin` など) は egg とサーバーが参照しているため、変更・削除しないでください。

## egg
- egg の `meta.update_url` は `https://raw.githubusercontent.com/morinoparty/fruits/refs/heads/main/eggs/<名前>/egg-fruits-<名前>.yaml` で、パネルの **Update Egg** はここから取得します。ファイルのパスや名前を変更しないでください。
- egg の `uuid` は変更しないでください。
- `main` にマージした内容がそのままパネルに配布されるため、egg の変更は 02_coding.md の確認手順を済ませてからマージしてください。
