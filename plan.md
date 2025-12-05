# Plan: spell-client

## 実装計画

このファイルは「完遂モード」で AI に渡す実装計画書です。

---

## 目的

spell-client リポジトリの初期実装を完遂する。

---

## 実装フェーズ

### Phase 1: 環境セットアップ

- [ ] Next.js プロジェクト作成（App Router）
- [ ] package.json に必要な依存関係を追加
  - `@simplewebauthn/browser`, `axios`, `zod`, `tailwindcss` など
- [ ] TypeScript 設定 (tsconfig.json)
- [ ] ESLint / Prettier 設定
- [ ] .env.example 作成

### Phase 2: SDK 実装

SDK は `spell-client/sdk` として独立パッケージ化

- [ ] loginWithPasskey()
  - spell-auth の /register, /login を呼び出し
  - JWT を localStorage に保存
- [ ] signIntent(intent)
  - spell-intent の /challenge を呼び出し
  - WebAuthn で署名
  - spell-intent の /verify を呼び出し
- [ ] execute(intent)
  - signIntent() を内部で呼び出し
  - spell-core の /execute を呼び出し
  - job_id を返却
- [ ] getJobStatus(jobId)
  - spell-core の /status/:jobId を呼び出し

### Phase 3: Web UI 実装

- [ ] ログイン画面 (`/login`)
  - Passkey registration
  - Passkey login
- [ ] ダッシュボード (`/dashboard`)
  - ジョブ一覧表示
  - ジョブステータス表示
- [ ] Intent 実行画面 (`/execute`)
  - Intent 入力フォーム
  - 実行ボタン（Passkey 署名が必要）

### Phase 4: UI コンポーネント

- [ ] PasskeyButton コンポーネント
- [ ] IntentForm コンポーネント
- [ ] JobList コンポーネント
- [ ] JobStatus コンポーネント

### Phase 5: テスト

- [ ] SDK ユニットテスト
- [ ] UI コンポーネントテスト
- [ ] E2E テスト（Playwright）

### Phase 6: ドキュメント

- [ ] SDK ドキュメント
- [ ] UI 使用方法
- [ ] 環境変数リスト

---

## ファイル構成（想定）

```
spell-client/
├── sdk/                       # SDK パッケージ
│   ├── src/
│   │   ├── index.ts           # SpellClient class
│   │   ├── auth.ts            # Auth methods
│   │   ├── intent.ts          # Intent signing
│   │   └── core.ts            # Core API calls
│   ├── package.json
│   └── tsconfig.json
├── app/                       # Next.js App Router
│   ├── layout.tsx
│   ├── page.tsx               # Landing page
│   ├── login/
│   │   └── page.tsx           # Login page
│   ├── dashboard/
│   │   └── page.tsx           # Dashboard
│   └── execute/
│       └── page.tsx           # Execute intent
├── components/
│   ├── PasskeyButton.tsx
│   ├── IntentForm.tsx
│   ├── JobList.tsx
│   └── JobStatus.tsx
├── lib/
│   └── spell-client.ts        # SDK instance
├── tests/
│   ├── sdk/
│   │   ├── auth.test.ts
│   │   └── intent.test.ts
│   └── e2e/
│       └── login.spec.ts
├── .env.example
├── next.config.js
├── tailwind.config.js
├── tsconfig.json
├── package.json
└── README.md
```

---

## 技術的決定事項

- **フレームワーク**: Next.js (App Router)
- **WebAuthn ライブラリ**: @simplewebauthn/browser
- **HTTP クライアント**: axios
- **UI ライブラリ**: TailwindCSS
- **バリデーション**: zod
- **テストフレームワーク**: vitest (SDK), Playwright (E2E)

---

## 注意事項

- SDK は spell-client とは独立してビルド可能にする
- JWT は localStorage に保存（本番環境では httpOnly cookie 推奨）
- WebAuthn は HTTPS 環境が必要（開発時は localhost で OK）
- 環境変数で spell-auth, spell-intent, spell-core の URL を設定可能にする

---

## 完遂モード実行時の指示

ticket.md のチケット内容と、この plan.md を参照して、上記フェーズを全て実装してください。
不明点は合理的な前提を置いて実装を完了させてください。
