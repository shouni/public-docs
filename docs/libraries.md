# ライブラリリファレンス

公開 Go ライブラリ群の**境界の地図**です。どのライブラリが何を引き受け、何を引き受けないか、
隣とどう違うか、なぜその層に置いたかを引くために置いています。

ここにあるのは、**1 つのリポジトリの中だけを見ていては書けないこと**だけです。それ以外は次にあります。

| 知りたいこと | 置き場 |
|---|---|
| 使い方、最初の 1 本 | 各リポジトリの README |
| シグネチャ、引数、エラー | godoc（pkg.go.dev） |
| そのライブラリ内部の設計判断、採らなかった案 | 各リポジトリの CLAUDE.md |

層の下から順に並んでいます。全体の設計思想と層の関係は
「[個人開発エコシステムの全体像](https://zenn.dev/snknsk/articles/shouni-go-ecosystem-overview)」を参照してください。
アプリケーションと MCP サーバーは [applications.md](applications.md) にあります。

---

## 🏗️ コア基盤ライブラリ群 (Core Infrastructure Libraries)

通信の安全性、リトライ、ストレージアクセス、素材の取得、音声バイナリ、通知、非同期ジョブの共通基盤です。
いずれも AI の話を一切含まないため、生成系以外のアプリケーションでもそのまま使えます。

### netarmor

外部通信の「安全性」だけを担当し、AI もクラウドサービスも知りません。

* **最下層に置いている理由**: SSRF 対策は「どのライブラリでも同じ書き方をしたい」ものであり、かつ
  **間違えても動いてしまう**部類だからです。プライベート IP への接続を 1 か所で塞ぎ忘れても、
  通常の利用では何も起きません。
* **外部依存を持たない**: `go.mod` の `require` は空で、`go.sum` も 0 バイトです。これだけの数の
  リポジトリが取り込む土台なので、ここに 1 行足すと使わないリポジトリのモジュールグラフにまで載ります。
* **`retry` は go-http-kit へ移した**: 指数バックオフは当初ここに同居していましたが、使っているのが
  `go-http-kit` と、元からそれに依存していた `go-web-reader` だけでした。移しても依存辺は 1 本も増えず、
  代わりに `require` が空になりました。
* **クラウドストレージのスキームは扱わない**: `gs://` や `s3://` は「名前解決せず通す」対象でしたが
  落としました。**接続しない URI を検証器が通すと、検証を通った事実が何も保証しなくなる**ためです。
  スキーム判定は、実際に読み書きする `go-remote-io` の仕事に寄せています。

依存の広がりはエコシステムで最大です。`go.mod` にこのモジュールが現れるリポジトリは 11 あり、
そのうち直接 require しているのは 7 で、残りは `go-http-kit` を経由して入ってきます。

このライブラリだけを、セキュリティ / アジリティ / レジリエンス / トレーサビリティの 4 つの柱という
切り口で書いた記事があります：[AIのカオスに対抗する、NetArmor「凡事徹底」の4つの柱と設計哲学](https://zenn.dev/snknsk/articles/netarmor-four-pillars)（執筆時点では `retry` も同居していました）。

[GitHub - shouni/netarmor](https://github.com/shouni/netarmor)

---

### go-utils

複数のプロジェクトで実際に重複していた小さな処理だけを集めたモジュールです。
`jobid`（ジョブ ID の採番・検証）、`jst`（日本標準時への変換）、`slogctx`（context に積んだ属性を
自動で付ける slog ハンドラ）、`strlist`（設定値の文字列リスト正規化）の 4 パッケージが、
互いに独立して入っています。

`utils` という名前は何でも受け入れてしまうため、**収録の可否を 3 つの条件で決めています**。

* **外部依存を持たない**: 標準ライブラリだけで完結すること。`go.mod` の `require` は 1 行もありません。
* **I/O やインフラに触れない**: ネットワークやクラウド SDK を扱うものは `go-remote-io` や `gcp-kit` など
  目的別のライブラリの担当です。
* **2 つ以上のプロジェクトから使われている**: 1 つでしか使わないものは、その利用者側の `internal/` に
  置きます。汎用に見えて、実際にはそのプロジェクト固有の判断に紐づいていることが多いためです。

3 つ目の条件が効いていて、この 4 パッケージは現在 10 リポジトリから直接 import されています。
基盤ライブラリの `gcp-kit` / `go-job-kit` / `go-notify` もここに依存しますが、逆向きの依存はありません。

[GitHub - shouni/go-utils](https://github.com/shouni/go-utils)

---

### go-serve-kit

HTTP サービスが**応答を返す側**で毎回書くことになる定型——防御的ヘッダー、表現の出し分け、
役割の宣言、埋め込みアセットの配信——だけを引き受けます。サーバーそのものは持ちません。
`respond` / `secureheaders` / `serverrole` / `staticfiles` の 4 パッケージが互いに独立して入っており、
**`go.mod` の `require` は空です**。

* **gcp-kit から切り出した**: いずれも GCP に依存しない処理だったためです。クラウドを使わない
  HTTP サービスが Cloud Run 向けのキットごと取り込む形になっていました。**両モジュールの間に
  依存はありません**（どちらの向きにも）。

[GitHub - shouni/go-serve-kit](https://github.com/shouni/go-serve-kit)

---

### gcp-kit

Cloud Run 上の Web アプリと、Cloud Tasks で動く非同期ワーカーが毎回同じように書くことになる部分
——ログイン、セッション、タスクの投入と受信、進行状況の記録、ログの体裁——だけを引き受けます。
6 つのパッケージ（`auth` / `cloudlog` / `cloudrun` / `jobstatus` / `tasks` / `worker`）は独立していて、
必要なものだけを import できます。

* **ジョブの一生を、順序ごと引き受ける**: `worker.Lifecycle` が「再配信ガード → 検証 → 実行 → 結末の記録」
  の順序を固定します。**5 つのアプリが同じ 20 行を写していて、うち 3 つで成功パスの切り離しが
  漏れていました**。ヘルパーの共通化では防げず、順序そのものをライブラリが持つと忘れる書き方が
  表現できなくなります。埋め方は [worker-convention.md](worker-convention.md) にあります。
* **進行状況の記録先は 2 つあってよい**: `jobstatus` が Firestore 版で、GCS 版は `go-job-kit` が持ちます。
  `Recorder` の形（`Begin` / `Record`）は同じで、上位からは区別しません。**選ぶ基準は一覧のコストであって、
  原子性ではありません**。プレフィックス走査で足りる規模なら GCS のままで困らず、履歴の絞り込みに
  クエリが要る規模で Firestore へ移ります。
* **GCP に依存しない処理は持たない**: 応答の出し分けと防御的ヘッダーは `go-serve-kit` へ出しました。

このキットが前提にしている Cloud Run 側の構成は、3 本に分けて書いています。

* 1 つのイメージを web / worker の 2 サービスに分け、役割で依存グラフを切り替える：[Cloud Run で 1 つのイメージを web/worker に分け、権限と設定を絞る](https://zenn.dev/snknsk/articles/cloudrun-web-worker-split)
* `tasks` / `worker` に相当する非同期処理を、Go と Python で並べた比較：[Cloud Run + Cloud Tasks の非同期処理を Go と Python で比較する](https://zenn.dev/snknsk/articles/cloudrun-cloudtasks-go-python)
* 環境変数やモデル名をアプリのビルドから切り離す：[モデル名を変えるたびに、アプリをフルビルドしていた。Cloud Run の設定を Terraform へ移す](https://zenn.dev/snknsk/articles/cloudrun-config-terraform-import)

[GitHub - shouni/gcp-kit](https://github.com/shouni/gcp-kit)

---

### go-http-kit

`netarmor/securenet` による SSRF 対策と指数バックオフを最初から組み込んだ `net/http` 互換の
HTTP クライアント（`httpkit`）と、その土台になる汎用リトライ（`retry`）の 2 パッケージです。

* **役割を 2 つに割る**: `retry` は「いつ・どれだけ待つか」だけを決め、「何を再試行する価値があるか」と
  「リクエストをどう再送するか」は `httpkit` が持ちます。HTTP の知識を `retry` 側へ漏らさないための
  線引きです。**この分離があるので、HTTP を介さない `go-web-reader` の取得ループも同じエンジンに
  乗れます**。

[GitHub - shouni/go-http-kit](https://github.com/shouni/go-http-kit)

---

### go-remote-io

Google Cloud Storage・Amazon S3・ローカルファイルシステムを、URI スキームの自動判定で
`Open(ctx, path)` 一本に揃える I/O ライブラリです。**成果物の置き場**を担当します。

* **コアはクラウド SDK を import しない**: 本体は抽象だけを持ち、GCS SDK は `gcs`、AWS SDK は `s3`
  パッケージだけが依存します。片方しか使わないアプリのビルドに、もう片方の SDK が入りません。
* **素材の取得は `go-web-reader`**: 両方 `gs://` を読みますが工程が違います。対照表は次項にあります。

このライブラリが書き出す先の設計——成果物とメタデータを分けて置く、署名付き URL で配信する、
削除で関連オブジェクトをまとめて消す——は「[AIアプリの成果物を履歴・再生・削除まで扱う設計](https://zenn.dev/snknsk/articles/gcs_object_design)」にまとめています。

[GitHub - shouni/go-remote-io](https://github.com/shouni/go-remote-io)

---

### go-web-reader

`https://` / `gs://` / `s3://` を同じ `Open(ctx, uri)` で読むライブラリです。
**AI に渡す素材を取ってくる側**を担当します。

go-remote-io と役割が紛らわしいので、線引きを明示しておきます。両方 `gs://` を読みますが、
担当している工程が違います。

| | go-remote-io | go-web-reader |
| --- | --- | --- |
| 方向 | 読み書き両方（+ 署名付き URL・一覧） | **読み取り専用** |
| 対象 | `gs://` / `s3://` / ローカル | `https://` / `gs://` / `s3://`（**ローカルは非対応**） |
| 立ち位置 | **成果物の置き場** | **素材の取得元** |
| HTML | バイト列としてそのまま | **本文だけを抽出**（広告・ナビゲーションを除去） |

* **接続直前の IP 検証は `netarmor` の担当**: 利用者が入力した URL を叩く以上、取得前の URI 検証だけでは
  足りません。検証を通ったあとに DNS が内部 IP へ解決される経路は、最下層で塞ぎます。

[GitHub - shouni/go-web-reader](https://github.com/shouni/go-web-reader)

---

### audio

音声バイナリと日本語の読みだけを扱うライブラリです。netarmor と同じく、AI もクラウドも知りません。

* **合成と結合を別のライブラリに分ける**: WAV の連結と読みの正規化はここ、音声合成は `go-voicevox`
  です。音声エンジンを差し替えても、結合側は変わりません。

読み正規化の実装は「[AIの誤読を許さない。プロンプトを安全な「読み形式」へトランスパイルする防衛戦術](https://zenn.dev/snknsk/articles/go-kagome-prompt-defense)」で 1 本にしています。

[GitHub - shouni/audio](https://github.com/shouni/audio)

---

### go-job-kit

Cloud Tasks へ投入し、オブジェクトストレージへ成果物を書き出す非同期ジョブの共通基盤です。
**成果物が音楽であれ動画であれ漫画であれ、「投入する → 進行状況を記録する → 履歴をページングする」の
骨格は同じ**になります。各アプリの `internal/` に同じ実装を抱えると少しずつ食い違っていくため、
その共通部分だけを抜き出しました。

* **成果物のドメインには踏み込まない**: 何を生成するかはアプリ側の関心で、共通なのは「状態をどう記録し、
  履歴としてどう見せるか」だけ、という線引きです。
* **Firestore 版は `gcp-kit/jobstatus`**: `Recorder` の形は同じで、選び方は前掲のとおり一覧のコストです。

履歴一覧の並列取得やキャッシュ、`job_id` の付け方といった運用側の判断は
「[AIアプリの成果物を履歴・再生・削除まで扱う設計](https://zenn.dev/snknsk/articles/gcs_object_design)」で扱っています。

[GitHub - shouni/go-job-kit](https://github.com/shouni/go-job-kit)

---

### go-notify

非同期パイプラインの実行結果を人へ届ける部分だけを担当します。CLI を持たず、
アプリケーションに組み込んで使います。

* **チャネルは本文の外側にある**: 本文の組み立ては標準的な Markdown で行い、Slack 固有の mrkdwn 記法への
  変換は `slack` パッケージが担当します。**本文を組み立てるコードはチャネルを意識しません。**

[GitHub - shouni/go-notify](https://github.com/shouni/go-notify)

---

## 🤖 AI抽象化ライブラリ群 (AI Abstraction Libraries)

LLM のプロンプト構築、レスポンス管理、キャラクター定義の共有、および画像・音声・動画生成を
抽象化するライブラリ群です。上位のオーケストレーターはこの層より下の SDK を直接 import しません。

### go-prompt-kit

AI へのプロンプト管理と、レスポンスのドキュメント化（Markdown / JSON から HTML へ）を担当します。
`prompts` / `resource` / `frontmatter` / `htmldoc` の 4 パッケージが独立して入っています。

* **メタデータの書式は解釈しない**: front matter の解析関数を呼び出し側から受け取るため、このモジュールは
  YAML ライブラリに依存しません。ライブラリが解析器を固定すると、乗り換えのたびに利用側とリリースを
  揃える必要が生じます。実際 `gopkg.in/yaml.v3` がアーカイブされ利用側が後継へ移ったとき、
  **このモジュールは 1 行も変えずに済みました**。

4 つのパッケージを入力から出力まで一本に並べて書いたのが「[モードを増やす作業を、Markdown 1枚に閉じ込める：Go でプロンプトをコードから引き剥がす](https://zenn.dev/snknsk/articles/go-prompt-kit-mode-per-file)」です。

[GitHub - shouni/go-prompt-kit](https://github.com/shouni/go-prompt-kit)

---

### go-gemini-client

Google Gemini API / Vertex AI 向けのデュアルバックエンド対応クライアントです。API Key 方式
（Google AI Studio）と Project ID / Location ID 方式（Vertex AI）を 1 つのクライアントで切り替えられます。
Vertex AI だけで足りる系統は `genai-kit` が担当しており、使い分けは次項の表にあります。

* **SDK の乗り換えで書き換わる範囲を、このモジュールに閉じる**: 上位が依存するのは `Generator` の
  1 メソッドだけです。ただし一部の型は `genai` の**型エイリアス**なので、効果は「別の SDK へただちに
  差し替わる」ことではなく、**手を入れる場所が上位の 5 リポジトリではなくこの 1 つに集まる**ことです。
* **`MusicRecipe` はエコシステムの共通通貨**: `lyria` パッケージが作る楽曲設計図を、
  **動画側のオーケストレーターも同じ形で入力に取ります**。

[GitHub - shouni/go-gemini-client](https://github.com/shouni/go-gemini-client)

---

### genai-kit

Vertex AI 専用のクライアントです。`gemini` / `lyria` / `veo` / `callguard` は `go-gemini-client` と
同じ構えで、加えて参照画像付きの画像生成 `imagegen` を持ちます。

APIキー方式を落とした分、参照画像の扱いが変わります。使い分けはここだけを見れば決まります。

| | go-gemini-client | genai-kit |
| --- | --- | --- |
| バックエンド | Gemini API（APIキー）と Vertex AI | **Vertex AI のみ** |
| 参照画像 | File API へ上げてキャッシュする経路を持つ | **`gs://` をモデル側に解決させる**（転送が起きない） |
| 画像生成 | `gemini-image-kit` へ委譲 | `imagegen` を内蔵 |

* **`gs://` 以外を受け付けない**: Vertex AI はモデル側で `gs://` を解決するので、取得もアップロードも
  バイト列の転送も発生しません。**HTTP からの取得・サイズ上限・再圧縮・アップロードのキャッシュが要る
  構成は `gemini-image-kit` の担当**で、こちらには経路そのものがありません。
* **流量制御は持たない**: クォータはプロジェクト単位なので、`callguard` のガードをテキスト生成と
  共有する形で上位に置きます。ライブラリごとに独立したレート制限を持たせると、合計がクォータを超えます。

[GitHub - shouni/genai-kit](https://github.com/shouni/genai-kit)

---

### gemini-image-kit

参照画像の**取得・再圧縮・キャッシュまで要る**画像生成を担当します
（`gs://` だけで足りる構成は `genai-kit` の `imagegen` が担います）。

* **外部 URL の取得経路は注入で決める**: 取得は `ports.Downloader` 経由に限定し、SSRF 対策や
  ドメイン制御はアプリケーション側で適用します。
* **流量制御は持たない**: レート制限・並列度・タイムアウトのオプションは撤去しました。クォータは
  プロジェクト単位なので、テキスト生成と同じ `callguard` のガードで包むのが呼び出し側の仕事です。

参照解決の選び方、MIME 推測、シードの扱いをコードまで降りて追ったのが「[Go言語で構築する堅牢なAI画像生成パイプライン：MIME推測・File APIキャッシュ・SSRF対策](https://zenn.dev/snknsk/articles/b694340e5cf15e)」です。

[GitHub - shouni/gemini-image-kit](https://github.com/shouni/gemini-image-kit)

---

### go-character-kit

キャラクターの `id` / `name` / `seed` / `reference_url` / `visual_cues` を JSON 定義として読み込み、
検証し、参照するだけの小さなライブラリです。

* **独立したライブラリにしている理由**は、Seed と参照アセットという一貫性の核が**漫画と動画の両方に
  必要**だからです。どちらかのオーケストレーターに置けば、もう一方がそれを import することになります。
  現在は `go-comic-kit`・`go-veo-orchestrator` を含む 5 リポジトリが同じ定義を読んでいます。

[GitHub - shouni/go-character-kit](https://github.com/shouni/go-character-kit)

---

### go-voicevox

`[]ScriptLine` を受け取り、結合済みの WAV バイト列を返すことだけを担当します。ファイル書き込みも
アップロードも行わず、それらに依存もしません。**保存先を決めるのは呼び出し側です。**

* **結合は `audio` に任せる**: 合成と結合を別のライブラリに分けているので、音声エンジンを差し替えても
  結合側は変わりません。

[GitHub - shouni/go-voicevox](https://github.com/shouni/go-voicevox)

---

## 🎨 生成オーケストレーター (Generation Orchestrators)

AI 抽象化層を組み合わせて「1 つの作品」を組み立てる層です。何を作るかのドメイン（作品の状態・
工程の順序・一貫性の維持）を持つのがここまでで、これより上のアプリケーションは受付・認証・
非同期実行だけを担います。

### go-comic-kit

キャラクターの一貫性を保ったまま、漫画を工程単位で組み立てるツールキットです。

* **プロンプトはすべてアプリが持つ（DI 必須）**: キットは内蔵プロンプトを持ちません。プロンプトは
  作品ごとに作り込む文言なので、キットに置くと 1 文字変えるたびにキットのリリースが必要になります。
* **重複排除は層が 2 つ要る**: 同一内容の同時実行を 1 回にまとめるのは `callguard` の singleflight で、
  これは**プロセス内**に対する重複排除です。時間をまたいだ再配信を止めるのは `go-job-kit` の
  再実行ガードの役目です。層が違うので、両方が要ります。

MangaState と部分再生成の実装は「[Go言語で構築するAI漫画生成：状態ドキュメント1枚で「3コマ目だけ描き直す」を成立させる設計](https://zenn.dev/snknsk/articles/8896d7672c5f68)」で 1 本にしています。

[GitHub - shouni/go-comic-kit](https://github.com/shouni/go-comic-kit)

---

### go-veo-orchestrator

**Music Recipe（楽曲構成書）** から動画カット列を構造化し、Google の動画生成 AI **Veo** へ渡す
バックエンドオーケストレーターです。AI の呼び出しは `genai-kit` 経由です。

* **入力は楽曲側と同じレシピ**: `MusicRecipe` の `sections` や `cuts` から尺と開始・終了位置を補完します。
  楽曲生成と動画生成が同じ形を読むので、間に変換を挟みません。
* **Veo への通信は実装を含まない**: 実通信は `ports.VideoRunner` という 1 メソッドの契約にだけ現れ、
  **実装はこのリポジトリに含まれません**（呼び出し側が注入します）。オーケストレーション・
  キーフレーム生成・メタデータ保存は、それぞれ別の関心として分かれています。

[GitHub - shouni/go-veo-orchestrator](https://github.com/shouni/go-veo-orchestrator)

---

### go-review-kit

Git の 2 つのリファレンス間の差分を取り、AI にレビューさせ、結果を公開して通知するまでを
1 本のパイプラインとして提供します。

* **AI SDK に依存しない**: このリポジトリ群では例外的に、**AI クライアントを一切同梱していません**。
  レビュアーは `WorkspaceReviewer` というポートとして定義され、実装は利用側が差し込みます。
  どの SDK でレビューするかはアプリの選択であり、ライブラリが抱えると全利用者にその依存が
  伝播するためです。直接依存は `go-git` のみです。

エージェント型のレビュアーを差し込んだ側の実践は「[エージェントは「差分の外」を読む：ADK for Go で AI レビューを作り直した実践](https://zenn.dev/snknsk/articles/adk-go-agent-review-practice)」に書いています。

[GitHub - shouni/go-review-kit](https://github.com/shouni/go-review-kit)
