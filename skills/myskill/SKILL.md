---
name: myskill
description: Claude Code の skill を作る・改善・レビューするときの自分の方法論（規範）。新しい skill を書く、既存 skill を直す、なぜ発火しないか調べる、references の分け方を決める、というときに使う。点検する観点は、frontmatter（argument-hint/arguments は引数を取る invocable skill のときだけ・任意項目は metadata 配下）、ディレクトリ構造、責務の分離（1 ファイル 1 責務）、skill のパターン分類（problem-first/tool-first、Pattern 1〜5）、ワークフロー型 skill の step 分割。各観点を criterion 形式（条件→良い例/悪い例→Why）で扱う。雛形生成・eval・description 最適化など作成作業そのものは skill-creator を併用する。
---

# skill 作成の方法論（自分用の規範）

Claude Code の skill を新規作成・改善するときは、この規範で点検する。
各項目は「満たすべき条件 → ✅ 良い例 / ❌ 悪い例 → Why」で書く。

## frontmatter

### `argument-hint` と `arguments` は、引数を取る invocable skill のときだけ入れる

`argument-hint`: 予想される引数を示すために、オートコンプリート中に表示されるヒント。例 `[issue-number]`、`[filename] [format]`。
`arguments`: skill コンテンツ内の `$name` 置換用の名前付き位置引数。スペース区切り文字列または YAML リストを受け付ける。名前は順序で位置にマップされる。

✅ ユーザーが `/` で起動して引数を渡す invocable skill では、両方を入れる:
```yaml
argument-hint: "[filename] [format]"
arguments: filename format
```
本文側で `$filename` `$format` を参照する。

❌ 引数を取らない（モデルが規範として読む自動発火型の）skill に、効果のない `argument-hint` / `arguments` を付ける。

Why: この 2 フィールドは optional で、ユーザーが `/` 起動で引数を渡す invocable skill のときだけ意味を持つ。`argument-hint` は補完時に「何を渡せばよいか」を提示し、`arguments` は本文中の `$name` をユーザー入力で置換する。引数を取らない skill に付けても何も起きず、frontmatter のノイズになるだけなので付けない。

### optional な項目は `metadata:` 配下にまとめる

✅
```yaml
metadata:
  author: ProjectHub
  version: 1.0.0
  mcp-server: projecthub
```
❌ `author` `version` `mcp-server` などをトップレベルに直書きする。

Why: トップレベルは予約フィールド（`name` `description` `argument-hint` `arguments`）の名前空間である。任意のメタ情報をそこに並べると、予約キーと混ざって判別できなくなる。`metadata:` 配下に束ねれば、「ここから下は任意メタ」と構造で明示でき、予約キーの名前空間を汚さない。

## ディレクトリ構造

### 規定の 4 構成に従う

```
<skill-name>/
├── SKILL.md      (required) YAML frontmatter 付きの Markdown 指示
├── README.md     (optional) 人間向け・skill 実行時には読まれない／直下に置ける唯一の例外
├── scripts/      (optional) 実行コード（Python, Bash など）
├── references/   (optional) 必要なときに読み込むドキュメント
└── assets/       (optional) 出力に使うテンプレート・フォント・アイコン
```

4 構成は `SKILL.md` ＋ `scripts/` `references/` `assets/`。`README.md` はこの 4 構成には含まれない、直下に許される唯一の例外ファイル。

### 直下には SKILL.md 以外のファイルを置かない（README.md は例外）

✅ 補助ファイルは役割ごとに `scripts/` `references/` `assets/` へ入れる。直下は `SKILL.md`（と必要なら `README.md`）のみ
❌ 直下に `schema.json` `template.html` `notes.md` などを並べる

Why: 直下にファイルが散らばると、それが実行コードなのか参照資料なのか出力素材なのかが構造から読み取れず、いつ読み込むべきか（progressive disclosure）の判断もできない。役割ごとのサブディレクトリに分ければ、用途と読み込みタイミングがディレクトリ位置から自明になる。`README.md` は人間向けのメタ情報で skill 実行時には読まれないため例外とする。

## 責務の分離

skill のファイルが複数になったら、責務を厳格に分離する。ファイルが増えるほど、同じことを複数ファイルに書く（重複）／ある責務の知識が隣のファイルへ漏れ出す、が起きやすい。これはワークフロー型に限らず、複数ファイルを持つすべての skill に効く一般原則である。

### 1 ファイル 1 責務を守り、重複と知識の漏れ出しを禁じる

✅ 各ファイルが単一の責務だけを持つ。ある知識・定義・手順は、ただ 1 つのファイルにだけ書く（Single Source of Truth）
❌ 同じ説明を複数ファイルに重複させる／あるファイルの責務が別ファイルへ滲み出す

Why: 1 箇所の変更が複数ファイルの変更を誘発するなら、責務分離ができていない（DDD と同じ判定基準）。重複は、片方だけ直して食い違う事故を生む。特に AI は責務を**漏れ出させる**傾向が強い — 隣のファイルの前提や知識を勝手に持ち込み、同じことを繰り返し書く。だからファイル境界 = 責務境界を物理的に保ち、各知識の置き場所を 1 つに固定する。

### skill をまたぐときは、他 skill の内部実装を利用側に書かない

✅ 他 skill を利用する側は、公開インターフェース（サブコマンド名・id・引数）だけを参照する。他 skill の内部（ファイル名・パス・URL・データ構造）は提供側 skill に隠蔽し、利用側から名指ししない
❌ 利用側の skill に、提供側の内部ファイル名・配信 URL・具体パスをハードコードする

Why: 「責務の分離」を skill 間の境界に広げたもの。他 skill の内部を利用側に書くと、提供側の変更が利用側へ波及し（二重管理・乖離）、境界が崩れる。利用側が依存してよいのは提供側の公開インターフェースだけで、内部実装は提供側が単独で変えられる状態に保つ。

## 記述する事実は根拠を確認してから書く

skill の SKILL.md・references・yaml に、実装や環境に依存する事実（パス、ファイル名、コマンド、仕様、階層構造）を書くときは、実物で裏を取ってから書く。

### 実装・構造・既存定義を確認してから事実を書く

✅ dest やバケットの階層、他ファイルの仕様など、実装・環境に依存する事実は、該当コード・実際の構造・既存ドキュメントを確認して書く。レビューや指摘に沿って直すときも、その指摘が実物と整合するか自分で検証してから反映する。確認できない事実は断定せず、不確実性を明示して確認を求める
❌ それらしい一般論・単純化で断定する／一部だけ見て「常に〜」と広げる／指摘を鵜呑みにして一般化する

Why: skill のドキュメントは動くコードと違い、テストや実行では検証されない。誤った記述はレビューまで露呈せず（フィードバックが遅い）、そのまま蓄積して手戻りを繰り返す。事実の裏取りを記述の時点で済ませることが、この遅いフィードバックを補う防波堤になる。指摘への追随でも検証を省くと、正しい記述を誤った一般化で上書きしてしまう。

## skill のパターン分類

新しい skill を作る前に、まずそれがどのパターンに当たるかを判定する。複数にまたがることもあるが、たいていは一つに寄る。出典は公式ガイド "The Complete Guide to Building Skills for Claude"（Ch.5 Patterns and troubleshooting）。

判定の入口として、まず問題起点（problem-first）か道具起点（tool-first）かを見る。

- **problem-first**: ユーザーは達成したい結果を言う（例「プロジェクトを立ち上げたい」）。skill が正しい順序でツールを束ねる。
- **tool-first**: ユーザーは既にツール接続を持つ（例「Notion MCP を繋いだ」）。skill が最適なワークフローと定石を教える。

そのうえで、以下の 5 パターンのどれかを判定する。

- **Pattern 1: Sequential workflow orchestration** — 多段処理を決まった順序で進める必要があるとき。明示的な step 順序・step 間依存・各段での検証・失敗時のロールバックを持つ。→ 該当する場合の詳細は下の「ワークフロー型 skill」。
- **Pattern 2: Multi-MCP coordination** — 処理が複数サービス（複数 MCP）にまたがるとき。フェーズを明確に分け、MCP 間でデータを受け渡し、次フェーズへ進む前に検証し、エラーを集約して扱う。
- **Pattern 3: Iterative refinement** — 出力が反復で品質を上げられるとき。明示的な品質基準・反復改善・検証スクリプト・止めどきの判断を持つ。
- **Pattern 4: Context-aware tool selection** — 同じ目的を文脈次第で違うツールで達成するとき。明確な判定基準・フォールバック・選択理由の透明化を持つ。
- **Pattern 5: Domain-specific intelligence** — ツールアクセスを超えた専門知識を skill が足すとき。ドメイン知識をロジックに埋め込み、行動の前に検証（コンプライアンス等）し、監査可能性を残す。

Why: パターンを先に確定させると、SKILL.md / references の分け方も検証の打ち方も変わる。自覚せずに書くと、順序が要る処理を反復型のように書くなど、構造とパターンがずれて破綻する。まず「自分はどのパターンか」を宣言してから中身を書く。

## ワークフロー型 skill（Pattern 1: Sequential workflow orchestration の詳細）

### 処理が A→B→C と段階的に進む skill は「ワークフロー型」だと自覚する

✅ SKILL.md 冒頭で「この skill は step を順に進めるワークフロー型である」と宣言し、step の一覧と遷移（A→B→C）を示す
❌ 段階的な処理を、ワークフローと自覚しないまま 1 枚の SKILL.md にべた書きする

Why: ワークフロー型だと自覚させないと、モデルは step の境界を意識せず、前後の step の知識を混ぜたり順序を飛ばしたりする。「これはワークフロー型で、いま自分はどの step か」を構造で持たせると、遷移が制御可能になる。

### SKILL.md はワークフロー管理だけを持ち、各 step の中身は references/ に委譲する

✅ SKILL.md には step の一覧・遷移条件・どの step でどの reference を読むか、だけを書く。各 step の具体的な処理は `references/` 側に置く
❌ SKILL.md に全 step の処理を詰め込む

Why: SKILL.md は起動時に丸ごと読まれる。全 step の処理を載せると、いま不要な step の詳細までコンテキストを食い、現在地が埋もれる。SKILL.md を「管理（どの順で・いつ何を読むか）」に限定し、処理本体を必要時にだけ読む reference に逃がすと、各 step 実行時のコンテキストが現在の step に集中する。

### 小〜中規模：step ごとに `references/*.md` へ分割し、1 ファイル 1 責務にする

✅ step ごとに `references/step-a.md` `references/step-b.md` … と 1 ファイルに分け、各ファイルはその step の処理だけを持つ
❌ 複数 step を 1 ファイルに混在させる／1 つの step の処理を複数ファイルに散らす

Why: これは「責務の分離」を step に適用したもの。step A の仕様を変えたら step A のファイルだけが変わるべきで、それが境界の正しさの判定基準になる。重複と知識の漏れ出しを防ぐ一般原則は「責務の分離」を参照。

### 大規模：`references/**/**.md` へ階層化し、`references/` 直下にファイルを直接置かない

✅ `references/<group>/<step>.md` のように 1 段以上のディレクトリで階層化する。直下は常にディレクトリのみ
❌ 規模が大きいのに `references/` 直下に多数の `.md` を平置きする／階層の深さが場所によってバラバラ

Why: 大規模で平置きすると、ファイル数が増えた瞬間に関係性が読めなくなる。グループ単位でディレクトリに束ね、**レイヤーの深さをできる限り揃える**と、どこに何があるかが構造から予測できる。多少冗長（中身が薄いディレクトリができる等）でも、レイヤーの一貫性を優先する。深さが揃っていることが、ナビゲーションと責務の見通しを支える。
