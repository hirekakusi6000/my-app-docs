# オフィス座席・従業員管理システム（ドキュメント）

オフィスの座席配置、出勤・在席状況、プロフィール、技術スキルを一元管理する Web アプリケーションのドキュメントです。
README と設計書はこのリポジトリにまとめています。コードのリポジトリの README は短い案内だけです。

## リポジトリ

| リポジトリ | 内容 | 説明 |
|---|---|---|
| `my-app` | API（Python / FastAPI / SQLite） | [API の README](api/README.md) |
| `my-app-web` | 画面（Vue 3 + Vite + TypeScript） | [画面の README](web/README.md) |
| `my-app-docs` | ドキュメント（このリポジトリ） | — |

3つは同じフォルダ（例：`projects\`）に並べて置きます。画面の自動テストは、隣の `my-app` を使って API を起動します。

```
projects\
├─ my-app\                       # API
├─ my-app-web\                   # 画面
├─ my-app-docs\                  # ドキュメント
└─ office-seat.code-workspace    # VS Code で3つをまとめて開くワークスペース
```

## 設計書

| 設計書 | 内容 |
|---|---|
| [01. 要件](design/01-requirements.md) | 機能・権限・制限・技術構成 |
| [02. アーキテクチャ](design/02-architecture.md) | 全体の構成、認証・認可、層の分け方 |
| [03. ディレクトリ](design/03-directory.md) | API のディレクトリ構成、命名規則 |
| [04. データベース](design/04-database.md) | テーブル定義、マイグレーション |
| [05. API](design/05-api.md) | エンドポイントの一覧と入出力 |
| [06. 画面](design/06-screens.md) | 画面の一覧と動き |
| [07. 公開（デプロイ）](design/07-deployment.md) | 環境変数、リバースプロキシ、ログ、バックアップ |

## 開発をはじめる

1. API の準備 → [API の README の「ローカル開発環境の構築」](api/README.md#ローカル開発環境の構築windows--powershell)
2. 画面の準備 → [画面の README の「ローカル開発環境」](web/README.md#ローカル開発環境windows--powershell)
3. VS Code で `office-seat.code-workspace` を開き、「ターミナル」→「タスクの実行」→「API と画面を起動」
4. ブラウザで http://127.0.0.1:5173 を開く

画面と API は同じオリジンで動かします（開発時は Vite が `/api` を FastAPI に転送し、公開時はリバースプロキシで振り分ける）。

## ブランチ運用（3つのリポジトリ共通）

- `master`: リリース用
- `develop`: 開発の統合用
- `feature/<名前>`: 作業ブランチ（必ず `develop` から作成し、`develop` に `--no-ff` でマージ。feature から feature は作らない）

コードを変えて README や設計書の更新が必要なときは、このリポジトリにも同じ名前の `feature/<名前>` を作って更新します。
