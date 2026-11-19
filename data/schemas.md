# スキーマと配置規約 v1.0

ドライラン検証（リポジトリだけを渡した第三者エージェントによる生成テスト）で見つかった不足を埋めるための規約。
install-prompt と function-spec は本書を前提とする。

## 0. アンカー（唯一のハードコード地点）

**`~/.palagial/config.yaml` だけはハードコードとする。** それ以外のパスは一切固定しない。

理由： 既存の稼働系がある環境でも、インストーラや別ベンダーのエージェントが「どこに在るか」を
推測で探すと、別の場所に二重に作ってしまう事故が起きる。全員が同じ1ファイルだけを見れば、
ベンダーが違っても在り処の合意が取れる。

```yaml
anchor_version: 1
install_root: "<絶対パス>"   # 稼働系のルート（house.yaml と SPEC/ を含むフォルダ）
installed_at: "<ISO8601>"
```

解決手順（全ラン・全ツール共通）：

1. `~/.palagial/config.yaml` を読む。無ければ「未インストール」として扱う（勝手に探さない）。
2. `install_root` を絶対パスとして解決し、その直下の `house.yaml` と `SPEC/bootstrap.yaml` から通常手順に入る。
3. cron スタブ・hook・監査などの恒常部品は、稼働系の絶対パスを直書きせず、**このアンカー経由でのみ**在り処を知る。

インストーラ（F8）の分岐：

- アンカーがあり `install_root` が実在する → 既存の稼働系がそこに在る。上書き／移行は必ずその場所に対して行い、別場所に新規作成しない。
- アンカーが無い → オーナーにインストール先を確認する。既定の提案は **`~/Application/PaLaGiAl/`**。決定したら先にアンカーを書き、そこへ生成する。
- アンカーはあるが `install_root` が実在しない → 移動・削除の可能性がある。オーナーに確認するまで生成しない。

## 1. インスタンスの配置規約（標準レイアウト）

稼働系（インスタンス）は1つのルートフォルダに置く。その在り処は §0 のアンカーが指す：

```
<instance-root>/
  house.yaml          # 家の設定（雛形: templates/house.yaml）
  avatar-props.yaml   # アバター属性プロパティ（任意。 §4）
  SPEC/
    bootstrap.yaml / status.yaml / current_version.yaml
    # 家の設定の正はルート直下の house.yaml。版フォルダには置かない
    # （置く場合はインストール時点の snapshot と明記し、正としない）
    spec_<yyyymmdd_hhmmss>/
      spec.yaml / environments.yaml / services.yaml
      projects/<slug>.yaml
    runtime/
      slots.yaml / queue.yaml / emergency-queue.yaml / doctor-log.md
```

## 2. 家名合成（F8 手順1の詳細）

**原則： 合成はプロンプトの裁量である。** オーナーが入力した名字とベース家名プールを材料に、プロンプトが「良きにはからって」良い感じに合成する。スクリプトや固定アルゴリズムではない。以下は再現性のための推奨手順（縛りではない）：

1. `data/house-names.yaml` から tone の合うベース名を3件程度選ぶ。
2. 合成： ベース名の `suffix` を保持し、オーナー名字から代表1字を頭に付ける。代表字は原則として先頭字だが、響きが悪い場合は名字の他の字から選んでよい（理由を synthesis に残す）。
3. `suffix` が空文字のベース名（複合名全体が材料）は、全体の頭字を代表字で置き換えるか、代表字を冠して使う。
4. 読みは合成結果全体に自然な音読み／訓読みで付ける。迷う場合は候補ごとに読みも併記する。
5. 候補は3件程度を提示し、各候補に材料（ベース名＋代表字）と読みを添える。オーナーが選ぶ。
6. 決定した合成の過程は house.yaml の `synthesis` に必ず残す（後から選び直せるように）。

## 3. レジストリの必須フィールドと空リスト規則（F2 検証の基準）

- project の必須フィールド： `id`, `name`, `status`, `overview`。`workspace`, `goals`, `rules_ref`, `chats`, `schedules`, `environments`, `services`, `staff`, `tasks` は欠けてもよいが、書く場合は下記の形に従う。
- **空リストは正**： インストール直後の `services: []`, `chats: []`, `tasks: []`, `schedules: []` は検証を通る。参照整合性は「書かれた参照が実在すること」のみを検査する。
- environments の必須： `id` と `routes`（各 route は `id`）。services の必須： `id`（`<provider>::<persona>` 形式）。認証情報の値は常に禁止。
- `staff.assignee` は null 可。機構だけを担うプロジェクトは null が正（機構は無名）。
- tasks の各要素は `id`, `name`, `schedule` が必須。任意フィールド `executor` の規定は §3.1。
- 初期プロジェクト global の記入例： `templates/spec/project-global.example.yaml`。

## 3.1 タスクの実行経路（executor）

- `executor` の値： `workflow`（既定、省略時）／ `live-browser`。
- `workflow`： 発火ペイロードを受け取った中継が、プロジェクトの保存ワークフローを起動して実行する。汎用サブエージェントで完結するタスク向け。
- `live-browser`： ライブブラウザ操作が必須のタスク。**ブラウザ操作手段を持たない汎用サブエージェント／ワークフローの子エージェントへ流してはならない**（実測障害： ワークフロー子エージェントにブラウザ手段がなく `live_cloud_browser_required` で全タスクが blocked になった、2026-11-14）。
- ディスパッチャの発火ペイロードは `executor` と `delegate_chat` を含む。`delegate_chat` はプロジェクトのピン留め先チャット（`trigger.delivery_chat`、無ければ `chats` の role=operations のもの）。
- **駆動方式（実測に基づく）**： hook の中継 worker には `browser.browse` も `chat.send_message` も存在しない（2026-11-14 プローブ実測： 使えるのは読み取り専用のページ取得、chat の list/read、ファイル書き込み、ワークフロー起動のみ）。ゆえに `live-browser` を含むプロジェクトのディスパッチャ判定は hook 中継ではなく、**`delegate_chat` に配送される driver cron が駆動する**：
  1. driver cron（短間隔、例： 5分）がディスパッチャスクリプトを直接実行して判定する（判定ロジックと発火状態は hook 駆動と同一のものを共用する。同時に両方を駆動してはならない）。
  2. SILENT なら何もせず終了する。WAKE なら、そのチャットのフルエージェントが executor に従ってその場で実行する： `live-browser` は access ピンのままブラウザを直列実行して状態ファイルを更新し、結果をチャットへ報告する。`workflow` のタスクは保存ワークフローを起動する。
  3. 代替方式として、ファイル委譲（中継 worker が `SPEC/runtime/delegations/<project>__<task>.json` に発火ペイロードを書き、pickup cron が拾って実行し削除する）も可とする。陳腐化した委譲は pickup 側が破棄し、再試行の周期は再発火が担う。
- 受け入れ確認は dry ではなく実発火で行う： 発火し、実行され、状態ファイルが更新されるまでを1セットで確認する。
- アクセス経路のピンは executor で変わらない（プロジェクト定義の access が正）。

## 3.2 blocked 報告の集約

- 同一プロジェクト＋同一タスク＋同一理由の blocked が連続する場合、報告は初回のみ。以後の同文繰り返しは抑制する。
- 抑制中は `SPEC/runtime/block-reports.json` に `{project, task, reason, first_at, last_at, count, last_report_at}` を保持する。理由が変わった場合、または前回報告から60分以上経過した場合は、件数つきの要約として再報告してよい。
- blocked が解消してタスクが成功したら、復旧を1回だけ報告し、当該エントリを消去する。

## 4. アバター属性プロパティのスキーマ（F12）

`avatar-props.yaml` の各エントリ（キーは英語、単位は cm / kg）：

| キー | 型 | 備考 |
|---|---|---|
| persona_id | string | data/personas.yaml の id |
| costume | string | 服装の意匠（制服のクラス統一＋プリセットのテーマ） |
| face | string | 顔立ちの傾向 |
| build | string | 体型 |
| hair_color / hair_style | string | |
| skin_tone | string | |
| eye_color | string | |
| three_sizes | string \| null | "B<数>-W<数>-H<数>"。不要・不適な場合は null でよい（全員に必須ではない） |
| height_cm / weight_kg | number \| null | |
| age | number \| "unknown" | 飛丸など年齢不詳は "unknown"。personas の age_bracket からの換算目安： 若年=16-19、若手=20-34、中年=35-49、高齢=65+、不詳="unknown" |

**優先順位の規定**： 個人の固定特徴は personas.yaml の `profile.appearance` が優先。プリセットは服装意匠・配色・全体の傾向に効き、個人の特徴と矛盾する場合は個人側を残し、矛盾しない範囲でプリセットを適用する。

## 5. 版IDの採番と初版の手続き（F4 / F8）

- 版IDは初版も含め常に `spec_<yyyymmdd_hhmmss>`（ローカル時刻）。時刻フィールド（since / activated_at）は実時刻を ISO8601 で書く。
- あわせて spec.yaml に人間可読な通し番号 `version_no` を持つ： 既存稼働系が無ければ 1 から、あれば既存の最終番号＋1（例： 旧来の v1, v2, v3… の続き）。フォルダID（日時）と通し番号は併記され、会話では通し番号（v4 等）で呼んでよい。
- **初版**（既存稼働系が無い環境）： MIGRATION を経ずに生成し、そのまま RUNNING で有効化してよい。
- **上書きインストール**（既存稼働系がある環境）： 必ず F4 の手順（MIGRATION→差分レビュー→承認→フリップ）を経る。

## 6. じいや・ばあやの「命名」の意味（F8 手順2の明確化）

- じいや・ばあやについて生成・命名するのは**名前（下の名前）だけ**である。**名字は家名**であり、フルネームは「家名＋名前」になる（例： 家名が山極印なら、じいやは「山極印 重蔵」、ばあやは「山極印 つる」）。
- personas.yaml の jiiya / baaya の `name` は名前だけの初期値である。インストール時にオーナーが命名した場合、house.yaml の `staff_singletons` の名前がそのインスタンスでの正式名となり、ペルソナの性格・語尾は personas.yaml 側をそのまま使う（差し替えるのは名前だけ）。
- 執事・メイド（マルチインスタンス）の姓名は personas.yaml の固定のものを使う（家名は付けない）。
- リセマラ（F11）では名前も再生成対象になる（飛丸は例外として不変。飛丸は姓名の概念なく「飛丸」のみ）。

## 7. ペルソナ解像度の記録

記録先は house.yaml の `persona_resolution`、表記は漢字とする。段階（塵劫記の小数名から採用。数としては小さくなるほど装飾は濃くなる）：

| 段階 | 10の冪 | 装飾 |
|---|---|---|
| 零 | — | 無装飾（推奨） |
| 模糊 | -13 | 語尾のみ |
| 須臾 | -15 | 口調に性格（仕事には影響しない） |
| 瞬息 | -16 | 姿： 立ち絵・表情アイコン等の視覚要素（初登場時の立ち絵、要所の表情アイコン） |
| 虚空 | -20 | 挙動にも性格（タスクに影響しない範囲で） |
| 阿摩羅 | -23 | 選択不能の見せラベル |

## 8. ダンプの形式（F14）

- ファイル名： `palagial-dump-<yyyymmdd_hhmmss>.tar.gz`（他形式でもよいが、manifest は必須）。
- 同梱物： `instance/`（install_root 一式。ただし `Logs/` は除く。下記）、`anchor/config.yaml`（アンカーの写し）、`manifest.yaml`。
- **ログはダンプに含めず、リストアもしない**： 復元した古いログが現行の履歴として混ざると、ドクター等の判定が偽の証拠を読むことになる。ログを残したい場合はアーカイブとして別途保管し、復元先のライブログへは混ぜない（§13）。
- manifest の必須項目： dump_version、created_at、source_install_root、house_name、spec_current、spec_state、file_count。アーカイブ自体の sha256 は外部の manifest にも残す。

## 9. service-health.yaml の形式（F15）

配置： `<install_root>/SPEC/runtime/service-health.yaml`（実行時状態。版管理の対象外）。

```yaml
services:
  - id: "<provider>::<persona>"
    last_ok: "<ISO8601 | null>"        # 最後に成功した時刻。未使用なら null
    last_check: "<ISO8601 | null>"
    last_error: null                    # または種別のみ: auth_failed / unreachable / rate_limited / unknown
    consecutive_failures: 0
```

## 10. 差分レビューレポートの形式（F16）

```yaml
diff_review:
  from_version: "<版ID>"
  to_version: "<版ID>"
  files: { added: [], removed: [], changed: [] }
  keys:
    added: ["<file>:<key path>"]
    removed: ["<file>:<key path>"]
    changed: ["<file>:<key path>"]
  protected:                            # environments / services / tasks / schedules / access
    unchanged: true                     # 変更がある場合は false にし、理由を添える
    changes_with_reason: []
  verdict:
    data_loss: none                     # 欠落がある場合はその詳細
```

## 11. 状態スナップショットの形式（F19）

`runtime/snapshot.json`。必須キー： generated_at、install_root、house（name / display_name / persona_resolution）、spec（current / version_no / state）、projects（id / name / status / assignee / schedules の要約 / **chats**）、runtime（slots / queue の件数要約）、service_health（サービスごとの last_ok と鮮度判定）、doctor（直近の checked_at / healthy / findings 件数）。認証情報の値は設計上存在しないため、スナップショットに秘匿項目は無い。

projects[].chats はプロジェクト⇄チャットの対応表示（GUI の対応表／チャットマップ台帳）のための読みモデルで、各要素は chat_id / name / role / state（インスタンスは open / closed、その他は省略可）/ instance_date（任意）/ last_activity（台帳等から引ける場合）を持つ。状態の意味は3層に分ける： プロジェクトの status（宣言）、インスタンスの state、最終活動。1つのラベルに潰さない。

## 12. ペルソナ解像度の伝播（F12 運用）

解像度の正は `house.yaml` の `persona_resolution`。各チャット／ランへは次の3経路で伝播する（完璧は求めない装飾層であり、推奨は零）：

- **pull（本命）**： エージェントは作業開始時（特に長期休止明け）に house.yaml の解像度と自プロジェクトの担当ペルソナの `speech_ending` を確認し、報告の装飾を合わせる。常駐ドキュメント（AGENTS.md 等）に規則を置く。
- **push（変更時）**： 解像度を変更したら、直近1週間アクティブなプロジェクトの最新チャットへ1回だけ通知する。送信記録は台帳 `SPEC/runtime/persona-sync.yaml`（chat_id / project / assignee / synced_resolution / synced_at / via / last_activity）に残す。
- **収束チェック（6時間毎）**： cron が台帳と現解像度を比べ、ズレていて直近1週間アクティブなチャットにだけ再 push する。定常時は無言で終了する。休眠プロジェクトは push 対象外（次回起動時の pull に任せる）。
- **適用範囲**： 装飾の対象は「エージェントがオーナーへ報告する文」だけ。キャラ本人の発話（運用対象のペルソナ自身の投稿・接客）と第三者向け文面（提示モード等の別規則があるもの）は対象外。
- **視覚要素（解像度が瞬息以上のときだけ有効）**：
  - **初登場の立ち絵**： そのチャットで担当ペルソナが装飾つきの報告を初めて行う時、立ち絵を1回だけ添付する。実施記録は台帳 `SPEC/runtime/persona-intros.yaml`（chat_id / persona_id / introduced_at / portrait）に残し、同じチャットでは二度出さない。立ち絵の所在はインスタンス側の `avatar-props.yaml` の `portrait_file` が指す。portrait_file が無いペルソナは対象外（用意できた者から有効になる）。
  - **表情アイコン**： 報告の要所に付ける小さな表情画像で、既存のリアクション機構の画像版として扱う。頻度は控えめに。表情セットの在庫が無いペルソナは使わない。
  - 解像度が瞬息未満（零／模糊／須臾）では視覚要素は一切出さない。push 通知で視覚段階が変わる場合は、その旨と立ち絵の有無も通知に含める。

## 13. ログ標準（F21）

- **置き場**： `<install_root>/Logs/<component>.log`。システム領域（/var/log 等）には置かない（アンカーによる移設可能性とインスタンス分離を保つため）。
- **分割単位**： ファイルは書き手コンポーネント（または拡張アプリ）ごとに分ける。コンポーネント名は既存の hook / cron / tool の ID を使う（例： spec-dispatcher、spec-doctor、service-health、house-report）。プロジェクト別には分けない： 代わりに各行へ識別子（project / task / service 等）をフィールドとして記録し、プロジェクト別の見方は機械的に抽出して作る。将来の拡張アプリは、同じ形式・同じローテーション規則で `Logs/<app-id>.log` を追加できる。
- **形式**： JSON Lines（1行1イベント）。必須フィールド： `ts`（ISO8601、タイムゾーンつき）、`level`（info / warn / error）、`component`、`event`。任意で `detail`（オブジェクト）。エラーイベントは種別に加えてエラーメッセージの抜粋（最大500字）を含めてよいが、認証情報・トークン等の値は記録禁止（F15 の規則を継承）。
- **ローテーション**： サイズ基準。1MB を超えたら世代交代し、保持は4世代。加えて90日を超えた世代は削除する。書き込みは共通ヘルパー（tools/log_event.py）経由に統一し、デーモンは置かない。
- **状態との分担**： runtime の yaml/json は「現在値」（状態）、Logs は「履歴とエラーの中身」。ドクターは findings の原文を Logs/spec-doctor.log へ残し、doctor-log.md は人間向け要約として併用する。
- **ダンプとの関係**： ログは F14 ダンプに含めない（§8）。保管はアーカイブ（世代ファイルの退避）で行い、リストアはしない。

## 14. 定期インスタンスチャット／プロジェクトスコープ・スキル（F22）

プロジェクト定義（§3 のプロジェクト YAML）への追加フィールド：

- `chat_template`： {name_pattern（例： `<プロジェクト名>_{yyyymmdd}`）、period（weekly 等）、anchor_weekday（基準日。例： sunday）、suffix_format（yyyymmdd）、create_when（基準日当日または準備開始時。当日作成なら当日の日付）、close_rule（期間終了の翌日に終了扱い＝アーカイブ。例： 月曜）}。テンプレートが無いプロジェクトは従来どおり固定チャットで運用する。
- `skills`： [{name、scope: project-only、path（ホスト環境のスキル実体への参照）}]。scope が project-only のスキルは、そのプロジェクトのチャット以外では使用しない。
- インスタンスチャットの記録： `chats` の各要素に role: instance と instance_date（基準日、YYYY-MM-DD）を持たせ、終了したインスタンスは state: closed として履歴に残す（行を消さない）。

## 15. ぱらぎストレージの構成（F23）

- **パス形式**： `<storage-root>/PaLaGial/<project-id>/<yyyymmdd_hhmmss>_<sidechat-name>/<自由>`
  - `PaLaGial`： 固定セグメント。綴りはパッケージ名と同一に統一する。
  - `<project-id>`： SPEC のプロジェクト ID（表示名ではない）。
  - `<yyyymmdd_hhmmss>`： その成果物バッチを格納した日時（ローカルタイム）。同一秒に複数バッチが衝突する場合は末尾に連番等を付けてよい。
  - `<sidechat-name>`： 成果物を出したサイドチャットの名前。バックエンドが禁止する文字（`/` 等）は `_` に置き換える。それ以外は原則そのまま使う。
  - 最下層（バッチフォルダの中身の構成）はプロジェクトごとに自由。仕様では固定しない。
- **バッチの定義**： 1つのサイドチャットが、1回の格納行為でまとめて置く成果物の集合を1バッチとする。
- **ストレージルートの解決順**： (1) インスタンスが事前に決めたルート（house.yaml の `storage.root`。バックエンドの種別も併記する）→ (2) デフォルト `workspace/Storage/`（インスタンスの workspace 配下）。
- **バックエンド**： 問わない。ルート配下の構成だけが共通仕様であり、ルート自体の置き場所（クラウドドライブ、NAS、ローカル等）はインスタンスの事情で決める。
- **遡及適用**： しない。既存の成果物の移行は各プロジェクトの判断に委ねる。

## 16. ぱらぎメモリ（F24）

プロジェクト定義（§3 のプロジェクト YAML）への追加フィールド：

- `code`： プロジェクト識別コード。プロジェクトID の sha256 先頭8桁（16進小文字）。ID から決定論的に再計算できるため、コード表そのものは正本を持たず、各プロジェクト定義への記録が正である。
- `memory_file`： そのプロジェクトのメモリファイルへのパス（1プロジェクト1ファイル）。global プロジェクトの memory_file はコアメモリ自身を指す。

メモリの層と規則：

- **コアメモリ**： 家全体で共有する最小限だけを持つ（当主の本人情報と呼び方、絶対ルール、全域の運用ルール、プロジェクト一覧＋各 memory_file へのポインタ）。プロジェクト固有の事実・経緯・人物詳細は持たない。
- **プロジェクトメモリ**： 各プロジェクトの学び・決定・経緯はそのプロジェクトの memory_file に書く。そのプロジェクトのチャットは作業開始時に自ファイルを読む（pull）。他プロジェクトのファイルは原則読まない（家の窓口と当主のメイン窓は横断のため例外）。
- **項目メタ行**： プロジェクトメモリの各項目は、見出しの直下に次の形式のメタ行を1行持つ。
  `<!-- pm: projs=<code>[,<code>...] [Shared] [Expire_YYYYMMDD] -->`
  - `projs`： その項目を所有するプロジェクトのコードを全てカンマ区切りで列挙する。項目の実体は、列挙された全プロジェクトの memory_file に同一内容で存在しなければならない。
  - `Shared`： projs が2件以上のときに付ける明示タグ（人間の目視用。機械判定は projs の件数で行う）。
  - `Expire_YYYYMMDD`： 有効期限。期限を過ぎた項目は削除候補であり、このタグ自体が削除の事前承認を兼ねる。監査（ドクター）が期限切れを info として検出し、所有プロジェクトのチャットが次回活動時に削除するか期限を更新する。
- **誤配置の再移動**： コアメモリ内にメタ行つきの項目があり、その projs に global のコードが含まれない場合、その項目は誤配置である。家の窓口は記載先プロジェクトの memory_file へ複製し、コアから削除する。メタ行のない項目の帰属は内容で判断する。
- **書き込み規則**： 新規の学びは所有プロジェクトの memory_file へメタ行つきで書く。コアへ追加できるのは家全体のルールだけであり、家の窓口または当主のメイン窓を経由する。
- **移行手順**： 導入は3段階で行う。(1) 仕様とコードの確定。(2) 全プロジェクト（休眠含む）への通達と、各プロジェクトによる自関係分の抽出（コアからは消さず複製し、メタ行を付ける）＋完了報告。(3) 全プロジェクト完了後、窓口が被覆表で未帰属の節が無いことを確認し、移行前コアを F23 ぱらぎストレージへバックアップ格納してから、コア以外を削除する。
- **対象外**： 人物・グループ等の別建て台帳ファイルは、初回移行の対象に含めない（その再編は F25／§17 で扱う）。ホスト環境が自動生成する要約類の注入は本仕様の制御外であり、コア削減に伴う変化を観察事項とする。

## 17. 人物・グループメモリの再編（F25）

台帳の置き場（人物フォルダ／グループフォルダの絶対パス）はインスタンスが定める。以下はその中での構成である。

人物ファイル（1人1ファイル）への追加フィールド（front matter）：

- `tags`： 関係性タグのリスト。語彙は固定で、本人／家族／親戚／関係者／リアル友達／ネット友達／リアル赤の他人／ツイッター知人／ツイッター友達／ツイッター他人／Facebook知人／Facebook友達／Facebook他人。複数可。未タグは移行完了時点では不可（監査の検出対象）。
- `personas`： 本人エントリのみ任意で持つ、ペルソナ（名前と説明）のリスト。ペルソナは独立した人物エントリにしない。

人物マップ登録簿（人物フォルダ直下の `map.yaml`）：

- 各エントリは `id`（人物ファイルの stem）、`file`（人物ファイルへのパス）、`tags`（front matter の tags と一致させる）、`projects`（その人物がキャッシュに入っているプロジェクトIDのリスト）、`updated`（正本の最終更新日）を持つ。
- 登録簿は人物ファイル群から機械的に再生成できること。食い違いがある場合は人物ファイル側（front matter）を正とする。

グループファイルの配置：

- 所有プロジェクトのフォルダ内に `groups/<group-id>.md` として置く。プロジェクト定義には任意で `groups` リスト（group-id の列挙）を追加し、所在を記録できる。
- 主所有が決まらないグループは台帳フォルダに残し、関係プロジェクトはポインタで参照する。

人物キャッシュ（プロジェクトフォルダ内の `people-cache/`）：

- 中身は人物ファイルの複製と `INDEX.md`（収録人物とタグの一覧）。複製の front matter には `cached_from`（正本のパス）と `cached_at`（複製した日時）を追加する。
- キャッシュは map.yaml の `projects` リストから機械的に生成する生成物であり、手編集しない。学びの書き込みは正本（人物ファイル）へ行い、キャッシュは再生成で追いつく。
- 本人（当主）のエントリは全プロジェクトのキャッシュに常時含める（map.yaml の projects リストに全プロジェクトが入っている状態を保つ）。

監査（ドクターの将来項目）：

- 未タグの人物、map.yaml に無い人物ファイル、front matter と map.yaml の tags 不一致、cached_at が正本 updated より古いキャッシュを info として検出する。

## 18. ブラウザプールの記録形式（F26）

- プロジェクトプール： そのプロジェクトの運用状態（ops state）の中に1節を持ち、少なくとも次を記録する： task_id（未登録ならその旨）、起動日時、最終利用。
- コモンプール： 家の実行時状態に1本持つ。形式の例：

```yaml
browser_pool:
  common:
    task_id: null        # 未登録なら null
    created_at: null
    last_used_at: null
```

- どちらのプールも「識別子は運用状態に保持し、ランをまたいで引き継ぐ」ことが要であり、保存場所の細部はインスタンスの流儀に従う。

## 19. お嬢ペルソナとキャラメイク（F27）

- house.yaml の `ojou` 節の形式：

```yaml
ojou:
  name: "<名前>"                 # キャラメイクで決定。名字は家名
  full_name: "<家名＋名前>"
  age: 11                        # 自動決定（下記の式）。オーナーの微調整可
  personality: "<軸集計から生成したプロフィール文>"
  quiz:
    mental_age: 30               # 正規化から復元した精神年齢 M
    params: { good_evil: 0.88, energy: 0.64, independence: 0.45, curiosity: 0.75 }  # 正規化パラメータ
    decided_at: "<日時>"
```

- パラメータと正規化： パラメータは mental_age（精神年齢）／good_evil（善悪）／energy（元気↔しっとり）／independence（甘えん坊↔しっかり者）／curiosity（好奇心↔慎重）。各質問の各選択肢は、任意のパラメータへの delta を持つ。抽出された10問のセットについて、パラメータ p 毎に次を計算する：
  - `min_p` = 10問それぞれの「p への delta の最小値（delta が無い選択肢は 0）」の合計。**負になり得る**。
  - `max_p` = 同じく最大値の合計。
  - `actual_p` = オーナーが選んだ選択肢の delta の合計。
  - 正規化値 `norm_p = (actual_p - min_p) / (max_p - min_p)`（∈ [0,1]）。これは各 delta に係数 `1 / (max_p - min_p)` を掛けて振れ幅を 1 にそろえるのと等価。`max_p = min_p`（振れ幅ゼロ）の場合は 0.5（中立）とする。
- 年齢マッピング（lerp）： 精神年齢 M は `M = 5 + 45 × norm_mental_age` で復元する。M が 50 以上（norm = 1 のとき）なら一律 18。M が 5 未満なら 5。それ以外は `5 + (M - 5) × (10 / 45)` を四捨五入する（M の定義域 5〜50 を、お嬢の年齢 5〜15 へ線形写像）。計算例： M=20 → 8歳、M=30 → 11歳、M=45 → 14歳、M=49 → 15歳、M=50 → 18歳（特則）。
- 性格： 正規化パラメータ（good_evil／energy／independence／curiosity）の値の傾向から、2〜3 文のプロフィールにする。
- `address_to_ojou` の `{ojou_name}` は、お嬢の name へ置換して使う。
- デフォルトプロジェクトの project yaml では、担当を `staff.assignee` ではなくお嬢を指す形で記録する（例： `persona: ojou`）。使用人のランダム割り当ての対象外である。
