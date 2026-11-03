---
title: "付録B　出典と書籍"
parent: "UI設計の基礎 — 読まなくても分かる画面の作り方"
grand_parent: "開発の心得"
nav_order: 91
---

# 付録B　出典と書籍

> 連載の入口 → [UI設計の基礎](README.md)

この連載で引いた出典の一覧です。すべて、連載を書いた時点で開けることを確認しています。

## 日本語で読める指針

**ソシオメディア「ヒューマンインターフェース ガイドライン」** — <https://www.sociomedia.co.jp/category/shig>

一項目が短く、そのまま画面レビューの観点にできます。この連載で引いた項目:

- [シンプルにする](https://www.sociomedia.co.jp/9008)
- [シグニファイア](https://www.sociomedia.co.jp/9043)
- [一貫性](https://www.sociomedia.co.jp/9055)
- [ユーザーの主導権](https://www.sociomedia.co.jp/9058)
- [視覚ゲシュタルト](https://www.sociomedia.co.jp/9099)
- [ユーザーの言葉を使う](https://www.sociomedia.co.jp/9108)
- [ユーザーの記憶に頼らない](https://www.sociomedia.co.jp/9111)
- [エラーを回避する](https://www.sociomedia.co.jp/9138)
- [フィッツの法則](https://www.sociomedia.co.jp/9146)
- [ヒックの法則](https://www.sociomedia.co.jp/9177)

**デジタル庁デザインシステム** — <https://design.digital.go.jp/>

部品ごとの使い方とアクセシビリティ。この連載で引いたページ:

- [ボタン: 使い方](https://design.digital.go.jp/components/button/usage/)、[ボタン: アクセシビリティ](https://design.digital.go.jp/components/button/accessibility/)
- [スイッチ](https://design.digital.go.jp/components/switch/)、[チェックボックス: 使い方](https://design.digital.go.jp/components/checkbox/usage/)、[テキスト入力: 使い方](https://design.digital.go.jp/components/input-text/usage/)
- [カラー](https://design.digital.go.jp/foundations/color/)、[タイポグラフィ](https://design.digital.go.jp/foundations/typography/)、[余白](https://design.digital.go.jp/foundations/spacing/)
- [ウェブアクセシビリティ](https://design.digital.go.jp/webaccessibility/)

**WAIC「WCAG 2.1（日本語訳）」** — <https://waic.jp/translations/WCAG21/>

**Apple「Human Interface Guidelines」（日本語）** — <https://developer.apple.com/jp/design/human-interface-guidelines/>

## NN/g（ニールセン・ノーマン・グループ）の記事

英語です。原則の名前と要点だけ押さえれば十分で、本文はブラウザの翻訳で読めます。

- [10 Usability Heuristics for User Interface Design](https://www.nngroup.com/articles/ten-usability-heuristics/) — 10原則の本体
- [Jakob's Law of Internet User Experience](https://www.nngroup.com/videos/jakobs-law-internet-ux/) — 利用者は他のサイトと同じ動きを期待する
- [How Users Read on the Web](https://www.nngroup.com/articles/how-users-read-on-the-web/) — 人は読まない、拾い読みする
- [Recognition vs. Recall in UX](https://www.nngroup.com/articles/recognition-and-recall/) — 記憶より認識
- [Icon Usability](https://www.nngroup.com/articles/icon-usability/) — アイコンにはテキストラベル
- [Toggle-Switch Guidelines](https://www.nngroup.com/articles/toggle-switch-guidelines/) — スイッチのラベルと使いどころ
- [Confirmation Dialogs Can Prevent User Errors — If Not Overused](https://www.nngroup.com/articles/confirmation-dialog/) — 確認は重大なときだけ、ボタンは結果が分かる言葉で
- [Error-Message Guidelines](https://www.nngroup.com/articles/error-message-guidelines/) — エラー文の書き方
- [The Principle of Proximity in Gestalt Psychology](https://www.nngroup.com/articles/gestalt-proximity/) — 近いものはまとまって見える
- [Fitts's Law and Its Applications in UX](https://www.nngroup.com/articles/fitts-law/) — 大きく近いものは押しやすい

## 書籍

| 書籍 | 向いている目的 |
| --- | --- |
| D. A. ノーマン『[誰のためのデザイン？ 増補・改訂版 — 認知科学者のデザイン原論](https://www.shin-yo-sha.co.jp/book/b455574.html)』新曜社、2015年 | 人が操作に迷う理由と、分かりやすいデザインの原理を理解する。シグニファイア・フィードバック・制約の出どころ |
| Jenifer Tidwell『[デザイニング・インターフェース 第2版](https://www.oreilly.co.jp/books/9784873115313/)』オライリー・ジャパン、2011年 | 画面構成・ナビゲーション・入力などの解決パターンを実務で引く。画面例には時代を感じるが、パターンの体系は今も使える |

前者は考え方の土台、後者はパターン集です。まず一冊なら前者です。

## 道具

- [eslint-plugin-jsx-a11y](https://www.npmjs.com/package/eslint-plugin-jsx-a11y)（React）、[eslint-plugin-vuejs-accessibility](https://www.npmjs.com/package/eslint-plugin-vuejs-accessibility)（Vue） — アクセシビリティの基本を lint で指摘する
- Chrome「[Lighthouse の概要](https://developer.chrome.com/docs/lighthouse/overview?hl=ja)」 — ブラウザからアクセシビリティ診断を実行する

---

連載の入口 → [UI設計の基礎](README.md)
