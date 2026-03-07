# Go SQLテスト実装ガイド（PostgreSQL）

## 目的
このガイドは、PostgreSQL を対象に SQL ファイル分離、スキーマ投入、初期データ管理、Go 統合テストを進めるための実装方針を定義する。

## 基本方針
- SQL は Go コードに直書きせず `.sql` ファイルとして管理する
- DB スキーマの正本は共通 SQL に寄せ、PostgreSQL 固有差分のみを分離する
- DB を使う検証は実 PostgreSQL に対する統合テストで行う
- 集計、絞り込み、JOIN、存在判定は、まず SQL で解決を検討する
- GORM で不自然になるクエリは、無理に組み立てず生クエリを使う

## 正本と差分の考え方
- 正本: `sql/spring_boot_resources/schema.sql`
- 正本: `sql/spring_boot_resources/data.sql`
- PostgreSQL 差分: `sql/dialect/postgres/*.sql`
- 共通検索 SQL: `sql/queries/*.sql`
- PostgreSQL 専用検索 SQL が必要な場合のみ `sql/dialect/postgres/*.sql` に分離する

## 推奨ディレクトリ構成
```text
project/
├── sql/
│   ├── spring_boot_resources/
│   │   ├── schema.sql
│   │   └── data.sql
│   ├── queries/
│   │   └── high_skill_users_search.sql
│   └── dialect/
│       └── postgres/
│           ├── schema_patch.sql
│           ├── data_patch.sql
│           └── high_skill_users_search.sql
├── scripts/
│   └── bootstrap-db.ps1
└── test/
    └── integration/
        └── high_skill_users_postgres_test.go
```

## PostgreSQL での実装ルール
- `SERIAL` / `IDENTITY`、真偽値、日時関数、`ON CONFLICT` などの方言差分は PostgreSQL 差分ファイルで吸収する
- 重複排除が不要な場合は `UNION ALL` を優先する
- テストで安定比較できるよう `ORDER BY` を必ず明示する
- `SELECT *` は使わず、取得カラムを明示する
- 実行計画が悪化する場合は 1 クエリに詰め込まず分割する

## スキーマ・初期データ管理
- `schema.sql` は DDL の正本として扱う
- `data.sql` は開発用の最小初期データとして扱う
- テスト固有データは `data.sql` に混ぜず、各テストまたは `test/testdata/*.sql` で投入する
- PostgreSQL でのみ必要な補正は `sql/dialect/postgres/schema_patch.sql` や `data_patch.sql` に切り出す

## SQL ファイルの書き方
- SQL ファイル冒頭に以下をコメントで書く
  - 目的
  - 出力カラム
  - 実装方針

```sql
-- 目的: skill_level >= 8 の社員をカテゴリ横断で取得する
-- 出力カラム: employee_id, employee_name, skill_category, skill_name, skill_level
-- 実装方針: カテゴリごとの結果を UNION ALL で結合し、employee_id, skill_category, skill_name でソートする
```

## Go テスト方針
- SQL 追加時は先に Go の `_test.go` を作成する
- `go test` で失敗を確認してから SQL を追加する
- クエリ文字列をテスト内に直書きせず、SQL ファイルを読み込んで実行する
- テスト関数名は英語、`t.Run` は日本語で書く
- 実 DB 接続で確認し、DB モックやリポジトリモックには逃げない

## 代表例
- 対象 SQL: `sql/queries/high_skill_users_search.sql`
- 対象テスト: `test/integration/high_skill_users_postgres_test.go`
- 目的: `skill_level >= 8` の社員をカテゴリ横断で抽出し、スキル名・レベルを返す

## チェックリスト
- [ ] 共通 SQL と PostgreSQL 差分の置き場を分けた
- [ ] `schema.sql` / `data.sql` を正本として扱っている
- [ ] PostgreSQL 固有差分のみ `sql/dialect/postgres` に閉じ込めた
- [ ] SQL 実装前に `_test.go` を作成した
- [ ] SQL を直書きせずファイルから読み込んだ
- [ ] `ORDER BY` を明示した
- [ ] 実 PostgreSQL で統合テストを通した
