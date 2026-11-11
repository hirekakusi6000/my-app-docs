# オフィス座席・従業員管理システム

オフィスの座席配置、出勤・在席状況、プロフィール、技術スキルを一元管理する Web アプリケーションです。
README と設計書はこのリポジトリにまとめています（コードのリポジトリの README は短い案内だけ）。

## リポジトリ

| リポジトリ | 内容 | 技術 |
|---|---|---|
| `my-app` | API | Python 3.12+ / FastAPI / SQLite（標準ライブラリ `sqlite3`、生SQL）/ Uvicorn / pip |
| `my-app-web` | 画面 | Vue 3 + Vite + TypeScript。移行前の画面は HTML / CSS / JavaScript（`legacy/`）で、画面ごとに順番に Vue へ移行中 |
| `my-app-docs` | README と設計書（このリポジトリ） | Markdown |

3つは同じフォルダに並べて置きます。画面の自動テストは、隣の `my-app` を使って API を起動します。

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
| [03. ディレクトリ](design/03-directory.md) | API と画面のディレクトリ構成、命名規則 |
| [04. データベース](design/04-database.md) | テーブル定義、マイグレーション |
| [05. API](design/05-api.md) | エンドポイントの一覧と入出力 |
| [06. 画面](design/06-screens.md) | 画面の一覧と動き |
| [07. 公開（デプロイ）](design/07-deployment.md) | 環境変数、リバースプロキシ、セキュリティヘッダー、ログ、バックアップ |

## 画面と API のつなぎ方

画面と API は**同じオリジン**（同じドメイン・ポート）で動かします。ログインの Cookie と CSP をそのまま使え、CORS の設定も要りません。

| 環境 | 画面 | `/api` |
|---|---|---|
| 開発（`npm run dev`） | Vite の開発サーバー http://127.0.0.1:5173 | Vite が FastAPI（http://127.0.0.1:8000）に転送 |
| 確認（`npm run preview`） | ビルドした `dist/` http://127.0.0.1:4173 | 同上 |
| 公開 | リバースプロキシが `dist/` を配信 | リバースプロキシが FastAPI に転送（[07. 公開](design/07-deployment.md)） |

API の URL は、画面のリポジトリで環境変数 `API_URL` を設定すると変えられます（既定は `http://127.0.0.1:8000`）。

## ローカル開発環境の構築（Windows / PowerShell）

### API（`my-app`）

```powershell
cd my-app

# 1. 仮想環境の作成と有効化
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. 依存ライブラリのインストール
pip install -r requirements-dev.txt

# 3. 設定ファイル（任意。なければ既定値で動きます）
Copy-Item .env.example .env
```

DB は起動時に自動で作成・マイグレーションされます。

### 画面（`my-app-web`）

Node.js 22.12 以上が必要です。

```powershell
cd my-app-web
npm install
npx playwright install chromium   # 画面の自動テスト用のブラウザ（初回のみ）
```

## 起動

VS Code で `office-seat.code-workspace` を開き、「ターミナル」→「タスクの実行」→「API と画面を起動」を選びます。
ターミナルで起動するときは、API と画面でターミナルを分けます。

```powershell
# ターミナル1：API
cd my-app
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload --reload-dir app

# ターミナル2：画面
cd my-app-web
npm run dev
```

- 画面: http://127.0.0.1:5173 （Vue の画面は保存するとすぐ反映される）
- API 仕様書: http://127.0.0.1:8000/docs
- 停止: それぞれのターミナルで Ctrl + C

## よく使うコマンド

### API（`my-app`）

```powershell
pytest                                 # テスト
pytest --cov=app                       # カバレッジ付きテスト
ruff check . ; ruff format .           # lint と整形
python -m app.db.migrate               # マイグレーションを手動で実行
python -m app.db.backup                # DB とプロフィール画像のバックアップ
```

### 画面（`my-app-web`）

```powershell
npm run dev            # 開発サーバー
npm run build          # 型チェックとビルド（dist/）
npm run preview        # ビルドした画面を、公開時と同じセキュリティヘッダー付きで確認
npm run typecheck      # 型チェックだけ
npm run check:legacy   # 移行前の画面の確認（文法・インラインスクリプト）
npm run test:e2e       # 画面の自動テスト（ビルドしてから実行）
npx playwright test --headed   # ブラウザを表示しながら自動テスト
```

## 画面の自動テスト

`my-app-web/e2e/` にあります（Playwright、TypeScript）。テストごとに、一時フォルダの DB を使う API サーバーと、ビルドした画面のサーバーを起動します。
画面は公開時と同じセキュリティヘッダー（CSP など）付きで開くので、CSP に違反した読み込みもエラーとして見つかります。

API のリポジトリは隣のフォルダ（`../my-app`）にあり、その `.venv` に依存ライブラリが入っている必要があります。場所が違うときは環境変数で指定します。

- `API_DIR`：API のリポジトリのフォルダ
- `API_PYTHON`：API を動かす Python（既定は `API_DIR` の `.venv` の Python）

## Vue の画面を追加・移行するとき

1. `my-app-web` のルートに URL と同じパスの HTML（例：`admin/organization.html`）、`src/pages/` に入口（`.ts`）と画面（`.vue`）を作る
2. `vite.config.ts` の `PAGES` に追加する
3. `legacy/` の同じ画面の HTML と `legacy/js/pages/` の JS を削除する

- 移行していない共通の処理（API の呼び出し・ヘッダー・ログイン確認など）は `@legacy/...` から読み込める
- CSP でインラインのスクリプト・スタイルを禁止しているので、CSS はファイル（または `.vue` の `<style>`）に書く

## CI（GitHub Actions）

`master` / `develop` / `feature/**` への push と、プルリクエストで自動実行します。

| リポジトリ | 内容 |
|---|---|
| `my-app` | lint と整形のチェック（ruff）、テスト（Python 3.12 と 3.14。カバレッジが 90% 未満なら失敗） |
| `my-app-web` | 移行前の画面の確認、型チェック・ビルド、画面の自動テスト（Playwright。失敗時はスクリーンショットと操作の記録を保存） |

画面の自動テストでは、API のリポジトリ（既定 `<オーナー>/my-app` の `develop`）を取得して起動します。
変えるときは `my-app-web` のリポジトリの変数 `API_REPOSITORY` / `API_REF` を、API のリポジトリが非公開のときはシークレット `API_REPOSITORY_TOKEN` を設定します。

ライブラリと GitHub Actions の更新は、Dependabot が `develop` 向けのプルリクエストで知らせます（`my-app`：pip、`my-app-web`：npm）。

## ブランチ運用（3つのリポジトリ共通）

- `master`: リリース用
- `develop`: 開発の統合用
- `feature/<名前>`: 作業ブランチ（必ず `develop` から作成し、`develop` に `--no-ff` でマージ。feature から feature は作らない）

コードを変えて README や設計書の更新が必要なときは、このリポジトリにも同じ名前の `feature/<名前>` を作って更新します。
