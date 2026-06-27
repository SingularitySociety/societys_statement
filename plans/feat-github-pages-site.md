# feat: GitHub Pages 化（Jekyll + just-the-docs）

## 目的

`societys_statement` リポジトリの Markdown 群を、GitHub Pages 上のドキュメントサイトとして公開する。

## 方式（ユーザー選択）

- ジェネレータ: **GitHub Pages 標準の Jekyll + just-the-docs テーマ**（`remote_theme`）
- ホスティング: **GitHub Pages**（`main` ブランチからのクラシックビルド）

## 重要な制約

Jekyll は **front matter を持つファイルだけを処理（HTML 化・レイアウト適用）** し、front matter の無い `.md` は静的ファイルとして生のまま配信する。
そのため、サイトに載せる各 Markdown には最小限の front matter（`title` など）が必要。just-the-docs のサイドバーも front matter（`title` / `parent` / `nav_order`）で構成されるため、ここで一括付与する。

## 実装

1. `_config.yml`: `remote_theme`、`baseurl: /societys_statement`、全文検索、`defaults` でレイアウト適用、不要物の `exclude`。
2. `Gemfile` / `.gitignore`: ローカルプレビュー用（GitHub のクラシックビルドには不要だが開発補助）。
3. front matter の一括付与（`scratchpad` のスクリプトで生成、冪等）:
   - トップレベル: 行動規範・カルチャー・ミーティング・Slack・ブートキャンプ・開発の心得 など
   - 階層: `開発の心得` > （`セキュリティ脅威マップ` / `Twitterクローンで学ぶWeb開発入門`）> 各ページ（3 階層 = grandparent まで）
   - ルート `README.md` は `jekyll-readme-index` でトップページ化（front matter 付与なし＝GitHub 上の見た目を維持）
4. ルート README 内の `topics.md` への絶対 GitHub URL を相対リンクへ修正（サイト内回遊のため）。

## トレードオフ

- front matter を付けたファイルは github.com 上で先頭にメタデータ表が表示される。最も閲覧される **ルート README とチュートリアル本文の可読性** を優先しつつ、ナビに必要な構造ページに付与する。
- `remote_theme` はバージョン未固定。安定後にタグ固定を推奨。

## 公開手順（マージ後）

- Settings → Pages → Source: Deploy from a branch → `main` / `(root)`、または `gh api` で有効化。
- 公開 URL: `https://singularitysociety.github.io/societys_statement/`
