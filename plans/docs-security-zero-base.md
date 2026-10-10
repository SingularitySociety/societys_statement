# 「セキュリティはゼロベースで考える」を追加する（Issue #104）

## ゴール

原稿「セキュリティはゼロベースで考える」を `development/security/security-zero-base.md` として追加する。
[セキュリティ脅威マップ](../development/security/README.md) は「攻撃者はどこから入るか」の分類。このページはその手前に置く、「そもそも何を持つか・誰に任せるか・何を信じるか」の考え方。

## 方針

- 原稿の構成（持たない／自前で作らない／過信しない／続ける／全体で守る／まとめ）と語り口は残す。直すのは語と文、足すのは説明・根拠・リンク
- 名前を挙げる制度・基準には実物へのリンクを付ける。開けることを確認したものだけ
- 原稿の主張のうち公表統計と食い違うもの（漏えいの原因）は、出典つきで言い直す
- 脅威マップの各入口ページへ繋ぎ、同じ話を二度書かない
- 連載の型に合わせて「関連ページ・出典」を末尾に置く。チェックリストは置かない（著者の判断）

## chaff の使い方

[plans/docs-twitter-supabase-chaff.md](docs-twitter-supabase-chaff.md) の手順に従う。ジャンルは `chaff.yaml` の `docs/manual`。

1. 原稿をそのまま保存し、`npx chaffjs <原稿> --compact` で検査する
2. `npx chaffjs fix-plan <原稿>` で直し方の計画を読む（指針・守ること・勧める深さ）
3. `npx chaffjs facts <原稿>` で事実の目録を取ってから書き直す
4. 書き直したら `npx chaffjs <新ファイル> --compact` と `npx chaffjs compare <原稿> <新ファイル>`。足した事実（URL・固有名詞・見出し）が、すべて意図したものか一つずつ確かめる
5. `npx chaffjs outline <原稿> <新ファイル>` で構成の変化を見る

## 作業

1. plans/ にこのファイル
2. `development/security/security-zero-base.md` を書く
3. `development/security/README.md`（ハブ）と `development/README.md` から辿れるようにする
4. 本文の全 URL の HTTP ステータスを確認する
5. PR を出す（Closes #104）
