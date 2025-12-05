# spell-client 機能仕様書（Web App + TypeScript SDK）

## 0. 文書情報

* 対象: `spell-client`
* 目的: Spell システムのユーザー接点（UI/SDK）を提供し、**Passkeyログイン（セッションJWT）** と **Intent署名導線** を用いて安全に操作できるようにする
* 関連サービス:

  * `spell-auth`：身元（user_id）確定 + セッション（Access JWT / Refresh Cookie）
  * `spell-core`：実行/ポリシー/ジョブ管理（最終判断を含む）
  * `spell-intent`：Intent署名検証（※ client は直接呼ばない）
* 本書の位置づけ: `spell-client` の機能仕様の正本

---

## 1. 非ゴール（Non-goals）

`spell-client` は以下を行わない（実装しない・責務外）：

* サーバ側WebAuthn検証、JWT署名発行（= `spell-auth` の責務）
* Intent署名のサーバ側検証（= `spell-intent` の責務。`spell-core` 経由で完結させる）
* 実行可否の最終判断、ジョブ分割、マルチリポ実行（= `spell-core` の責務）
* 課金計算・決済導線の最終版（= `spell-billing` の責務）
* 長時間/重処理の実行（= `spell-runner` の責務）
* OAuth/SAML など外部IdP連携

---

## 2. 前提条件（Prerequisites）

### 2.1 動作環境（MUST）

* WebAuthn対応ブラウザ/OS（Passkey利用可能）
* JavaScript有効

### 2.2 デプロイ/ドメイン前提（MUST）

* `spell-client` と `spell-auth` は **同一“site”（同一 eTLD+1）配下**で運用する
  例: `app.example.com` と `auth.example.com`
  ※ Refresh Cookie（SameSite=Lax）でのセッション維持を成立させるため

### 2.3 依存APIが提供されていること（MUST）

* `spell-auth` API（register/login/refresh/logout/passkeys…）
* `spell-core` API（whoami / intent start / intent execute / jobs status…）

---

## 3. 制約（Constraints）

### 3.1 認証情報の保存（MUST）

* Access JWT は **永続保存しない（localStorage禁止）**

  * メモリ（アプリ状態）にのみ保持
  * ページ再訪時は refresh により再取得する
* Refresh Token は **HttpOnly Cookie** 前提（JSから参照不可）

### 3.2 ネットワーク呼び出し（MUST）

* `spell-auth` 呼び出しは `fetch(..., { credentials: "include" })` を必須
  （Set-Cookie受領とCookie送信のため）
* `spell-core` 呼び出しは `Authorization: Bearer <access_token>` を使用

### 3.3 自動リトライ（MUST）

* `spell-core` へ送ったリクエストが **401** の場合：

  * refresh を1回だけ試みる → access_token再取得 → 元リクエストを1回だけ再試行
* refresh 失敗時：

  * 未ログイン状態へ遷移（ログイン画面へ）

### 3.4 セキュリティ（MUST）

* token / assertion をURLクエリに載せない
* token / assertion をconsole/logへ出さない
* 署名が必要な操作は **署名前に Intent 内容を表示**する（ユーザー確認なしに署名させない）

### 3.5 責務境界（MUST）

* `spell-client` は `spell-intent` を直接呼ばない
  （最終実行の文脈は `spell-core` に統一）

---

## 4. 成功条件（Success Criteria：全体）

* 初回登録/ログインがUIから完了し、Access JWT を取得できる
* 再訪時に自動 refresh が成功すれば、再サインイン無しで利用できる
* 危険操作では Intent プレビュー → Passkey署名 → 実行確定までが成立する
* 401/403/ネットワーク障害/ユーザー署名キャンセル時に、安全側の挙動で復帰導線がある

---

## 5. 機能一覧（提供機能）

* FC-1 セッション初期化（起動時の自動refresh）
* FC-2 ユーザー登録（Passkey作成）
* FC-3 ログイン（Passkey認証）
* FC-4 ログアウト
* FC-5 Passkey管理（一覧/追加/削除）
* FC-6 通常操作（JWTで `spell-core` を呼ぶ）
* FC-7 Intent実行（プレビュー/署名/実行/進捗/結果）
* FC-8 異常系（401→refresh、403(BAN)、WebAuthn非対応、署名キャンセル等）
* FC-9 SDK提供（UIから利用する統一API）

---

# 6. 機能仕様（詳細）

## FC-1 セッション初期化（アプリ起動時）

**目的**: 再訪時に再サインイン不要を実現する

* 入力:

  * アプリ起動（初回ロード/再訪）
* 処理:

  * `spell-auth` の refresh を試行し access_token を取得する
* 出力:

  * 成功: ログイン済み状態（Access JWTがメモリに載る）
  * 失敗: 未ログイン状態（ログインUIを表示）

**前提条件**

* `spell-auth` が稼働している
* refresh cookie が存在する場合はブラウザが送信可能

**制約**

* `credentials: "include"` 必須
* 失敗時は無限リトライしない（1回で未ログイン扱い）

**成功条件**

* access_token を取得でき、以後の `spell-core` 呼び出しが可能になる

---

## FC-2 ユーザー登録（Passkey作成）

**目的**: 初回ユーザーがPasskeyでアカウント作成しログイン状態になる

* 入力:

  * UI入力: `email?`, `display_name?`
  * ユーザー操作: OS/ブラウザのPasskey作成を許可

* 処理（I/O）:

  1. `POST {AUTH}/api/auth/register/options`

     * 入力: `{ email?, display_name? }`
     * 出力: `{ challenge_id, publicKey }`
  2. `navigator.credentials.create({ publicKey })`

     * 出力: `credential`（PublicKeyCredential）
  3. `POST {AUTH}/api/auth/register/verify`

     * 入力: `{ challenge_id, email?, display_name?, credential }`
     * 出力: `{ user, access_token }` + refresh cookie（Set-Cookie）

* 出力:

  * UI: `user` 表示、ログイン状態へ遷移
  * 状態: access_token をメモリに保持

**前提条件**

* WebAuthn対応環境
* `spell-auth` のOrigin制御に許可されている

**制約**

* `register/verify` は `credentials: "include"` 必須（cookie受領）
* access_token を永続保存しない

**成功条件**

* access_token を受け取り、以後の whoami が通る状態になる

---

## FC-3 ログイン（Passkey認証）

**目的**: 既存ユーザーがPasskeyでログインしセッションを得る

* 入力:

  * UI入力: `user_hint?`（例: email）
  * ユーザー操作: OS/ブラウザのPasskey認証を許可

* 処理（I/O）:

  1. `POST {AUTH}/api/auth/login/options`

     * 入力: `{ user_hint? }`
     * 出力: `{ challenge_id, publicKey }`
  2. `navigator.credentials.get({ publicKey })`

     * 出力: `assertion`（PublicKeyCredential）
  3. `POST {AUTH}/api/auth/login/verify`

     * 入力: `{ challenge_id, user_hint?, credential: assertion }`
     * 出力: `{ user, access_token }` + refresh cookie（Set-Cookie）

* 出力:

  * UI: `user` 表示、ログイン状態へ遷移
  * 状態: access_token をメモリに保持

**前提条件**

* WebAuthn対応環境
* `spell-auth`が稼働

**制約**

* `login/verify` は `credentials: "include"` 必須
* BAN（403）時は凍結画面へ（FC-8）

**成功条件**

* access_token を取得し、再訪はFC-1で復元可能

---

## FC-4 ログアウト

**目的**: セッションを破棄し、再訪しても自動復元しないようにする

* 入力:

  * ユーザー操作: 「ログアウト」クリック
* 処理（I/O）:

  * `POST {AUTH}/api/auth/logout`（`credentials: "include"`）
* 出力:

  * access_token をメモリから破棄
  * UIを未ログイン状態へ

**前提条件**

* `spell-auth` 稼働

**制約**

* ログアウト失敗時もローカル状態は破棄してよい（安全側）

**成功条件**

* refresh cookie が失効し、次回起動時のFC-1が失敗する

---

## FC-5 Passkey管理（一覧/追加/削除）

**目的**: 複数端末/鍵の運用をUIから可能にする

### FC-5a 一覧

* 入力: ユーザー操作「Passkey一覧」
* 処理: `GET {AUTH}/api/auth/passkeys`（Bearer access_token）
* 出力: passkey配列の表示

前提条件: ログイン済み
制約: 401時はFC-1/refresh→再試行
成功条件: passkey一覧が表示される

### FC-5b 追加

* 入力: ユーザー操作「Passkey追加」
* 処理:

  1. `POST {AUTH}/api/auth/passkeys/add/options`（Bearer）
  2. `navigator.credentials.create(...)`
  3. `POST {AUTH}/api/auth/passkeys/add/verify`（Bearer）
* 出力: 追加完了の表示

前提条件: WebAuthn対応 + ログイン済み
制約: 署名キャンセル時は追加されない
成功条件: 一覧に新しいpasskeyが出る

### FC-5c 削除

* 入力: ユーザー操作「削除」＋確認
* 処理: `POST {AUTH}/api/auth/passkeys/remove`（Bearer, `{passkey_id}`）
* 出力: 削除完了の表示

前提条件: ログイン済み
制約: **最後の1本削除は強警告UI**（ただしサーバ側は許可しうる）
成功条件: 対象passkeyが一覧から消える

---

## FC-6 通常操作（JWTでspell-coreを呼ぶ）

**目的**: 通常のread系/軽い操作を署名なしで快適に行う

* 入力: ユーザー操作（画面遷移/データ取得）
* 処理:

  * `GET {CORE}/api/whoami` などを `Authorization: Bearer <access_token>` で呼ぶ
  * 401時は refresh→再試行（FC-1ルール）
* 出力:

  * UIにデータ表示 / 未ログイン遷移 / 403凍結遷移

**前提条件**

* `spell-core` 稼働

**制約**

* 401→refresh は **1回だけ**
* 403は凍結画面（FC-8）

**成功条件**

* whoami等が取得でき、UIが正しいログイン状態を表示する

---

## FC-7 Intent実行（プレビュー/署名/実行/進捗/結果）

**目的**: 高リスク操作を「内容表示→署名→実行」で安全に行う

### Intentモデル（クライアント内部）

* 入力（UI/SDKが生成）:

```json
{
  "op": "string",
  "args": { },
  "ts": 1733300000,
  "exp": 1733300060,
  "nonce": "string"
}
```

### FC-7a Intentプレビュー

* 入力: 危険操作の実行ボタン
* 出力: Intentの要点（op/対象/主要args/期限/警告）を表示し、署名/キャンセル選択を提示

前提条件: ログイン済み
制約: プレビュー無しに署名へ進ませない
成功条件: ユーザーが内容を認知して署名へ進める

### FC-7b 署名開始（coreからoptions取得）

* 入力: 「署名して実行」クリック、Intent本文
* 処理（I/O）:

  * `POST {CORE}/api/intent/start`

    * 入力: `{ intent }` + Bearer
    * 出力: `{ intent_id, publicKey }`
* 出力: `publicKey` を取得

前提条件: `spell-core` 稼働
制約: 401→refresh→再試行1回
成功条件: WebAuthn署名に必要なoptionsを受け取れる

### FC-7c Passkey署名（WebAuthn）

* 入力: `publicKey`（RequestOptions）
* 処理:

  * `navigator.credentials.get({ publicKey })`
* 出力:

  * 成功: `assertion` 取得
  * キャンセル: 署名中断（実行しない）

前提条件: WebAuthn対応
制約: キャンセル時は **実行リクエストを送らない**
成功条件: assertion を取得できる

### FC-7d 実行確定（署名付き）

* 入力: `{ intent_id, intent, assertion }`
* 処理（I/O）:

  * `POST {CORE}/api/intent/execute`

    * 入力: `{ intent_id, intent, credential: assertion }` + Bearer
    * 出力: `{ job_id }`（推奨）/ もしくは `{ result }`
* 出力:

  * job_id を保持し進捗画面へ

前提条件: `spell-core` 稼働
制約: clientは署名検証しない（coreが最終）
成功条件: 実行が開始され job_id が得られる（または結果が返る）

### FC-7e 進捗/結果表示

* 入力: job_id
* 処理（I/O）:

  * `GET {CORE}/api/jobs/{job_id}` をポーリング
* 出力:

  * status/progress/result/error を表示

前提条件: `spell-core` がジョブ状態を提供
制約: ポーリング間隔は固定（例 1〜3秒）でよい（v0）
成功条件: 成功/失敗がユーザーに明確に表示される

---

## FC-8 異常系（共通）

### 8.1 401（セッション切れ）

* 入力: core/auth 呼び出しで 401
* 出力: refresh→再試行（1回）→失敗なら未ログイン状態

成功条件: 自動復帰できるか、復帰不可ならログイン導線が出る

### 8.2 403（BAN/凍結）

* 入力: auth/core から 403（凍結）
* 出力: 凍結画面を表示し、以後の自動refreshループを止める

成功条件: ユーザーが「凍結である」ことを理解でき、無限再試行しない

### 8.3 WebAuthn非対応

* 入力: `window.PublicKeyCredential` なし等
* 出力: 「この環境はPasskey非対応」画面 + 代替案は別要件（ここでは表示のみ）

成功条件: ユーザーが進めない理由を理解できる

### 8.4 署名キャンセル

* 入力: ユーザーがOSダイアログでキャンセル
* 出力: 実行しない／プレビュー画面へ戻す

成功条件: 危険操作が実行されない

---

## FC-9 SDK提供（TypeScript）

**目的**: UI/CLI/将来クライアントから同一の手順で呼べるようにする

### 公開API（最小）

* 入力/出力は TypeScript 型で保証する

```ts
export type Intent = {
  op: string;
  args: Record<string, any>;
  ts: number;
  exp: number;
  nonce: string;
};

export type AuthUser = { id: string; email?: string; display_name?: string };

export interface SpellClient {
  // session
  bootstrap(): Promise<void>;            // FC-1
  logout(): Promise<void>;                // FC-4
  getAccessToken(): string | null;        // in-memory only

  // auth
  registerPasskey(input: { email?: string; display_name?: string }): Promise<{ user: AuthUser }>; // FC-2
  loginPasskey(input: { user_hint?: string }): Promise<{ user: AuthUser }>;                       // FC-3

  // passkeys
  listPasskeys(): Promise<Array<{ id: string; created_at: string; last_used_at?: string | null }>>; // FC-5a
  addPasskey(): Promise<void>;                                                                        // FC-5b
  removePasskey(passkey_id: string): Promise<void>;                                                    // FC-5c

  // core normal
  whoami(): Promise<{ user_id: string }>;                                                              // FC-6

  // intent
  executeIntent(intent: Intent): Promise<{ job_id: string }>;                                          // FC-7b〜7d
  getJob(job_id: string): Promise<any>;                                                                // FC-7e
}
```

**前提条件**

* `NEXT_PUBLIC_AUTH_ORIGIN` / `NEXT_PUBLIC_CORE_ORIGIN` が設定済み

**制約**

* SDK内部で `credentials: "include"` を適切に付与
* 401→refresh→再試行は SDK が統一実装
* token/assertion をログ出力しない

**成功条件**

* UIはSDK関数を呼ぶだけで、ログイン〜intent実行が成立する

---

## 7. 受け入れ基準（Acceptance）

* 登録: UIからPasskey登録→ログイン状態になる
* ログイン: Passkeyログイン→ログイン状態になる
* 再訪: ページ再訪→自動refreshで復帰できる（期限内）
* Intent: プレビュー→署名→実行→結果確認ができる
* 異常系: 401/403/署名キャンセル/非対応環境が安全に扱われる
* 制約: access_tokenが永続化されていない（検査可能）

---

必要なら次に、`spell-core` 側に要求する `/api/intent/start` / `/api/intent/execute` の **機能仕様（入力/出力/エラーコードまで）** を同じ形式で“固定契約”として書き切って、client/core間のブレを永久に潰せる。
