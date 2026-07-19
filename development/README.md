---
title: "開発の心得"
nav_order: 9
has_children: true
---

# 　開発の心得

- [Gitを使った開発について](./git.md)
  - [Git/GitHub 入門（非エンジニア向け）](./git_basics.md) — git init〜commit/push/pull と「ブランチ→PR→マージ」の流れを、非エンジニア向けにイラストと実行結果の例つきで。チーム開発とコンフリクトの解消まで
- [Vibe Coding について](./vibe_coding.md)
  - [Vibe Coding 入門（非エンジニア向け）](./vibe_coding_simple.md)
- [リファクタリング & プロダクト化（Vibe Coding時代の心得）](./refactoring.md) — きれいなコードの原則（テスト可能性/DRY/小ささ/疎結合/条件整理）＋AI時代の追加＋いつやるか、そして「ブラウザだけの試作→プロダクト対応」の実践ロードマップ（テスト→整理→UI/ロジック分離→サーバー移設→TS化→per-userデータ→分離・課金）
- [Twitterクローンで学ぶWeb開発入門（Supabase）](./twitter_supabase/README.md) — 認証・データ分離・タイムライン・フォロー・スケールを「Twitterの挙動」から実装まで（完全未経験〜初心者向け、全13章＋付録7）
- [ChatGPTクローンで学ぶ LLMアプリ開発入門](./chatgpt_clone/README.md) — APIキーの守り方・会話の記憶（ステートレス→ログ→要約→メモリ）・REST/SSE・ツール・使用量制限を「ChatGPTの挙動」から実装まで（完全未経験〜初心者向け、全14章＋付録A〜J）
- [自作CLIエージェントで学ぶ AIエージェント開発入門](./cli_agent/README.md) — ツール（道具）・エージェントループ・許可と安全（human-in-the-loop／最小権限）・スキル・コンテキスト／コストを「Claude Codeの挙動」から実装まで。TypeScript＋Anthropic SDKでCLIエージェントを自作（完全未経験〜初心者向け、全15章＋付録A〜G）
- [コンピューティングの選択肢 — オンプレ/IaaS/コンテナ/PaaS/サーバーレス](./compute_options/README.md) — 「アプリをどこで動かすか」を AWS・Firebase・Vercel・Supabase など具体例で総合比較。用途・コスト・セキュリティ・長期メンテ・ベンダーロックイン・ペルソナ別（大企業/中堅/スタートアップ/個人）まで（初心者向け、全12ページ＋早見表/フローチャート/用語辞典）
- [データの置き場所ガイド — RDB/NoSQL/キャッシュ/ストレージ](./data_stores/README.md) — 「データをどこに置くか」を RDB（SQLite/自前/RDS/Aurora）・NoSQL（DynamoDB/MongoDB/Firestore）・キャッシュ（Redis）・オブジェクトストレージ（S3系）で総合比較。データの形×運用の任せ方の2軸、ペルソナ別、選び方フローチャート・早見表・用語辞典つき（初心者向け、全12ページ＋ハブ。compute_options の姉妹編）
- [フロントエンドの作り方 — 素のJS/React/Vue と SPA/SSR/SSG](./frontend/README.md) — 「UIをどう作り・どこで描画するか」を 素のTS/JS・React・Vue・SPA・SSR・SSG・Next.js/Astro（RSC=Reactの一部がバックエンド）まで総合比較。jQueryの歴史、ビルド（Vite）/ホスティング、ペルソナ/用途別、フローチャート・早見表・用語辞典つき（初心者向け、全12ページ＋ハブ。compute_options/data_stores の姉妹編）
- [MCPサーバーを作って学ぶ AIに道具を持たせる入門](./mcp_server/README.md) — 「AIに道具を後付けする共通規格」MCP のサーバーとクライアントを自作。stdio と Streamable HTTP、`console.log` でサーバーが壊れる理由、MCP Inspector とログでの切り分け、パスの封じ込めと DNS リバインディング対策まで。TypeScript で「自分のメモを読める Claude」を作り、第2弾の ChatGPTクローンにも繋ぐ（完全未経験〜初心者向け、全14章＋付録A〜D）
- [GitHub ActionsではじめるCI/CD](./github_actions/README.md) — 「安全に・自動で“届ける”」。CI/CDで何が嬉しいか、GitHub Actionsのしくみ（on→jobs→steps/YAML）、実例カタログ（テスト・CodeQL・レビューBot・Dependabot・デプロイ・リリース・cron・通知）、シークレットと安全、マトリクス/キャッシュ/コスト、選び方早見表・用語辞典つき（初心者向け、全8ページ＋ハブ。シリーズ姉妹編）
- [AIハーネス入門 — AIに安全に良い仕事をさせる環境づくり](./harness/README.md) — 「口頭の注意」を「毎回自動で効く環境」に変える。CLAUDE.md の書き方（置き場所の使い分け・網羅的テンプレートつき）、lint とは何かから始める AI 向けガチガチ ESLint 設定（Vue/React + Express、MulmoClaude の実設定ベース、インストール手順つき）、subagent と CI 上の別AIによる cross review（判定マーカー・レビューループ運用）、フックと権限（破りようがない強制力）、Dependabot / CodeQL / CodeRabbit・CI codex レビュー実物解剖・レビュースキル2本の付録まで（実例つき、全4章＋付録3＋ハブ）
- [Product Manageについて](./pm.md)
- [Productについて](./Product.md)
- [セキュリティ（攻撃の入口を知る教育ドキュメント）](./security/README.md)

