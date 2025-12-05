# spell-client 要件定義書（Client/UI + SDK）

## 0. 文書情報

* 対象リポジトリ: `spell-client`
* 目的: Spell システムの **ユーザー接点（UI/SDK）** を提供し、Passkeyログイン（セッションJWT）と Intent署名フローを用いて Spell を安全に操作可能にする
* ステータス: 確定（`spell-client` の要件の唯一の正本）

---

## 1. 背景 / 目的

`spell-client` は Spell システムにおいて、ユーザーが行いたい操作を **UI/SDK として表現**し、以下を満たす：

* **ログイン体験の最適化**
  一度ログインしたら、一定期間は再サインイン無しで利用できる（Refreshベースのセッション維持）
* **危険操作の安全化**
  重要/高リスク操作は、実行前に **Intent（操作内容）を提示し、Passkeyでの署名（同意）** を要求する
* **責務分離の維持**

  * 身元・セッション: `spell-auth`
  * Intent署名検証: `spell-intent`
  * 実行・ポリシー判断・ジョブ管理: `spell-core`
  * 課金・利用量: `spell-billing`
  * 重い実行: `spell-runner`
    `spell-client` はそれらを **UI/SDKとして統合**するが、機能を侵食しない

---

## 2. スコープ（やること / やらないこと）

### 2.1 やること（In Scope）

`spell-client` は以下を提供する。

#### A. Webアプリ（ユーザーUI）

* Passkey登録/ログインUI
* セッション維持（refreshによる自動再ログイン）
* Passkey管理（追加/一覧/削除）
* Intent作成・プレビュー・署名UI（危険操作用）
* Spell操作UI（最低限：実行開始・実行状況・結果表示）
* エラー/状態表示（ログイン切れ、署名失敗、BAN、ネットワーク障害等）

#### B. TypeScript SDK（`@spell/client` 相当）

* `spell-auth` / `spell-core` への呼び出しをラップするSDK
* `WebAuthn` 実行（options受領→`navigator.credentials.*`）を抽象化
* トークン更新・リトライを統一（401→refresh→再試行など）

※ WebアプリはSDKを内部利用してもよい（推奨）。

---

### 2.2 やらないこと（Non-goals）

* `spell-auth` が担うべきユーザー識別/認証ロジックの実装（サーバ側のWebAuthn検証、JWT署名など）
* `spell-intent` が担うべき Intent署名検証（サーバ側の署名検証、credential台帳など）
* `spell-core` が担うべき 実行ポリシー判断・ジョブ分割・オーケストレーション
* 料金計算/課金UIの完成版（最低限の「コスト表示」までは可、決済は別要件）
* 長時間実行やGit操作の実働（runnerの責務）
* 外部IdP/OAuthログイン（別要件）

---

## 3. 対象プラットフォーム / サポート範囲

### 3.1 サポート（MUST）

* Web（PC/モバイル）

  * 最新版 Chrome / Safari / Edge を対象（Passkey対応環境）
  * iOS/Android の標準ブラウザで Passkey が動作すること

### 3.2 非サポート（当面）

* レガシーブラウザ（WebAuthn非対応）
* ネイティブアプリ（要件外。必要なら別途）

---

## 4. システム境界 / 依存（Integration）

`spell-client` が依存する外部サービスは以下。

* `spell-auth`（身元/セッション）
* `spell-core`（実行/状態/結果）
* （`spell-intent` は **直接依存しない**。`spell-core` が内部で呼ぶ前提で統一する）
  ※ clientがintent検証オラクルを叩くと信頼境界が曖昧になるため、実行側で一元化する。

---

## 5. 重要設計原則（Design Principles）

1. **デプロイファースト**
   UIが常に動作し、最小導線（ログイン→実行→結果）が途切れないことを最優先とする
2. **セキュリティファースト**

   * Access JWT を localStorage に永続保存しない
   * Refresh は HttpOnly cookie を前提に、JS から触れない
3. **責務境界を壊さない**
   client は “表示・操作・署名開始” に集中し、最終判断や実行を内包しない
4. **操作の意味（Intent）をユーザーに見せる**
   署名前に「何をするか」を表示し、意図せぬ実行を防ぐ
5. **失敗に強い**
   refresh失敗・署名失敗・BAN・ネットワーク障害でも明確に復帰導線を出す

---

## 6. ユースケース（User Stories）

### US-1 初回登録

* ユーザーはメール（任意）と表示名（任意）で Passkey を登録し、アカウントを作成できる

### US-2 ログイン

* ユーザーは Passkey でログインできる（パスワード不要）

### US-3 セッション維持

* ユーザーはページを閉じて再訪しても、一定期間は再ログイン無しで利用できる
  （裏で refresh が成功する限り）

### US-4 Passkeyの追加

* ユーザーは別端末で Passkey を追加し、同じアカウントとして利用できる

### US-5 危険操作の実行

* 重要操作（例：複数リポ変更/高コスト処理）を行う際、Intent を確認し Passkey 署名してから実行できる

### US-6 実行状況の確認

* 実行の開始・進捗・結果（成功/失敗）を確認できる

---

## 7. 機能要件（Functional Requirements）

### FR-1 認証（登録/ログイン）

* MUST: 登録フロー（options→`navigator.credentials.create`→verify）
* MUST: ログインフロー（options→`navigator.credentials.get`→verify）
* SHOULD: `user_hint`（email）を入力できるUIを持つ
* MUST: 失敗時にエラー理由（最低限の分類）と復帰導線を提示する

### FR-2 セッション維持（refresh）

* MUST: アプリ起動時に **自動 refresh** を試みる（ログイン維持）
* MUST: API呼び出しが 401 の場合、1回だけ refresh→再試行する
* MUST: refresh 失敗時、ログイン画面へ遷移（またはログイン要求状態にする）
* MUST: Refresh cookie は JS から参照不可である前提で動くこと（HttpOnly）

### FR-3 Passkey 管理

* MUST: Passkey一覧表示
* MUST: Passkey追加（ログイン済みで add/options→create→add/verify）
* MUST: Passkey削除（削除の確認UIを必須）
* MUST: 最後の1本削除は「警告・確認」付きで許可（以後ログイン不能になり得る）

### FR-4 通常操作（JWTのみで可能な操作）

* MUST: `spell-core` の read系操作を JWT で呼び出せる（例：whoami、ジョブ一覧など）
* MUST: JWT の存在だけを前提にせず、常に refresh による回復を試みる

### FR-5 Intent 操作（危険操作の署名導線）

* MUST: state-changing/高リスク操作は Intent として構造化する（UI内部のモデル）
* MUST: 署名前に Intent をユーザーに表示する（プレビュー画面）

  * 操作名（op）
  * 対象（例：リポ一覧）
  * 主要引数（argsの要点）
  * 期限（exp）
  * 予想コスト/警告（あれば）
* MUST: `spell-core` が返す WebAuthn RequestOptions を使い、`navigator.credentials.get` を実行する
* MUST: 署名結果（assertion）を `spell-core` に送って実行を確定させる
* MUST: 署名キャンセルは「実行しない」で確定し、状態を安全側に戻す

### FR-6 実行状況/結果表示

* MUST: 実行開始後にステータス確認ができる（最小：ポーリング）
* MUST: 成功/失敗/キャンセルを表示
* SHOULD: 実行ログ（要約）を表示
* SHOULD: 失敗時に再試行導線（同じIntent再署名 or 新Intent生成）を出す

### FR-7 BAN / アクセス拒否

* MUST: `spell-auth`/`spell-core` から BAN 由来の 403 を受けたら、明示的に「凍結」画面を表示
* MUST: refresh / login でも復帰できないことを示す（サポート導線などは別要件）

---

## 8. 外部I/F要件（API連携：入力/出力）

### 8.1 `spell-auth` 連携（MUST）

`spell-client` は以下を利用する（I/Oは `spell-auth` 仕様に準拠）。

* `POST /api/auth/register/options` → CreationOptions + challenge_id
* `POST /api/auth/register/verify` → user + access_token + refresh cookie
* `POST /api/auth/login/options` → RequestOptions + challenge_id
* `POST /api/auth/login/verify` → user + access_token + refresh cookie
* `POST /api/auth/token/refresh` → access_token（cookie送信が必要）
* `POST /api/auth/logout` → refresh失効（cookie削除）
* `GET /api/auth/verify` → user（デバッグ/確認用）
* `GET /.well-known/jwks.json` → （SDK内部で必要なら取得するが、通常は core 側が利用）

**制約（MUST）**

* refresh/logout は `credentials: "include"` を付ける
* Origin allowlist 前提（同一レジストラブルドメイン配下を推奨）

### 8.2 `spell-core` 連携（MUST）

`spell-client` は `spell-core` に対して以下の “クライアント契約” を要求する。
（※ spell-core の機能仕様で確定させる前提。client側はこれを前提に実装する。）

#### a) セッション確認

* `GET /api/whoami`

  * 入力: `Authorization: Bearer <access_token>`
  * 出力: `{ user_id, ... }` または 401/403

#### b) Intent署名開始

* `POST /api/intent/start`

  * 入力: `{ intent: {...} }` + `Authorization: Bearer <access_token>`
  * 出力: `{ intent_id, publicKey: PublicKeyCredentialRequestOptions }`
  * 目的: 署名用 challenge/options を提供する（内部でspell-intentと連携してよい）

#### c) Intent実行確定

* `POST /api/intent/execute`

  * 入力: `{ intent_id, intent, credential: assertion }` + `Authorization: Bearer <access_token>`
  * 出力: `{ job_id }`（実行開始）or `{ result }`（同期実行の場合）
  * 目的: 署名検証（内部でspell-intent）→ポリシー→実行開始

#### d) 実行状況

* `GET /api/jobs/:job_id`

  * 入力: Bearer
  * 出力: `{ status, progress?, result?, error? }`

**制約（MUST）**

* `spell-client` は core の 401 を受けた場合、refresh→再試行を1回行う
* core の 403(BAN等) を受けた場合は凍結画面へ

---

## 9. クライアント側データ取り扱い（Storage/State）

### 9.1 トークン保持（MUST）

* Access JWT は **永続保存しない**（localStorage禁止）
* 推奨挙動（MUST）:

  * メモリ上（状態管理）に保持
  * 起動時と 401 時に refresh を試みることで再取得する
* Refresh Token は HttpOnly Cookie であり、JS から参照しない

### 9.2 永続化してよい情報（SHOULD）

* UI設定（テーマ、表示密度等）
* 最後に選択したリポやフォームの下書き（個人情報を含めない範囲）
* 監査目的に使える情報は **保存しない**（サーバ側ログの責務）

---

## 10. セキュリティ要件（Client）

* MUST: CSP（Content Security Policy）を設定し、インラインスクリプトを極力避ける
* MUST: XSS 対策（dangerouslySetInnerHTMLを避ける、入力値をHTMLとして解釈しない）
* MUST: 認証情報（access token）を URL クエリに載せない
* MUST: passkey 署名時はユーザーに「何を実行するか」を提示する
* MUST: `navigator.credentials.*` の失敗（ユーザーキャンセル含む）を安全に処理する（実行しない）
* MUST: 例外時に機密情報（token/assertion）をログ出力しない

---

## 11. UX / UI要件（最低限）

* MUST: ログイン状態が明確（ログイン済み/未ログイン/セッション切れ/凍結）
* MUST: 署名が必要な操作は “署名を求める理由” を表示する
* MUST: Intentプレビューで「取り消し」導線を必ず用意する
* SHOULD: エラーは分類して表示（ネットワーク、認証、署名、サーバ、権限）
* SHOULD: 実行中は進捗/待機を表示し、ユーザーが不安にならないUI

---

## 12. 非機能要件（NFR）

* パフォーマンス（MUST）:

  * 初回表示（ログイン画面）を軽く保つ
  * refresh は非同期で行いUIブロックしない
* 可用性（MUST）:

  * auth/core が落ちている場合のフォールバック表示（メンテ/再試行）
* アクセシビリティ（SHOULD）:

  * キーボード操作、フォーカス管理、読み上げに配慮
* 監視/計測（SHOULD）:

  * Sentry等でフロント例外を収集（tokenを含めない）

---

## 13. 運用 / 設定（Environment）

`spell-client` は以下を環境変数で切替できる（MUST）。

* `NEXT_PUBLIC_AUTH_ORIGIN`（例: `https://auth.<root-domain>`）
* `NEXT_PUBLIC_CORE_ORIGIN`（例: `https://core.<root-domain>`）
* `NEXT_PUBLIC_APP_ORIGIN`（任意、origin表示用）
* `NEXT_PUBLIC_BUILD_SHA`（任意、サポート用）

---

## 14. 受け入れ基準（Acceptance Criteria）

以下が end-to-end で成立したら `spell-client` v0 は合格とする。

1. 登録:

* UIから register/options → Passkey作成 → register/verify が成功し、ログイン状態になる

2. ログイン:

* UIから login/options → Passkey認証 → login/verify が成功し、ログイン状態になる

3. セッション維持:

* タブを閉じて再訪しても、自動 refresh によりログイン状態に復帰できる（期限内）

4. Passkey管理:

* ログイン後に passkey 一覧が見れる
* passkey 追加ができる
* passkey 削除ができる（警告付き）

5. Intent署名操作:

* 署名が必要な操作で Intentプレビューが表示される
* Passkey署名を実行できる
* 署名結果を core に送って実行が開始され、結果が確認できる

6. 異常系:

* refresh 失敗時にログイン要求へ戻れる
* BAN（403）時に凍結画面が出る
* 署名キャンセル時に実行されない

---

この要件で `spell-client` は「UI/SDKとしての責務」に集中し、`spell-auth` / `spell-intent` / `spell-core` の境界を壊さずに、**“ログインは楽、危険操作は署名で確定”** を一貫して実現できます。
