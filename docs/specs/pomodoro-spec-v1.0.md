---
id: pomodoro-spec-v1-0
title: PWA ポモドーロタイマー — 仕様書 (v1.0)
version: "1.0"
status: draft
created_at: "2026-02-24"
srs_ref: docs/specs/srs.yaml
---

# PWA ポモドーロタイマー — 仕様書 (v1.0)

> **権威ある仕様:** `docs/specs/srs.yaml`  
> **コーディングエージェントブリーフ:** `docs/specs/coding-agent-brief-v1.0.md`  
> **オーケストレーションプラン:** `docs/specs/orchestration-plan-v1.0.md`

---

## 0. スコープ・背景

本仕様書は PWA ポモドーロタイマーアプリ v1.0 の全機能・非機能要件を定義する。  
ポモドーロ・テクニック（作業 25 分 → 休憩 5 分のサイクル）をベースとした生産性向上ツールであり、タスク管理・作業ログ・プッシュ通知を統合する。

---

## 1. 技術スタック

| レイヤー        | 技術                                        |
| -------------- | ------------------------------------------ |
| フロントエンド  | React + TypeScript, Vite, vite-plugin-pwa  |
| ホスティング   | Cloudflare Pages                           |
| バックエンド    | Go (Cloudflare Workers, syumai/workers, chi) |
| データベース   | Cloudflare D1 (SQLite)                     |
| 認証           | JWT (email / password)                     |
| インフラ        | Terraform + Terragrunt (dev / prod)        |
| CI/CD         | GitHub Actions                             |

---

## 2. 機能仕様

### 2.1 ユーザー認証 (FR-AUTH)

| ID           | タイトル                        | 優先度 |
| ------------ | ------------------------------- | ------ |
| FR-AUTH-001  | メール＋パスワードによるユーザー登録 | must   |
| FR-AUTH-002  | ログイン / ログアウト            | must   |
| FR-AUTH-003  | JWT リフレッシュ                 | should |

#### FR-AUTH-001: ユーザー登録
- メールアドレス + パスワード（≥8 文字）で登録する。
- 既存メールで重複登録した場合は `409 Conflict` を返す。
- パスワードは bcrypt でハッシュ化して保存する。

#### FR-AUTH-002: ログイン / ログアウト
- 正しい認証情報でアクセストークン（15 分）＋リフレッシュトークン（7 日）を発行する。
- ログアウトでリフレッシュトークンを無効化する。
- 無効トークンでのアクセスは `401 Unauthorized` を返す。

#### FR-AUTH-003: JWT リフレッシュ
- アクセストークン期限切れ時、リフレッシュトークンで自動更新する。
- リフレッシュトークンは HttpOnly Cookie で管理する。

---

### 2.2 タイマー機能 (FR-TIMER)

| ID            | タイトル                    | 優先度 |
| ------------- | -------------------------- | ------ |
| FR-TIMER-001  | ポモドーロサイクル          | must   |
| FR-TIMER-002  | タイマーカスタマイズ         | should |
| FR-TIMER-003  | バックグラウンド動作         | must   |
| FR-TIMER-004  | タイマー操作                | must   |

#### FR-TIMER-001: ポモドーロサイクル
```
作業(25min) → 短休憩(5min) → 作業 → 短休憩 → ... → 長休憩(15min) → 繰り返し
```
- デフォルト: 作業 25 分、短休憩 5 分、長休憩 15 分、長休憩インターバル 4 サイクル。
- サイクル完了時にプッシュ通知を送信する。

#### FR-TIMER-002: タイマーカスタマイズ
| パラメータ         | 範囲     | デフォルト |
| ----------------- | -------- | --------- |
| 作業時間          | 1–60 分  | 25 分     |
| 短休憩時間         | 1–30 分  | 5 分      |
| 長休憩時間         | 1–60 分  | 15 分     |
| 長休憩インターバル  | 2–8 回   | 4 回      |

#### FR-TIMER-003: バックグラウンド動作
- Service Worker で `startTime` + `duration` を保存し、画面復帰時に残り時間を計算する。
- バックグラウンドでもセッション完了時に Web Push 通知を送信する。

#### FR-TIMER-004: タイマー操作
- **開始/一時停止:** ToggleButton  
- **リセット:** 現在のセッションを最初から再開  
- **スキップ:** 次のフェーズ（休憩/作業）へ即移行

---

### 2.3 タスク管理 (FR-TASK)

| ID           | タイトル                      | 優先度 |
| ------------ | ----------------------------- | ------ |
| FR-TASK-001  | タスクの CRUD                 | must   |
| FR-TASK-002  | タスクとポモドーロの関連付け   | should |
| FR-TASK-003  | タスク完了                    | should |

#### FR-TASK-001: CRUD
- タスク名を入力して追加 / 一覧表示 / 編集 / 削除できる。
- 認証済みユーザーの所有タスクのみ操作可能。

#### FR-TASK-002: タスクとポモドーロの関連付け
- タイマー開始前にタスクを選択できる（任意）。
- セッション完了時に選択タスクの `pomodoro_count` をインクリメントする。

#### FR-TASK-003: タスク完了
- `completed` フラグをトグルしてタスクを完了/未完了にする。
- 完了タスクはリスト末尾または別セクションに表示する。

---

### 2.4 作業ログ・統計 (FR-LOG)

| ID          | タイトル                    | 優先度 |
| ----------- | -------------------------- | ------ |
| FR-LOG-001  | ポモドーロセッションの自動記録 | must   |
| FR-LOG-002  | 統計表示                    | should |

#### FR-LOG-001: 自動記録
- セッション完了時に `POST /api/v1/sessions` を呼び出し、開始時刻・終了時刻・タスクIDを保存する。
- オフライン時は IndexedDB に一時保存し、オンライン復帰後にバックグラウンド同期を実行する。

#### FR-LOG-002: 統計表示
- 今日の完了ポモドーロ数・作業時間を表示する。
- 今週のサマリーを棒グラフで表示する。
- タスク別の消化ポモドーロ数を表示する。

---

### 2.5 プッシュ通知 (FR-NOTIF)

| ID            | タイトル      | 優先度 |
| ------------- | ------------ | ------ |
| FR-NOTIF-001  | Web Push 通知 | must   |
| FR-NOTIF-002  | 通知音        | could  |

#### FR-NOTIF-001: Web Push 通知
- ユーザーが通知を許可した場合のみ送信する。
- 作業セッション完了 → 「休憩しましょう！」
- 休憩セッション完了 → 「作業を再開しましょう！」
- 通知クリックでアプリをフォーカスする。

#### FR-NOTIF-002: 通知音
- 設定からオン/オフ切り替え（デフォルト: オン）。
- ブラウザの AudioContext API を使用する。

---

### 2.6 設定 (FR-SETTINGS)

| ID               | タイトル           | 優先度 |
| ---------------- | ----------------- | ------ |
| FR-SETTINGS-001  | ダークモード       | should |
| FR-SETTINGS-002  | タイマー設定の永続化 | must   |

---

## 3. 非機能要件

| ID           | 要件                                              |
| ------------ | ------------------------------------------------- |
| NFR-PWA-001  | PWA インストール (manifest.json, Service Worker)  |
| NFR-PWA-002  | オフライン対応 (コア UI はオフラインで使用可能)     |
| NFR-PERF-001 | Lighthouse スコア ≥ 90 (Performance/Accessibility/PWA) |
| NFR-SEC-001  | HTTPS 必須、JWT は HttpOnly Cookie                |
| NFR-A11Y-001 | WCAG 2.1 AA 準拠                                 |
| NFR-RESP-001 | レスポンシブ (375px–1440px)                       |

---

## 4. データモデル

### users
| カラム         | 型        | 制約               |
| ------------- | --------- | ------------------ |
| id            | TEXT      | PK, UUIDv4         |
| email         | TEXT      | UNIQUE, NOT NULL   |
| password_hash | TEXT      | NOT NULL           |
| created_at    | DATETIME  | NOT NULL           |
| updated_at    | DATETIME  | NOT NULL           |

### settings
| カラム                | 型        | デフォルト |
| -------------------- | --------- | --------- |
| user_id              | TEXT      | PK, FK→users |
| work_minutes         | INTEGER   | 25        |
| short_break_minutes  | INTEGER   | 5         |
| long_break_minutes   | INTEGER   | 15        |
| long_break_interval  | INTEGER   | 4         |
| sound_enabled        | INTEGER   | 1 (true)  |
| theme                | TEXT      | 'system'  |
| updated_at           | DATETIME  |           |

### tasks
| カラム          | 型        | 制約             |
| -------------- | --------- | ---------------- |
| id             | TEXT      | PK, UUIDv4       |
| user_id        | TEXT      | FK→users, NOT NULL |
| title          | TEXT      | NOT NULL         |
| completed      | INTEGER   | DEFAULT 0        |
| pomodoro_count | INTEGER   | DEFAULT 0        |
| created_at     | DATETIME  | NOT NULL         |
| updated_at     | DATETIME  | NOT NULL         |

### pomodoro_sessions
| カラム      | 型        | 制約                                  |
| ---------- | --------- | ------------------------------------- |
| id         | TEXT      | PK, UUIDv4                            |
| user_id    | TEXT      | FK→users, NOT NULL                    |
| task_id    | TEXT      | FK→tasks, NULL許可                    |
| type       | TEXT      | work \| short_break \| long_break     |
| started_at | DATETIME  | NOT NULL                              |
| ended_at   | DATETIME  | NULL許可                              |
| completed  | INTEGER   | DEFAULT 0                             |
| created_at | DATETIME  | NOT NULL                              |

---

## 5. API エンドポイント一覧

`Base URL: /api/v1`

### 認証
| メソッド | パス            | 説明               | 認証要否 |
| ------- | --------------- | ------------------ | ------- |
| POST    | /auth/register  | ユーザー登録        | 不要    |
| POST    | /auth/login     | ログイン・JWT発行   | 不要    |
| POST    | /auth/logout    | ログアウト          | 必要    |
| POST    | /auth/refresh   | トークンリフレッシュ | 不要    |

### 設定
| メソッド | パス        | 説明     | 認証要否 |
| ------- | ----------- | -------- | ------- |
| GET     | /settings   | 設定取得  | 必要    |
| PUT     | /settings   | 設定更新  | 必要    |

### タスク
| メソッド | パス           | 説明           | 認証要否 |
| ------- | -------------- | -------------- | ------- |
| GET     | /tasks         | タスク一覧取得  | 必要    |
| POST    | /tasks         | タスク作成      | 必要    |
| PUT     | /tasks/{id}    | タスク更新      | 必要    |
| DELETE  | /tasks/{id}    | タスク削除      | 必要    |

### セッション
| メソッド | パス              | 説明               | 認証要否 |
| ------- | ----------------- | ------------------ | ------- |
| GET     | /sessions         | セッション履歴取得   | 必要    |
| POST    | /sessions         | セッション記録      | 必要    |
| GET     | /sessions/stats   | 統計情報取得        | 必要    |

---

## 6. 画面構成

| ID      | 画面名            | ルート      |
| ------- | ---------------- | ----------- |
| SCR-001 | ログイン画面       | /login      |
| SCR-002 | ユーザー登録画面   | /register   |
| SCR-003 | タイマー画面 (メイン) | /         |
| SCR-004 | タスク管理画面     | /tasks      |
| SCR-005 | 履歴・統計画面     | /history    |
| SCR-006 | 設定画面          | /settings   |

---

## 7. 受け入れ基準 (高レベル)

- [ ] 登録・ログイン・ログアウトが正常に動作する
- [ ] タイマーが作業/休憩サイクルを正しくカウントダウンする
- [ ] バックグラウンドでもタイマーが継続し、完了時に通知される
- [ ] タスクの CRUD 操作が正常に動作する
- [ ] セッション完了時に作業ログが自動保存される
- [ ] 統計画面に日別・週別のポモドーロ数が表示される
- [ ] ダークモードが設定から切り替えられる
- [ ] Lighthouse PWA スコアが 90 以上
- [ ] オフライン時もコア UI が使用できる

---

_End of specification._
