# クライアントサイドとブラウザのハンティング

#### このファイルを使う場面

意味のある信頼判断や信頼できないコンテンツのレンダリングがブラウザ内で行われる場合にこのファイルを使う。対象は、シングルページアプリ、ブラウザ拡張、埋め込み webview、Service Worker、オフラインアプリ、そして攻撃者が影響できるコンテンツを DOM にレンダリングするコード、ウィンドウ間メッセージを受け取るコード、ブラウザストレージを使うコードである。これらの経路には、URL フラグメント、`window.name`、`postMessage`、以前にキャッシュされたコンテンツなど、サーバーが一切見ないソースが含まれる。

`ATTACK-CLASSES.md` と併用する。このファイルは、ブラウザのソースとシンク、オリジン境界、ブラウザの永続化、クロスサイトの状態オラクルを扱う。webview ブリッジのネイティブ側には `DESKTOP-MOBILE-AND-LOCAL-IPC.md` を、サーバー側の CSRF、セッション、認証コールバックには `WEB-PROTOCOL-AND-AUTH.md` を使う。

## 基本規律 (この領域のすべてのエージェントプロンプトに含める)

```
- A client-side candidate needs a controllable source and an executing or disclosing sink. Name both and show attacker-influenced data reaching the sink.
- The impact must reach a victim's session, another origin, or shared persistence. Self-injection and disclosure of the attacker's own data are not findings.
- Framework escaping, browser same-origin policy, CSP, COOP/CORP, service-worker scope, and modern noopener defaults are real controls. Verify them before assigning impact.
- Browser storage and caches are shared by origin and may outlive login state. Identify who writes, who reads, and which account, tenant, or worker lifecycle clears each record.
- Use `confirmed` only for complete source evidence plus bounded local browser tests. Use `needs_validation` when renderer, extension permission, deployed header, or browser-policy behavior is required but unavailable.
```

## DOM とオブジェクト状態の攻撃クラス (subagent_type: `general`)

**DOM-based XSS**
`location` のフィールド、`document.referrer`、`window.name`、メッセージデータ、ストレージ、ブラウザが制御するドキュメント状態を、`innerHTML`、`outerHTML`、`document.write`、文字列を評価する API、実行可能な URL、jQuery の HTML API、フレームワークのエスケープハッチまで追跡する。フレームワークがエスケープする補間は指摘ではない。

**DOM clobbering**
攻撃者が注入した `id` や `name` 属性が、後でコードが信頼するグローバル、フォームプロパティ、設定オブジェクト、初期化フラグを覆い隠す。属性を保持するマークアップ経路と、上書きされた値のセキュリティ上重要な利用の両方を要件とする。

**プロトタイプ汚染とガジェットチェーン**
攻撃者が制御するキーが、ディープマージやパス代入のような再帰的書き込みに到達し、プロトタイプの状態を変更する。その後、到達可能なガジェットが汚染されたプロパティを消費し、認可、実行、ナビゲーション、レンダリングを変える。`JSON.parse`、浅いコピー、ガジェットのない汚染だけでは足りない。

## クロスオリジンのメッセージングとネットワークの攻撃クラス (subagent_type: `general`)

**`postMessage` のオリジンとソースの信頼**
ハンドラが、厳密なオリジン許可リストなしに、また複数のフレームが同じオリジンを共有する場合には期待する `event.source` の確認なしに、`event.data` を使って機密アクションを行う。送信側では、`*` に送られた機密データが意図しない埋め込み元に届く。部分文字列、前方一致、後方一致、アンカーのない正規表現による弱いオリジン照合は、オリジンチェックではない。

**クロスサイト WebSocket リクエストの利用**
WebSocket のアップグレードが、`Origin` チェックやチャネル固有のトークンなしに信頼できないオリジンからのアンビエントな Cookie を受け入れ、被害者のセッションでデータを読み取りまたは変更できる。アップグレードの挙動と、セキュリティ上重要なメッセージハンドラの両方を確認する。

**クレデンシャル付き CORS の信頼**
サーバーがクレデンシャルを許可しつつ `Origin` を反射するか弱く照合し、機密の応答を返す。クレデンシャル付きの単純なワイルドカードはブラウザが拒否する。実際に反射・許可されるオリジンの経路と、クロスオリジンでのデータ取得または変更のみを報告する。

## Service Worker とブラウザストレージの攻撃クラス (subagent_type: `general`)

**Service Worker の登録とスコープの乗っ取り**
攻撃者が影響できるコンテンツが、登録される Worker スクリプトになりうるか、過度に広い `Service-Worker-Allowed` スコープを受け取るパスを制御できるか、完全性の制御なしに更新時のインポートを変更できる。最終的なスクリプト URL、応答の MIME タイプ、オリジン、スコープ、インポートされる各スクリプトを誰が制御するかを検証する。意図したスコープを持つ通常の同一オリジン Worker は欠陥ではない。

**Service Worker のキャッシュと ID の混同**
Worker が、アカウント、テナント、認可状態、リクエストモードをポリシーに含めずにパーソナライズされた応答をキャッシュし、アカウント切り替えやログアウト後にそれを返す。fetch イベントのルーティング、キャッシュ名とキー、ナビゲーションのフォールバック、キャッシュの掃除、エラー/オフライン経路が他ユーザーの過去の応答を返すかをレビューする。

**ブラウザストレージの開示と古い認可**
トークン、非公開の応答、下書きデータ、認可判断が、`localStorage`、`sessionStorage`、IndexedDB、Cache Storage、拡張機能のストレージ、クライアント状態に残り、別のアカウントや信頼度の低い同一オリジンのコンポーネントから読み取れるようになる。トークンを保存しているだけでは指摘ではない。より権限の低い現実的な読み取り主体、または失効/ログアウト後の継続利用を要件とする。

**コンテキスト間のストレージとブロードキャストの混同**
`storage` イベント、`BroadcastChannel`、Shared Worker、オリジン全体のキャッシュが、現在のセッションに束縛されないまま ID やコマンドをタブ間で運ぶ。アカウント切り替え、プライベート/通常ウィンドウ、テナント変更、新しい認証状態を上書きしうる古いタブを確認する。

## クロスサイトの情報漏洩クラス (subagent_type: `general`)

**XS-Leaks とクロスオリジンの状態オラクル**
ブラウザが被害者のクレデンシャルを付与する状態で、攻撃者のページが、リソースの load/error イベント、フレームやウィンドウの状態、リダイレクトの挙動、タイミング、キャッシュ状態、応答サイズを通じて、保護されたクロスオリジンの状態を区別できる。非公開オブジェクト、ロール、アカウントが存在するかどうかのような、秘密を含む具体的な述語を 1 つ要件とする。一般的なタイミングのばらつきや公開リソースの可用性は指摘ではない。

**ウィンドウと opener の状態の開示**
クロスオリジンのウィンドウで許可されたメタデータやナビゲーション結果が保護された状態を明かす、または保持された opener/名前付きウィンドウの関係により、攻撃者が制御するページが特権的なナビゲーションに影響できる。COOP、フレーム保護、`noopener`、厳密なオリジン、観測可能な状態が機密かどうかを確認する。

## UI 偽装 (UI redress) とナビゲーションの攻撃クラス (subagent_type: `general`)

**クリックジャッキング**
フレームに埋め込まれた状態変更アクションに、有効な `frame-ancestors`、`X-Frame-Options`、または同等の UI 分離がない。機密アクションを要件とし、フレーム内の状態でそれを完了できることを確認する。読み取り専用コンテンツでのヘッダー欠如はハードニングの注記である。

**クライアントサイドのナビゲーション混同**
クライアント側のソースが、スキームと遷移先のポリシーなしに、実行可能な `javascript:` や `data:` の遷移先を含めて、リダイレクトやナビゲーションを制御する。リバースタブナビングが当てはまるのは、コードが明示的に `window.opener` を保持する場合、分離なしに `window.open` を使う場合、暗黙の `noopener` がないブラウザをサポートする場合に限る。

## 共通の手順 (上記すべてに適用)

- DOM、ナビゲーション、Worker、メッセージ、ストレージのシンクから始め、ブラウザのみのソースとサーバーが制御するソースへと逆向きに追跡する。その経路を止めるべきブラウザポリシーを記録する。
- アカウント切り替え、ログアウト、Worker の更新、オフラインのフォールバック、古いタブの状態を、ローカルのテスト用オリジンとダミーアカウントでテストする。本番のユーザー、オリジン、共有サービスは使わない。
- XS-Leaks では、ソースとローカルのブラウザ挙動で証明された述語のみを挙げる。そのうえで、オラクルを取り除く応答ヘッダーやレンダリング上の選択を特定する。

## 検証ルール (ここでの指摘を報告する前に必ず適用)

1. ソース、シンク、ブラウザポリシー、影響を受けるオリジン/セッション、観測可能な変更または開示を示す。
2. プロトタイプ汚染では、再帰的書き込みとセキュリティ上重要なガジェットを証明する。DOM clobbering では、マークアップが残ることと、覆い隠された値が使われることを証明する。
3. Service Worker とストレージでは、ライフサイクル上の到達可能性を証明する。攻撃者が制御する書き込みやキャッシュエントリが、別のアカウント、テナント、または後の認可状態に到達しなければならない。
4. メッセージング、CORS、WebSocket、XS-Leaks では、厳密なオリジン/ソース検証と、露出する保護された状態やアクションを示す。CSP、COOP/CORP、Cookie、SameSite ポリシーがすでにそれを阻止していないことを確認する。
5. `confirmed` の指摘は、完全なクライアント経路と範囲を限ったローカルの証拠がある場合にのみ返す。オーナーが検証すべき、デプロイ済みヘッダー、拡張機能の権限、ブラウザのバージョン、レンダラの挙動を正確に添えて `needs_validation` を返す。
