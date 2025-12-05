# spell-client

**UI / SDK / CLI**

## 責務

**人間と世界の接点**

- Web フロント（ログイン画面、ジョブ実行画面、Intent ベース UI）
- SDK（ブラウザ / Node 向け）
- CLI（余裕が出たら）

JWT も Intent 署名も全部よしなに包む層

## Components

### Web Frontend

- ログイン画面（Passkey registration/login）
- ダッシュボード
- ジョブ実行 UI
- ジョブステータス表示

### SDK

```typescript
import { SpellClient } from 'spell-client';

const client = new SpellClient({
  authUrl: 'https://auth.spell.dev',
  intentUrl: 'https://intent.spell.dev',
  coreUrl: 'https://core.spell.dev'
});

// Login with passkey
await client.loginWithPasskey();

// Execute intent
const result = await client.execute({
  op: 'repo.create',
  args: { name: 'my-project' }
});
```

### CLI (Future)

```bash
spell login
spell create my-project
spell status <job-id>
```

## Architecture

```
┌──────────────────┐
│   Web Frontend   │
│   (Next.js/Vite) │
└────────┬─────────┘
         │
┌────────▼─────────┐
│   Spell SDK      │
│                  │
│ - Auth wrapper   │
│ - Intent signer  │
│ - Core client    │
└──┬───┬───┬───────┘
   │   │   │
   │   │   └──────► spell-core
   │   └──────────► spell-intent
   └──────────────► spell-auth
```

## Tech Stack

- Web: Next.js / React / TypeScript
- SDK: TypeScript (dual ESM/CJS)
- CLI: Commander.js (future)
