---
id: changelog-v1-0
title: PWA ポモドーロタイマー — チェンジログ (v1.0)
version: "1.0"
created_at: "2026-02-24"
---

# PWA ポモドーロタイマー — チェンジログ (v1.0)

## 概要

v1.0 は PWA ポモドーロタイマーの初版リリースです。  
Spec-Driven Development (SDD) アプローチを採用し、SRS YAML を単一のソースオブトゥルースとして、以下の仕様・実装ドキュメントを整備します。

---

## 追加 (v1.0)

### Spec-kit ドキュメント
- `docs/specs/srs.yaml` — SRS (Software Requirements Specification) YAML を追加。全機能要件・非機能要件・データモデル・API 一覧を集約したシングルソース。
- `docs/specs/pomodoro-spec-v1.0.md` — SRS から生成されたメイン仕様書。機能仕様・受け入れ基準を詳述。
- `docs/specs/coding-agent-brief-v1.0.md` — コーディングエージェント向けブリーフ。技術スタック・グラウンドルール・マイルストーン・DoD を記述。
- `docs/specs/orchestration-plan-v1.0.md` — オーケストレーションプラン。実装順序・テスト戦略・CI/CD パイプラインを定義。
- `docs/specs/CHANGELOG-v1.0.md` — 本ファイル。

### GitHub インテグレーション
- `.github/PULL_REQUEST_TEMPLATE.md` — 仕様 ID 参照を必須とする PR テンプレート。
- `.github/workflows/spec-traceability.yml` — PR の仕様参照を検証する GitHub Actions ワークフロー。

---

## 定義された機能要件 (v1.0)

| 要件 ID          | 機能名                       | 優先度 |
| ---------------- | ---------------------------- | ------ |
| FR-AUTH-001      | メール＋パスワードによるユーザー登録 | must   |
| FR-AUTH-002      | ログイン / ログアウト         | must   |
| FR-AUTH-003      | JWT リフレッシュ              | should |
| FR-TIMER-001     | ポモドーロサイクル            | must   |
| FR-TIMER-002     | タイマーカスタマイズ          | should |
| FR-TIMER-003     | バックグラウンド動作          | must   |
| FR-TIMER-004     | タイマー操作                 | must   |
| FR-TASK-001      | タスクの CRUD                | must   |
| FR-TASK-002      | タスクとポモドーロの関連付け  | should |
| FR-TASK-003      | タスク完了                   | should |
| FR-LOG-001       | セッションの自動記録          | must   |
| FR-LOG-002       | 統計表示                     | should |
| FR-NOTIF-001     | Web Push 通知                | must   |
| FR-NOTIF-002     | 通知音                       | could  |
| FR-SETTINGS-001  | ダークモード                 | should |
| FR-SETTINGS-002  | タイマー設定の永続化          | must   |

---

## 定義された非機能要件 (v1.0)

| 要件 ID      | 内容                              |
| ------------ | --------------------------------- |
| NFR-PWA-001  | PWA インストール対応              |
| NFR-PWA-002  | オフライン対応                    |
| NFR-PERF-001 | Lighthouse スコア ≥ 90            |
| NFR-SEC-001  | HTTPS + HttpOnly Cookie           |
| NFR-A11Y-001 | WCAG 2.1 AA 準拠                  |
| NFR-RESP-001 | レスポンシブ (375px–1440px)       |

---

## Speckit 導入の目的・メリット

### 目的
1. **仕様の一元管理** — `srs.yaml` を Single Source of Truth とし、機能要件・データモデル・API を一箇所で管理する。
2. **PR トレーサビリティ** — PR テンプレートと GitHub Actions により、全 PR に仕様 ID を紐付け、変更の追跡可能性を確保する。
3. **コーディングエージェント対応** — エージェントブリーフとオーケストレーションプランにより、AI コーディングエージェントが仕様に沿った実装を自律的に行える環境を整備する。

### メリット
- **開発効率の向上** — 仕様・実装・テストの対応関係が明確になり、手戻りが減少する。
- **ドキュメント連携** — 仕様書・実装・PR が連携し、プロジェクトの状態を常に把握できる。
- **品質保証** — AC (受け入れ基準) が仕様に明記されており、テストの基準が統一される。
- **オンボーディング効率化** — 新規参加者やエージェントが仕様書から即座に実装を開始できる。

---

_End of changelog._
