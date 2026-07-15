---
title: "付録B 実物解剖: MulmoClaudeのCI codexレビュー"
parent: "AIハーネス入門 — AIに安全に良い仕事をさせる環境づくり"
grand_parent: "開発の心得"
nav_order: 6
---

# 付録B 実物解剖 — MulmoClaude の CI codex レビュー workflow

第3章で載せたのは、要点だけ残した簡略版でした。この付録では、実際に全 PR で稼働している本物 — [MulmoClaude の codex_review.yaml](https://github.com/receptron/mulmoclaude/blob/main/.github/workflows/codex_review.yaml)(約220行)を解剖します。**簡略版との差分にこそ、運用で学んだ知恵が詰まっています**。

## 全体の流れ

```text
PR が作られる / 更新される
  │
  ├─ Dependabot の PR なら skip(理由は下記)
  ▼
Node.js を用意 → codex CLI をバージョン固定でインストール(npm キャッシュつき)
  ▼
Azure OpenAI 用の設定ファイル(~/.codex/config.toml)を生成
  ▼
codex exec: gh CLI で diff と既存スレッドを読み、レビューコメントを投稿
  ▼
最後に必ず「CODEX VERDICT: LGTM / CHANGES REQUESTED」を投稿
```

## 解剖ポイント

### 1. コスト設計 — 「制限」ではなく「集約」で守る

この workflow は draft もドキュメントだけの PR も**全部レビューします**。実物のコメントには設計判断がそのまま書いてあります: 「ユーザーの方針: 極力制限しないでどんどん使える」。それでもコストが暴れない理由は3つ:

- `concurrency` + `cancel-in-progress` — push を連打しても、レビューは最新コミットに1回だけ
- npm キャッシュ — CLI の再ダウンロードを省く
- **流量の上限はモデル側(Azure デプロイの TPM 上限)で効かせる** — workflow 側で制限を複雑にしない

「ドキュメントだけの PR にも回す」判断にも理由が書いてあります: 安いし、タイポやリンク切れを拾うことがあるから。

### 2. 認証まわり — 失敗の記録が設定になっている

このプロジェクトは OpenAI 直ではなく Azure OpenAI 経由で codex を動かしています。実物にはこうコメントされています:

> Codex CLI 0.125.0 は `OPENAI_BASE_URL` を直接尊重しない — 最初の試行(iter-1)では環境変数が黙って無視され、api.openai.com に向かって 401 になった。だから workflow が明示的に `~/.codex/config.toml` を書く。

つまり**一度実際に失敗して、その原因と対策がコメントとして残っている**わけです。設定ファイルの生成にも芸があります:

```yaml
# クォートしたヒアドキュメント(<<'TOML')= 完全なリテラル。
# TOML本文がシェルとして解釈されることはなく、
# envsubst が許可した3変数だけを埋め込む。
envsubst '${AZURE_MODEL} ${BASE_URL} ${AZURE_API_VERSION}' \
  > "$HOME/.codex/config.toml" <<'TOML'
model_provider = "azure"
model = "${AZURE_MODEL}"
...
TOML
```

テンプレートに任意のシェル展開を許さず、**許可リストに載せた変数だけ**を差し込む — CI で外部入力を扱うときの安全な書き方の見本です。

> 💡 自分のリポジトリで OpenAI を直接使うなら、この config.toml 生成ステップは丸ごと不要です(`OPENAI_API_KEY` を渡すだけ)。第3章の簡略版はその形になっています。

### 3. サンドボックスの罠 — `danger-full-access` の本当の理由

```text
デフォルトの workspace-write サンドボックスは bwrap でネットワーク名前空間を
作ろうとするが、GitHub Actions のランナーでは kernel 制限で失敗する。
bwrap なしでは codex はシェルコマンドを一切実行できない(gh も動かない)。
ランナーVM自体が使い捨て・隔離されているので、二重サンドボックスを
外すのはこの文脈では安全。
```

これも実物のコメントの要約です。物騒なフラグに見えても、**「なぜ安全と判断したか」が根拠つきで書いてある**ので、あとから見た人(や AI)が判断を検証できます。

### 4. Dependabot をスキップする理由

GitHub は Dependabot が開いた PR にはリポジトリの secrets を渡しません(セキュリティ仕様)。つまり `AZURE_OPENAI_API_KEY` が空文字になり、codex は数秒で「環境変数がない」と落ちて、**チェックが赤くなるだけ**。だから最初から skip します。依存更新 PR は他の網(CI・CodeRabbit・付録Aの各種チェック)が見るので十分、という判断も書き添えられています。

### 5. プロンプト設計 — 2周目以降を賢くする

第3章の簡略版にも入れましたが、実物のプロンプトの核心はこの3つです:

1. **先に既存スレッドを全部読め** — 過去の自分(や他ボット)の指摘が最新コミットで解決済みなら再指摘しない。他ボットと同じ指摘の繰り返しもしない。「既存スレッドに対して自分が足せる差分に集中しろ」
2. **判定マーカーを必ず出せ** — `CODEX VERDICT: LGTM / CHANGES REQUESTED`。これが後続の自動化(付録Cの gh-review-loop)の機械可読な信号になる
3. **修正はするな** — レビュアーは報告だけ。直すのは開発側。役割を混ぜない

### 6. いちばんの学び — workflow のコメントに「運用の歴史」を書く文化

この workflow が優れているのは YAML の技巧ではなく、**すべての設定に「なぜ」がコメントで残っている**ことです。401 になった最初の失敗、ランナーの kernel 制限、コスト方針、skip の理由 — 全部その場に書いてある。

これは第1章で見た原則「**AI は What を書くが Why を持たない → 意図を残す**」の実践です。次にこのファイルを触る人(人間でも AI でも)は、コメントを読むだけで「この行は消していいか?」を判断できます。**設定ファイルのコメントは、未来の自分たちへのハーネス**です。

## 合わせて読む

- [第3章 subagent と cross review](./03-cross-review.md) — 簡略版と全体の考え方
- [付録C cross review を手元で回すスキル](./appendix-c-review-skills.md) — この workflow の verdict を受けて回すループ
- [GitHub ActionsではじめるCI/CD](../github_actions/README.md) — workflow 構文の基礎
