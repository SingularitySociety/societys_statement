# UI設計の基礎（非エンジニア向け連載）を追加する（Issue #133）

## ゴール

`development/ui_design/` に、UI 設計の基礎を初めての非エンジニア向けに解説する連載（ハブ＋7章＋付録）を追加する。挙がった例（一貫性・説明を読まなくても分かる・肯定形と否定形をまぜない）に限らず、定番の原則を一通りそろえ、AI に画面を作らせるときの指示とレビューの観点として使える形にする。

## 方針

- 既存の連載（フロントエンドの作り方・Vibe Coding 入門・AIハーネス入門）と同じ型。ハブ README に読み方・章一覧・到達点、各章は parent / grand_parent の3階層ナビ
- 各原則は「ひとことで → なぜ → 避けたい例と改善例 → 出典」の順。例は画面の文言で示す（表にする）
- 出典は実物へのリンクだけ。書く前に HTTP ステータスで開けることを確認したものに限る。本文で名前を挙げる記事・書籍には必ずリンクか書誌を付ける
- 英語の記事は、原則の名前を日本語にし、原文の名前を添える
- 「ノンバーバル」は「説明文を読まなくても分かる」と捉え、「アイコンだけにする」とは区別する（NN/g と デジタル庁の指針に沿う）
- 図は mermaid が使えないので、テキストの図か表で示す
- 既存ページ（Vibe Coding 入門・AIハーネス入門の CLAUDE.md の書き方・フロントエンドの作り方）へ相互リンクする

## 章立て

```
development/ui_design/README.md   ハブ（parent: 開発の心得）
  01-consistency.md      一貫性 — 同じものは同じ見た目・同じ場所・同じ言葉
  02-self-evident.md     読まなくても分かる — 手がかり（シグニファイア）、記憶に頼らない、アイコンには言葉
  03-labels-switches.md  言葉とスイッチ — 肯定形と否定形をまぜない、ラベルは短く、スイッチとチェックボックス、ボタンは動詞
  04-feedback-errors.md  状態を伝える・間違いを防ぐ — フィードバック、取り消せる、エラー文、確認ダイアログ
  05-layout.md           配置と量 — 近いものはまとまる、大きく近いものは押しやすい、選択肢を減らす、余白
  06-accessibility.md    誰でも使える — 色だけで区別しない、コントラスト、文字サイズ、キーボード
  07-ai-review.md        AI に画面を作らせるときの見方 — 指示文とレビュー観点、チェックリスト
  apx-a-glossary.md      用語ミニ辞典
  apx-b-sources.md       出典と書籍
```

## 出典

- ソシオメディア「ヒューマンインターフェース ガイドライン」 — 一貫性 / シグニファイア / ユーザーの言葉を使う / ユーザーの記憶に頼らない / エラーを回避する / フィッツの法則 / ヒックの法則 / 視覚ゲシュタルト / ユーザーの主導権 / シンプルにする
- NN/g — 10 Usability Heuristics / Toggle-Switch Guidelines / Icon Usability / Error Message Guidelines / Confirmation Dialogs / Fitts's Law / Gestalt: Proximity / Recognition vs Recall
- デジタル庁デザインシステム — ボタン / スイッチ / チェックボックス / テキスト入力
- WAIC「WCAG 2.1 日本語訳」、Apple「Human Interface Guidelines」（日本語）、Material Design 3
- 書籍 — ノーマン『誰のためのデザイン？ 増補・改訂版』（新曜社）、Tidwell『デザイニング・インターフェース 第2版』（オライリー・ジャパン）

## 検査

1. `npx chaffjs development/ui_design --compact`（ジャンルは chaff.yaml の docs/manual）。指摘ごとに「直す / その場だけ stet / ルールを relax」のどれかを選び、理由を残す
2. 本文の全 URL の HTTP ステータスを確認する。相対リンクは実在を確認する
3. `development/README.md` に一覧行を足す

## 作業

1. plans/ にこのファイル
2. 出典を読み、引用する文を控える
3. ハブと各章を書く
4. 検査（上）
5. PR を出す（Closes #133）
