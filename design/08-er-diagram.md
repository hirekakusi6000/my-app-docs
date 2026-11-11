# 08. ER図

データベースのテーブルと、テーブルどうしの関係の図。マイグレーション `0002_drop_troubles.sql` まで適用した状態。
テーブルの説明は [04. DB設計](04-database.md)、実際の定義は API のリポジトリの `migrations/` を参照。

図は Mermaid で書いている。GitHub ではそのまま図として表示される。VS Code では Markdown のプレビュー（Ctrl + Shift + V）で表示できる（拡張機能「Markdown Preview Mermaid Support」が必要）。

## 全体

```mermaid
erDiagram
    organizations ||--o{ users : "所属する"
    organizations ||--o{ positions : "候補"
    organizations ||--o{ job_types : "候補"
    organizations ||--o{ skills : "候補"
    organizations ||--o{ floors : "持つ"
    floors ||--o{ rooms : "持つ"
    rooms ||--o{ furniture : "置く"

    users ||--o{ sessions : "ログイン"
    users ||--o| profiles : "プロフィール"
    users ||--o| presence : "在席状態"
    users ||--o{ user_skills : "持つ"
    skills ||--o{ user_skills : "持たれる"

    positions |o--o{ profiles : "役職"
    job_types |o--o{ profiles : "職種"
    furniture |o--o{ profiles : "デフォルト座席"
    furniture |o--o| presence : "今の座席"

    organizations {
        INTEGER id PK
        TEXT name "1〜100文字"
        TEXT created_at
    }
    users {
        INTEGER id PK
        INTEGER organization_id FK
        TEXT email UK "システム全体で重複不可（大文字・小文字を区別しない）"
        TEXT password_hash "Argon2"
        TEXT role "master / general"
        INTEGER must_change_password "0 / 1"
        TEXT created_at
        TEXT updated_at
    }
    sessions {
        TEXT token_hash PK "トークンの SHA-256"
        INTEGER user_id FK
        TEXT expires_at
        TEXT created_at
    }
    profiles {
        INTEGER user_id PK, FK
        TEXT name "1〜50文字"
        TEXT birth_date "YYYY-MM-DD（年齢はここから計算）"
        INTEGER position_id FK "削除されたら NULL"
        INTEGER job_type_id FK "削除されたら NULL"
        TEXT avatar_path "プロフィール画像"
        INTEGER default_seat_id FK "削除されたら NULL"
        TEXT updated_at
    }
    positions {
        INTEGER id PK
        INTEGER organization_id FK
        TEXT name "組織の中で重複不可"
    }
    job_types {
        INTEGER id PK
        INTEGER organization_id FK
        TEXT name "組織の中で重複不可"
        INTEGER is_engineer "0 / 1"
    }
    skills {
        INTEGER id PK
        INTEGER organization_id FK
        TEXT category "language / framework / database / cloud"
        TEXT name "組織・分類の中で重複不可"
    }
    user_skills {
        INTEGER user_id PK, FK
        INTEGER skill_id PK, FK
        INTEGER years "0〜50"
    }
    floors {
        INTEGER id PK
        INTEGER organization_id FK
        TEXT name "組織の中で重複不可"
        INTEGER sort_order
    }
    rooms {
        INTEGER id PK
        INTEGER organization_id FK
        INTEGER floor_id FK
        TEXT name "フロアの中で重複不可"
        INTEGER width "5〜200マス"
        INTEGER height "5〜200マス"
        INTEGER sort_order
    }
    furniture {
        INTEGER id PK
        INTEGER organization_id FK
        INTEGER room_id FK
        TEXT type "desk / chair / wall / door / other（chair が座席）"
        TEXT label "例：A-1"
        INTEGER x
        INTEGER y
        INTEGER width
        INTEGER height
        INTEGER rotation "0 / 90 / 180 / 270"
    }
    presence {
        INTEGER user_id PK, FK
        INTEGER organization_id FK
        TEXT status "not_working / present / away / remote"
        INTEGER seat_id FK, UK "1つの座席に1人だけ"
        TEXT comment "100文字まで"
        TEXT updated_at
    }
```

## 記号の読み方

| 記号 | 意味 |
|---|---|
| `\|\|--o{` | 1 対 0以上（例：1つの組織に、ユーザーが0人以上） |
| `\|\|--o\|` | 1 対 0か1（例：1人のユーザーに、プロフィールは0か1つ） |
| `\|o--o{` | 0か1 対 0以上（例：役職は、0人以上のプロフィールから選ばれる。プロフィール側は役職なしでもよい） |
| `\|o--o\|` | 0か1 対 0か1（例：1つの座席に、在席している人は0か1人） |
| PK / FK / UK | 主キー / 外部キー / 重複不可 |

## 図だけでは分からないルール

- **組織ごとの分離**：`organization_id` を持つテーブルは、必ず組織で絞り込んで読み書きする（図では、`organizations` から `rooms`・`furniture`・`presence` への線は省略している）
- **削除の連鎖**：組織を削除すると、その組織のデータはすべて削除される（`ON DELETE CASCADE`）。ユーザーを削除すると、セッション・プロフィール・スキル・在席状態も削除される
- **候補・座席の削除**：役職・職種・デフォルト座席が削除されたら、プロフィールの側は空（NULL）になる
- **在席状態と座席**：`present`（在席中）・`away`（離席中）のときだけ `seat_id` が入る（CHECK 制約）。`seat_id` は UNIQUE なので、同じ座席に2人は座れない
- **初回登録**：`profiles` の行があれば、プロフィールの初回登録が済んでいる
- **マスター**：マスターはプロフィールと在席状態を持たない（マップに表示しない）
