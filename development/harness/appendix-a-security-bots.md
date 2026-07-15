---
title: "付録A GitHubの既製セキュリティチェック"
parent: "AIハーネス入門 — AIに安全に良い仕事をさせる環境づくり"
grand_parent: "開発の心得"
nav_order: 5
---

# 付録A GitHub の既製セキュリティチェック — Dependabot・CodeQL・CodeRabbit

第3章では自前の AI レビュー workflow を組みましたが、**網は自作するだけではありません**。GitHub には「有効にするだけ」で使える既製のチェックがそろっていて、セキュリティの網を実質ゼロ工数で増やせます。ここでは代表3つ + おまけ1つを紹介します。

## Dependabot — 依存ライブラリの見張り番

**何者?** GitHub 内蔵の bot です。仕事は2つ:

1. **脆弱性アラート** — あなたのプロジェクトが使っているライブラリ(依存)を、既知の脆弱性データベース(GitHub Advisory Database)と常に照合。危ないバージョンを使っていたら警告し、**修正版へ上げる PR を自動で作ってくれます**
2. **バージョン更新** — 古くなった依存を定期的にチェックして、更新 PR を出してくれます

**なぜ AI 時代に重要?** 自分のコードがどれだけ安全でも、**依存ライブラリの穴からやられます**(サプライチェーン攻撃)。そしてこの種の問題は、人間のレビューでも AI のレビューでも diff を見ているだけでは**絶対に見つかりません** — 脆弱性データベースとの照合が必要で、それは bot にしかできない仕事です。さらに AI は学習時点の知識でコードを書くため、**古いバージョンのパッケージを選びがち**という事情もあり、依存を新しく保つ仕組みの価値が上がっています。

**導入:** 脆弱性アラートはリポジトリの Settings → Advanced Security まわりでワンクリック。バージョン更新は `.github/dependabot.yml` を1枚置くだけです:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

**運用の注意:** Dependabot の PR も CI とレビューを通します(「バージョンを上げただけで壊れる」は普通に起きます)。また第3章で見たとおり、**GitHub はセキュリティ上の理由で bot の PR に secrets を渡しません**。API キーが必要な自前の AI レビュー workflow は Dependabot の PR では動かないので、スキップ設定が必要です(第3章・付録B参照)。

## CodeQL — コードの中の「危険な流れ」を追う静的解析

**何者?** GitHub の code scanning エンジンです。lint がコードを「行のパターン」で見るのに対し、CodeQL はプログラムを**意味のレベルで解析**します。たとえば「ユーザーの入力が、無害化されないまま SQL やコマンド実行に流れ着く経路はないか」を、関数をまたいでデータの流れごと追跡します。

**第2章の security lint との関係** — 競合ではなく、網の目が違います:

| | eslint-plugin-security(第2章) | CodeQL |
| --- | --- | --- |
| 見るもの | 1行〜数行の危険パターン | プログラム全体のデータの流れ |
| 速さ | 一瞬(エディタで即時) | 数分(PR ごと・定期) |
| 深さ | 浅く広く | 深い(関数をまたぐ経路も追う) |

**導入:** Settings → Code security → Code scanning → **Default setup** のワンクリックで有効になります。実は**このドキュメントサイトのリポジトリでも全 PR に回っています**(PR のチェック欄に出る「CodeQL / Analyze」がそれです)。公開リポジトリは無料です。

## CodeRabbit — 入れるだけの AI レビュアー

**何者?** 第3章で「GitHub App を入れるだけ」の方式として触れた、既製の AI レビューサービスの代表格です。インストールすると、PR のたびに変更の要約・図解・行単位のレビューコメントを自動投稿してくれます。

**導入:** GitHub Marketplace から App をインストールして対象リポジトリを選ぶだけ。設定ファイル(`.coderabbit.yaml`)は任意です。OSS(公開リポジトリ)には無料枠があります。

**運用の注意:**

- 要約・ポエムなど定型の前置きはノイズとして読み飛ばしてOK(本体はインラインの指摘)
- 利用量の上限(rate limit)でレビューが走らない回があります。「今回はレビューなし」として扱い、依存しすぎない
- 指摘の受け方は第3章のトリアージ原則そのまま: **盲信せず、本物/nit/誤検知に分類**してから対応

## おまけ: secret scanning — コミットに紛れた鍵を検出

API キーや秘密鍵を**うっかりコミットしてしまう**事故は、AI 開発でも定番です(AI は「動く」を優先して鍵を直書きしがち — [refactoring.md](../refactoring.md) の「事実→だから」参照)。GitHub 標準の Secret scanning に加えて、`gitleaks` のようなスキャナを CI に置くと、コミット履歴全体を毎 push 検査できます。

MulmoClaude の実例([secret-scan.yml](https://github.com/receptron/mulmoclaude/blob/main/.github/workflows/secret-scan.yml))は参考になります: gitleaks のバイナリを**バージョン固定 + SHA256 検証つき**でダウンロードして実行しています。セキュリティ道具そのものの供給元も検証する — サプライチェーン対策のお手本です。

## まとめ — 導入コスト対効果の表

| 道具 | 見張るもの | 導入の手間 | 費用 |
| --- | --- | --- | --- |
| **Dependabot** | 依存ライブラリの脆弱性・古さ | Settings ワンクリック + yml 1枚 | 無料 |
| **CodeQL** | コード内の危険なデータの流れ | Settings ワンクリック | 公開リポジトリは無料 |
| **CodeRabbit** | PR の diff(AI レビュー) | App をインストール | 無料枠あり |
| **secret scanning / gitleaks** | コミットに紛れた鍵・秘密情報 | GitHub 標準 + workflow 1枚 | 無料 |

どれも「入れない理由がない」水準の手間です。新しいリポジトリを作ったら、コードを書き始める前に最初の3つは有効にしておきましょう。

## 合わせて読む

- [第3章 subagent と cross review](./03-cross-review.md) — これらの bot を含むレビューループの回し方
- [付録B 実物解剖: MulmoClaude の CI codex レビュー](./appendix-b-codex-ci.md)
- [セキュリティ(攻撃の入口を知る)](../security/README.md) — そもそも何から守るのか
