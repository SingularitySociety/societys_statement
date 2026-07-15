---
title: "第3章 subagent と cross review"
parent: "AIハーネス入門 — AIに安全に良い仕事をさせる環境づくり"
grand_parent: "開発の心得"
nav_order: 3
---

# 第3章 subagent と cross review — AIの仕事を別のAIがチェックする

lint と型チェック(第2章)は「機械的に判定できる問題」を止めます。しかし「このエラー処理、この場合に漏れてない?」「この設計、既存のコードと重複してない?」のような判断は lint には書けません。そこで登場するのが**レビュー**です。

この章では、レビューを人間だけに頼らず、**書いたAIとは別のAIにチェックさせるしくみ**を、実際に運用されている構成([MulmoClaude](https://github.com/receptron/mulmoclaude) の CI)を題材に組み立てます。

## なぜ「別の目」が必要か

1. **書いた本人は自分に甘い** — AI も人間と同じで、自分が書いたコードのレビューでは自分の前提を疑えません。「さっき自分が正しいと判断したこと」をもう一度正しいと判断しがちです
2. **同じモデルは同じ盲点を持つ** — Claude が見落とすパターンは、もう一度 Claude に聞いても見落とされる可能性が高い。**別の会社のモデル(GPT系、Gemini系)は訓練が違うので、盲点の位置も違います**。これが cross(交差)review の核心です
3. **AIレビュアーは疲れない** — 人間は3,000行の diff を深夜に読めませんが、AI は毎 push ごとに同じ集中力で読みます

ゴールのイメージは**三重の網**です。この章はその2枚目の話です。

```text
あなた + Claude(書く)
   │
   ▼
🤖 機械の網 …… lint / 型 / テスト / CI(第2章)
   │              「機械的に判定できる問題」を全部止める
   ▼
👀 別のAIの網 …… subagent / cross review(この章)
   │              バグ・設計・セキュリティを「別の目」で見る
   ▼
🧑 人間の網 …… 最終判断してマージ
                 AIの指摘の採否を決めるのは常に人間
```

---

## レベル1: subagent — 同じAIの「別の脳」に見せる

**subagent(サブエージェント)**は、Claude Code がメインの会話とは**別の真っさらな文脈(コンテキスト)**で起動するもう1つの Claude です。

ポイントは「真っさら」なところです。メインの会話には「さっきこう実装すると決めた」という**思い込みの履歴**が溜まっています。subagent はその履歴を持たずに、コードだけを見ます。同じモデルでも、**先入観ゼロの初見レビュー**ができるわけです。

### 使い方1: 頼むだけ

Claude Code は依頼に応じて自動で subagent を使います。明示的に頼むこともできます:

```text
実装が終わったら、サブエージェントを使って
この変更を先入観なしでレビューして。バグとエッジケース中心で。
```

Claude Code には diff をレビューする `/code-review` コマンドも組み込まれています。大きめの変更を仕上げたら習慣として回しましょう。

### 使い方2: レビュー専用エージェントを定義しておく

`.claude/agents/` にエージェント定義を置くと、観点を固定した専属レビュアーを作れます。**書く人と同じルール(CLAUDE.md)を読む**ので、プロジェクトの文脈を踏まえた指摘ができます。

```markdown
<!-- .claude/agents/code-reviewer.md -->
---
name: code-reviewer
description: コード変更のレビュー専用。diff からバグ・セキュリティ・設計の問題を探すときに使う
tools: Read, Grep, Glob, Bash
---

あなたはこのリポジトリのコードレビュアーです。

- 変更された diff を読み、次の観点で問題を探す:
  正しさ(エッジケース・エラー処理漏れ)、セキュリティ(XSS・インジェクション)、
  設計(既存コードとの重複・責務の持ちすぎ)、テストの過不足
- 指摘は「重要度 / 場所(ファイル:行) / 理由 / 修正案」の形式で報告する
- スタイルの好みは指摘しない(lint の担当)
- 修正はしない。報告だけを行う
```

### subagent の限界

subagent は「別の脳」ですが「別のモデル」ではありません。**モデル自体の盲点(苦手パターン)は共有しています**。そこを補うのが次のレベルです。

---

## レベル2: 手元で cross review — 別の会社のAIに見せる

PR を出す前に、手元で**別ベンダーのAI CLI**に diff をレビューさせます。たとえば OpenAI の codex CLI:

```bash
codex exec "このリポジトリの現在のブランチと main の差分をレビューして。
git diff main... で差分を確認し、バグ・セキュリティ・エッジケース漏れを
重要度つきで指摘して。修正はしないで報告だけ。"
```

Gemini CLI など、別モデルの CLI なら何でも同じ構図が作れます。ここで1つ、後で効いてくる工夫を導入します — **判定マーカー**です。レビューの最後に機械可読な1行を必ず出させます:

```text
- 問題がなければ最終行に: CODEX VERDICT: LGTM
- 問題があれば最終行に:   CODEX VERDICT: CHANGES REQUESTED(+ 指摘の箇条書き)
```

人間にとってはただの行ですが、**スクリプトや別のAIが「レビュー通過したか」を確実に判定できる**ようになります。これが次のレベルの自動化の土台です。

> LGTM = "Looks Good To Me"(私はOKだと思う)。レビュー承認の定番スラングです。

---

## レベル3: CI cross review — push のたびに自動で回す(本命)

手元のレビューは「やり忘れ」ます。そこで GitHub Actions に載せて、**PR を出す/更新するたびに別のAIが自動レビュー**するようにします。以下は MulmoClaude で実際に全 PR に回っている workflow の簡略版です(実物は [codex_review.yaml](https://github.com/receptron/mulmoclaude/blob/main/.github/workflows/codex_review.yaml))。

```yaml
# .github/workflows/ai_review.yaml
name: AI cross review

on:
  pull_request:
    types: [opened, synchronize, reopened]   # PR作成・更新のたびに
    branches: [main]

permissions:
  contents: read
  pull-requests: write   # レビューコメントを書き込むため
  issues: write

# 連続pushしたら古いレビューは中止して最新コミットだけ見る(コスト対策)
concurrency:
  group: ai-review-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    # Dependabot の PR は secrets が渡らず失敗するのでスキップ
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0          # diff を見るために全履歴が必要
          persist-credentials: false

      - uses: actions/setup-node@v6
        with:
          node-version: 22.x

      - name: Install codex CLI(バージョンを固定して再現性を確保)
        run: npm install -g "@openai/codex@0.125.0"

      - name: Run AI review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO: ${{ github.repository }}
        run: |
          codex exec --sandbox danger-full-access "$(cat <<PROMPT
          Review PR #${PR_NUMBER} in ${REPO}.

          gh CLI で diff を確認すること:
            gh pr view ${PR_NUMBER} --json title,body
            gh pr diff ${PR_NUMBER}

          先に既存のレビュースレッドを読むこと:
            gh api repos/${REPO}/issues/${PR_NUMBER}/comments --paginate
          過去の指摘が最新コミットで解決済みなら再指摘しない。
          他のボットが既に指摘した内容の繰り返しもしない。

          観点: 正しさ、エッジケース、セキュリティ(XSS / SSRF /
          パストラバーサル / プロンプトインジェクション)、テストの過不足、
          既存コードとの一貫性。スタイルの好みは指摘しない。

          指摘は gh pr comment / gh api で PR にコメントとして投稿すること。

          最後に必ず、次のマーカーで始まるコメントを1件投稿すること:
            問題なし → 'CODEX VERDICT: LGTM'
            要修正   → 'CODEX VERDICT: CHANGES REQUESTED' + 箇条書き

          修正はしないこと。レビューと報告だけを行う。
          PROMPT
          )"
```

### この workflow に詰まっている運用の知恵

実物から学べるポイントを整理します。どれも「AIレビューを回しっぱなしにする」ための工夫です。

| 工夫 | 理由 |
| --- | --- |
| `concurrency` + `cancel-in-progress` | 5分で3回 push しても、レビューは最新コミットに対して1回だけ。**コストが暴れない** |
| CLI のバージョン固定(`@0.125.0`) | ある日突然レビューの挙動が変わる事故を防ぐ。上げるときは明示的に |
| 「既存スレッドを先に読め」 | 2回目以降のレビューで**同じ指摘を繰り返させない**。解決済みの指摘は流す |
| 「修正はするな、報告だけ」 | レビュアーとしての役割を固定。直すのは開発側(あなた+Claude)の仕事。責任の分離 |
| 判定マーカーの強制 | 「全ボットが LGTM を出したか」を機械的に判定できる → マージ条件を自動化できる |
| Dependabot をスキップ | GitHub は bot の PR に secrets を渡さないため、動かして失敗させるだけ無駄 |
| `timeout-minutes: 10` | レビューが無限に回ってお金が溶けるのを防ぐ保険 |

> ⚠️ `--sandbox danger-full-access` は名前が物騒ですが、GitHub Actions のランナー自体が使い捨ての隔離VMなので、この文脈では二重サンドボックスを外しているだけです。**手元のPCで使う設定ではありません**。

### 選択肢はほかにもある

自前 workflow を書かなくても、レビューAIを追加する方法はあります。組み合わせも自由です(MulmoClaude は自前 codex + CodeRabbit + Sourcery の3本立て)。

| 方式 | 例 | 特徴 |
| --- | --- | --- |
| **GitHub App を入れるだけ** | CodeRabbit、Sourcery | 設定ほぼゼロで PR に自動レビューが付く。無料枠あり。細かい制御はしにくい |
| **自前 workflow + 他社CLI** | 上の codex 例 | プロンプト・観点・マーカーを完全に自分で設計できる |
| **Claude をレビュアー側に置く** | anthropics/claude-code-action | 開発を別のAI(または人間)がして、レビューを Claude にやらせる逆向き構成 |

大事なのは製品名ではなく構図です: **書くAIとレビューするAIのベンダーを分ける**こと。

---

## 指摘をどう受けるか — 「盲信しない」が鉄則

AIレビューの指摘は**そのまま全部適用してはいけません**。AIレビュアーは間違えますし、複数のボットは互いに矛盾する提案をします。開発側(あなた+Claude)は、指摘を1件ずつ**分類**します。

| 分類 | 対応 |
| --- | --- |
| **本物のバグ・セキュリティ問題** | 修正して、再発防止のテストも足す |
| **正しいけど細かい指摘(nit)** | 安く直せるなら直す。意図的な設計なら「意図的です」と返信 |
| **誤検知(false positive)** | コードで検証した上で、理由を書いてスキップ。**黙って無視しない**(次のレビューでまた指摘される) |
| **矛盾する提案** | どちらが正しいかコードに照らして判断。両方を機械的に満たそうとしない |

そして修正を push すると、AIレビュアーが**もう一度**レビューします。このループを回します:

```text
push → 🤖 AIレビュー → 指摘を分類 → 修正を commit / 誤検知には返信
  ↑                                        │
  └────────── もう一度 push ←──────────────┘

終了条件: 全レビュアーが LGTM + CI が緑 + 人間が納得 → マージ
```

運用のコツ:

- **そもそも PR を小さく保つ**([Gitのルール](../git.md))。生成は速くても、レビューできる量が律速です。diff が小さいほど AI のレビューも人間のレビューも深くなり、指摘の精度が上がる
- 修正コミットは「fix: address CodeRabbit review comments」のように**どのボットの指摘対応か分かる名前**にする
- ループの最後に「対応した指摘 / 意図的に見送った指摘」のまとめコメントを PR に残す。**人間のレビュアーがボットのスレッドを全部読み直さなくて済む**ようにするため
- このループ自体を Claude Code に任せることもできます(「ボットの指摘を全部トリアージして、修正して、再レビューを待って」)。ただし採否の最終判断は人間に残すこと

---

## コストの感覚

「全 PR にAIレビューなんて高いのでは?」— 実際の運用では、上の表の工夫(concurrency で連続 push を1回に集約、タイムアウト、モデル側の流量制限)だけで**現実的なコストに収まります**。ドキュメントだけの PR も、タイポやリンク切れを拾ってくれるので回す価値があります。

判断の物差しはシンプルです: **AIレビューの月額 < バグが本番に漏れたときの損害 + 人間がレビューに費やす時間**。ほとんどのチームで前者が圧倒的に安くつきます。

---

## まとめ — ハーネス全体像のなかで

| 網 | 何を止めるか | 設定するもの |
| --- | --- | --- |
| 機械(第2章) | 機械的に判定できる問題すべて | ESLint / tsc / テスト / CI |
| 別のAI(この章) | バグ・設計・セキュリティの「判断が要る」問題 | subagent / cross review workflow |
| 人間 | 「そもそも何を作るべきか」のずれ | PR レビューとマージ権限 |

AIに書かせる量が増えるほど、人間の役割は「全行を読む人」から「**網を設計し、最終判断をする人**」に変わっていきます。ハーネスづくりは、その新しい役割の中心スキルです。

## 合わせて読む

- [シリーズ表紙: AIハーネス入門](./README.md) — 全体像
- [第4章 フックと権限](./04-hooks.md) — 破りようがない強制力
- [Git/GitHub 入門(非エンジニア向け)](../git_basics.md) — PR・マージの基本
- [GitHub ActionsではじめるCI/CD](../github_actions/README.md) — workflow ファイルの読み書き
- [Vibe Coding について](../vibe_coding.md) — AIとの開発全体の心得
