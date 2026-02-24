# pomodoro-timer

PWA ポモドーロタイマー — ポモドーロ・テクニックに基づく生産性向上アプリ。

## 技術スタック

| レイヤー       | 技術                                           |
| ------------- | ---------------------------------------------- |
| フロントエンド | React + TypeScript + Vite + PWA (Cloudflare Pages) |
| バックエンド   | Go + chi + syumai/workers (Cloudflare Workers)  |
| データベース   | Cloudflare D1 (SQLite)                          |
| 認証           | JWT (email / password)                          |
| インフラ       | Terraform + Terragrunt (dev / prod)             |
| CI/CD         | GitHub Actions                                  |

## Spec-kit ドキュメント

本プロジェクトは **Spec-Driven Development (SDD)** を採用しています。  
`docs/specs/srs.yaml` を単一のソースオブトゥルースとして、以下のドキュメントを管理します。

| ドキュメント                                                             | 説明                                 |
| ----------------------------------------------------------------------- | ------------------------------------ |
| [`docs/specs/srs.yaml`](docs/specs/srs.yaml)                            | SRS YAML — 要件定義のシングルソース  |
| [`docs/specs/pomodoro-spec-v1.0.md`](docs/specs/pomodoro-spec-v1.0.md)  | メイン仕様書                         |
| [`docs/specs/coding-agent-brief-v1.0.md`](docs/specs/coding-agent-brief-v1.0.md) | コーディングエージェントブリーフ |
| [`docs/specs/orchestration-plan-v1.0.md`](docs/specs/orchestration-plan-v1.0.md) | オーケストレーションプラン       |
| [`docs/specs/CHANGELOG-v1.0.md`](docs/specs/CHANGELOG-v1.0.md)          | チェンジログ                         |

## コントリビューション

PR を作成する際は、`.github/PULL_REQUEST_TEMPLATE.md` に従い、  
実装した仕様 ID (`docs/specs/srs.yaml` 参照) を PR 本文に記載してください。