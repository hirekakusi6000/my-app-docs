# オフィス座席・従業員管理システム

オフィスの座席配置、出勤・在席状況、プロフィール、技術スキルを一元管理する Web アプリケーションです。
README と設計書はこのリポジトリにまとめています（コードのリポジトリの README は短い案内だけ）。

## リポジトリ

| リポジトリ | 内容 | 主な技術（詳しくは「技術スタック」） |
|---|---|---|
| `my-app` | API | Python / FastAPI / SQLite |
| `my-app-web` | 画面 | Vue 3 + Vite + TypeScript、nginx の設定 |
| `my-app-docs` | README と設計書（このリポジトリ） | Markdown |

3つは同じフォルダに並べて置きます。画面の自動テストは、隣の `my-app` を使って API を起動します。

```
projects\
├─ my-app\                       # API
├─ my-app-web\                   # 画面
├─ my-app-docs\                  # ドキュメント
└─ office-seat.code-workspace    # VS Code で3つをまとめて開くワークスペース
```

## 技術スタック

バージョンは `my-app` の `requirements.txt` / `requirements-dev.txt` と、`my-app-web` の `package.json` で固定しているもの。

### 全体の構成

```
ブラウザ ──▶ nginx ──┬─ 画面（Vue 3 + TypeScript をビルドした静的ファイル）
                     └─ /api ──▶ Uvicorn + FastAPI ──▶ SQLite
```

開発時も公開時と同じ nginx が入口になる（違いは HTTPS かどうかとポートだけ。[画面と API のつなぎ方](#画面と-api-のつなぎ方)）。

### API（`my-app`）

| 技術 | バージョン | 用途 |
|---|---|---|
| Python | 3.14（3.12 以上で動作） | 開発言語 |
| FastAPI | 0.162.0 | Web フレームワーク（REST API、入力チェック、API 仕様書の自動生成） |
| Uvicorn | 0.54.0 | ASGI サーバー（API の起動） |
| Pydantic | 2.x（FastAPI に同梱） | API の入出力の型と入力チェック |
| pydantic-settings | 2.16.2 | 設定の読み込み（環境変数・`.env`） |
| SQLite（`sqlite3`） | Python 標準 | データベース。ORM は使わず生 SQL で操作 |
| argon2-cffi | 25.1.0 | パスワードのハッシュ化（Argon2） |
| email-validator | 2.3.0 | メールアドレスの形式チェック |
| python-multipart | 0.0.33 | ファイルのアップロード（プロフィール画像・CSV） |

### 画面（`my-app-web`）

| 技術 | バージョン | 用途 |
|---|---|---|
| Vue | 3.5.45 | 画面のフレームワーク（Composition API、`<script setup>`） |
| TypeScript | 5.9.3 | 型付きの JavaScript（`strict`） |
| Vite | 8.3.8 | ビルド（複数ページ）。開発時は保存するたびに自動でビルドし直す。画面の自動テストでは preview で画面を返す |
| @vitejs/plugin-vue | 6.0.10 | Vite で `.vue` ファイルを扱う |
| vue-tsc | 3.3.14 | `.vue` を含めた型チェック |
| Node.js | 22.12 以上 | ビルド・テスト・開発用の起動スクリプトの実行 |
| HTML / CSS / JavaScript（`legacy/`） | ― | まだ Vue に移行していない画面（ES モジュール、Fetch API、`<dialog>`、CSS カスタムプロパティ） |
| SVG | ― | オフィスマップの描画と編集（ドラッグ、範囲選択、回転） |

### Web サーバー・公開

| 技術 | バージョン | 用途 |
|---|---|---|
| nginx | 1.30（安定版） | 入口（開発時も公開時も）。画面のファイルの配信、`/api` の転送、セキュリティヘッダー、公開時は HTTPS（設定は `my-app-web/deploy/nginx/`） |
| Let's Encrypt（certbot） | ― | HTTPS の証明書 |
| systemd | ― | API（Uvicorn）を常駐させる |
| Caddy | ― | nginx の代わりに使う場合の設定例のみ |

### 設計・アーキテクチャ

詳しくは [02. アーキテクチャ](design/02-architecture.md) を参照。

| 項目 | 内容 |
|---|---|
| リポジトリの分け方 | API・画面・ドキュメントの3つ。画面と API は同じオリジンで動かす（CORS 不要） |
| レイヤー構成（API） | router → service（業務ルール）→ repository（SQL）の一方向。DB アクセスと業務ロジックを分離 |
| マルチテナント | すべてのデータを `organization_id` で組織ごとに分離。他の組織のデータは 404 |
| マイグレーション | 番号付きの SQL ファイル（`migrations/`）＋ `PRAGMA user_version` による自作の仕組み |
| 認証 | セッション Cookie（`HttpOnly`・`SameSite=Lax`）。トークンは DB にハッシュ値で保存 |
| 認可 | FastAPI の `Depends` による権限チェック（マスター／一般、初回設定の完了） |
| データの保護 | UNIQUE・CHECK・外部キーの制約（座席の重複防止など）、プレースホルダーによる SQL インジェクション対策 |
| 画面の移行 | 1画面ずつ Vue に移行。移行前の部品（API の呼び出し・ヘッダーなど）は `@legacy` から再利用 |

### セキュリティ・運用

詳しくは [07. 公開](design/07-deployment.md) を参照。

| 項目 | 内容 |
|---|---|
| セキュリティヘッダー | CSP（インラインのスクリプト・スタイルを禁止）、X-Frame-Options、HSTS（HTTPS 時）、Referrer-Policy、Permissions-Policy。API は FastAPI、画面は nginx が付ける |
| 試行回数の制限 | ログインと新規登録（メモリ上のスライディングウィンドウ） |
| ログ | Python 標準の `logging`。リクエストID、アクセスログ、監査ログ、ファイルのローテーション |
| バックアップ | SQLite のバックアップ機能（動作中でも安全）＋世代管理（`python -m app.db.backup`） |

### テスト・品質

| 技術 | バージョン | 用途 |
|---|---|---|
| pytest | 9.2.0 | API のテスト（単体・API。カバレッジ約 97%） |
| pytest-cov | 7.1.0 | カバレッジの計測（CI では 90% 未満で失敗） |
| httpx（FastAPI の `TestClient`） | 0.28.1 | API のテスト |
| Ruff | 0.17.4 | Python の lint と整形 |
| Playwright（`@playwright/test`） | 1.65.0 | 画面の自動テスト（Chromium。テストごとに API と画面を起動し、CSP の違反も検出） |
| vue-tsc | 3.3.14 | 画面の型チェック |
| `check-legacy.mjs`（`node --check`） | ― | 移行前の画面の JavaScript の文法と、インラインスクリプトがないことの確認 |
| `.gitattributes`（3つ共通）/ `.editorconfig`（`my-app`） | ― | 文字コード（UTF-8）・改行コード（LF）の統一 |

### 開発環境・CI

| 技術 | 用途 |
|---|---|
| pip / venv | Python のライブラリの管理と仮想環境 |
| npm | 画面のライブラリの管理（`package-lock.json` で固定） |
| VS Code（`office-seat.code-workspace`） | 3つのリポジトリをまとめて開く。API・画面・nginx の起動とテストをタスクで実行 |
| Git | バージョン管理。`master` ← `develop` ← `feature/*`、`--no-ff` でマージ |
| GitHub Actions | `my-app`：lint・テスト（Python 3.12 / 3.14）、`my-app-web`：型チェック・ビルド・画面の自動テスト |
| Dependabot | ライブラリ（pip・npm）と GitHub Actions の更新のお知らせ |

## 設計書

| 設計書 | 内容 |
|---|---|
| [01. 要件](design/01-requirements.md) | 機能・権限・制限・技術構成 |
| [02. アーキテクチャ](design/02-architecture.md) | 全体の構成、認証・認可、層の分け方 |
| [03. ディレクトリ](design/03-directory.md) | API と画面のディレクトリ構成、命名規則 |
| [04. データベース](design/04-database.md) | テーブル定義、マイグレーション |
| [05. API](design/05-api.md) | エンドポイントの一覧と入出力 |
| [06. 画面](design/06-screens.md) | 画面の一覧と動き |
| [07. 公開（デプロイ）](design/07-deployment.md) | 環境変数、nginx（リバースプロキシ・HTTPS）、セキュリティヘッダー、ログ、バックアップ |

## 画面と API のつなぎ方

画面と API は**同じオリジン**（同じドメイン・ポート）で動かします。ログインの Cookie と CSP をそのまま使え、CORS の設定も要りません。

開発時も公開時も、入口は同じ nginx の設定（`my-app-web/deploy/nginx/office-seat/`）です。開発中から CSP などのセキュリティヘッダーが公開時と同じように付くので、違反があればすぐ分かります。

| 環境 | 画面 | `/api` |
|---|---|---|
| 開発（`npm run dev`） | nginx が `dist/` を配信 http://127.0.0.1:8080（保存するたびに自動でビルドし直す） | nginx が FastAPI（http://127.0.0.1:8000）に転送 |
| 公開 | nginx が `dist/` を配信（HTTPS） | nginx が FastAPI に転送（[07. 公開](design/07-deployment.md)） |
| 画面の自動テスト | Vite の preview が `dist/` を配信（同じセキュリティヘッダー付き） | Vite がテスト用の FastAPI に転送 |

自動テストだけ nginx を使わないのは、CI（GitHub Actions）に nginx を入れずに済ませるためです。

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

Node.js 22.12 以上と nginx が必要です。

1. nginx for Windows（[nginx.org](https://nginx.org/en/download.html) の Stable 版の zip）を `%LOCALAPPDATA%\Programs\nginx-<版>\` に展開する
2. ライブラリと、画面の自動テスト用のブラウザを入れる

```powershell
cd my-app-web
npm install
npx playwright install chromium
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

- 画面: http://127.0.0.1:8080 （`npm run dev` が、ビルド → nginx の起動 → 保存するたびに自動でビルドし直す、までを行う。ブラウザは再読み込みで反映）
- API 仕様書: http://127.0.0.1:8000/docs （nginx 経由の http://127.0.0.1:8080/docs でも開ける）
- 停止: それぞれのターミナルで Ctrl + C（`npm run dev` を止めると nginx も止まる）
- VS Code のタスクをゴミ箱のボタンで終わらせると nginx が残ることがあります。そのときは「nginx を止める」タスクか `npm run nginx:stop` で止めます

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
npm run dev            # 開発用の起動（ビルド → nginx → 保存するたびに自動でビルド。http://127.0.0.1:8080）
npm run build          # 型チェックとビルド（公開用の dist/）
npm run typecheck      # 型チェックだけ
npm run check:legacy   # 移行前の画面の確認（文法・インラインスクリプト）
npm run test:e2e       # 画面の自動テスト（ビルドしてから実行）
npm run nginx:reload   # deploy/nginx/ の設定を変えたあと、nginx に読み込み直させる
npm run nginx:stop     # nginx が残っていたら止める
npx playwright test --headed   # ブラウザを表示しながら自動テスト
```

## 開発時の nginx について

- 設定は公開時と同じ `my-app-web/deploy/nginx/office-seat/` を使います。違うのは、ポート（8080）と HTTPS を使わないことだけです
- nginx の作業フォルダ（設定・ログ）は `%LOCALAPPDATA%\office-seat-nginx` に作られます（日本語を含むパスを避けるため）。ログは `logs\access.log` と `logs\error.log`
- `dist/` はジャンクションでつながっているので、自動のビルドの結果がそのまま返ります
- 開発時のビルドにはソースマップが付くので、ブラウザの開発者ツールで元の `.vue` / `.ts` を見られます
- nginx の場所が違うときは環境変数 `NGINX_PATH`、ポートを変えるときは `NGINX_PORT` で指定します

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
