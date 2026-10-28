# 脆弱性ハンティング

### Phase 2: カバレッジ主導のハンティングウェーブを実行する

Parent は `planned` の台帳単位を `general` エージェントに割り当てる。関連のない境界をまとめずに単位をカバーできるだけの、焦点を絞ったハンターを使う。1 つのハンターが 1 つのサブシステム内の密接に関連する単位を受け持ってもよい。エージェント数の上限を理由に、どの単位も黙って未割り当てにしてはならない。予算で届かない単位は、理由 `budget_cannot_reserve_critics_and_validation` とともに明示的に `deferred` にする。

予算やプロファイルでハンター数に上限がある場合は、優先順位の順に単位を割り当て、その順序付けの根拠を台帳に記録する。順位付けは次のとおりである。(1) 未認証または最も信頼の低い入口サーフェスを、認証済みのものより先にする。(2) 最も価値の高いリソース (認証情報、テナント間のデータ、コード実行、リリース権限) を守る境界。(3) 過去の実行のギャップ、再検証の対象、変更されたソースを、同一ソースの再パスより先にする。(4) このターゲット種別で確定した指摘を歴史的に生んできたクラスの単位を、推測的なものより先にする。同順位は `coverage_id` の辞書順で決め、実行が決定的になるようにする。

起動前に、Parent は割り当てた単位を `in_progress` に変更し、正規の小文字の `agent_id` を設定し、そのエージェントの `scratch/` と Parent 所有の `artifacts/` を作成する。ハンターはソースと Parent が提供したコンテキストを読み、固有の `scratch/` にのみ書き込み、Task tool を通じて構造化された結果を 1 つ返す。保持用の成果物を書き込んだり、ターゲットのソース、`architecture.md`、`coverage-ledger.json`、`findings.json`、他のエージェントのファイルを編集したりすることは決してない。

## 必須のハンタープロンプト

すべてのハンタープロンプトは、次の部分をこの順序で含む。

1. 2 文の役割の前置き: ハンターの目的は、割り当てられた単位において、ソースに根拠のあるセキュリティ不変条件の破綻を見つけることであり、このプロンプトの末尾にある構造化結果の契約に一致する JSON オブジェクトをちょうど 1 つ返さなければならない。
2. `architecture.md` をそのまま。
3. 割り当てられたカバレッジ ID、サブシステム、境界、リポジトリ相対の開始パス、および `coverage-ledger.json` にある各単位の割り当てブロックマップ。
4. 選択した正確なブロックをそのままコピーしたもの: `ATTACK-CLASSES.md` から選択した各通常の攻撃クラスブロック、および選択した各コンパニオンからその `Core discipline`、選んだ各攻撃クラスの小節、`Universal moves`、`Validation rules`。通常のブロックは自己完結しており、コンパニオン形式の `Core discipline`、`Universal moves`、`Validation rules` の節は持たない。ブロック名やコンパニオン名だけを送らない。
5. 除外した通常ブロックとコンパニオンブロックを明示し、それぞれに除外の理由を付ける。
6. 以下の基本ハンティング手法と、それに続く昇格手順ブロック。
7. 以下の基本検証規則。
8. 引き継いだ同一ソースの過去の confirmed の除外項目 (それぞれフィンガープリント、タイトル、根本原因に限る)、およびこのハンターが重複してはならない、他のハンターが所有する現在のカバレッジ ID。
9. 固有の scratch/artifact パス、安全なエージェント ID、事前宣言した昇格の許可リストとバイト上限、および構造化結果の契約。契約には、以下の「構造化されたハンター結果」ブロックと、`report-schema.json` の `confirmed` および `needs_validation` の分岐をそのままコピーしたものを含める。

同じ経路が複数のドメインにまたがる場合、プロンプトは複数のコンパニオンブロックを選択してよい。それらの制約はまとめておく。スコープはハンターのカバレッジ上の義務であり、除外された作業を重複してよいという許可ではない。想定外の別の境界が現れた場合は、それを `uncovered` に入れて返し、Parent が安定した台帳単位を作成して次のウェーブで割り当てられるようにする。

#### 基本ハンティング手法 — すべてのハンタープロンプトに含める

```text
## Defensive vulnerability-finding method

Your goal is to find source-grounded security invariant failures and the smallest fix,
not to expand harm beyond the boundary result. Stay within source review and bounded local execution.
Do not contact deployed endpoints, provider APIs, registries, identity systems,
message brokers, shared services, or other users. Use local dummy data only.

READ THE CODE AT DEPTH. Follow each assigned input through parsing, identity,
authorization, normalization, state, derived copies, and the final sink. Read sibling,
legacy, batch, retry, cancellation, migration, and error paths that produce the same
effect. Compare sibling controls for equivalence, not only presence, and compare what
one component guarantees with what the next component assumes.

WORK FROM A CONCRETE INVARIANT:
1. Name the lower-trust principal and starting capability.
2. Name the accepted value, action, state transition, or resource selector.
3. Locate the control that should reject, bind, isolate, limit, or revoke it.
4. Trace the exact source path after that decision.
5. Stop at the smallest affected dummy record, wrong return value, process-integrity
   effect, or locally observable shared-resource effect.
6. State a source-level change and regression case that enforce the invariant.

DEPTH BOUND: trace only paths that can reach your assigned boundary or whose
guarantees that boundary relies on. Stop a line of investigation as soon as the
invariant is settled either way, and record the result in your structured output —
a covered, candidate, or blocked disposition, or an `uncovered` entry — instead of
continuing to search.

TEST SAD PATHS AND DISAGREEMENTS. Check absent, empty, zero, negative, maximum,
over-limit, duplicate, mixed encoding, stale, revoked, reordered, concurrent,
partially migrated, failed dependency, and rollback state only where the interface
accepts them. Compare canonicalization and units at every parser or policy handoff.
For multi-step issues, treat each output as a prerequisite and do not assume a later
boundary. If any prerequisite is not established, record a blocker.

When a proposed high or critical candidate reveals a reusable root cause, search paths
owned by the assigned coverage IDs for lexical, structural, and logical variants.
Consolidate the same root cause, but establish each variant's conditions and impact
independently. Do not investigate peer-owned units. Return a variant with no current
coverage unit as `uncovered`.

USE THE NARROWEST LOCAL CHECK THAT SETTLES THE CLAIM. Target-controlled builds,
tests, processes, browsers, emulators, fuzzers, and fixture processing may run only
inside the parent-approved OS-enforced sandbox. It must disable external networking,
start from an empty allowlisted environment, expose target and tools read-only, permit
writes only to your scratch directory, and apply low CPU, memory, process, file-size,
disk, and wall-clock limits. Isolated loopback is allowed only for a local fixture.
If any control is unavailable, do not execute: return needs_validation with that exact
blocker. Prefer an existing unit test, minimal function harness, dummy-tenant service
call, small malformed fixture, deterministic race schedule, or locally rendered policy.
Do not install or fetch tools.

Record the exact input, command, limits, and minimum result. For the environment,
record only allowlisted variable names and safe non-secret values needed to reproduce
the check. Never capture the ambient environment, inherited variables, credentials,
authentication state, or unrelated host paths. The target-controlled process writes
only in scratch. After the sandbox and all its processes terminate, only trusted
parent-side code may promote predeclared scratch-relative files, following the
promotion procedure block included verbatim in this prompt. You and target code never
write retained artifacts. If promotion is unavailable or fails for decisive evidence,
return needs_validation with the exact promotion blocker.
Never stress availability, invoke a live target, use a real credential, publish an
artifact, or continue past the minimum observed effect.

A deployment, browser, provider, broker, OS, proxy, package, secret, or identity fact
outside source is not proof either way. If one such fact is decisive, return a
needs_validation record with the exact missing observation and safe owner-observed check.
```

#### 昇格手順 — この昇格手順をすべてのハンタープロンプトにそのままコピーする

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

#### 基本検証規則 — すべてのハンタープロンプトに含める

```text
## Candidate gate

1. A candidate needs a complete repository-relative source trace and evidence for the
   claimed root cause, including the strongest source-visible control.
2. A proposed confirmed record needs a bounded local observed result, meaningful impact
   across a stated boundary, complete conditions, and no visible preventing layer.
3. Do not strengthen a crash into code execution, ordinary work into shared availability,
   or a same-principal action into privilege gain.
4. If a required fact is not source-visible or locally observable, use
   needs_validation. Name exact blockers; do not give it severity or speculative completion.
5. A missing best practice with no affected principal/resource is excluded or hardening,
   not a finding. A candidate disproved by source is not needs_validation.
6. Use the same source-derived fingerprint for the same root cause in every state.
   It must match `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` and must not include a line,
   wave, agent, severity, or verdict.
7. Return an empty candidate array when nothing survives these gates.
```

## ローカル検証の境界

ローカル実行は確認のためであり、影響を拡大するためではない。

- **必須の OS サンドボックス内でのみ許可:** 既存の依存関係を使うオフラインビルド、ダミーの状態を使う分離ループバックのプロセス、単体テストと統合テスト、小さなフィクスチャの処理、サニタイザー、範囲を限定したファジング/回帰テスト、決定的な並行性チェック、ダミーアカウントを使うローカルのブラウザ/エミュレータテスト、ダミーの ID を使う生成済みマニフェストとポリシー評価、外部呼び出しや有料呼び出しのモック。
- **禁止:** 稼働中またはデプロイ済みの環境へのトラフィック、この分離チェックのために起動したのではないサービスへのリクエスト、ネットワーク経由の依存関係のインストール、実際のアカウントや認証情報、本番データ、共有のキュー、クラウドリソース、ランナー、レジストリ、署名サービスやリリースサービス、公開、ストレス、飽和、コストの発生、最小限のダミーデータによる境界上の結果を得た後のあらゆる作業。

サンドボックスは空の環境で開始し、ターゲットのコードに外部ネットワークもホスト上の書き込み可能なパスも与えず、高コストが見込まれるチェックだけでなくすべてのチェックに対して明示的で低いリソースと時間の上限を強制する。scratch の出力は終了後もターゲットが制御するものである。それを昇格するのは、`SKILL.md` にある、リンクをたどらず、パスを閉じ込め、通常ファイルに限り、サイズを限定するホスト側の手順でのみ行う。サンドボックスや昇格の機能が欠けていても、ソースに根拠のある候補は消えない。正確なブロッカーを `needs_validation` で表す。

## 構造化されたハンター結果

前後に文章を付けず、JSON オブジェクトをちょうど 1 つ返す。

```json
{
  "units": [
    {
      "coverage_id": "one assigned ID",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["repo/relative/path"],
      "checks": [
        {
          "agent_id": "canonical owner of this check",
          "reviewed_paths": ["repo/relative/path owned by this check"],
          "invariant": "specific control checked for this unit",
          "method": "source|local",
          "result": "what source or the bounded check established",
          "artifact": "agents/<agent-id>/artifacts/file for local, null for source"
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["concrete non-finding note"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "reason": "why this needs its own deterministic coverage unit"
    }
  ]
}
```

各 `candidates` エントリは、`verdict` の代わりに `proposed_verdict` を使う点を除いてスキーマに従った形である。

- `proposed_verdict: "confirmed"`: `report-schema.json` の `confirmed` 分岐が要求するフィールドのうち、`verdict` 以外のすべてを含める。`fingerprint`、title、description、`root_cause`、`intended_behavior`、順序付きの `trace`、`evidence`、`conditions`、ターゲット中立の `execution`、`remediation`、`severity`、`confidence` である。実行手順には、すでに実施した範囲を限定したローカルのチェックだけを記述する。`payloads` には最小限のテスト入力、フィクスチャ、またはネイティブの呼び出しを入れる。`observed_result` には実際のローカルの出力を記録する。全体の深刻度は観測された影響を超えてはならない。
- `proposed_verdict: "needs_validation"`: そのスキーマ分岐が要求するフィールドのうち、`verdict` 以外のすべてを含める。`fingerprint`、title、description、`claimed_root_cause`、順序付きの `trace`、`evidence`、空でない `blockers`、および該当する `local` または `deployment` のステップを少なくとも 1 つ含む `validation_plan` である。該当しないコンテキストをでっち上げない。severity、execution、remediation、reason、確定した `root_cause` は含めない。`deployment` はオーナーが観測するチェックであり、稼働中のターゲットを調べる依頼ではない。

割り当てられた各カバレッジ ID は `units` にちょうど 1 回現れる。`covered` の単位には、オーナー、空でない `reviewed_paths` と `checks` が必要で、未解決の事実も候補もあってはならない。`candidate` の単位は同じ所有された証拠を持ち、リンクされたフィンガープリントを持てる唯一の状態である。`blocked` の単位は所有された部分的なレビューであり、空でないパス、チェック、未解決の事実を持つが、フィンガープリントは持たない。ソースのパスはすべてリポジトリ相対であり、絶対パスやトラバーサルを含むパスであってはならない。複数のエントリを持つトレースは `entrypoint` で始まり、`sink` で終わり、途中のステップには `propagation` のラベルを付ける。各チェックは独自の正規の小文字の `agent_id` と空でない `reviewed_paths` を持つ。単位レベルのリストは、それらの所有パスの和集合とちょうど一致する。`source` のチェックは `artifact: null` を使う。`local` のチェックは、`agents/<check.agent_id>/artifacts/` 配下に Parent が昇格に成功した通常ファイルを 1 つ使う。これにより、検証者はハンターから所有権を奪うことなく、独立して所有する証拠を追加できる。scratch、出力ルートのファイル、他のチェックオーナーの成果物には決してリンクしない。

## Parent による統合と台帳の更新

Parent は各単位の結果を検証し、割り当てた `coverage_id` のちょうど 1 つに対応付け、その台帳単位だけを更新する。重複した ID や欠けた ID、安全でない単位またはチェックのエージェント ID、成果物を持つ source のチェック、信頼された Parent 側のコードがチェックオーナーの artifacts サブツリーに昇格していないローカルの成果物は拒否する。単位の `reviewed_paths`、その `checks` (単位の `local_checks` へ)、リンクされた成果物のパス、候補のフィンガープリント、未解決の事実を台帳にコピーする。各ハンターの `hardening` リストは、Phase 6 で報告できるように、関連する単位の Parent 用の管理フィールド (意味上のフィールドの外) に保持する。失敗した結論、形式が不正な結論、裏付けのない結論の場合、その単位は再割り当てのために `planned` のままにする。予算やプロファイルにより手が付けられなかった単位は、証拠が空で理由を持つ未割り当ての `deferred` 単位になる。部分的な証拠を `deferred` に隠さない。更新後に `validate-coverage-ledger.cjs` を実行する。不正な台帳で次の割り当てを行ってはならない。この単位ごとの契約により、1 つのハンターが 1 つの単位を完了させながら、別の単位について候補やブロッカーを返せる。

候補エントリはフィンガープリントで、次に根本原因で統合する。複数の入口経路や影響を露出する 1 つの根本原因は、最も強い完全なトレースを持つ 1 つの候補である。関連はあるが独立した制御の欠如には、別々のフィンガープリントを使う。重複したフィンガープリントは関連する台帳単位に記録し、重複した候補は検証に送らない。

## カバレッジ批評ウェーブ

各ハンターウェーブの直後に、確保しておいた起動枠を使って、新しい `research` のウェーブ後カバレッジ批評者を 1 つ起動する。批評者には `architecture.md`、各割り当てブロックマップを含む完全なカバレッジ台帳、現在の候補のフィンガープリントと状態、過去の台帳のギャップの要約を渡す。批評者はソースを読むが、書き込みやターゲットの実行は行わない。正確に次の JSON を要求する。

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "source-backed coverage gap"
    }
  ],
  "reassign_ids": ["existing-id-that-did-not-close"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

批評者は、対応付けられていない入口、チェックされていない並行経路、欠けているライフサイクルモード、単位のない選択済みコンパニオンクラス、正当化されていない除外、パスやチェックなしで閉じられた単位、どの単位も扱っていない過去の `needs_validation` やソース変更のギャップを確認する。批評者が提案するのはカバレッジであり、指摘ではない。`stop` は批評者自身の評価であり、受理される `missing_units` も `reassign_ids` もない場合にだけ `true` とする。別のウェーブを実行するかどうかを決めるのは、`stop` 単独ではなく、以下の Parent のループ条件である。`resolved_prior_leads` の各フィンガープリントについて、Parent はリンクされた単位または過去のリードのエントリを解決済みとし、批評者のソースに基づく理由を記録する。

Parent は、レビューのスコープまたはソース/ローカルの境界の外にある提案単位を拒否し、受理した単位の正規 ID を導出し、現在の単位と重複排除する。過去の同一ソースで完了した単位は証拠を提供してよい。過去の `deferred`、`blocked`、`out_of_scope`、またはソースが変更された単位は現在の作業となり、受理された単位を抑制することは決してない。正規 ID が衝突した場合はマージせずに失敗させる。稼働中の `blocked`、`covered`、`candidate` の証拠を持つ正当な各 `reassign_id` について、その終端レコードをそのまま単位の `attempts` に追記し、批評者のソースに基づく `reassignment_reason` を付ける。そのオーナー、チェック、成果物、フィンガープリント、未解決の事実はそのアーカイブ内に保存する。稼働中の `wave` を 1 増やす。次のハンターは新しいオーナーでなければならず、稼働中の証拠が空の `in_progress` 単位を受け取る。ハンターの終端結果は、新しい証拠だけを稼働中のフィールドに書き込む。アーカイブされたオーナーのチェックや成果物を新しい稼働中の試行に決してコピーしない。次の割り当ての前に ID をソートし、台帳を検証する。`standard` と `deep` では、ウェーブ後批評者が受理される `missing_units` も正当な `reassign_ids` も報告せず、`planned` の単位が残っていない場合に、別に確保しておいた起動枠を使って別個の最終クリーン批評者を起動する。その批評者も受理される作業を返さなかった場合にだけ、カバレッジを完了とする。作業が見つかった場合は、それをキューに入れ、ウェーブ、ウェーブ後批評者、最終クリーンの処理を繰り返す。時間やリソースのために早期停止せざるを得ない場合は、手が付けられていないすべての単位を `deferred` にし、批評者の理由を保持し、そのギャップをレポートで開示する。ウェーブやエージェントの暗黙の上限を、完全なカバレッジの証拠として決して使わない。

実行プロファイルがこのループを制限する。`quick` の実行では、ハンターウェーブをちょうど 1 回実行した後に、最終批評パスをちょうど 1 回実行する。受理された各 `missing_unit` を現在の台帳に追加し、理由 `quick_profile_final_critic` とともに `deferred` にする。証拠を持つ正当な各 `reassign_id` について、稼働中の終端状態を `attempts` にアーカイブし、`wave` を 1 増やし、稼働中の状態を、証拠が空で理由が `quick_profile_final_critic` の未割り当ての `deferred` にする。2 回目のハンターウェーブや別の批評者は起動しない。スコープ限定の実行でも、批評者は気付いたスコープ外のギャップを報告するが、Parent はそれを割り当てる代わりに、批評者の理由とともに `out_of_scope` として記録する。上記の早期停止の規則は同じ仕組みである。`quick` は事前に宣言された早期停止であり、完全なカバレッジの証拠ではない。

予算も同じようにこのループを制限する。各ウェーブを割り当てる前に、残りの予算を、そのハンター数、検証の確保分、直後のウェーブ後批評者、残しておく最終クリーン批評者と比較する (`quick` が確保するのは、その 1 回の最終ウェーブ後批評者だけである)。優先順位の順に単位を取り、ハンターウェーブを収まるように縮小する。これらの必須の確保分が収まらない場合は、そのウェーブのハンターを起動せず、その計画済み単位を理由 `budget_cannot_reserve_critics_and_validation` とともに `deferred` にする。批評者が提案した単位は、予算を拡張するのではなく、同じ順位付けされたキューに入る。生き残った候補が検証の確保分を超える場合は、`SKILL.md` の未完了実行の規則に従う。ハンティングを停止し、フィンガープリント順に検証し、未検証の単位を未解決の候補として保持し、それらを指摘として決して提示しない。
