# CLAUDE.md — Chrome拡張開発ガイド

## 基本方針

- **機能を追加する前に、本当に必要か問う**。使い勝手が微妙な機能は作らない
- コードはシンプルに保つ。抽象化・共通化は3箇所以上で重複してから検討する
- コメントは「なぜ」が非自明な箇所だけ書く。何をしているかはコードを読めばわかる

## テスト（必須）

**新しいロジックを追加・変更したら必ずテストを書く。テストなしのロジック変更は認めない。**

```bash
npm test
npm run test:coverage
```

- `chrome.*` API はすべて `global.chrome` でモックする
- モックは `beforeEach` で初期化し、個別ケースで上書きする
- テストケース名は日本語で「何をテストしているか」が一目でわかるように書く

## Chrome拡張の制約

### 許可色（tabGroups）

`blue` / `cyan` / `green` / `grey` / `orange` / `pink` / `purple` / `red` / `yellow`

`teal` は Chrome API では使えない。追加時は必ずこのリストから選ぶ。

### 権限は最小限に

`manifest.json` の `permissions` は実際に使うものだけ。追加するときは理由を確認する。

### `chrome://` と `chrome-extension://` は操作対象外

タブ操作の前に必ずフィルタする。

## ファイル構成のルール

### CommonJS と ESM の2ファイル並行管理

コアロジックを CommonJS（Jest用）と ESM（popup用）に分けている場合、**片方を変えたら必ずもう片方も変える**。

```
src/tab-organizer.js          # CommonJS（テスト用）
src/tab-organizer.browser.js  # ESM（popup用）
```

### popup.js は UI と Chrome API の橋渡しに徹する

ロジックは `src/` に置く。popup.js に判定・変換ロジックを書かない。

## 設定の保存

`chrome.storage.local` を使う。`localStorage` は使わない。

- 読み込みはデフォルト値とセットで行う（`storage.get(defaultSettings)`）
- 保存値は読み込み時に検証・サニタイズする（外部入力として扱う）

## ドキュメント

- 実装を変えたら `docs/` も同じタイミングで更新する
- 書くのは「コードから読み取れない情報」だけ。コードの写経は書かない

## やらないこと

- バックグラウンド常駐での自動実行（ユーザーの明示的な操作がトリガー）
- 手動で作ったタブグループへの干渉
- 確認なしの破壊的操作（タブ・グループの削除）
