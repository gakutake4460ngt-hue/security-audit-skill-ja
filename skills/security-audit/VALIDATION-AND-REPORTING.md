# 検証、構造化出力、最終確認、報告

### Phase 3: すべての候補を独立に検証する

クリーンな coverage-critic パス、または明示的に記録された早期停止の後、Phase 2 の候補と、引き継いだ同一ソースの過去の確認済み結果を、安定した fingerprint と根本原因で統合する。提案された一意の `confirmed` および `needs_validation` 候補はすべて、その候補を探索していない新しい `general` verifier に渡す。引き継いだ過去の確認済み結果も、hunter がその変更のない根本原因を除外していても、同じ現行の検証経路をたどる。verifier は hunter や過去の成果物を読んでもよいが、引用された現行のソース箇所をすべて読み直し、安全に再現できる決定的なチェックはすべて独立に実行しなければならない。

各 verifier には正規の小文字の一意な ID と、`<output-dir>/agents/<verifier-id>/scratch/` および親が所有する `artifacts/` を割り当てる。verifier は `scratch/` にのみ書き込み、保持される成果物には決して書き込まない。verifier が受け取るのは、候補、それに紐づく coverage unit のチェックと成果物のパス、経路の解釈に必要なアーキテクチャ上の事実、関連するコンパニオンの検証ブロックそのもの、後述の promotion procedure ブロック、ソース/ローカル実行の境界、`report-schema.json` の `confirmed`・`needs_validation`・`rejected` 分岐を逐語的にコピーしたもの、同じ fingerprint を持つ過去のレコードのみである。他の verifier の結論を受け取ってはならない。

#### 候補 verifier 用プロンプト

```text
You did not write this candidate. Try to refute it from repository source and bounded
local evidence. Do not contact deployed endpoints or external/shared services. Run
target-controlled code only inside the approved OS-enforced sandbox: no external
network, empty allowlisted environment, read-only target and tools, scratch-only
writes, and explicit low resource and wall-clock limits. If any control is unavailable,
do not execute; retain the exact missing capability as a needs_validation blocker.
Treat every scratch entry as target-controlled after execution. After the sandbox and
all its processes terminate, only trusted parent-side code may promote a predeclared
scratch-relative file, following the promotion procedure block included verbatim in
this prompt. You and target code never write retained artifacts. If promotion is
unavailable or fails, do not use that file as evidence.

1. Verify every trace and evidence file, positive line number, scope, and description.
   Confirm the first entry is a real lower-trust entrypoint and the last is the
   claimed sink or boundary effect.
2. Reconstruct the strongest source-visible validation, identity, authorization,
   normalization, lifecycle, framework, and containment controls on the path.
   Where the architecture summary names a comparable baseline, note whether it
   shares the pattern — as calibration, never as grounds to dismiss.
3. For a proposed confirmed candidate, independently reproduce the minimum observed
   result when possible. Verify inputs, interface shape, conditions, and affected
   dummy principal/resource. Do not infer a stronger result or continue after it.
4. Verify that likelihood, impact, confidence, and the proposed source fix match only
   what the evidence establishes.
5. For a proposed needs_validation candidate, decide whether the blocker is genuinely
   outside source/local observation. If source refutes the trace, reject it. If the
   missing fact remains decisive, keep needs_validation and make the local and
   owner-observed plans exact and non-destructive.
6. Preserve the fingerprint for the same source-derived root cause across every state.

Return exactly one JSON object and no surrounding prose:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
where record exactly matches the decision's verdict branch of the schema included
in this prompt. A corrected record replaces the hunter's wording.
```

次の promotion procedure を、すべての候補 verifier 用プロンプトに逐語的にコピーする。

```text
Artifact promotion procedure (trusted parent-side code only):
Reference only for you: the parent performs these steps; you never perform them.

Before execution, the parent opens and retains trusted, non-inheritable directory
descriptors for the agent's scratch/ and artifacts/ roots, and records an allowlist
of expected scratch-relative artifact files plus explicit per-file and cumulative
byte limits. Never pass those descriptors to the agent or sandbox. After the sandbox
and all its processes terminate, trusted parent-side code promotes each allowlisted
file separately:

1. Validate the declared relative path: reject absolute, empty, `.`, `..`, or
   symlinked components.
2. Walk each parent component from the retained scratch-root descriptor with
   no-follow directory-relative operations; never reopen by path.
3. Open the leaf no-follow and nonblocking.
4. Verify with `fstat` that it is a regular file with link count exactly one and
   within the recorded per-file and cumulative byte limits.
5. Enforce those limits again while reading from that descriptor.
6. Copy exactly the verified size, repeat `fstat`, and reject a changed identity,
   type, link count, or size.
7. For the destination, walk every parent component from the retained
   artifacts-root descriptor with no-follow directory-relative operations; require
   each existing component to be a real directory, and create any missing directory
   exclusively before reopening and verifying it no-follow.
8. Create the leaf exclusively without following links, verify that the opened
   destination is a regular file with link count exactly one, and copy from the
   verified source descriptor without reopening either path.
9. Use equivalent race-safe APIs on non-POSIX systems.
10. Never recursively copy or glob scratch, extract an archive into artifacts, or
    open or promote a symlink, FIFO, socket, device, directory, hard-linked file,
    changing file, or file that exceeds its bound.
11. If any check is unavailable, cannot be enforced, or fails, discard the scratch
    entry; if it is decisive evidence, retain `needs_validation` with the exact
    promotion blocker.
```

verifier が `needs_validation` を `confirmed` に昇格できるのは、完全な経路と範囲を限った観測結果を独立に立証した後に限る。特定のデプロイ環境やランタイムの事実が不明なままなら、提案された確認を `needs_validation` に降格する。ソース、ローカルでの挙動、目に見える制御、意味のある影響の欠如、または成立し得ない前提条件によって主張が反証された場合は `rejected` とする。`needs_validation` は推測的なアイデアの置き場所では決してない。

親は、各 verifier が同じ fingerprint を返したことを確認する。ただし真に異なる根本原因を特定した場合は除く。修正をマージし、紐づくすべての coverage unit に判定を記録し、fingerprint ごとに最終レコードが 1 件だけになるようにする。不正な形式の、あるいは散文で包まれた verifier の結果は修復せずに破棄する。予算が許せば新しい verifier でその候補を再実行し、許さなければ不完全な実行に関するルールに従って、未検証の ledger 候補のまま残す。

verifier の証拠によって ledger のチェックを更新するときは、そのチェックの `agent_id` を verifier の正規 ID に設定し、空でないリポジトリ相対の `reviewed_paths` を列挙する。unit レベルの `reviewed_paths` は、全チェックの和集合と等しく保つ。ソースのみのレビューには `method: "source"` と `artifact: null` を使う。`method: "local"` は、信頼された親側のコードによって `agents/<check.agent_id>/artifacts/` 配下へ正常に昇格されたファイルがある場合にのみ使う。unit は元の割り当て所有者を保持するので、所有者の異なる hunter のチェックと verifier のチェックが共存できる。引き継いだ過去レコード用に作られた `planned` unit には過去の所有者がいない。それを再チェックする verifier が unit の割り当て所有者となり、その再チェックが unit の最初のチェックとなって、unit を引き継いだ fingerprint とともに `candidate` へ移す。

厳格な総エージェント予算ですべての候補をカバーできない場合は、実行ステータスを incomplete に設定し、`SKILL.md` の決定的な予算ルールに従う。未検証の候補は ledger にのみ残る。どの判定であれ `findings.json` には入らない。

### Phase 4: `findings.json` を書き、検証する

親は、独立に判定されたすべてのレコードを fingerprint 順にソートして `<output-dir>/findings.json` に書く。含めるものは次のとおり。

- `confirmed`: ソースに根拠を持つ脆弱性で、完全なローカル実行の証拠、条件、具体的な修正策、likelihood/impact/全体の severity、confidence を備えたもの。
- `needs_validation`: ソースに根拠を持つ候補で、未解決の正確な blocker と、適用可能なローカルまたはオーナー観測によるデプロイ環境の計画を少なくとも 1 つ備えたもの。
- `rejected`: ソースに根拠を持つが検証中に反証された候補。証拠が変わらないまま、将来の実行が根拠のない主張を繰り返さないよう保持する。

書く直前に `report-schema.json` を読む。このスキーマは `additionalProperties: false` を使うので、hunter のラッパーフィールドをレコードに持ち込んではならない。次の判定ごとの契約を区別して守る。

- `confirmed` レコードは `root_cause`、`intended_behavior`、`conditions`、`execution`、`remediation`、`severity`、`confidence` を使う。`claimed_root_cause`、`blockers`、`validation_plan`、`reason` を使ってはならない。`execution` はターゲットに依存せず、ターゲット固有のインターフェースを使う。該当するものとして、API/HTTP 入力、CLI 呼び出し、ライブラリ呼び出し、メッセージ、ファイルフィクスチャ、ブラウザ操作、レンダリングされたポリシー、ローカルハーネスがある。`observed_result` は空でなく、事実に即したものとする。
- `needs_validation` レコードは `claimed_root_cause`、`trace`、`evidence`、`blockers` と、空でない `validation_plan.local` または `validation_plan.deployment` フィールドを少なくとも 1 つ使う。両方を含めるのは、両方のコンテキストがそれぞれ別の事実を解決できる場合に限る。severity、execution、remediation、reason、確認済みの root cause を使ってはならない。
- `rejected` レコードは `claimed_root_cause`、`trace`、`evidence`、`reason` を使う。severity、execution、remediation、blockers、validation plan、確認済みの root cause を使ってはならない。

すべてのレコードは、安定した fingerprint、title、description、リポジトリ相対のソースパスを持つ。複数ステップの trace は `entrypoint` で始まり `sink` で終わり、`propagation` はその間でのみ使う。1 エントリの trace は `entrypoint` か `sink` を使う。全体の severity は実証された impact を超えてはならない。

次を実行する。

```sh
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

先に進む前に、構造上および意味上のエラーをすべて修正する。findings validator は 5 MiB、トップレベルの finding 1,000 件、ネスト 64 レベルを超える入力を拒否し、報告するエラー出力を 100 メッセージに制限する。validator の成功が証明するのは、形式と ledger の整合性だけである。

### Phase 5: 最終レコードを新しい目で確認する

最終的な `confirmed` および `needs_validation` レコードごとに、新しい `research` verifier を 1 つずつ並列に起動する。この verifier は hunter の記述ではなく構造化レコードを確認し、ソース/ローカルの境界内にとどまる。

`quick` 実行では Phase 3 と Phase 5 を統合する。Phase 3 の verifier がこれらのレコードチェックも行い、スキーマ形式の最終レコードを返すので、各候補は 2 人ではなく 1 人の新しい独立したレビュアーを得る。その他のプロファイルでは 2 つのパスを分けたままにする。どのプロファイルでも、`confirmed` レコードの独立レビューを決して省略しない。

`confirmed` については、次を確認させる。

1. リポジトリ相対の trace/evidence のパス、行、スコープ、記述された操作のすべて。
2. 実際のエントリーインターフェースと、正確なローカル入力の形。
3. すべての条件、パーサー/ポリシーの各ステップ、ソース上で見える防御層、ローカルで観測された結果。
4. 影響を受ける principal/resource と、実証された impact。
5. severity の分離: 現実的な likelihood、実証された impact、全体は impact 以下であること。
6. 修正方針と `code_changes` があればその内容。修正が単に信頼を移すだけでなく、不変条件を強制しているかを含む。

`needs_validation` については、次を確認させる。

1. ソースの経路が実在し、記述された `claimed_root_cause` だけを裏付けていること。
2. 列挙された各 blocker が決定的であり、ローカルですでに答えられるものではないこと。
3. 候補が、一般的な懸念ではなく、境界と起こり得る具体的な結果を示していること。
4. validation-plan のフィールドが少なくとも 1 つあり、正確であること。`local` は範囲を限ったフィクスチャを使い、`deployment` はオーナーに設定、アイデンティティ、ルート、ポリシー、ランタイムの事実の観測を依頼する。該当しないコンテキストのために計画をでっち上げてはならず、デプロイ環境に監査トラフィックを送ってはならない。
5. fingerprint が、同じ根本原因に対する過去/現行のレコードと一致していること。

各 verifier は、`{"decision":"verified","fingerprint":"..."}` または `{"decision":"replace","reason":"...","record":{...}}` のいずれか 1 つの JSON オブジェクトだけを、前後に散文を付けずに返す。置換レコードは、その `confirmed`、`needs_validation`、`rejected` のスキーマ分岐に一致しなければならない。不正な形式の、あるいは散文で包まれた Phase 5 の結果は Phase 3 と同様に扱う。修復せずに破棄し、予算が許せば新しい verifier で再実行する。

Phase 5 の置換が、レコードをより強い判定に昇格させる場合 (`confirmed` への昇格を含む)、または根本原因、trace、実行入力や観測結果、実証された impact、severity を実質的に変更する場合は、その置換を最終として適用してはならない。その完全な置換を、探索も Phase 3 の検証も Phase 5 の置換提案も行っていない新しい独立した verifier に渡す。新しい verifier は現行のソースを再確認し、決定的なローカル結果を実行境界の下で独立に再現した上で、`verified` または別の置換を返す。実質的な置換は、この新たな検証の後にのみ適用する。さらに実質的な置換が生じた場合は、新しい verifier で繰り返す。予算や独立性が確保できない場合は、争点のあるレコードを `findings.json` から除き、その ledger unit を未解決の候補として残し、正確な `incomplete_reason` とともに `run_status: "incomplete"` を設定する。意味や証拠を変えない、実質的でない文言やリポジトリ行の修正は直接適用してよい。

置換を適用するたびに両方の validator を再実行し、紐づく ledger の判定を更新する。最終 verifier が別の根本原因を特定した場合は、新しい fingerprint を割り当て、含める前に独立した候補検証にかける。`run_status: "complete"` を設定するのは、すべての ledger 候補が独立した最終処分を持ち、保持されるすべてのレコードが Phase 5 を通過した場合に限る。

`confirmed` レコードだけを確認してはならない。誤解を招く `needs_validation` の引き渡しはオーナーの時間を浪費し、誤った前提を温存しかねない。

### Phase 6: 最終レコードからターゲット非依存のレポートを作成する

`findings.json` に保持されたすべてのレコードが Phase 5 を通過した後にのみ、最終レコード、ledger、ledger の管理情報に保持された hunter の `hardening` メモから散文を導出する。不完全な実行でも独立に検証されたレコードは報告してよいが、未解決の ledger 候補をそれぞれ明示し、それを finding として提示してはならない。散文ファイルが判定、severity、blocker、実証された impact を変えることは決してない。

#### `REPORT.md`

次を書く。

1. 実行プロファイル、スコープ、予算 (設定されていれば) と計画に対して消費したエージェント数、ソース ref、サンドボックス内でのソースとローカルのみの実行である旨の記述、過去の実行の利用、明示的な deferred および out-of-scope のカバレッジ。引き継いだ同一ソースの確認済み結果と、ソース変更による再検証を明記する。`quick`、スコープ限定、予算制限付き、または不完全な実行は、部分的なパスであることを率直に述べる。候補検証で厳格な予算を使い切った場合は、実行が不完全であることを述べ、未検証の fingerprint と紐づく unit をすべて列挙する。それらの候補を finding として記述してはならない。予算のために必須の critic が実行できなかった場合は、どの critic が実行されなかったかを述べ、クリーンなカバレッジを主張しない。
2. 短いセキュリティ態勢の要約 1 つ。
3. 確認済み finding の表: severity、title、影響を受ける境界、1 行の観測結果。
4. 各確認済み finding: リポジトリのソース位置、低信頼の principal、ターゲット固有の範囲を限った再現手順、条件、実際の結果、impact、優先度の根拠、最小のソース修正。
5. 別個の `NEEDS VALIDATION` 表。各リードの title、リポジトリの trace、正確な blocker、範囲を限ったローカルでの次の手順、安全なオーナー観測によるデプロイ環境チェックを示す。severity を付けたり、確認済みの脆弱性と呼んだりしてはならない。
6. 別個の hardening メモと、良好なソースのパターン。
7. ledger に基づくカバレッジの要約: covered、candidate、blocked、deferred の件数、重要な除外事項、最終的な critic の結果。

rejected レコードを finding として記述してはならない。その fingerprint に触れるのは、過去の不一致やカバレッジの判断を説明する場合だけにする。

#### `FINDINGS-DETAIL.md`

確認済みの `medium`、`high`、`critical` レコードそれぞれについて、完全なソースの経路とターゲット非依存のローカル再現手順をコピーする。

- 順序付きのリポジトリ相対の trace と evidence。
- ダミーの攻撃者/principal と、影響を受けるダミーのリソース。
- 固有の入力、呼び出し、またはフィクスチャと、範囲を限った正確な手順。
- 観測された出力と、それが証明するセキュリティ不変条件。
- 条件と封じ込め。
- ソースレベルの修正策と回帰ケース。

#### `NEEDS-VALIDATION.md`

未解決のレコードごとに、ソースの trace、検証済みの evidence、正確な blocker、影響を受ける境界、適用可能な範囲を限ったローカルまたはオーナー観測による解決計画をそれぞれコピーする。これらは severity を付けない優先順位付きのリードとして扱う。ライブテストの手引きに変えたり、欠けているデプロイ環境の事実を仮定したりしてはならない。

HTTP は固有のインターフェースの 1 つにすぎず、デフォルトではない。ライブラリの finding は関数呼び出し、パーサーはフィクスチャ、CLI はコマンド、デスクトップアプリは IPC やファイル操作、インフラはローカルでレンダリングしたポリシーを使ってよい。ターゲットが持たないエンドポイント、外部アカウント、ライブ環境を要求してはならない。

レポートは証拠に見合った分量に保つ。クリーンな実行では確認済みレコードが 0 件でもよい。その結果と、残るカバレッジ/検証の限界を述べ、LOW の finding をでっち上げてはならない。
