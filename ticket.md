# Ticket: spell-client

## チケット情報

- **リポジトリ**: spell-client
- **技術スタック**: TypeScript / Next.js / React / TailwindCSS
- **仕様書パス**: ../README.md, ./README.md
- **ビルドコマンド**: `pnpm build`
- **テストコマンド**: `pnpm test`
- **その他コマンド**: `pnpm lint`

---

## 実装対象チケット

（ここにチケット本文を貼り付けてください）

例:
```
[SPELL-CLIENT-001] Passkey ログイン UI + SDK 実装

## 概要
ユーザーが Passkey でログインし、Intent を署名・実行できる UI と SDK を実装する。

## 受け入れ条件
- [ ] ログイン画面 (Passkey registration/login)
- [ ] ダッシュボード画面
- [ ] Intent 実行 UI
- [ ] SDK: loginWithPasskey()
- [ ] SDK: signIntent(intent)
- [ ] SDK: execute(intent)
- [ ] エラーハンドリング

## 技術仕様
- Next.js App Router
- @simplewebauthn/browser を使用
- SDK は spell-client/sdk として独立パッケージ化
```

---

## 補足情報

- 不明点は合理的な前提を置いて実装してください
- 既存コードスタイルに従ってください
- 破壊的変更が必要な場合は理由を明記してください
