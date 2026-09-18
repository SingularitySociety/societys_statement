---
title: "エンジニアの法則・名言集"
parent: "開発の心得"
nav_order: 19
---

# エンジニアの法則・名言集 — 出典つき

ソフトウェアエンジニアが、どこかで一度は聞いたことがあるはずの「法則」と「言葉」を集めました。中堅・ベテランには懐かしく、若い世代には初耳かもしれないものが並んでいます。

このページの狙いは**格言を集めることではなく、出典に戻れるようにすること**です。孫引きされるうちに言葉は短くなり、元の文脈は落ち、ときには言った人まで入れ替わります。ここでは可能な限り**一次資料（本・論文・エッセイ・RFC・メーリングリスト投稿）へのリンク**を付けました。気になったものは、ぜひ原文を読んでみてください。**短い格言より、それが書かれた前後の数段落のほうが、たいてい面白い**です。

## このページの読み方

各項目は次の形式です。

- **原文（英語）** — 引用ブロックで、言われたとおりに
- **誰が／いつ／どこで** — 人名と初出（本・論文・エッセイなど）
- **リンク** — 原文が読める場所
- **日本語の説明** — 意味と、いつ効くか

出典の確かさは 3 段階で示します。

| 記号 | 意味 |
| --- | --- |
| ◎ | 一次資料がオンラインで読める（リンク先が原文） |
| ○ | 本人の著書・講演・投稿だと分かっているが、原文がオンラインで読めない（書誌情報のみ） |
| △ | 伝聞・二次資料しかない。初出がはっきりしない |

**収録の基準**：実在の人物が実際に発した（書いた）言葉だけを載せています。フィクション作品からの引用は入れていません。

---

## 1. ソフトウェア開発の古典

### プログラマの三大美徳 — 怠惰・短気・傲慢

> **Laziness**: The quality that makes you go to great effort to reduce overall energy expenditure.
>
> **Impatience**: The anger you feel when the computer is being lazy.
>
> **Hubris**: ... the quality that makes you write (and maintain) programs that other people won't want to say bad things about.

◎ **Larry Wall**（Perl の作者）／『Programming Perl』初版（O'Reilly, 1991）巻末の用語集 — [perlglossary（現在も同じ定義が載っている）](https://perldoc.perl.org/perlglossary#laziness)

三つとも**普通なら悪徳**なのが肝。**怠惰**は「サボる」ではなく「**総エネルギーを減らすために大変な努力をする**」こと（だから労力を省くプログラムを書き、質問に何度も答えずに済むようドキュメントを書く）。**短気**はコンピュータの遅さに腹を立て、要求される前に先回りするプログラムを書くこと。**傲慢**は他人に悪く言われないコードを書き、保守し続けること。皮肉のようでいて、良いエンジニアの行動をよく言い当てています。

### DRY — 同じ知識を二度書かない

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

◎ **Andrew Hunt / David Thomas**／『The Pragmatic Programmer』（1999）Tip 15 — [原著の Tips 一覧](https://pragprog.com/tips/)

よく「コードをコピペするな」と縮められますが、原文が言っているのは**知識（knowledge）**です。同じ判断・同じルール・同じ事実が 2 か所にあると、片方だけ直してバグる。逆に、**たまたま似ているだけのコード**を無理に 1 つにまとめるのは DRY ではありません。→ [リファクタリング & プロダクト化](./refactoring.md)

### Brooks の法則 — 遅れているプロジェクトに人を足すと、もっと遅れる

> Adding manpower to a late software project makes it later.

○ **Frederick P. Brooks Jr.**／『The Mythical Man-Month』（1975）第 2 章 — [archive.org の書籍ページ](https://archive.org/details/mythicalmanmonth0000broo)

人月（man-month）は人と月が交換可能であるかのような単位だが、交換できない。新しい人には教育が必要で、しかも**コミュニケーション経路は人数の 2 乗**で増える。「遅れているから増員する」は、たいてい火に油です。

### 銀の弾丸はない

> There is no single development, in either technology or management technique, which by itself promises even one order-of-magnitude improvement within a decade in productivity, in reliability, in simplicity.

◎ **Frederick P. Brooks Jr.**／"No Silver Bullet — Essence and Accidents of Software Engineering"（IFIP 1986 / IEEE Computer 1987）— [PDF](https://worrydream.com/refs/Brooks_1986_-_No_Silver_Bullet.pdf)

ソフトウェアの難しさを **essence（本質的な複雑さ）** と **accident（道具に由来する偶発的な複雑さ）** に分け、道具の改善で消せるのは後者だけだと論じた。新しい言語・新しいフレームワーク・新しい AI が「10 倍にする」と言われるたびに読み返す価値があります。

### 一つは捨てるつもりで作れ／セカンドシステム症候群

> Plan to throw one away; you will, anyhow.

> The second is the most dangerous system a man ever designs.

○ **Frederick P. Brooks Jr.**／『The Mythical Man-Month』（1975）第 11 章・第 5 章 — [archive.org](https://archive.org/details/mythicalmanmonth0000broo)

最初のシステムは捨てることになる（どうせなら最初からそのつもりで作れ）。そして**2 番目のシステムがいちばん危ない** — 1 作目で我慢した機能を全部盛りしたくなるからです。

### フローチャートより、データ構造を見せろ

> Show me your flowcharts and conceal your tables, and I shall continue to be mystified. Show me your tables, and I won't usually need your flowcharts; they'll be obvious.

○ **Frederick P. Brooks Jr.**／『The Mythical Man-Month』（1975）第 9 章

> Bad programmers worry about the code. Good programmers worry about data structures and their relationships.

◎ **Linus Torvalds**／git メーリングリスト、2006 年 7 月 27 日 — [LWN の記事](https://lwn.net/Articles/193245/)

30 年をまたいで同じことを言っています。**設計の本体はデータ構造で、手続きはその帰結**。設計の議論が煮詰まったら、まずデータの形を描き直すと解けることが多い。

### Conway の法則 — システムは組織のコピーになる

> organizations which design systems (in the broad sense used here) are constrained to produce designs which are copies of the communication structures of these organizations

◎ **Melvin E. Conway**／"How Do Committees Invent?"（Datamation, 1968 年 4 月）— [本人のサイトにある原文](https://www.melconway.com/Home/Committees_Paper.html)

4 チームでコンパイラを作れば 4 パスのコンパイラができる。**アーキテクチャを変えたければ組織を変えろ**という「逆 Conway 作戦」の元ネタでもあります。

### 早すぎる最適化は諸悪の根源

> We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

◎ **Donald E. Knuth**／"Structured Programming with go to Statements"（ACM Computing Surveys 6-4, 1974 年 12 月, p.268）— [PDF](https://pic.plover.com/knuth-GOTO.pdf)

**後半まで引用されることが稀な、代表的な「切り取られた名言」**。Knuth は最適化するなとは言っておらず、「**残り 3% の勝負どころを見逃すな**」と続けています。要は、測らずに勘で速くしようとするなという話。

### 証明はしたが、動かしてはいない

> Beware of bugs in the above code; I have only proved it correct, not tried it.

◎ **Donald E. Knuth**／1977 年のメモ — [本人の FAQ ページ](https://www-cs-faculty.stanford.edu/~knuth/faq.html)

理屈で正しいことと、実際に動くことは別。**「ロジック的には合っているので大丈夫です」と言いたくなったときに思い出す言葉**です。

### ソフトウェア設計の二つの道

> There are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies, and the other way is to make it so complicated that there are no obvious deficiencies. The first method is far more difficult.

○ **C. A. R. Hoare**／"The Emperor's Old Clothes"（1980 年 ACM チューリング賞受賞講演、CACM 24-2, 1981）— [ACM Digital Library](https://dl.acm.org/doi/10.1145/358549.358561)

「**明らかに欠陥がない**」と「**明らかな欠陥がない**」は一字違いで大違い。前者は圧倒的に難しい、という話です。

### 10 億ドルの過ち — null 参照

> I call it my billion-dollar mistake. It was the invention of the null reference in 1965.

◎ **C. A. R. Hoare**／QCon London 2009 の講演 — [InfoQ の録画](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/)

1965 年に ALGOL W へ null 参照を入れた本人による述懐。「実装が簡単だったから入れた」結果、以後数十年のヌルポインタ例外を生んだ。最近の言語が `Optional` や null 安全を持つ理由がここにあります。

### Postel の法則（頑健性原則）

> be conservative in what you do, be liberal in what you accept from others

◎ **Jon Postel**／RFC 761（1980 年 1 月）§2.10、のちに RFC 793 にも — [RFC 761 原文](https://datatracker.ietf.org/doc/html/rfc761)

自分が送るものは厳格に、受け取るものは寛容に。インターネットが動き続けている理由のひとつであると同時に、「寛容すぎる受け入れ」がセキュリティホールと仕様の曖昧化を生むという**批判もセット**で知っておくとよい法則です。

### Kernighan の法則 — デバッグは書くより 2 倍難しい

> Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you can be when you write it, how will you ever debug it?

◎ **Brian W. Kernighan / P. J. Plauger**／『The Elements of Programming Style』第 2 版（1978）— [archive.org](https://archive.org/details/the-elements-of-programming-style-second-edition)

**よく流布している「だから、あなたは定義上、それをデバッグできるほど賢くない」という言い回しは後世の改変**で、原文は上のとおり問いかけの形です。意味は変わりません — **書けるギリギリの賢さで書くな**。

### テストはバグの存在しか示せない

> Program testing can be used to show the presence of bugs, but never to show their absence!

◎ **Edsger W. Dijkstra**／"Notes on Structured Programming"（EWD249, 1970）— [PDF](https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD249.PDF)

テストが通った＝バグがない、ではない。だからこそ「**何をテストしたか**」を言える形に設計する意味があります。→ [テストと設計の話](./refactoring.md)

### GOTO 文は有害である

◎ **Edsger W. Dijkstra**／"Go To Statement Considered Harmful"（CACM 11-3, 1968 年 3 月 / EWD215）— [EWD215 の書き起こし](https://www.cs.utexas.edu/~EWD/transcriptions/EWD02xx/EWD215.html)

構造化プログラミングの号砲。ちなみに **"Considered Harmful" というタイトルは編集者の Niklaus Wirth が付けたもの**で、以後「〇〇 considered harmful」という定型句が生まれました。

### 謙虚なプログラマ

> The competent programmer is fully aware of the strictly limited size of his own skull; therefore he approaches the programming task in full humility.

◎ **Edsger W. Dijkstra**／"The Humble Programmer"（EWD340, 1972 年チューリング賞受賞講演）— [書き起こし](https://www.cs.utexas.edu/~EWD/transcriptions/EWD03xx/EWD340.html)

**自分の頭の容量が小さいと知っているからこそ、小さく単純に作る**。「1 関数 20 行以下」のような指針の、そもそもの理由がこれです。

### UNIX 哲学 — 一つのことをうまくやれ

> Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new "features".

◎ **Doug McIlroy**／Bell System Technical Journal 1978 年 7-8 月号「UNIX Time-Sharing System」の Foreword — [archive.org](https://archive.org/details/bstj57-6-1899)

続けて「**テキストストリームを共通のインタフェースにせよ**」と言っています。パイプの思想であり、いまのマイクロサービスや CLI ツール設計にもそのまま効きます。

### Linus の法則 — 目玉の数さえあれば、バグは深くない

> Given enough eyeballs, all bugs are shallow.

◎ **Eric S. Raymond**（Linus Torvalds に因んで命名）／『The Cathedral and the Bazaar』（1997）— [原文](http://www.catb.org/~esr/writings/cathedral-bazaar/cathedral-bazaar/)

**言ったのは Linus 本人ではなく ESR** である点に注意。オープンソースの品質を支える主張ですが、Heartbleed 以降「見ている目玉が本当にあるのか」という反証もよく引かれます。

### 早くリリースし、しょっちゅうリリースせよ

> Release early. Release often. And listen to your customers.

> Every good work of software starts by scratching a developer's personal itch.

◎ **Eric S. Raymond**／『The Cathedral and the Bazaar』（1997）— [原文](http://www.catb.org/~esr/writings/cathedral-bazaar/cathedral-bazaar/)

CI/CD と継続的デリバリの思想的な祖先。後者（**自分のかゆいところを掻け**）は、そのままスタートアップのアイデア論（後述の Paul Graham）に繋がります。

### Talk is cheap. Show me the code.

> Talk is cheap. Show me the code.

◎ **Linus Torvalds**／Linux カーネルメーリングリスト、2000 年 8 月 25 日 — [LKML のアーカイブ](https://lkml.org/lkml/2000/8/25/132)

議論より動くコード。ただし原文は**具体的な技術論争の文脈で**出た一言であり、「議論は無価値」という意味ではありません。

### Worse is Better

> ... the right thing and the New Jersey approach ... it is slightly better to be simple than correct.

◎ **Richard P. Gabriel**／"Lisp: Good News, Bad News, How to Win Big"（1991）の "The Rise of Worse is Better" 節 — [原文](https://www.dreamsongs.com/WorseIsBetter.html)

MIT 流の「正しいもの（the right thing）」より、実装が単純で移植しやすい New Jersey 流（＝ C と UNIX）のほうが結果的に勝つ、という観察。**技術的に劣ったほうが普及する現象**を説明する古典で、著者自身がその後も賛否両論の続編を書き続けています。

### 技術的負債

> Shipping first time code is like going into debt. A little debt speeds development so long as it is paid back promptly with a rewrite.

◎ **Ward Cunningham**／OOPSLA '92 の経験報告「The WyCash Portfolio Management System」— [原文](https://c2.com/doc/oopsla92.html)、[本人による解説](http://wiki.c2.com/?WardExplainsDebtMetaphor)

**「汚いコード」を正当化する言葉ではない**、と本人が後年はっきり言っています。元の意味は「**そのとき理解していたことに基づいて出荷し、理解が進んだら書き直す**」。理解のズレを放置したまま作り続けると利息で潰れる、という比喩です。

### 人間が読めるコードを書け

> Any fool can write code that a computer can understand. Good programmers write code that humans can understand.

○ **Martin Fowler**／『Refactoring』（1999）第 2 章 — [書籍ページ](https://martinfowler.com/books/refactoring.html)

### 3 度目で共通化する（Rule of Three）

> Three strikes and you refactor.

○ **Martin Fowler**／『Refactoring』（1999）

1 回目は書く。2 回目は重複に目をつぶって書く（でも気づく）。**3 回目で共通化する**。早すぎる抽象化への処方箋です。

### YAGNI — たぶん要らない

> Always implement things when you actually need them, never when you just foresee that you need them.

◎ **Ron Jeffries**（Extreme Programming）— [原文](https://ronjeffries.com/xprog/articles/practices/pracnotneed/)、[Martin Fowler の解説](https://martinfowler.com/bliki/Yagni.html)

"You Aren't Gonna Need It"。**将来使うかもしれない機能**のコストは、作る手間だけでなく、以後ずっと読まれ・保守され・邪魔をし続けるコストです。

### 動かす → 正しくする → 速くする

> Make it work, make it right, make it fast.

△ **Kent Beck** に帰されることが多い／[c2 wiki の項目](http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast)（初出ははっきりせず、Beck 以前の類似表現もある）

順番が肝。**正しくする前に速くしない**（＝ Knuth）、**動かす前に正しくしようとしない**。

### ボーイスカウト・ルール

> Always leave the campground cleaner than you found it.

◎ **Robert C. Martin**／『97 Things Every Programmer Should Know』（2010）所収 — [原文](https://97-things-every-x-should-know.gitbooks.io/97-things-every-programmer-should-know/content/en/thing_08/)

触ったファイルを、来たときより少しだけきれいにして帰る。**大掃除は永遠に来ない**ので、ついでに少しずつ直すしかない、という現実的な戦略です。

### コンピュータサイエンスの二つの難問

> There are only two hard things in Computer Science: cache invalidation and naming things.

△ **Phil Karlton**（Netscape）に帰される。**本人が書いた一次資料は見つかっていない** — [Martin Fowler による出典の調査](https://martinfowler.com/bliki/TwoHardThings.html)

「…と off-by-one エラー」と続ける派生ジョークが無数にあります。命名がここに並んでいるのは冗談ではなく、**名前は設計そのもの**だからです。

### ゼロから書き直すな

> They did it by making the single worst strategic mistake that any software company can make: They decided to rewrite the code from scratch.

◎ **Joel Spolsky**／"Things You Should Never Do, Part I"（2000 年 4 月 6 日）— [原文](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/)

Netscape の全面書き直しを題材に。**汚く見える古いコードの一行一行は、たいてい誰かが踏んだバグの記録**である、という指摘が本体です。

### 漏れのある抽象化の法則

> All non-trivial abstractions, to some degree, are leaky.

◎ **Joel Spolsky**／"The Law of Leaky Abstractions"（2002 年 11 月 11 日）— [原文](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/)

ORM も TCP も、うまく隠してくれている間はいいが、**壊れた瞬間に下の層の知識を要求してくる**。だから抽象化は学習時間を減らさない、という結論が辛辣です。

### 正規表現で問題が 2 つになる

> Some people, when confronted with a problem, think "I know, I'll use regular expressions." Now they have two problems.

◎ **Jamie Zawinski**／Usenet の alt.religion.emacs、1997 年 8 月 — [スレッド](https://groups.google.com/g/alt.religion.emacs/c/DR057Srw5-c/m/drsDEeIIE5kJ)

ただし**この言い回し自体には先行例があり**（awk を槍玉に挙げた 1988 年の投稿など）、jwz が正規表現版で広めた、というのが正確なところです。

### Zawinski の法則

> Every program attempts to expand until it can read mail. Those programs which cannot so expand are replaced by ones which can.

◎ **Jamie Zawinski**／Jargon File に収録 — [原文](http://www.catb.org/jargon/html/Z/Zawinskis-Law.html)

いまなら「すべてのアプリはチャットを実装するまで膨張する」でしょうか。**機能追加圧力の普遍性**についての観察です。

### Atwood の法則

> Any application that can be written in JavaScript, will eventually be written in JavaScript.

◎ **Jeff Atwood**／Coding Horror, 2007 年 7 月 — [原文](https://blog.codinghorror.com/the-principle-of-least-power/)

2007 年時点では冗談でした。現在は事実です。

### Greenspun の第 10 法則

> Any sufficiently complicated C or Fortran program contains an ad hoc, informally-specified, bug-ridden, slow implementation of half of Common Lisp.

◎ **Philip Greenspun** — [本人のサイト](https://philip.greenspun.com/research/)

第 1〜9 法則は存在しません。大きなプログラムは、放っておくと**その場しのぎの独自インタプリタ／独自 DI ／独自設定言語**を内側に育ててしまう、という話です。

### Wirth の法則

> Software is getting slower more rapidly than hardware becomes faster.

◎ **Niklaus Wirth**／"A Plea for Lean Software"（IEEE Computer, 1995 年 2 月）— [PDF](https://cr.yp.to/bib/1995/wirth.pdf)

### Moore の法則

◎ **Gordon E. Moore**／"Cramming more components onto integrated circuits"（Electronics, 1965 年 4 月 19 日）— [PDF](https://newsroom.intel.com/wp-content/uploads/sites/11/2018/05/moores-law-electronics.pdf)

上の Wirth の法則と**セットで**覚えるのが正しい使い方です。ハードが速くなった分をソフトが食い潰す。

### Gall の法則 — 動く複雑なシステムは、動く単純なシステムから育つ

> A complex system that works is invariably found to have evolved from a simple system that worked. ... A complex system designed from scratch never works and cannot be patched up to make it work.

○ **John Gall**／『Systemantics』（1975）— [archive.org（1977 年版）](https://archive.org/details/systemanticshows00gall)

**最初から複雑なものを設計しても動かない**。まず動く単純なものを作り、それを育てるしかない。マイクロサービスから始めるべきでない理由として、いまもよく引かれます。

### The Zen of Python

> Beautiful is better than ugly. Explicit is better than implicit. Simple is better than complex. ... Readability counts.

◎ **Tim Peters**／PEP 20（1999 年投稿 / 2004 年 PEP 化）— [原文](https://peps.python.org/pep-0020/)

Python を起動して `import this` と打つと出てきます。**19 行しかなく、20 行目は今も空のまま**（「Guido に分かることは 1 つ明らかだが、それは彼が青くなるまで言わない」という註つき）。

### Perlis のエピグラム

> It is easier to write an incorrect program than understand a correct one.

> A language that doesn't affect the way you think about programming, is not worth knowing.

◎ **Alan J. Perlis**（初代チューリング賞受賞者）／"Epigrams on Programming"（ACM SIGPLAN Notices 17-9, 1982 年 9 月）— [全 130 個](http://www.cs.yale.edu/homes/perlis-alan/quotes.html)

短いものが 130 個並んでいます。**この節だけでもページを閉じて読みに行く価値がある**古典です。

### デメテルの法則 — 友達とだけ話せ

> Only talk to your immediate friends.

◎ **Karl Lieberherr** ほか（Northeastern University）／OOPSLA '88 — [論文 PDF](https://www2.ccs.neu.edu/research/demeter/papers/law-of-demeter/oopsla88-law-of-demeter.pdf)

`a.getB().getC().doSomething()` のような**数珠つなぎの呼び出し**を禁じる指針。結合度を下げ、内部構造の変更が遠くまで波及しないようにします。

### Rob Pike のプログラミング 5 つのルール

> Rule 1. You can't tell where a program is going to spend its time. ... Measure.
>
> Rule 5. Data dominates. If you've chosen the right data structures ... the algorithms will almost always be self-evident.

◎ **Rob Pike**／"Notes on Programming in C"（1989）— [原文](http://doc.cat-v.org/bell_labs/pikestyle)

ルール 1・2 は Knuth の言い換え、ルール 5 は Brooks の言い換え。**同じことが繰り返し言われている**という事実自体が、これらの重要さの証拠です。

### 驚き最小の原則は「私の驚き」最小の原則

> The principle of least surprise is not for _you_ only. The principle of least surprise means principle of least _my_ surprise.

> For me the purpose of life is partly to have joy. Programmers often feel joy when they can concentrate on the creative side of programming.

◎ **まつもとゆきひろ（Matz）**／Bill Venners によるインタビュー "The Philosophy of Ruby"（Artima, 2003 年 9 月 29 日）— [原文](https://www.artima.com/articles/the-philosophy-of-ruby)

Ruby が「驚き最小の原則（principle of least surprise）」で設計されている、と言われることへの本人の回答。**万人にとって驚きが少ないものは作れない。作者である自分にとっての驚きを最小にした結果、多くの人にとっても自然になった**という順序です。しかも「Ruby を十分に習得したあとで驚きが少ない」という条件つき。設計の指針を借りてくるときは、**それが誰にとっての最適化なのか**を確かめよ、という話でもあります。

そしてもう一つ、Ruby の目的は**プログラマを楽しくすること**だと繰り返し語っています。

### MINSWAN / MINASWAN — Matz is nice so we are nice

> MINSWAN: Matz is Nice, so we are nice.

△ **Martin Fowler** が言い出したとされる。出典は Bart Eisenberg の連載 "Software Designers — The People Behind the Code" 第 34 回「Yukihiro "Matz" Matsumoto: Ruby Inventor」（Software Design 2012 年 2 月号 / gihyo.jp）— [記事](https://gihyo.jp/dev/serial/01/software_designers/0034)。**Fowler 本人が書いた一次資料は見つかっていない**

**もとの形は "and" の入らない MINSWAN** でした。2008 年の時点でもこの綴りで使われています（[Pat Eyler のブログ](http://on-ruby.blogspot.com/2008/03/gracious-dave-and-minswan.html)：「MINSWAN (Matz is nice, so we are nice) was the order of the day」）。のちに **MINASWAN**（Matz is nice **and** so we are nice）の綴りが広まり、いまはこちらが主流です。

**技術コミュニティの空気は、中心にいる人の振る舞いで決まる**という観察。コードの話ではないのに、いちばん再現性のある「法則」かもしれません。逆に言えば、中心にいる人が刺々しければコミュニティもそうなる。

### 文句を言われる言語と、誰も使わない言語

> There are only two kinds of languages: the ones people complain about and the ones nobody uses.

◎ **Bjarne Stroustrup**（C++ の作者）— [本人の引用集ページ](https://www.stroustrup.com/quotes.html)

言語批判を見たときの解毒剤。

---

## 2. スタートアップ・プロダクト

### 未来を予測する最良の方法は、それを発明することだ

> The best way to predict the future is to invent it.

◎ **Alan Kay**／1971 年、Xerox パロアルト研究所（PARC）の会議での発言。本人が後年の記事 "Predicting The Future"（Stanford Engineering, Vol.1 No.1, 1989 年秋, pp.1-6）で経緯を書いている — [記事全文](https://www.ecotopia.com/webpress/futures.htm)

Xerox の経営陣に「コンピューティングの未来はどうなるのか」と問われたときの答え。**予測は当てるものではなく、自分で作るもの**。シンギュラリティ・ソサエティがやろうとしていることそのものです。

### 視点は IQ 80 点分の価値がある

> Point of view is worth 80 IQ points.

○ **Alan Kay**／講演で繰り返し語られ、活字の初出は 1984 年ごろ — [出典の追跡（Quote Investigator）](https://quoteinvestigator.com/2018/05/29/pov/)

頭の良さで殴るより、**問題の見え方を変えるほうが効く**。難問に詰まったら、まず問題の立て方を疑うべき、という話です。

### ソフトウェアに本気なら、自分でハードウェアを作れ

> People who are really serious about software should make their own hardware.

△ **Alan Kay**（1982 年ごろの発言とされる）／2007 年の iPhone 発表で **Steve Jobs が引用**して有名になった

垂直統合の思想。いまなら「AI に本気なら自分でチップを作れ」でしょうか。

### 人々が欲しがるものを作れ

> Make something people want.

◎ **Paul Graham / Y Combinator** のモットー（2005 年の YC 創業時から）。PG のエッセイ "Be Good"（2008）にもそのまま出てくる — [原文](https://paulgraham.com/good.html)

身も蓋もないが、**スタートアップが死ぬ理由のほとんどはこれを外したこと**です。

### アイデアは「考え出す」ものではなく「気づく」もの

> The way to get startup ideas is not to try to think of startup ideas. It's to look for problems, preferably problems you have yourself.

> The very best startup ideas tend to have three things in common: they're something the founders themselves want, that they themselves can build, and that few others realize are worth doing.

◎ **Paul Graham**／"How to Get Startup Ideas"（2012 年 11 月）— [原文](https://paulgraham.com/startupideas.html)、姉妹編 ["Organic Startup Ideas"（2010）](https://paulgraham.com/organic.html)

**人々が欲しがるものを作る一番の方法は、自分が欲しいものを作ること**。自分がユーザーなら、欲しいものが正しいか毎日検証できる。ESR の「開発者自身のかゆみ」（前節）と同じことを、プロダクトの側から言っています。

### スケールしないことをやれ

> The most common unscalable thing founders have to do at the start is to recruit users manually. Nearly all startups have to. You can't wait for users to come to you. You have to go out and get them.

◎ **Paul Graham**／"Do Things that Don't Scale"（2013 年 7 月）— [原文](https://paulgraham.com/ds.html)

最初のユーザーは**一人ずつ手で捕まえる**。Airbnb の創業者がニューヨークで一軒ずつ写真を撮って回った話が出てきます。「自動化できないから間違っている」のではなく、**最初は自動化しないほうが正しい**。

### 100 万人にまあまあ好かれるより、100 人に愛されるほうがいい

> It's better to have 100 people that love you than a million people that sort of like you.

△ **Paul Graham** が Airbnb に語った助言。**書かれたエッセイではなく口頭の助言**で、Brian Chesky が繰り返し紹介したことで広まった — [CNBC の記事](https://www.cnbc.com/2023/05/24/airbnb-ceo-brian-chesky-heres-the-best-piece-of-advice-i-ever-got.html)

熱狂している少数を作れれば、そこから広げられる。**なんとなく好かれている多数からは、何も分からない**。

### 本物のアーティストは出荷する

> Real artists ship.

◎ **Steve Jobs**／1983 年、Apple の社内リトリートでの発言。Macintosh の開発者 Andy Hertzfeld が記録している — [folklore.org](https://www.folklore.org/Real_Artists_Ship.html)

作品は出して初めて作品。ただし**当時これに追い立てられた開発陣の消耗も同じページに書いてある**ので、セットで読むのが誠実です。

### 顧客体験から始めて、技術に戻る

> You've got to start with the customer experience and work backwards to the technology.

◎ **Steve Jobs**／WWDC 1997 の質疑応答 — [動画](https://www.youtube.com/watch?v=oeqPrUmVz-o)

批判的な質問に答えた有名なくだり。「**この素晴らしい技術で何ができるか**」から始めると失敗する、という順序の話です。

### 最初のバージョンが恥ずかしくないなら、リリースが遅すぎる

> If you are not embarrassed by the first version of your product, you've launched too late.

△ **Reid Hoffman**（LinkedIn 共同創業者）／講演などで繰り返し語られているが、**一次資料は特定できていない**

### 顧客との最初の接触で、計画は生き残らない

> No plan survives first contact with customers.

◎ **Steve Blank**／"No Plan Survives First Contact With Customers"（2010 年 4 月 8 日）、原型は『The Four Steps to the Epiphany』（2005）— [原文](https://steveblank.com/2010/04/08/no-plan-survives-first-contact-with-customers-%E2%80%93-business-plans-versus-business-models/)

元ネタはモルトケの「いかなる作戦計画も敵との最初の接触を生き延びない」。だから **"Get out of the building"（建物の外に出て顧客に会え）** が処方箋になります。

### MVP と「構築 → 計測 → 学習」

> The minimum viable product is that version of a new product which allows a team to collect the maximum amount of validated learning about customers with the least effort.

◎ **Eric Ries**／"Minimum Viable Product: a guide"（2009 年 8 月）、のちに『The Lean Startup』（2011）— [原文](http://www.startuplessonslearned.com/2009/08/minimum-viable-product-guide.html)

MVP は「**手抜きの製品**」ではなく「**最小の労力で最大の学びを得るための実験**」。目的が学習である点を外すと、ただの雑なリリースになります。

### ソフトウェアが世界を食い尽くす

> Software is eating the world.

◎ **Marc Andreessen**／Wall Street Journal, 2011 年 8 月 20 日 — [a16z による全文](https://a16z.com/why-software-is-eating-the-world/)

あらゆる産業がソフトウェア企業に置き換わる、という予言。2011 年には大げさだと言われました。

### 重要なのはプロダクト・マーケット・フィットだけ

> The only thing that matters is getting to product/market fit.

◎ **Marc Andreessen**／"The Pmarca Guide to Startups, part 4"（2007 年 6 月）— [原文](https://pmarchive.com/guide_to_startups_part4.html)

市場が良ければ悪いチームでも生き残り、市場が悪ければ最高のチームでも死ぬ。**PMF の前と後では、やるべきことが全部違う**という区分けを広めた文章です。

### パラノイアだけが生き残る

> Only the paranoid survive.

○ **Andrew S. Grove**（インテル元 CEO）／『Only the Paranoid Survive』（1996）— [archive.org](https://archive.org/details/onlyparanoidsurv00grov)

**戦略的転換点（strategic inflection point）**、つまり事業の前提が変わる瞬間を見逃すな、という本。成功している最中ほど見逃します。

### イノベーターのジレンマ

○ **Clayton M. Christensen**／『The Innovator's Dilemma』（1997）— [archive.org](https://archive.org/details/innovatorsdilemm0000chri)

**優良企業が、優良な経営をしたからこそ負ける**という理論。既存顧客の声を聞き、利益率の高い市場に集中する合理的判断が、下から来る破壊的技術への対応を遅らせる。

### キャズム

○ **Geoffrey A. Moore**／『Crossing the Chasm』（1991）— [archive.org](https://archive.org/details/crossingchasmmar00moor)

アーリーアダプターとアーリーマジョリティの間には**深い溝（chasm）**がある。新しもの好きに受けたことと、普及することは別問題です。

### 競争は敗者のためのもの

> Competition is for losers.

◎ **Peter Thiel**／Wall Street Journal（2014 年 9 月）／『Zero to One』（2014）— [スタンフォード "How to Start a Startup" 講義の動画](https://www.youtube.com/watch?v=3Fx5Q8xGU8k)

完全競争では利益がゼロに収束する。だから**競争を避けられる独自の場所**を作れ、という主張。賛否ある議論ですが、議論の出発点としてよく参照されます。

### ドッグフードを食え

> Eating our own dogfood.

△ 1988 年、マイクロソフトの **Paul Maritz** が社内メールの件名に使ったのが広まったとされる

自社製品を自分たちで日常的に使う。**使っていなければ気づけない不便が必ずある**。

### いつまでも Day 1 でいる

> Day 2 is stasis. Followed by irrelevance. Followed by excruciating, painful decline. Followed by death. And that is why it is always Day 1.

◎ **Jeff Bezos**／2016 年の株主への手紙 — [原文](https://www.aboutamazon.com/news/company-news/2016-letter-to-shareholders)

Day 1 を保つ方法として、**顧客への執着・プロセスの代理化を避ける・外部トレンドへの素早い適応・高速な意思決定**の 4 つを挙げています。

### Musk の 5 ステップ — 要件を疑い、消し、単純にし、速くし、最後に自動化する

> Make the requirements less dumb. The requirements are definitely dumb; it does not matter who gave them to you.

> ... the most common error of a smart engineer is to optimize something that should not exist.

◎ **Elon Musk**／Everyday Astronaut（Tim Dodd）による Starbase でのインタビュー、2021 年 7 月 30 日 — [記事全文](https://everydayastronaut.com/starbase-tour-and-interview-with-elon-musk/)。Walter Isaacson の伝記『Elon Musk』（2023）では "the algorithm" として紹介されている

**順番を守ることが本体**の 5 ステップです。

1. **要件を「より愚かでない」ものにする** — 要件は必ずどこか愚かで、**誰が出したかは関係ない**。むしろ賢い人が出した要件のほうが危ない（疑わずに受け入れてしまうから）。「誰の要件か」は**部署名ではなく個人名**で記録せよ — 部署には理由を聞けないが、人には聞ける
2. **部品・工程を消す** — 消したもののうち **10% くらいは後で戻すことになる。戻していないなら消し足りない**
3. **単純化・最適化する** — ここが 3 番目なのが肝。**賢いエンジニアが犯す最大の間違いは、そもそも存在すべきでないものを最適化すること**
4. **サイクルタイムを速くする** — ただし上の 3 つを終えるまで速くするな
5. **自動化する** — いちばん最後

ソフトウェアに読み替えると、そのまま効きます。**存在すべきでない機能を高速化する／消せるコードをリファクタリングする／要らない手順を CI で自動化する**のは、どれも順番を間違えた形です。1 と 2 をやらずに 5 から入るのが、いちばんよくある失敗。

---

## 足りないものを募集しています

このページは**未完成であることを前提**にしています。「これが無いのはおかしい」というものが必ずあるはずです。

**X（旧 Twitter）の [@SingularitySoci](https://x.com/SingularitySoci) まで教えてください。**

教えていただくときに、できれば次の 3 つを添えてもらえると助かります（分かる範囲で結構です）。

1. **原文**（英語なら英語のまま）
2. **誰の言葉か**
3. **初出**（本・論文・エッセイ・講演・メーリングリストなど）と、読める URL

**「この項目の出典、本当はこれです」という訂正も大歓迎**です。上の表で △（伝聞のみ）になっているものを ◎（一次資料あり）にできる情報は、とくにありがたいです。

### まだ入っていないジャンル

次のジャンルは、今回は意図的に入れていません。ここを埋める提案も歓迎します。

- **人・組織・チームの法則** — Parkinson の法則、Hofstadter の法則、90-90 ルール、Chesterton の柵、Goodhart の法則、bus factor など
- **AI 時代の言葉** — The Bitter Lesson（Rich Sutton）、Amara の法則、Hyrum の法則など
- **セキュリティ** — Kerckhoffs の原理、Schneier の法則など
- **日本語圏で生まれた言葉**

---

## 関連ドキュメント

- [リファクタリング & プロダクト化（Vibe Coding 時代の心得）](./refactoring.md) — DRY・小さい関数・テスト容易性を、実際の書き方に落としたもの
- [Product について](./Product.md)
- [Product Manage について](./pm.md)
