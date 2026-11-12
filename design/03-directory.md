# 03. ディレクトリ構成

## API（`my-app`）
```
my-app/
├─ app/
│  ├─ main.py                    # アプリの組み立て（ルーター、例外ハンドラー、起動時のマイグレーション）
│  ├─ core/
│  │  ├─ config.py               # 設定（.env）
│  │  ├─ security.py             # パスワードのハッシュ化、セッショントークン
│  │  ├─ errors.py               # 業務エラーと HTTP エラーへの変換
│  │  └─ rate_limit.py           # ログイン・新規登録の回数制限
│  ├─ db/
│  │  ├─ connection.py           # 接続、PRAGMA 設定、get_db
│  │  └─ migrate.py              # migrations/*.sql を番号順に実行
│  ├─ entities/                  # 内部で使うデータ型（dataclass）
│  ├─ schemas/                   # API の入出力（Pydantic）
│  ├─ repositories/              # 生SQL（organization_id で必ず絞り込む）
│  ├─ services/                  # 業務ルール
│  └─ api/
│     ├─ deps.py                 # 共通の Depends
│     └─ v1/
│        ├─ router.py
│        └─ endpoints/           # auth / me / users / profiles / skills / master_data /
│                                # floors / rooms / presence / health
├─ migrations/                   # 0001_init.sql, 0002_xxx.sql ...
├─ tests/
│  ├─ conftest.py
│  ├─ unit/                      # service・DB のテスト
│  └─ api/                       # API のテスト
├─ docs/                         # 設計書
├─ data/                         # app.db、uploads/（git 管理外）
├─ requirements.txt / requirements-dev.txt
├─ pyproject.toml                # pytest・ruff の設定
└─ .env.example
```

## 画面（`my-app-web`）
```
my-app-web/
├─ *.html / admin/*.html      # 画面の入口（URL と同じパス。中身は <div id="app"> とスクリプトの読み込みだけ）
├─ src/
│  ├─ entries/                # 画面ごとの入口のスクリプト（ログインと権限を確認してから画面を表示する）
│  ├─ pages/                  # 画面（XxxPage.vue）
│  ├─ components/             # 画面の部品（ヘッダー、ダイアログ、プロフィールの入力欄など）
│  │  └─ map/                 # マップの描画（部屋・家具・座っている人）
│  ├─ api/                    # API の呼び出し（client.ts：fetch の共通処理、index.ts：リソースごとの呼び出し）
│  ├─ lib/                    # 共通の処理（ログイン確認、画面の表示、エラー、お知らせ、マップの計算）
│  ├─ types/                  # API の入出力の型
│  └─ styles/                 # base.css, components.css, map.css
├─ e2e/                       # 画面の自動テスト（Playwright）
├─ deploy/nginx/              # nginx の設定（office-seat/ は公開時もローカルでも使う、sites/ は公開サーバー用）
├─ scripts/                   # check-html.mjs（HTML の確認）、dev.mjs（開発用の起動）、nginx.mjs（ローカルの nginx の起動・停止）
├─ vite.config.ts             # ビルドの設定（PAGES：画面の一覧）、preview の設定（自動テスト用。/api の転送）
├─ security-headers.ts        # CSP などのセキュリティヘッダー
├─ playwright.config.ts
└─ package.json / package-lock.json
```

## ドキュメント（`my-app-docs`）
```
my-app-docs/
├─ README.md                  # 全体の案内・開発の手順
└─ design/                    # 設計書（01〜07）
```

## 命名規則
- Python：ファイル・関数は `snake_case`、クラスは `PascalCase`。ファイル名は `<リソース>_repository.py` / `<リソース>_service.py`
- API のパス：複数形の名詞、単語の区切りは `-`（例：`/job-types`）
- JSON のキー：`snake_case`
- TS：ファイルは `kebab-case.ts`、関数・変数は `camelCase`
- Vue：コンポーネントのファイルは `PascalCase.vue`。画面は `XxxPage.vue`
