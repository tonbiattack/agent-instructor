# AGENTS.md

## Writing Style Rule

- 記事作成では、読みやすさと管理しやすさを優先する。
- 記事ファイルのファイル名と、記事内のタイトル（H1）は同じ文言にそろえる。
- タイトルは内容が過不足なく伝わる表現にする。
- 導入で「この記事で何を扱うか」を先に明示する。
- タイトル（H1）の直後に「## はじめに」セクションを追加する。公開時にタイトルを本文から切り離すため、H1 は独立させ、本文は「はじめに」から始める。

- Do not use markdown bold markers like `**...**` for emphasis.
- Do not use separator lines such as `---` as visual dividers in article text.
- Avoid AI-like repetitive emphasis patterns and symbolic decoration.
- Prefer natural Japanese prose that reads like it was written by a person.
- Keep formatting simple and practical: headings, short paragraphs, and bullet lists when needed.
- 図が必要な場合は Mermaid を使用する。

- **コード内のコメント（行コメント・ブロックコメント・ドキュメンテーションコメント）はすべて日本語で記載する**

## Tone and Voice（文体定義）

文体の参照元は `public/` フォルダ内の公開済み記事とする。
特に次の2記事を基準にする。

- `public/なぜ壊れた権限設計が生まれるのか（RBAC設計の実務）.md`
- `public/クリーンアーキテクチャでモックはどこまで書くべきか.md`

これらの記事から抽出した文体ルールは次のとおり。

- です・ます調で統一する（だ・である調は使わない）
- 一文を短く保つ。一段落に詰め込みすぎない
- 情報の列挙には箇条書きを使う。「次の〜です」「次のとおりです」で導入する
- 見出しの直後に結論・概要を置き、詳細は後に続ける
- 接続詞や冗長な修飾語を省いてシンプルに書く
- 「〜だと思います」「〜かもしれません」など過度な留保表現は避ける
