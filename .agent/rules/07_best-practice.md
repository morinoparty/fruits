# 推奨される書き方

## JVM フラグ
- GC は PaperMC が推奨する G1GC を基本にしてください (Paper は Aikar's flags、Velocity は Velocity の tuning guide のフラグ)。
- `-Dterminal.jline=false -Dterminal.ansi=true` は、パネルのコンソールで色を出し、`Advanced terminal features are not available` の警告を消すために付けています。外さないでください。
- 特定の Java バージョンでしか動かないフラグを使う起動コマンドは、名前に `Java 25 - ...` / `Java 21 - ...` と対象を明記してください。`Default` はすべてのイメージで動くものにしてください。
- 警告を隠すためだけのフラグ (例: `--sun-misc-unsafe-memory-access=allow`) は追加しないでください。

## イメージ
- egg に載せるイメージを増やす場合は、README の Runtime comparison のように実測した結果を根拠にしてください。
- 上流タグに追従するイメージ (Temurin / Zulu / Semeru) でバージョンを固定しないでください。固定が必要なのは tarball から入れる GraalVM だけです。

## インストールスクリプト
- バージョンやビルド番号は完全一致で比較してください (`4.2` が `4.2.0` にマッチしないように)。
- 再インストールで既存のサーバーデータ (設定ファイルなど) を上書きしないでください。
- ワイルドカードの jar 名を使う場合は、古い jar を `.jar.old` にリネームして、マッチするファイルが常に 1 つになるようにしてください。

## Test
- 確認は網羅的にするよりも、変更が影響するイメージと起動コマンドの組み合わせを漏れなく実際に動かすことを優先してください。
