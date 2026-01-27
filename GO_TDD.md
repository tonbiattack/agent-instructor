# Go TDD ガイド（GORM + MySQL + Cobra）

## 技術スタック
- **Go**: 標準testing パッケージ
- **GORM**: ORM（データベース操作）
- **MySQL**: データベース
- **Cobra**: CLI フレームワーク

## 基本サイクル: Red-Green-Refactor

### Red - 失敗するテストを書く
1. 小さな機能単位でテストを作成
2. `go test` を実行して失敗を確認
3. 何を実装すべきか明確にする

### Green - テストを通す最小実装
1. テストが通る最小限のコードを書く
2. `go test` ですべてのテストが成功することを確認
3. 美しさより動作を優先（まず動かす）

### Refactor - コードを改善
1. 重複を排除し、構造を改善
2. インターフェースを活用し、疎結合に
3. `go test` を再実行して成功を確認
4. テストコード自体もリファクタリング対象

## テストの書き方

### 単体テスト（標準testing）
```go
package user

import "testing"

func TestGetUserByID(t *testing.T) {
    // Arrange（準備）
    userID := 1
    
    // Act（実行）
    user, err := GetUserByID(userID)
    
    // Assert（検証）
    if err != nil {
        t.Errorf("expected no error, got %v", err)
    }
    if user.ID != userID {
        t.Errorf("expected user ID %d, got %d", userID, user.ID)
    }
}
```

### テーブル駆動テスト
```go
func TestValidateUser(t *testing.T) {
    tests := []struct {
        name    string
        input   User
        wantErr bool
    }{
        {"valid user", User{Name: "John", Email: "john@example.com"}, false},
        {"empty name", User{Name: "", Email: "john@example.com"}, true},
        {"invalid email", User{Name: "John", Email: "invalid"}, true},
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

### GORM のモック（インターフェース）
```go
// リポジトリインターフェース
type UserRepository interface {
    FindByID(id int) (*User, error)
    Create(user *User) error
}

// テスト用モック
type MockUserRepository struct {
    users map[int]*User
}

func (m *MockUserRepository) FindByID(id int) (*User, error) {
    if user, ok := m.users[id]; ok {
        return user, nil
    }
    return nil, errors.New("user not found")
}
```

### データベーステスト
```go
func TestUserRepository_Create(t *testing.T) {
    // テスト用データベース接続
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)
    
    repo := NewUserRepository(db)
    user := &User{Name: "Test", Email: "test@example.com"}
    
    err := repo.Create(user)
    if err != nil {
        t.Fatalf("failed to create user: %v", err)
    }
    
    // 作成されたことを確認
    found, err := repo.FindByID(user.ID)
    if err != nil {
        t.Fatalf("failed to find user: %v", err)
    }
    if found.Name != user.Name {
        t.Errorf("expected name %s, got %s", user.Name, found.Name)
    }
}

func setupTestDB(t *testing.T) *gorm.DB {
    dsn := "root:password@tcp(localhost:3306)/test_db?parseTime=true"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("failed to connect database: %v", err)
    }
    
    // マイグレーション
    db.AutoMigrate(&User{})
    return db
}

func cleanupTestDB(t *testing.T, db *gorm.DB) {
    db.Exec("DELETE FROM users")
}
```

### Cobra コマンドのテスト
```go
func TestUserCreateCommand(t *testing.T) {
    cmd := NewUserCreateCommand()
    cmd.SetArgs([]string{"--name", "John", "--email", "john@example.com"})
    
    err := cmd.Execute()
    if err != nil {
        t.Errorf("expected no error, got %v", err)
    }
}
```

## ディレクトリ構成例
```
project/
├── cmd/
│   └── root.go              # Cobra コマンド
├── internal/
│   ├── domain/
│   │   └── user.go          # ドメインモデル
│   ├── repository/
│   │   ├── user.go          # リポジトリ実装
│   │   └── user_test.go     # リポジトリテスト
│   └── service/
│       ├── user.go          # ビジネスロジック
│       └── user_test.go     # サービステスト
├── test/
│   └── integration/
│       └── user_test.go     # 統合テスト
└── go.mod
```

## 作業ルール
- Red-Green-Refactor を厳密に守る
- 一度に一つの機能に集中（小さなステップ）
- テストは常にグリーン（成功）を維持
- リファクタリングはテスト成功時のみ
- 各サイクル完了時にコミット

## テストコマンド
```bash
# すべてのテストを実行
go test ./...

# カバレッジ付き
go test -cover ./...

# 詳細表示
go test -v ./...

# 特定のパッケージのみ
go test ./internal/service

# テストファイルを監視（air など使用）
air -c .air.toml
```

## ベストプラクティス
- データベースは各テストで独立した状態にする
- トランザクションを使ってテスト後にロールバック
- テスト用の設定ファイルを分離（config.test.yml）
- 環境変数でテスト用DBを切り替え（TEST_DB_NAME など）
- モックは必要最小限に（本物のDBでテストする方が安全）
- Cobra コマンドはロジックを分離してテストしやすく

## サイクル例
```
1. Red:     user_test.go に FindByID のテスト作成 → 失敗確認
2. Green:   user.go に FindByID の最小実装 → テスト成功
3. Refactor: エラーハンドリング追加 → テスト成功

4. Red:     user_test.go にバリデーションテスト追加 → 失敗確認
5. Green:   user.go にバリデーション実装 → すべて成功
6. Refactor: 共通処理を抽出 → すべて成功
```

## チェックリスト
- [ ] Red: テストケースを書いた（`_test.go` ファイル）
- [ ] Red: `go test` で失敗を確認した
- [ ] Green: 最小限の実装を書いた
- [ ] Green: `go test` ですべてのテストが成功した
- [ ] Refactor: コードを改善した
- [ ] Refactor: `go test` で成功を確認した
- [ ] `go fmt` でフォーマットした
- [ ] コミットした
