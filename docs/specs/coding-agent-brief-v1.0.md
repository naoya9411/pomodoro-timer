---
id: coding-agent-brief-v1-0
title: PWA ポモドーロタイマー — コーディングエージェントブリーフ (v1.0)
version: "1.0"
status: draft
created_at: "2026-02-24"
srs_ref: docs/specs/srs.yaml
spec_ref: docs/specs/pomodoro-spec-v1.0.md
---

# PWA ポモドーロタイマー — コーディングエージェントブリーフ (v1.0)

**リポジトリ:** `naoya9411/pomodoro-timer`  
**権威ある仕様:** `docs/specs/pomodoro-spec-v1.0.md`  
**実装ガイド:** `docs/specs/orchestration-plan-v1.0.md`

> 本ブリーフに従って v1.0 仕様を正確に実装すること。  
> 仕様との乖離がある場合は PR にその理由を明記すること。

---

## 0. グラウンドルール

- **フレームワーク:** React + TypeScript + Vite (frontend) / Go + chi + syumai/workers (backend)
- **ランタイム:** Cloudflare Workers (backend) / Cloudflare Pages (frontend)
- **データベース:** Cloudflare D1 (SQLite)。スキーマは SQL マイグレーションファイルで管理する。
- **認証:** JWT (email / password)。アクセストークンは 15 分、リフレッシュトークンは 7 日。
- **インフラ:** Terraform + Terragrunt (dev / prod 完全分離)
- **CI/CD:** GitHub Actions (lint, build, test, deploy)
- **セキュリティ:** JWT は HttpOnly Cookie 管理。シークレット類はリポジトリに含めない。
- **PWA:** Service Worker で バックグラウンドタイマー動作 + オフライン対応を実現する。

---

## 1. 環境変数

### フロントエンド (.env)
```
VITE_API_BASE_URL=https://api.pomodoro-timer.example.com
VITE_VAPID_PUBLIC_KEY=<VAPID公開鍵>
```

### バックエンド (Cloudflare Workers Secrets / wrangler.toml)
```
JWT_SECRET=<JWTシークレット>
VAPID_PRIVATE_KEY=<VAPID秘密鍵>
VAPID_SUBJECT=mailto:admin@pomodoro-timer.example.com
```

---

## 2. 作業分解 (マイルストーン + AC)

### M1 — 認証基盤 (FR-AUTH)
**実装内容:** JWT 認証 API + フロントエンド認証画面  
**AC:**
- `POST /api/v1/auth/register` でユーザー登録できる
- `POST /api/v1/auth/login` でアクセストークン + リフレッシュトークンが発行される
- `POST /api/v1/auth/refresh` でアクセストークンが更新される
- ログイン・登録画面 (SCR-001, SCR-002) が正常に動作する

### M2 — タイマー機能 (FR-TIMER)
**実装内容:** タイマーロジック + Service Worker + タイマー画面  
**AC:**
- 作業/休憩サイクルが正しくカウントダウンされる (FR-TIMER-001)
- バックグラウンド時もタイマーが継続する (FR-TIMER-003)
- 開始/一時停止/リセット/スキップ操作が正常に動作する (FR-TIMER-004)
- タイマー画面 (SCR-003) で現在のセッションタイプと残り時間が表示される

### M3 — タスク管理 (FR-TASK)
**実装内容:** タスク API + タスク管理画面  
**AC:**
- タスクの CRUD が正常に動作する (FR-TASK-001)
- タスクとセッションを関連付けられる (FR-TASK-002)
- タスク完了フラグをトグルできる (FR-TASK-003)
- タスク管理画面 (SCR-004) が正常に動作する

### M4 — 作業ログ・統計 (FR-LOG)
**実装内容:** セッション記録 API + 統計 API + 履歴画面  
**AC:**
- セッション完了時に自動で `POST /api/v1/sessions` が呼ばれる (FR-LOG-001)
- オフライン時は IndexedDB に保存し、Background Sync で同期する
- 今日/今週の統計が `GET /api/v1/sessions/stats` で取得できる (FR-LOG-002)
- 履歴・統計画面 (SCR-005) が正常に動作する

### M5 — プッシュ通知 (FR-NOTIF)
**実装内容:** VAPID キー設定 + Service Worker Push ハンドラ + 通知許可 UI  
**AC:**
- ユーザーが通知を許可後、セッション完了時に通知が届く (FR-NOTIF-001)
- 通知クリックでアプリにフォーカスする
- 通知音のオン/オフが設定から変更できる (FR-NOTIF-002)

### M6 — 設定・PWA (FR-SETTINGS, NFR-PWA)
**実装内容:** 設定 API + 設定画面 + PWA 設定  
**AC:**
- ダークモードが設定から切り替えられる (FR-SETTINGS-001)
- タイマー設定がサーバーに永続化される (FR-SETTINGS-002)
- Lighthouse PWA スコア ≥ 90 (NFR-PERF-001)
- ホーム画面へのインストールが可能 (NFR-PWA-001)
- オフラインでコア UI が使用できる (NFR-PWA-002)

### M7 — インフラ・CI/CD
**実装内容:** Terraform/Terragrunt + GitHub Actions  
**AC:**
- dev / prod 環境が Terragrunt で完全分離されている
- PR 時に lint / build / test が自動実行される
- main マージ時に Cloudflare Workers / Pages へ自動デプロイされる

---

## 3. ディレクトリ構成

```
/ (repo root)
├── frontend/                      # React + Vite + PWA
│   ├── src/
│   │   ├── components/            # UIコンポーネント
│   │   ├── pages/                 # 画面コンポーネント (SCR-001〜006)
│   │   ├── hooks/                 # カスタムフック
│   │   ├── store/                 # 状態管理 (Zustand 等)
│   │   ├── api/                   # API クライアント
│   │   ├── lib/
│   │   │   ├── timer/             # タイマーロジック
│   │   │   ├── pwa/               # Service Worker ユーティリティ
│   │   │   └── notifications/     # Web Push ユーティリティ
│   │   └── sw.ts                  # Service Worker エントリ
│   ├── public/
│   │   └── manifest.json          # PWA マニフェスト
│   └── vite.config.ts
│
├── backend/                       # Go + Cloudflare Workers
│   ├── cmd/worker/                # エントリポイント
│   ├── internal/
│   │   ├── handler/               # HTTP ハンドラ
│   │   ├── middleware/            # JWT 検証等
│   │   ├── model/                 # DB モデル
│   │   ├── repository/            # D1 アクセス層
│   │   └── service/               # ビジネスロジック
│   ├── migrations/                # SQL マイグレーション
│   └── wrangler.toml
│
├── infra/                         # Terraform + Terragrunt
│   ├── modules/                   # 再利用可能モジュール
│   ├── envs/
│   │   ├── dev/                   # dev 環境設定
│   │   └── prod/                  # prod 環境設定
│   └── terragrunt.hcl
│
├── docs/
│   └── specs/                     # Spec-kit ドキュメント
│       ├── srs.yaml
│       ├── pomodoro-spec-v1.0.md
│       ├── coding-agent-brief-v1.0.md
│       ├── orchestration-plan-v1.0.md
│       └── CHANGELOG-v1.0.md
│
└── .github/
    ├── workflows/                 # CI/CD + スペックトレーサビリティ
    └── PULL_REQUEST_TEMPLATE.md
```

---

## 4. 実装上の重要事項

### 認証
- パスワードは `bcrypt` (コスト 12) でハッシュ化する。
- JWT はアクセストークン (15 分) + リフレッシュトークン (7 日) の 2 トークン方式。
- リフレッシュトークンは `HttpOnly; Secure; SameSite=Strict` Cookie で管理する。

### Cloudflare D1 (SQLite)
- スキーマは `backend/migrations/` に SQL ファイルで管理する。
- Drizzle は使用しない。`database/sql` 互換のドライバを使用する。

### Service Worker / PWA
- `vite-plugin-pwa` (Workbox) を使用する。
- タイマーのバックグラウンド動作は Service Worker + `startTime` 保存方式で実装する。
- オフラインセッション同期は Background Sync API を使用する。

### Cloudflare Workers
- `syumai/workers` + `chi` でルーティングを構成する。
- D1 バインディングは `wrangler.toml` で設定する。

### テスト
- フロントエンド: Vitest + React Testing Library
- バックエンド: Go 標準の `testing` パッケージ
- E2E: Playwright (主要フロー: 登録 → ログイン → タイマー → ログ確認)

---

## 5. セキュリティ要件

- JWT シークレット、VAPID 秘密鍵はリポジトリに含めない (GitHub Secrets / Cloudflare Secrets で管理)。
- CORS は許可オリジンを明示的に設定する (ワイルドカード不可)。
- SQL インジェクション対策: プレースホルダを使用する。
- 全 API エンドポイントに適切な認可チェックを実装する。

---

## 6. 成果物と完了定義 (DoD)

- 全マイルストーン (M1〜M7) の AC を満たすコード
- SQL マイグレーションファイル
- Vitest / Go テスト (カバレッジ ≥ 70%)
- Playwright E2E テスト (主要フロー)
- Terraform / Terragrunt 設定
- GitHub Actions ワークフロー
- 更新済み `README.md`

---

### コピー&ペースト用エージェントプロンプト
> `docs/specs/pomodoro-spec-v1.0.md` に従い、React + TypeScript + Vite (PWA) フロントエンド / Go + chi + syumai/workers バックエンド / Cloudflare D1 (SQLite) / JWT 認証を実装してください。  
> 認証・タイマー・タスク管理・作業ログ・プッシュ通知・設定の全機能を含め、Service Worker によるバックグラウンド動作とオフライン対応を実装してください。  
> `docs/specs/orchestration-plan-v1.0.md` のマイルストーン順に実装し、`docs/specs/coding-agent-brief-v1.0.md` の DoD を満たしてください。
