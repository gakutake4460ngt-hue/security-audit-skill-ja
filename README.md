# security-audit

コーディングエージェントをセキュリティ監査担当に変えるスキル。互いに隔離したエージェントを指揮し、偵察、カバレッジ主導のハンティング、候補の検証、構造化出力、レコードの独立検証、対象に依存しないレポート作成までを進める。

このスキルは、[Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness) で紹介した Cloudflare の脆弱性発見ハーネスの元になったものである。ハーネスはその後、複数ステージからなるフリート全体のシステムへ成長した。このスキルは、その出発点となった単一リポジトリ向けの版である。

## 何をするか

このスキルは構造化された監査を 6 つのフェーズで実行する。

1. **偵察 (Reconnaissance)** -- アーキテクチャ、信頼境界、入力面、既存の証拠、決定的なカバレッジを `architecture.md` と `coverage-ledger.json` にまとめる。
2. **カバレッジ主導のハンティング** -- 台帳 (ledger) の単位ごとに隔離したハンターを割り当て、行ったチェックを記録させ、カバレッジ批評役 (coverage critic) で抜けを見つける。
3. **候補の検証** -- 一意な候補それぞれを、反証を試みる新しい検証役に渡す。
4. **構造化出力** -- `confirmed`、`needs_validation`、`rejected` のレコードを `findings.json` に書き、`report-schema.json` に照らして検証する。
5. **レコードの独立検証** -- 新しいエージェントが最終的なソース上の主張を検証する。実質的な差し替えがあった場合は、別の独立した検証役がもう一度検証する。
6. **対象に依存しないレポート作成** -- 検証済みのレコードとカバレッジ台帳から `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md` を作る。

親エージェントは、台帳を作った直後と、その後に台帳を更新するたびに `validate-coverage-ledger.cjs` を実行する。`validate-findings.cjs` はフェーズ 4 で実行し、フェーズ 5 で差し替えがあるたびに再度実行する。

判定はそれぞれ別物である。`confirmed` は完全なソーストレースと範囲の限定された観測結果を持つ。`needs_validation` は未解決の具体的な事実を 1 つ持ち、深刻度を持たない。`rejected` は反証された候補を記録する。

同じリポジトリに対する複数回の実行は積み上がる。スキルは以前の台帳と指摘を使って抜けを狙い、変更されたソースを再検証し、現行ソースに基づく証拠を引き継ぐ。古い作業や未解決の作業をカバー済みとして扱うことはない。

## ファイル

| ファイル | 目的 |
|------|---------|
| `SKILL.md` | セットアップ、基本原則、プラットフォーム用語、ワークフローの概要、監査のアンチパターン |
| `RECONNAISSANCE.md` | フェーズ 1 の偵察プロンプトと統合の手順 |
| `HUNTING.md` | フェーズ 2 の指揮、ハンティング手法、検証ルール |
| `ATTACK-CLASSES.md` | コア、ワイルドカード、「当たり前のこと」の攻撃プロンプト |
| `MEMORY-SAFETY-AND-BINARY.md` | ネイティブ対象向けのメモリ安全性、バイナリ、カーネルのハンティングクラス |
| `AI-AND-LLM.md` | LLM を使う対象向けのプロンプトインジェクション、エージェント/ツール、出力処理のハンティングクラス |
| `WEB-PROTOCOL-AND-AUTH.md` | HTTP プロトコルと認証の対象向けの HTTP リクエストフレーミング、キャッシュ、認証プロトコルのハンティングクラス |
| `CLIENT-SIDE.md` | クライアントサイド/ブラウザ対象向けの DOM インジェクション、メッセージングの信頼、UI リドレス、プロトタイプ汚染のハンティングクラス |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依存関係、CI、リリース、署名、アップデート、プラグイン、拡張機能のハンティングクラス |
| `CLOUD-AND-DEPLOYMENT.md` | IAM、Infrastructure as Code、コンテナ、サーバーレス、イングレス、ランタイム設定のハンティングクラス |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、シリアライズ、キュー、ブローカー、Webhook、ストリーミングプロトコルのハンティングクラス |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共有リソース、クォータ、キュー、ワーカー、運用者の費用負担のハンティングクラス |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | テナント分離、キャッシュ、検索、エクスポート、バックアップ、マイグレーション、削除、リストアのハンティングクラス |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | ネイティブアプリ、ディープリンク、WebView、エクスポートされたコンポーネント、ヘルパー、デーモン、ローカル IPC のハンティングクラス |
| `VALIDATION-AND-REPORTING.md` | フェーズ 3〜6 の候補検証、構造化出力、レコード検証、レポート作成 |
| `report-schema.json` | `findings.json` の 3 種類の判定すべてに対応する JSON スキーマ |
| `validate-findings.cjs` | フェーズ 4 と 5 で使う、依存ゼロの `findings.json` バリデータ |
| `validate-findings.test.cjs` | 指摘バリデータのテストと、生成側と互換性のあるフィクスチャのチェック |
| `validate-coverage-ledger.cjs` | フェーズ 1〜5 で使う、依存ゼロの `coverage-ledger.json` バリデータ |
| `validate-coverage-ledger.test.cjs` | カバレッジ台帳バリデータのテスト |

## インストール

[Skills CLI](https://skills.sh) でスキルをインストールする。

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit
```

ユーザー単位でインストールするには `--global` を付ける。

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global
```

エージェントの選択や非対話モードのオプションは `npx skills --help` で確認できる。

## 使い方

監査したいコードベースで (またはそこを指定して) コーディングエージェントを起動し、セキュリティ監査を依頼する。

```
security audit this codebase
```

```
find security vulnerabilities in ./src
```

```
do a security review, output to ~/audits/my-project
```

依頼がトリガー (セキュリティ監査、脆弱性の発見、コードのペンテストなど) に合うと、スキルは自動的に有効になる。コードベースの監査やペンテストを直接依頼した場合はフル監査モードになる。セキュリティに関する質問や対象を絞った脆弱性調査は、レポート成果物を求めない限りガイダンスモードになる。フル監査モードで出力ディレクトリを指定しない場合、既定は `~/security-audit-skill/<repo-name>/run-<N>` である。ワークフローが対象リポジトリ内に書き込むのは、バージョン管理で無視されるディレクトリを明示的に選んだときだけである。

## 必要なもの

- ツール使用と並列サブエージェントに対応したモデルを使うコーディングエージェント
- 依存ゼロの指摘バリデータとカバレッジ台帳バリデータを動かすための Node.js
- 対象が制御するビルド、テスト、プロセス、ブラウザ、エミュレータ、ファザー、フィクスチャのための、OS が強制するサンドボックス。外部ネットワークを無効にし、許可リストで浄化した環境を使い、リソース制限を強制し、割り当てられた scratch パスにだけ書き込みを許す必要がある。これらの制御がない場合、ワークフローは対象のコードを実行せず、その手がかりを `needs_validation` のまま残す。

## 設計原則

- **確立された境界の破綻だけを確定させる。** ソースに根拠があるが検証が止まった手がかりは、未解決の具体的な事実を添えて `needs_validation` として残す。
- **敵対的な検証。** 指摘をチェックするエージェントは、決してそれを見つけたエージェントではない。
- **深刻度には影響が必要。** 可能性 x 影響で決める。チェックリストからの逸脱では決めない。
- **多層防御の欠落は脆弱性ではない。** レイヤー A が攻撃を防いでいるなら、レイヤー B がないことは堅牢化のメモである。
- **複数回の実行でカバレッジが上がる。** 私たちのテストでは、1 回の実行で見つかったのは、繰り返し実行で見つかった脆弱性全体のおよそ半分だった。

## 連絡先

AI を使ったセキュリティツールについての質問、フィードバック、知見の共有: security-ai-research@cloudflare.com

## ライセンス

MIT -- [LICENSE](../LICENSE) を参照。
