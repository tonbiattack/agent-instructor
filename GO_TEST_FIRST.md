# Go テストファースト開発ガイド（GORM + MySQL + Cobra）

## 技術スタック
- **Go**: 標準testing パッケージ
- **GORM**: ORM（データベース操作）
- **MySQL**: データベース
- **Cobra**: CLI フレームワーク

## テスト手法の方針（古典派）
- **データベースアクセスは実DBを使用**（モックを使用しない）
- **モックは外部API等の外部結合のみ**に限定
- 内部で閉じている実装（リポジトリ、サービス層など）に対してモックを使用しない
- 実際のMySQLを使ってテストすることで、テストの信頼性を向上
- トランザクションやロールバックを活用してテストを独立させる

## 基本原則
実装コードを書く前に、必ずテストコードを先に作成する。

## 作業手順

### 1. 要件を理解する
- ユーザーストーリーや仕様を確認
- 期待される動作を明確にする
- データベーススキーマを設計

### 2. テストケースを設計する
- 正常系、異常系、境界値をリストアップ
- データベース操作のテストケースを洗い出す
- Cobra コマンドの入出力を定義

### 3. テストコードを作成する
- `_test.go` ファイルに失敗するテストを書く
- テーブル駆動テストで複数のケースをカバー
- モックやテストDBを準備

### 4. インターフェースを定義する
- テストから必要な関数・メソッドのシグネチャを決める
- リポジトリのインターフェースを定義
- 実装すべきAPIを明確にする

## テストの書き方

### 単体テスト（標準testing）
```go
// user_test.go（実装前に作成）
package user

import "testing"

func TestGetUserByID(t *testing.T) {
    // まだ GetUserByID は存在しない状態
    userID := 1
    
    user, err := GetUserByID(userID)
    
    if err != nil {
        t.Errorf("expected no error, got %v", err)
    }
    if user.ID != userID {
        t.Errorf("expected user ID %d, got %d", userID, user.ID)
    }
}

// この時点で go test を実行すると失敗する（関数が未定義）
```

### テーブル駆動テスト
```go
// user_test.go
func TestValidateUser(t *testing.T) {
    tests := []struct {
        name    string
        input   User
        wantErr bool
    }{
        {"valid user", User{Name: "John", Email: "john@example.com"}, false},
        {"empty name", User{Name: "", Email: "john@example.com"}, true},
        {"invalid email", User{Name: "John", Email: "invalid"}, true},
        {"long name", User{Name: string(make([]byte, 256)), Email: "test@example.com"}, true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateUser(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### リポジトリインターフェース定義（実装前）
```go
// repository/user.go（インターフェースのみ先に定義）
package repository

type UserRepository interface {
    FindByID(id int) (*User, error)
    FindByEmail(email string) (*User, error)
    Create(user *User) error
    Update(user *User) error
    Delete(id int) error
}

// この時点ではインターフェースのみで、実装はまだない
```

### GORM リポジトリのテスト（実装前に作成）
```go
// repository/user_test.go
package repository

import (
    "testing"
    "gorm.io/gorm"
)

func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewUserRepository(db) // まだ実装されていない
    user := &User{Name: "Test", Email: "test@example.com"}
    
    err := repo.Create(user)
    if err != nil {
        t.Fatalf("failed to create user: %v", err)
    }
    
    if user.ID == 0 {
        t.Error("expected user ID to be set")
    }
}

func TestUserRepository_FindByID(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewUserRepository(db)
    
    // 事前データ投入
    user := &User{Name: "Test", Email: "test@example.com"}
    repo.Create(user)
    
    // テスト実行
    found, err := repo.FindByID(user.ID)
    if err != nil {
        t.Fatalf("failed to find user: %v", err)
    }
    if found.Name != user.Name {
        t.Errorf("expected name %s, got %s", user.Name, found.Name)
    }
}
```

### Cobra コマンドのテスト（実装前）
```go
// cmd/user_test.go
package cmd

import (
    "bytes"
    "testing"
)

func TestUserCreateCommand(t *testing.T) {
    cmd := NewUserCreateCommand() // まだ実装されていない
    
    buf := new(bytes.Buffer)
    cmd.SetOut(buf)
    cmd.SetArgs([]string{"--name", "John", "--email", "john@example.com"})
    
    err := cmd.Execute()
    if err != nil {
        t.Errorf("expected no error, got %v", err)
    }
    
    output := buf.String()
    if !strings.Contains(output, "User created successfully") {
        t.Errorf("expected success message, got %s", output)
    }
}
```

### テストヘルパー関数
```go
// test/helper.go
package test

import (
    "testing"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func SetupTestDB(t *testing.T) *gorm.DB {
    dsn := "root:password@tcp(localhost:3306)/test_db?parseTime=true"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("failed to connect database: %v", err)
    }
    
    // マイグレーション
    db.AutoMigrate(&User{})
    
    return db
}

func CleanupTestDB(t *testing.T, db *gorm.DB) {
    db.Exec("DELETE FROM users")
}

func CreateTestUser(t *testing.T, db *gorm.DB, name, email string) *User {
    user := &User{Name: name, Email: email}
    if err := db.Create(user).Error; err != nil {
        t.Fatalf("failed to create test user: %v", err)
    }
    return user
}
```

## ディレクトリ構成例
```
project/
├── cmd/
│   ├── root.go
│   ├── user.go              # 実装（テスト後）
│   └── user_test.go         # テスト（実装前）
├── internal/
│   ├── domain/
│   │   └── user.go          # ドメインモデル
│   ├── repository/
│   │   ├── interface.go     # インターフェース（実装前）
│   │   ├── user.go          # リポジトリ実装（テスト後）
│   │   └── user_test.go     # リポジトリテスト（実装前）
│   └── service/
│       ├── user.go          # ビジネスロジック（テスト後）
│       └── user_test.go     # サービステスト（実装前）
├── test/
│   ├── helper.go            # テストヘルパー
│   └── integration/
│       └── user_test.go     # 統合テスト（実装前）
└── go.mod
```

## ルール
- テストは実装前に必ず作成
- `_test.go` ファイルを先に作る
- インターフェースを先に定義
- テストコードは読みやすく保守しやすい構造
- **データベースアクセスは実DBを使用（モック禁止）**
- **モックは外部API等の外部結合のみに限定**
- **内部実装（リポジトリ、サービス層）はモックしない**
- 一つの実装に複数のテストファイルがあってもOK
  - 単体テスト: `user_test.go`
  - 統合テスト: `user_integration_test.go`
  - E2Eテスト: `user_e2e_test.go`

## テストコマンド
```bash
# テストを実行（失敗することを確認）
go test ./...

# カバレッジ付き
go test -cover ./...

# 詳細表示
go test -v ./...

# 特定のパッケージのみ
go test ./internal/repository
```

## 開発の流れ例

### 例: ユーザー作成機能

#### ステップ1: テストを書く（実装前）
```go
// repository/user_test.go
func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewUserRepository(db)
    user := &User{Name: "Test", Email: "test@example.com"}
    
    err := repo.Create(user)
    if err != nil {
        t.Fatalf("failed to create user: %v", err)
    }
}
```

#### ステップ2: インターフェースを定義
```go
// repository/interface.go
type UserRepository interface {
    Create(user *User) error
}
```

#### ステップ3: `go test` を実行
```bash
$ go test ./repository
# undefined: NewUserRepository
# FAIL
```

#### ステップ4: 実装する（テスト後）
```go
// repository/user.go
type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(user *User) error {
    return r.db.Create(user).Error
}
```

#### ステップ5: `go test` を再実行
```bash
$ go test ./repository
# PASS
```

## ベストプラクティス
- **データベースは実DBを使用**（モック禁止）
- データベースは各テストで独立した状態にする
- トランザクションやロールバックでデータをクリーンアップ
- テスト用の設定ファイルを分離（config.test.yml）
- 環境変数でテスト用DBを切り替え（TEST_DB_NAME など）
- **インターフェースは外部API用に定義**（内部実装のモック用ではない）
- **モックは外部API等の外部結合のみ**に限定
- テストヘルパー関数を用意して重複を削減
- `t.Helper()` を使ってヘルパー関数を明示

## チェックリスト
- [ ] 要件を理解した
- [ ] テストケースを列挙した
- [ ] `_test.go` ファイルにテストを作成した
- [ ] インターフェースを定義した
- [ ] `go test` で失敗を確認した（実装前）
- [ ] テストが要件を網羅している
- [ ] テストヘルパー関数を用意した
