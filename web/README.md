# 画面（リポジトリ `my-app-web`）

オフィスの座席配置・在席状況・プロフィール・スキルを管理する Web アプリの画面です。
API（FastAPI）は別のリポジトリ（`my-app`）にあります → [API の README](../api/README.md)。設計書は [design/](../design/) にあります。

## 技術構成

- Vue 3 + Vite + TypeScript
- 移行前の画面は HTML / CSS / JavaScript（`legacy/`）。画面ごとに順番に Vue へ移行中
- 画面の自動テスト：Playwright（`@playwright/test`）

## 画面と API のつなぎ方

画面と API は**同じオリジン**（同じドメイン・ポート）で動かします。ログインの Cookie と CSP をそのまま使え、CORS の設定も要りません。

| 環境 | 画面 | `/api` |
|---|---|---|
| 開発（`npm run dev`） | Vite の開発サーバー http://127.0.0.1:5173 | Vite が FastAPI（http://127.0.0.1:8000）に転送 |
| 確認（`npm run preview`） | ビルドした `dist/` http://127.0.0.1:4173 | 同上 |
| 公開 | リバースプロキシが `dist/` を配信 | リバースプロキシが FastAPI に転送 |

API の URL は環境変数 `API_URL` で変えられます（既定は `http://127.0.0.1:8000`）。

## ローカル開発環境（Windows / PowerShell）

Node.js 22.12 以上が必要です。

```powershell
npm install
```

起動は、API と画面でターミナルを分けます。

```powershell
# ターミナル1：API（API のリポジトリで）
cd ..\my-app
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload --reload-dir app

# ターミナル2：画面（このリポジトリで）
npm run dev
```

ブラウザで http://127.0.0.1:5173 を開きます。Vue の画面は保存するとすぐ反映されます。

## よく使うコマンド

```powershell
npm run dev            # 開発サーバー
npm run build          # 型チェックとビルド（dist/）
npm run preview        # ビルドした画面を、公開時と同じセキュリティヘッダー付きで確認
npm run typecheck      # 型チェックだけ
npm run check:legacy   # 移行前の画面の確認（文法・インラインスクリプト）
npm run test:e2e       # 画面の自動テスト（ビルドしてから実行）
npx playwright test --headed   # ブラウザを表示しながら自動テスト
```

### 画面の自動テスト

テストごとに、一時フォルダの DB を使う API サーバーを起動します。API のリポジトリが隣のフォルダ（`../my-app`）にあり、その `.venv` に依存ライブラリが入っている必要があります。
場所が違うときは環境変数で指定します。

- `API_DIR`：API のリポジトリのフォルダ
- `API_PYTHON`：API を動かす Python（既定は `API_DIR` の `.venv` の Python）

テスト用のブラウザは、初回だけ `npx playwright install chromium` でインストールします。

## Vue の画面を追加・移行するとき

1. ルートに URL と同じパスの HTML（例：`admin/organization.html`）、`src/pages/` に入口（`.ts`）と画面（`.vue`）を作る
2. `vite.config.ts` の `PAGES` に追加する
3. `legacy/` の同じ画面の HTML と `legacy/js/pages/` の JS を削除する

- 移行していない共通の処理（API の呼び出し・ヘッダー・ログイン確認など）は `@legacy/...` から読み込める
- CSP でインラインのスクリプト・スタイルを禁止しているので、CSS はファイル（または `.vue` の `<style>`）に書く

## ディレクトリ

```
├─ admin/*.html          # Vue の画面の入口（URL と同じパス）
├─ src/
│  ├─ pages/             # 画面ごとの入口（.ts）と画面（.vue）
│  ├─ api/               # API の呼び出し（型付き）
│  ├─ lib/               # 共通の処理
│  └─ types/             # API の応答の型
├─ legacy/               # 移行前の画面（そのまま dist/ にコピーされる）
├─ e2e/                  # 画面の自動テスト
├─ scripts/              # 確認用のスクリプト
├─ vite.config.ts        # ビルド・開発サーバーの設定（PAGES、/api の転送）
├─ security-headers.ts   # CSP などのセキュリティヘッダー
└─ playwright.config.ts
```

## 公開

`npm ci && npm run build` で作った `dist/` を、リバースプロキシで配信します。
`/api/` は FastAPI に転送し、画面には `security-headers.ts` と同じヘッダーを付けます。

Caddy の例：

```
example.com {
	handle /api/* {
		reverse_proxy 127.0.0.1:8000
	}
	handle {
		root * /srv/office-web/dist
		file_server
		header {
			Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' blob: data:; connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'"
			X-Content-Type-Options nosniff
			X-Frame-Options DENY
			Referrer-Policy same-origin
			Permissions-Policy "camera=(), microphone=(), geolocation=()"
			Cross-Origin-Opener-Policy same-origin
			Strict-Transport-Security "max-age=31536000; includeSubDomains"
			Cache-Control no-cache
		}
	}
}
```

API 側の設定は [design/07-deployment.md](../design/07-deployment.md) を参照してください。

## CI（GitHub Actions）

画面のリポジトリの `.github/workflows/ci.yml`：移行前の画面の確認、型チェック・ビルド、画面の自動テスト。
自動テストでは API のリポジトリ（既定 `<オーナー>/my-app` の `develop`）を取得して起動します。
変えるときはリポジトリの変数 `API_REPOSITORY` / `API_REF` を、API のリポジトリが非公開のときはシークレット `API_REPOSITORY_TOKEN` を設定します。

## ブランチ運用

- `master`: リリース用
- `develop`: 開発の統合用
- `feature/<名前>`: 機能ごとの作業ブランチ（`develop` から作成し、`develop` にマージ）
