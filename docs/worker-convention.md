# ワーカー規約

Cloud Run 上の Web 面が Cloud Tasks へ投入し、worker 面が受け取って成果物を作る
アプリケーション群で、ジョブ 1 件の「一生」をどう扱うかを決める文書です。
想定する読み手は、新しいワーカーアプリを起こす人と、既存アプリにコマンドを足す人。
読み終えると「投入から結末の記録までにどの順で何をするか」を規約から機械的に決められ、
既存アプリのどこが規約から外れているかを自分で判定できます。

URL の切り方は [url-naming-convention.md](url-naming-convention.md)、キューと
タイムアウトのインフラ側の判断は `ap-infra/docs/conventions.md` が持ちます。
この文書はその間、アプリのコードが持つ部分だけを扱います。

---

## 1. なぜ揃えるのか

ジョブが黙って消える事故は、どれも「順序」か「context」の取り違えで起きました。

- 投入してから queued を書くと、数十ミリ秒で届いたワーカーが書いた running を、
  遅れて来た queued が上書きする（2 アプリが本番で踏んだ）。
- 失敗パスだけを切り離した ctx で記録し、成功パスを生の ctx で記録すると、期限と
  前後して完了したジョブが running のまま固着する（3 アプリで同時に見つかった）。
- panic を回復しないと、HTTP 側が 500 を返すだけで結末が記録されない。キューは
  `max_attempts = 1` なので再試行も来ず、ジョブは永久に running になる（1 アプリだけ
  回復していた）。

どれも「書き間違い」ではなく「片方への適用忘れ」です。ヘルパーの共通化では防げず、
順序そのものをライブラリが持つ形にすると、忘れる書き方が表現できなくなります。
その形が `gcp-kit/worker.Lifecycle` で、この規約はそれをどう埋めるかの取り決めです。

---

## 2. 規約

### 2.1 ジョブの一生

```
Web 面                                 worker 面（gcp-kit/worker.Lifecycle）
────────────────────────────────       ──────────────────────────────────────────
1. ジョブ ID を採番する                  4. Prepare   ログの相関キーを ctx に載せる
2. queued を記録する（Recorder.Save）    5. Begin     再配信ガード＋running の記録
3. Cloud Tasks へ投入する                6. Validate  入力の検証（失敗は Permanent）
   → 202 と Location: /jobs/{jobID}      7. Run       本体（Timeout はここだけ）
                                         8. Finish    結末の記録と通知（切り離した ctx）
```

| 段 | 決定 | 理由 |
|---|---|---|
| 2 → 3 の順 | **queued を書いてから投入する** | 逆順だと、ワーカーが書いた running を遅れてきた queued が上書きする。投入の失敗時は queued を取り消す |
| 5 | `Recorder.Begin` を使う。前回の記録を 1 回読んで「完了済みなら打ち切り、未完了なら running を記録」を行う | `AlreadySucceeded` と `Record` を分けると、その間に別の配信が割り込む |
| 5 の失敗 | **状態を読めなければ実行しない**。通知先があれば記録なしの通知を出し、エラーを返す | 「未完了」に倒すと完了済みを作り直し、「完了済み」に倒すと未完了が ACK されて二度と走らない。どちらにも倒さず人に知らせる |
| 5 → 6 の順 | Begin を検証より前に置く | 全試行が Attempts に載り、検証で落ちたジョブも running を経由して failed に至る |
| 6 | 配り直しても直らない失敗（入力不正、上限超過、参照先が無い）は `worker.Permanent` に包む | 包まないと 500 として再配信され、同じ失敗通知が 2 通届く。`max_attempts = 1` でも配線しておく。上げたときに直す場所を探さなくて済む |
| 7 | 実行時間の上限は `Lifecycle.Timeout`（`PIPELINE_TIMEOUT`）で Run にだけ掛ける | 呼び出し元の ctx に掛けると、打ち切られた直後の Finish まで期限切れの ctx で走る。値の大小関係（三段）はインフラ規約 |
| 7 の panic | ライブラリが回復して `worker.ErrPanicked` として Finish へ渡す | アプリで recover を書かない。書き忘れる場所を残さない |
| 8 | 成功も失敗も同じ Finish で、記録 → 通知の順 | 記録できていないジョブの成功を人に知らせない |
| 8 の ctx | ライブラリが `context.WithoutCancel` で切り離し、30 秒の上限を与える | 打ち切りこそが終端の理由である場面では、元の ctx は既に Done |
| 8 の失敗の扱い | Finish は cause をそのまま返す。Permanent なら 2xx で打ち切り、それ以外は 500 で再配信 | 記録と通知は済んでいるので、利用者から見た結果は変わらない |

Finish が呼ばれないのは、Begin が「完了済み」か「読めない」を返したときだけです。

### 2.2 ペイロード

| 項目 | 決定 |
|---|---|
| 型 | 投入側（`gcp-kit/tasks.Enqueuer[T]`）と受信側（`worker.Handler[T]`）で同じ `domain.Task` を共有する。契約は型で、JSON の写しを 2 箇所に持たない |
| `job_id` | 必須。GCS のパスと Firestore のドキュメント ID になるので、`go-utils/jobid` で検証してから使う |
| `command` | ジョブの種別。コマンドが 1 つしか無いアプリでも持つ。状態の記録と一覧の絞り込み（`?kind=`）がこれを読む |
| 大きな入力 | ペイロードに載せず、先に保存してから ID だけを渡す（Cloud Tasks の 1MB 上限。台本・レシピ） |
| 受け口 | `POST /tasks/<動詞>`。パスは `internal/domain/routes.go` の定数 1 箇所で持ち、投入側と受信側の両方がそれを読む |

### 2.3 タスク名（重複排除）

決定的なタスク名（`EnqueueWithName` / `WithTaskID`）を付けるのは、**同じ名前で届いた 2 通目が
「同じ仕事」であるとき**だけです。Cloud Tasks の重複排除は完了後もしばらく効き続けるので、
名前が同じなら投げ直しは `ALREADY_EXISTS` として黙って捨てられます。

- 付ける: 同じジョブを継続するタスク（カット単位の継続生成）。二重に走らせたくない。
- 付けない（または投入時刻を名前に含める）: 作り直し・再生成。投げ直しが正しい操作。

「対象だけを名前にする」と、作り直しの投入が前回の完了済みタスクに吸われます。
ap-story がこれを踏んで、まとめてよい重複だけに範囲を絞りました。

### 2.4 状態の保存先

Firestore（`gcp-kit/jobstatus`）と GCS（`go-job-kit/jobstatus`）の 2 つを許します。
`Recorder` の形（`Begin` / `Record`）は同じで、Lifecycle からは区別しません。
どちらを選ぶかの基準は一覧のコストで、原子性ではありません（GCS に残るのは、一覧が
prefix 走査で足りる規模のアプリだけ）。

### 2.5 ファイルと名前

| 対象 | 決定 |
|---|---|
| パッケージ | `internal/pipeline`。ワーカーの本体は adapters に置かない（adapters は外部サービスの境界だけ） |
| 入口 | `pipeline.go` に `Runner` 型と `NewRunner` を置き、`Lifecycle` を組み立てて返す |
| 工程の単位 | `Step`（`Name()` と `Execute(ctx, *Context)`）。工程間で引き継ぐ値は `Context` 1 つに載せ、引数で渡し回さない |
| 工程のファイル | 1 工程 1 ファイル、`step_<名前>.go`。工程が多くパッケージが膨らむなら `step/` サブパッケージに移し、その中では接頭辞を付けない（`step/scene_split.go`） |
| コマンド → 工程 | `planner.go` の `Planner.Plan(task) ([]Step, error)` だけが対応を持つ。1 コマンド 1 工程のアプリでも同じ形にする。`Runner.run` は返った列を順に実行するだけで、`command` で分岐しない。本体がライブラリ呼び出し 1 つで工程に分かれないアプリ（adk-review）は `Step` も `planner.go` も持たず、`Run` から直接呼ぶ |
| 状態の記録 | `job_status.go`（`begin` / `markSucceeded` / `markFailed`） |
| 通知 | `notify.go` |
| 型名 | アプリ名を重ねない（`AdkReviewRunner` のようにしない）。パッケージ名（`internal/pipeline`）が既にアプリを言っている |

工程の呼び名（Filter / Workflow / Step）は作った年代で違っていましたが、揃えました。
接頭辞で種類が分かれば読めるという案もありましたが、grep が 3 通りになり、新しいアプリを
起こす人がどれを写すか迷う点で、1 つに決めるほうが安く済みます。

---

## 3. 採らなかった案

### 3.1 `worker.Handler` 側で実行時間を打ち切る

```text
採用: Lifecycle.Timeout（Run にだけ掛ける）。Handler の WithTimeout は削除
理由: Handler の上限は executor 全体に掛かるので、結末の記録まで期限切れの ctx で
      走る。記録が要るのは打ち切られた場面なので、上限は記録より内側に無いと
      いけない。2 箇所に上限があると、どちらが効いているかも読めない
再検討の条件: なし
```

### 3.2 状態を読めないときに「進む」

```text
採用: 進まない。通知して、エラーを返す
理由: 1 アプリが「記録が読めないことを理由に止めるより二重実行のほうが回復可能」
      として進んでいた。だが進むと、完了済みのジョブを再配信で作り直し、ガードが
      防ぐはずの費用を自分で払う。読めない事態は稀で、人が投げ直せば済む
再検討の条件: 二重実行の費用が無視できるアプリだけ。そのときは Begin の中で
      決めて、規約の例外として書く
```

### 3.3 `recordOutcome` をアプリごとに書く

```text
採用: Lifecycle.Finish に渡す。順序と ctx の切り離しはライブラリが持つ
理由: 5 アプリが同じ 20 行を写しており、うち 3 つで成功パスの切り離しが漏れた。
      写す限り、次に増えるアプリでも漏れる
再検討の条件: なし
```

### 3.4 `max_attempts` を 2 以上にする

```text
採用: 1（voice-queue だけ 2）
理由: 再試行は「何が起きたか」を隠す。失敗はその場で記録と通知に残して、人が
      投げ直す。2 にするなら Permanent の配線が前提で、それは 2.1 の 6 で済ませてある
再検討の条件: 一時障害での損失が手作業の投げ直しより高くつくアプリが出たとき
```

---

## 4. 規約から外れたルートの見つけ方

```bash
# Lifecycle を使わずに TaskExecutor を実装している
grep -rn 'func (.*) Execute(ctx context.Context' internal --include='*.go' | grep -v _test
# アプリ側に残った recover / WithoutCancel（Lifecycle が持つので、Finish の経路に残っていれば二重）
# 工程の途中で成果物を退避する保存（打ち切られた場面こそ残す価値がある下書き・部分保存）は例外
grep -rn 'recover()\|WithoutCancel' internal/pipeline --include='*.go' | grep -v _test
# 工程の型名が Step 以外の呼び名で定義されている（ライブラリ由来の Workflows などは対象外）
grep -rn 'type \w*\(Filter\|Workflow\)\b' internal/pipeline --include='*.go' | grep -v _test
# コマンド → 工程の対応が planner.go 以外にある（Runner.run の switch が疑わしい）
grep -rn 'switch .*Command' internal/pipeline/pipeline.go
# 投入より先に queued を書いているか（Enqueue の直前に Save があること）
grep -rn -B6 '\.Enqueue(ctx' internal/server/handlers --include='*.go' | grep -c 'Save\|recordQueued'
```

---

## 更新履歴

- 2026-09-04: 初版。5 アプリのワーカー入口を突き合わせ、揃っていた順序（queued → 投入、
  Begin → 検証 → 実行 → 記録）を `gcp-kit/worker.Lifecycle` に移して規約にした。
  差があった 3 点（panic の回復、`Permanent` の配線、読めないときに進む方針）は
  ライブラリと規約の側で 1 つに決めた。工程の呼び名（Filter / Workflow / Step）と
  コマンド → 工程の置き場（`planner.go`）も 1 つに揃えた。
