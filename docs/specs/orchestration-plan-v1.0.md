---
id: orchestration-plan-v1-0
title: PWA ポモドーロタイマー — オーケストレーションプラン (v1.0)
version: "1.0"
status: draft
created_at: "2026-02-24"
srs_ref: docs/specs/srs.yaml
spec_ref: docs/specs/pomodoro-spec-v1.0.md
---

# PWA ポモドーロタイマー — オーケストレーションプラン (v1.0)

**目的:** 本ファイルはコーディングエージェント・開発者に「**何を**」「**どのように**」「**どう検証するか**」を正確に伝える。  
v1.0 仕様への実装を拘束し、セキュリティ・テスト品質・デプロイフローを保証する。

---

## 0. 参照ドキュメント

| 種別          | パス                                             |
| ------------- | ------------------------------------------------ |
| SRS YAML      | `docs/specs/srs.yaml`                            |
| 仕様書        | `docs/specs/pomodoro-spec-v1.0.md`               |
| エージェントブリーフ | `docs/specs/coding-agent-brief-v1.0.md`     |
| チェンジログ   | `docs/specs/CHANGELOG-v1.0.md`                   |

> コードは必ず **v1.0 仕様**に準拠すること。乖離は PR で理由を明記すること。

---

## 1. 技術スタック・規約

| 項目                | 詳細                                                         |
| ------------------- | ------------------------------------------------------------ |
| フロントエンド       | React 18 + TypeScript strict + Vite 5                         |
| PWA                 | vite-plugin-pwa (Workbox)                                    |
| 状態管理            | Zustand                                                      |
| スタイリング         | Tailwind CSS                                                 |
| フロントテスト       | Vitest + React Testing Library + Playwright (E2E)            |
| バックエンド         | Go 1.22+, chi v5, syumai/workers                             |
| DB アクセス          | Cloudflare D1 (SQL マイグレーション管理, Drizzle 不使用)     |
| 認証                | JWT (RS256 or HS256), HttpOnly Cookie                        |
| バックエンドテスト   | Go `testing` + `testify`                                     |
| CI/CD               | GitHub Actions                                               |
| IaC                 | Terraform 1.7+ + Terragrunt                                  |

---

## 2. ディレクトリ構成

```
/ (repo root)
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── store/
│   │   ├── api/
│   │   └── lib/
│   │       ├── timer/
│   │       ├── pwa/
│   │       └── notifications/
│   ├── public/manifest.json
│   └── vite.config.ts
├── backend/
│   ├── cmd/worker/
│   ├── internal/
│   │   ├── handler/
│   │   ├── middleware/
│   │   ├── model/
│   │   ├── repository/
│   │   └── service/
│   ├── migrations/
│   └── wrangler.toml
├── infra/
│   ├── modules/
│   ├── envs/dev/
│   └── envs/prod/
├── docs/specs/
└── .github/
    ├── workflows/
    └── PULL_REQUEST_TEMPLATE.md
```

---

## 3. 環境・シークレット

```
# フロントエンド
VITE_API_BASE_URL          # バックエンド API の URL
VITE_VAPID_PUBLIC_KEY      # VAPID 公開鍵

# バックエンド (GitHub Secrets / Cloudflare Secrets)
JWT_SECRET                 # JWT 署名シークレット
VAPID_PRIVATE_KEY          # VAPID 秘密鍵
VAPID_SUBJECT              # VAPID サブジェクト (mailto:...)
CLOUDFLARE_API_TOKEN       # Cloudflare デプロイ用トークン
CLOUDFLARE_ACCOUNT_ID      # Cloudflare アカウント ID
```

> **シークレット類はリポジトリに含めない。** GitHub Secrets または Cloudflare Secrets で管理する。

---

## 4. データモデル (D1 マイグレーション)

`backend/migrations/` に以下のファイルを作成する:

```
0001_create_users.sql
0002_create_settings.sql
0003_create_tasks.sql
0004_create_pomodoro_sessions.sql
```

各テーブルの定義は `docs/specs/pomodoro-spec-v1.0.md` §4 に従う。

---

## 5. マイルストーン・実装順序

### M1: 認証基盤 (FR-AUTH-001〜003)
```
backend/
  migrations/0001_create_users.sql
  internal/handler/auth.go
  internal/middleware/auth.go
  internal/service/auth.go
  internal/repository/user.go
frontend/
  src/pages/LoginPage.tsx        # SCR-001
  src/pages/RegisterPage.tsx     # SCR-002
  src/api/auth.ts
  src/store/authStore.ts
```

**検証:**
- `POST /api/v1/auth/register` → 201 Created + JWT
- `POST /api/v1/auth/login` → 200 OK + アクセストークン + Cookie
- `POST /api/v1/auth/refresh` → 200 OK + 新アクセストークン
- 無効トークン → 401 Unauthorized
- ログイン/登録画面の Vitest ユニットテスト

---

### M2: タイマー機能 (FR-TIMER-001〜004)
```
frontend/
  src/lib/timer/timerEngine.ts   # タイマーロジック
  src/lib/timer/useTimer.ts      # カスタムフック
  src/sw.ts                      # Service Worker (バックグラウンド動作)
  src/pages/TimerPage.tsx        # SCR-003
  src/store/timerStore.ts
```

**検証:**
- 25/5/15 分のサイクルが正確にカウントダウンされる
- バックグラウンド遷移後も残り時間が正確に維持される
- 開始/一時停止/リセット/スキップが正常動作する
- タイマーロジックの Vitest ユニットテスト

---

### M3: タスク管理 (FR-TASK-001〜003)
```
backend/
  migrations/0003_create_tasks.sql
  internal/handler/task.go
  internal/repository/task.go
frontend/
  src/pages/TaskPage.tsx         # SCR-004
  src/api/tasks.ts
  src/store/taskStore.ts
```

**検証:**
- タスク CRUD の REST API テスト (Go `testing`)
- タスク完了トグルの動作確認
- タスクとタイマーの関連付けテスト

---

### M4: 作業ログ・統計 (FR-LOG-001〜002)
```
backend/
  migrations/0004_create_pomodoro_sessions.sql
  internal/handler/session.go
  internal/repository/session.go
frontend/
  src/pages/HistoryPage.tsx      # SCR-005
  src/api/sessions.ts
  src/lib/pwa/backgroundSync.ts  # Background Sync
```

**検証:**
- セッション完了時に自動で API が呼ばれる
- オフライン → IndexedDB 保存 → Background Sync で同期
- `GET /api/v1/sessions/stats` で日別・週別の統計が取得できる

---

### M5: プッシュ通知 (FR-NOTIF-001〜002)
```
frontend/
  src/lib/notifications/push.ts  # VAPID + Push API
  src/lib/notifications/sound.ts # Web Audio API
  src/sw.ts                      # push イベントハンドラ追加
```

**検証:**
- 通知許可後、セッション完了時にブラウザ通知が届く
- 通知クリックでアプリがフォーカスされる
- 通知音のオン/オフ切り替えが機能する

---

### M6: 設定・PWA (FR-SETTINGS, NFR-PWA)
```
backend/
  migrations/0002_create_settings.sql
  internal/handler/settings.go
  internal/repository/settings.go
frontend/
  src/pages/SettingsPage.tsx     # SCR-006
  public/manifest.json           # PWA マニフェスト
  vite.config.ts                 # vite-plugin-pwa 設定
```

**検証:**
- ダークモード切り替えが設定に反映される
- 設定が API に永続化される
- Lighthouse PWA スコア ≥ 90
- `Add to Home Screen` が機能する

---

### M7: インフラ・CI/CD
```
infra/
  modules/cloudflare-workers/
  modules/cloudflare-pages/
  modules/cloudflare-d1/
  envs/dev/terragrunt.hcl
  envs/prod/terragrunt.hcl
.github/workflows/
  ci.yml                         # lint + build + test
  deploy-frontend.yml            # Cloudflare Pages
  deploy-backend.yml             # Cloudflare Workers
  terraform.yml                  # Terraform plan/apply
```

**検証:**
- PR 時に lint / typecheck / build / test が自動実行される
- `main` マージ時に prod 環境へ自動デプロイされる
- `develop` マージ時に dev 環境へ自動デプロイされる
- `terraform plan` が PR 時に実行される

---

## 6. テスト戦略

### フロントエンド
| テスト種別        | ツール                    | カバレッジ目標 |
| ---------------- | ------------------------- | -------------- |
| ユニット          | Vitest + React Testing Library | ≥ 70%     |
| E2E              | Playwright                | 主要フロー     |
| アクセシビリティ  | axe-core (Playwright 統合) | WCAG 2.1 AA   |
| PWA              | Lighthouse CI              | スコア ≥ 90   |

**主要 E2E フロー:**
1. ユーザー登録 → ログイン → タイマー開始 → セッション完了確認
2. タスク追加 → タイマー開始(タスク選択) → ポモドーロ数確認
3. オフライン時のタイマー動作確認

### バックエンド
| テスト種別  | ツール              | カバレッジ目標 |
| ---------- | ------------------- | -------------- |
| ユニット    | Go `testing` + `testify` | ≥ 70%     |
| 統合        | D1 テスト用バインディング | 主要エンドポイント |

---

## 7. CI/CD パイプライン

### PR 時 (`ci.yml`)
```yaml
steps:
  - Frontend: eslint + tsc + vitest
  - Backend:  go vet + golangci-lint + go test
  - Spec:     spec-traceability check (仕様参照の確認)
```

### main マージ時 (prod デプロイ)
```yaml
steps:
  - Frontend build → Cloudflare Pages deploy
  - Backend build (WASM) → Cloudflare Workers deploy
  - terraform apply (prod)
```

### develop マージ時 (dev デプロイ)
```yaml
steps:
  - Frontend build → Cloudflare Pages deploy (dev)
  - Backend build (WASM) → Cloudflare Workers deploy (dev)
  - terraform apply (dev)
```

---

## 8. PR ラベル・テンプレート

**ラベル:** `fr-auth`, `fr-timer`, `fr-task`, `fr-log`, `fr-notif`, `fr-settings`, `nfr-pwa`, `infra`, `ci-cd`, `spec`

**PR テンプレート必須項目:**
- 実装した仕様 ID (例: FR-TIMER-001, FR-TIMER-003)
- マイグレーションの有無
- スクリーンショット / GIF (UI 変更がある場合)
- テスト追加の有無
- 仕様からの逸脱とその理由

---

## 9. 非交渉事項 (Non-Negotiables)

- **Drizzle 不使用。** SQL マイグレーション + D1 ドライバのみ。
- **シークレットをリポジトリに含めない。**
- **全 API に認可チェックを実装する。**
- **Service Worker のバックグラウンドタイマーは必須機能。**
- **Lighthouse PWA スコア ≥ 90 を維持する。**

---

## 10. エージェント実行プロンプト (コピー&ペースト用)

> `docs/specs/pomodoro-spec-v1.0.md` に従い、  
> React + TypeScript + Vite (PWA) フロントエンド / Go + chi + syumai/workers バックエンド /  
> Cloudflare D1 (SQLite) / JWT 認証 を実装してください。  
> `docs/specs/orchestration-plan-v1.0.md` のマイルストーン M1→M7 の順で実装し、  
> 各マイルストーンの AC を満たした後に次へ進んでください。  
> シークレット類はリポジトリに含めず、GitHub Secrets で管理してください。
