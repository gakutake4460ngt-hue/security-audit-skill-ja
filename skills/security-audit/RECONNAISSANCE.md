# 偵察

### Phase 1: ソースをマッピングし、カバレッジを計画する

親は `run-metadata.json` を初期化し、`SKILL.md` の厳格な偵察前の予算ゲートを適用した後、探索の前にエージェントの scratch ルートと共有 ledger を作成する。ゲートを通過しなかった場合は、metadata に incomplete ステータスを記録し、偵察エージェントを起動しない。偵察で読むのは、ターゲットとローカルで利用可能なビルド/設定の状態だけである。デプロイ済みのエンドポイント、外部のアイデンティティプロバイダー、レジストリ、ブローカー、クラウド API、その他の共有サービスには接続しない。

複数の `research` エージェントを並列に起動する。これらは構造化された事実を親に返し、ファイルは書かない。

**Agent 1a: プロダクト、スタック、ローカルでの動作**

```text
Read the target at <target>. Do not use network access. Return:
1. Product type, users, operators, and ordinary trust-sensitive actions.
2. Languages, frameworks, build system, runtimes, and locally visible deployment models.
3. Repository-relative entry points and subsystem boundaries.
4. Exact build and test commands that could run offline with local dependencies, their expected write locations, and the target-controlled inputs they process. Do not run them during reconnaissance.
5. Comparable software or protocol visible from local documentation and dependencies. If no useful comparison is source-grounded, say so.
6. Missing local toolchains or runtime facts that limit bounded execution.
Return only source facts with repository-relative file:line references.
```

**Agent 1b: principal、権限、制御**

```text
Read all source that establishes identity, authorization, isolation, and privilege. Map:
1. Each lower-trust principal and the actions it has by design.
2. Authentication or peer identity at each entry surface.
3. Per-resource authorization and tenant/owner scope.
4. Process, browser, workload, CI, plugin, model/tool, device, or local-IPC authority.
5. Privilege changes, confirmation, revocation, recovery, and fallback paths.
6. Which controls are source-visible and which depend on an unobserved deployment fact.
Return trust boundaries and control locations with repository-relative file:line references. Do not infer live reachability.
```

**Agent 1c: エントリーサーフェス、コピー、シンク**

```text
Inventory every source-visible place external or lower-trust input enters:
- HTTP/browser, RPC/message/protocol, files/archive/document, CLI/env/config, plugins/dependencies/CI, cloud events/IAM selectors, model context/tool arguments, mobile/deep-link/webview, and local IPC.
For each surface, follow major transformations, stored or derived copies, and security-relevant sinks. Record source-visible limits and parallel paths to the same effect.
Return repository-relative paths and line numbers. Be complete, but do not execute or send inputs.
```

**Agent 1d: ローカル実行とデプロイ環境の可視性**

```text
Read tests, build definitions, manifests, packaging, and maintained environment overlays. Return:
1. Small offline tests or existing fixtures that could validate trust boundaries with dummy data inside the required OS-enforced sandbox.
2. Processes that could use an isolated loopback network namespace without external or shared dependencies.
3. Commands that would fetch dependencies, publish artifacts, contact paid/provider APIs, or affect shared state; mark them prohibited for this run.
4. Deployed controls and attachments that source cannot establish and therefore require needs_validation if decisive.
5. The final active source path for each deployment mode only where the repository selects it deterministically.
6. Whether the local platform can enforce an empty allowlisted environment, no external network, read-only target/toolchain mounts, scratch-only writes, and explicit CPU, memory, process, file-size, disk, and wall-clock limits. Missing controls block target-controlled execution.
7. Whether trusted parent-side code can promote predeclared scratch files with path-confined no-follow descriptor traversal, nonblocking regular-file checks, no-follow traversal of every destination parent, exclusive regular-file destination creation, and explicit per-file and cumulative size bounds. Missing promotion controls block use of scratch files as evidence.
```

これら 4 つでマッピングできない、実質的に異なるデプロイモードやサブシステムには、焦点を絞った偵察エージェントを追加する。黙って省略してはならない。`SKILL.md` の予算ゲートが焦点を絞ったエージェントを阻む場合は、そのエージェントは何も起動せず、マッピングされていない領域を reason `budget_cannot_reserve_critics_and_validation` 付きの `deferred` ledger unit として登録し、そのギャップをレポートで開示する。

## 過去の実行からの入力

作業を選ぶ前に、親は同じリポジトリについて利用可能な過去の `coverage-ledger.json` と `findings.json` をすべて読む。

- 過去のすべてのレコードと unit について、ソース位置、制御、条件、ソースから導かれたアイデンティティを現行のソースと比較する。
- 変更のない過去の `confirmed` レコードを同じ fingerprint のまま現行の候補セットに引き継ぐのは、関連するソース、条件、根拠となる証拠がまだ当てはまる場合に限る。それを `prior_status: "prior_confirmed_same_source"` 付きの現行の `planned` unit に紐づけ、その根本原因だけを hunter の除外リストに載せる。引き継いだレコードを再検証する Phase 3 の verifier がその unit の割り当て所有者となり、そのソース再チェックが unit の最初のチェックとなって、unit を引き継いだ fingerprint とともに `candidate` へ移す。
- 関連するソースや条件が変わった場合は、現行の planned な `prior_confirmed_changed_source` 再検証 unit を作る。その根本原因を探索から除外したり、過去の判定がまだ当てはまると仮定したりしてはならない。
- 過去の `needs_validation`、`deferred`、`blocked`、`out_of_scope`、およびソースが変わった unit のすべてについて、現行の作業 unit を作る。これらの状態は優先度の入力であり、重複排除や抑制のキーでは決してない。
- まだブロックされている過去の `needs_validation` レコードを同じ fingerprint のまま引き継ぐのは、現行のソースがその trace を裏付けた後に限る。それを `prior_status: "prior_needs_validation"` 付きの現行の `planned` unit に紐づける。レコードは未解決の blocker を保持する。引き継いだレコードを再チェックする Phase 3 の verifier がその unit の割り当て所有者となり、その再チェックが unit の最初のチェックとなって、unit を引き継いだ fingerprint とともに `candidate` へ移す。このレコードを最終確認に含める。
- 過去の rejected レコードは、現行の証拠が失敗した trace や欠けていた条件を変えない限り、古い主張として扱う。変更のない rejection が抑制するのはその主張そのものだけであり、coverage unit のレビューは抑制しない。
- 欠けている、または互換性のない ledger は、空のカバレッジとして扱わずに記録する。

使用したパスとソース ref は `run-metadata.json` に記述する。`architecture.md` にはカバレッジへの影響だけを要約する。

## アーキテクチャ要約とコンパニオンの選択

親は `<output-dir>/architecture.md` を、約 1,000 語を厳格な上限としてまとめる。含めるものは次のとおり。

1. プロダクト、principal、通常の権限、保護対象のリソース。
2. Agent 1a による比較対象ソフトウェアのベースライン (ソースに根拠がある場合): 比較対象がどのようなセキュリティ上のトレードオフを受け入れているか。これは労力と severity の調整に使い、実証された finding を退ける根拠には決して使わない。比較対象が、実際に問題となった欠陥パターンを共有しているなら、それは finding を強める。意味のある比較対象がなければこの項目は省く。
3. 技術スタック、ソース上で見えるデプロイ経路、オフラインでのビルド/テストの制約。
4. エントリーサーフェスと、重要な source-to-sink またはライフサイクルの経路。
5. 信頼境界と、それぞれにおけるソース上で見える最も強い制御。
6. リポジトリ相対の起点パス。
7. 過去のカバレッジのギャップ、ソース変更およびブロックされた再検証の対象、同一ソースの確認済み除外。
8. [ATTACK-CLASSES.md](ATTACK-CLASSES.md) から導いた短いコンパニオン選択の要約: 選択したファイルと、それを必要とするソース上で見える境界。

割り当てレベルの通常ブロック、選択したコンパニオンブロック、理由付きの除外ブロックは、`architecture.md` ではなく各 ledger unit に保持する。これにより、大規模な実行でもアーキテクチャの上限を守れ、hunter プロンプトの正確な対応関係を機械的に検証できる。

言語名や依存関係名が出現するというだけでコンパニオンファイルを選択してはならない。偵察で、そのファイルの `When to use this file` セクションに記述された信頼に敏感な境界が見つかったから選択するのである。別のエージェントが関連するクラスをレビューするという理由だけで、目に見える境界を除外してはならない。

## 決定的な coverage ledger

親は `<output-dir>/coverage-ledger.json` をトップレベルの JSON 配列として書く。エントリーサーフェス、信頼境界、サブシステム、適用可能な通常またはコンパニオンの攻撃クラスの実質的な組み合わせごとに、実行プロファイルが定める粒度で 1 つの unit を導出する (`quick` はスコープ内のすべてを 1 つにまとめたサブシステムのアイデンティティを使い、`deep` はライフサイクルモードを加える)。スコープ限定の実行では、スコープ内のサーフェスを割り当て用に登録し、発見したスコープ外のサーフェスは `out_of_scope` unit として保持して、後のフル実行がそれを現行の作業にできるようにする。

各次元は、人間向けのラベルと、`canonical_refs` 内の安定したソース由来の値を持つ。同じソースオブジェクトには、表示ラベルが変わっても、実行をまたいで同じ正規参照を使う。適切な参照には、リポジトリ相対のエントリーパスとエクスポートされたスコープ、ソースで定義されたルートやメッセージのアイデンティティ、境界を定義するソース上の制御、リポジトリのパッケージパス、正確な攻撃クラスのブロック参照がある。ブロック参照は `FILE.md#` に、そのファイル内で太字または見出しとして書かれたとおりの正確なクラス名を付けたものである。これはファイルのテキストと照合される安定した識別子であり、レンダリングされた HTML アンカーではない。コンパニオンのセクションブロックには、括弧書きの修飾語より前の見出しテキストを使う (例: `Core discipline`)。表示ラベルを小文字化したりスラッグ化したりして参照を導出してはならない。

`coverage_id` は、情報が失われるスラッグを使わずに導出する。

1. すべての参照について、Unicode NFC で有効なスカラー値から成り、可視の内容を持ち、制御文字、書式文字、行/段落区切り、default-ignorable なコードポイントを含まず、前後に空白がないことを要求する。
2. その UTF-8 バイト列を RFC 3986 のパーセントエンコーディングでエンコードする: `A-Z a-z 0-9 - . _ ~` だけをエスケープせずに残し、その他のすべてのバイトには大文字の `%HH` を使う。
3. エンコードした `surface`、`boundary`、`subsystem`、`attack_class` の参照を `::` で連結する。`lifecycle` があれば、エンコードしたものを末尾に付け加える。

quick プロファイルの粗くまとめたサブシステム次元には、固定の正規値 `profile/quick/all-in-scope-subsystems` を使う。参照や ID に、ウェーブ番号、エージェント、判定、severity、行番号を含めてはならない。各割り当ての前に unit を `coverage_id` の辞書順でソートする。重複する ID があればすべて失敗とする。重複する ID が異なる意味フィールドを持つ場合は、正規アイデンティティの衝突として扱い、決してマージしたり黙って上書きしたりしない。validator は、1 つの意味的タプルが異なる正規参照で表されている場合も拒否する。

各 unit は次を記録する。

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/router.ts#POST /users/:id",
    "boundary": "src/authz.ts#requireOwner",
    "subsystem": "packages/api",
    "attack_class": "ATTACK-CLASSES.md#Access control"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["repo/relative/path"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Access control",
  "selected_companion_blocks": ["FILE.md#section"],
  "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
  "prior_status": "new|prior_confirmed_same_source|prior_confirmed_changed_source|prior_needs_validation|prior_deferred|prior_blocked|prior_out_of_scope|prior_covered_same_source|prior_covered_changed_source|prior_rejected_claim_changed|none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

`lifecycle` が実質的な場合は、`canonical_refs.lifecycle` と人間向けの `lifecycle` フィールドの両方を追加する。`ordinary_attack_class_block` が null になるのは、適用できる通常ブロックがない場合だけである。選択したコンパニオンのリストには、適用可能な各クラスに加えて、そのコンパニオンの `Core discipline`、`Universal moves`、`Validation rules` を含める。`excluded_blocks` には、検討したが選択しなかったすべてのブロックと、それを除外するソース上の事実を記録する。

親は管理用のフィールドを追加してよいが、上記の意味フィールドは安定に保つ。`prior_status` において、`new` は互換性のある過去の ledger が存在するときに今回の実行で初めて見つかったサーフェスを示し、`none` は互換性のある過去の ledger がないときに登録された unit を示す。過去の `deferred`、`blocked`、`out_of_scope`、およびソースが変わった unit は、現在スコープ内であれば現行の `planned` 作業として初期化する。過去に同一ソースで covered だった unit も現行の ledger に見える形で残す。ソースの変更、重要なライフサイクル経路、正確な矛盾を先に割り当て、その後 coverage critic を使ってもう一度パスが必要かを判断する。

`attempts` は、coverage critic が再オープンした、証拠を伴う割り当ての追記専用アーカイブである。再割り当ての前に、以前の unit の正確な `wave`、`status`、`agent_id`、`reviewed_paths`、`local_checks`、`result_fingerprints`、`unresolved` を、critic によるソースに裏付けられた `reassignment_reason` とともに追記する。アーカイブできるのは `blocked`、`covered`、`candidate` の状態だけである。アーカイブされた attempt は、ライブの unit と同じ状態と証拠の不変条件を保持し、現在のウェーブより小さい厳密に増加するウェーブを使い、それを生み出した所有者と成果物を保持する。次の割り当てでは `wave` を増やし、新しい所有者を使い、ライブの証拠を空にして開始する。プロファイルや予算のために次の割り当てができない場合は、`wave` を増やし、所有者を null、証拠を空にし、停止理由を付けたライブの `deferred` 状態を使う。アーカイブされた所有者のチェックや成果物をライブの状態にコピーしてはならない。後のライブの終了状態には新しい attempt の証拠だけを含め、アーカイブは変更しない。

次の状態表を厳密に適用する。

| 状態 | unit の `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | null | 空 | 空 | 空 |
| `not_applicable`, `out_of_scope`, `deferred` | null | 空 | 空 | 空でない理由 |
| `in_progress` | 正規の所有者 | 空 | 空 | 空 |
| `blocked` | 正規の所有者 | 両方とも空でない、所有者自身の部分的な証拠 | 空 | 空でない blocker |
| `covered` | 正規の所有者 | 両方とも空でない | 空 | 空 |
| `candidate` | 正規の所有者 | 両方とも空でない | 空でない | 任意 |

正規のエージェント ID は `^[a-z0-9][a-z0-9_-]{0,63}$` に一致し、Windows のデバイス名ではない。小文字は必須なので、1 つの ledger に大文字小文字違いの別名が含まれることはない。unit の `agent_id` は割り当て所有者を記録する。各チェックは自身の `agent_id` と空でない `reviewed_paths` を記録し、unit レベルの `reviewed_paths` はそれらの和集合とちょうど等しい。ソースのみのチェックは `artifact: null` を使う。ローカルチェックには、信頼された親側のコードによってのみ、正確に `agents/<check.agent_id>/artifacts/` 配下へ昇格された通常ファイルが必要である。これにより、hunter と verifier のチェックが 1 つの unit 内に共存できる。scratch のパス、出力ルートのファイル、シンボリックリンク、特殊ファイル、別のチェック所有者の成果物は証拠にならない。

ledger こそがカバレッジの主張である。アーキテクチャ要約、エージェント数、「auth をレビューした」といった一般的な一文は、カバレッジの証拠ではない。Phase 2 は、hunter の構造化された結果に含まれるパスとチェックだけに基づいて unit を閉じる。

登録の後、親による更新のたび、そして Phase 6 の前に、`node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json` を実行する。validator は、5 MiB、ネスト 64 レベル、10,000 unit、ネストされたコレクション内の 1,000 エントリ、走査値 500,000 を超える入力を拒否し、報告する検証エラーを 100 件に制限する。実際には 5 MiB のバイト上限で現実的な unit がおよそ 2,000〜5,000 件収まるので、10,000 unit の上限より先にこちらに達する。作業を割り当てたりカバレッジを主張したりする前に、エラーをすべて修正する。
