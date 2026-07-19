---
title: "サービスの広め方"
parent: "ブートキャンプ"
nav_order: 5
---

# サービスの広め方 — 作ったものを世に出す

作っただけでは、誰にも知られません。ここでは Web サービス・アプリを広めるチャネルを、**日本向け**と**グローバル向け**に分けて整理します。

全部やる必要はありません。**「誰に届けたいか」で選ぶ**のが唯一のコツです。

---

## 大原則（日本・グローバル共通）

### 1. スパイク型と複利型を、両方持つ

| 種類 | チャネル | 性質 |
| --- | --- | --- |
| **スパイク型** | Product Hunt / Hacker News / Reddit / はてなブックマーク | 一気に跳ねるが、数日で減衰する |
| **複利型** | SEO・技術ブログ / GitHub / ニュースレター / ディレクトリ被リンク | 立ち上がりは遅いが、資産として積み上がる |

スパイク型だけを追うと、**毎回ゼロから集客し直す**ことになります。複利型を並行して仕込んでください。

### 2. 最初の 100 人は「スケールしないこと」で集める

チャネル戦略よりも、**直接の声かけ・ユーザーインタビュー・手動オンボーディング**の方が速いです。最初はチャネルを増やすより、一人ひとりに会いに行きましょう。

### 3. 計測する

チャネルごとに **UTM パラメータ**を付け、**流入数ではなく「登録・課金」で評価**します。感覚で「Xが効いた気がする」と判断しないこと。

### 4. AI 検索時代への対応

2026年現在、**検索の約6割はゼロクリック**（AI が答えて終わる）です。つまり **LLM に引用されること**自体が新しい流入経路になっています。Product Hunt やディレクトリへの掲載は、単発の流入よりも **被リンクと AI 引用（AI citations）** を生むことに価値があります。「SEO」を検索エンジン対策だけと捉えないでください。

### 5. コミュニティは「先に貢献」する

ローンチ前からコミュニティで活動していた人は、コールドアウトリーチ比で **3〜5倍のコンバージョン**という報告があります。宣伝より、質問に答えて信頼を作る方が効きます。

### 6. ⚠️ やってはいけないこと（アカウント・ドメインを失います）

- **upvote の購入、投票リング、身内の応援コメント（サクラ）**
  Hacker News のリング検知は強力で、**URL のシャドウバン、最悪ドメイン永久BAN**。Reddit も即BANです。
- **各コミュニティのルールを読まずに投稿する**
  同じ内容でも、あるサブレでは歓迎され、別のサブレでは即BANです。

---

## 日本向け

国内ユーザーが対象なら、主戦場はここです。

| チャネル | 向いているもの | 効果 |
| --- | --- | --- |
| **Zenn** | 技術記事のメイン | ★★★★★ |
| **Qiita** | エンジニアへのリーチ | ★★★★☆ |
| **note** | 「なぜ作ったか」の物語・検索流入 | ★★★★☆ |
| **はてなブックマーク** | 増幅装置（ホットエントリ） | ★★★★★ |
| **X（日本語）** | 開発ログ・デモ | ★★★★★ |
| **YouTube Shorts** | 一般向け・説明が要るもの | ★★★★☆ |

### Zenn — 技術記事の主軸に

はてなブックマーク3件以上の記事数で見ると、**2024年に Zenn が Qiita を逆転**しており、成長性でも優位です。技術的な内容はまず Zenn に置くのが効率的。

### Qiita — リーチはまだ大きい

エンジニアへの到達という意味では依然有効。Zenn と併用する人も多いです。

### note — 「なぜ作ったか」を書く

技術ではなく **背景・想い・意思決定**を書く場所。検索流入も拾えます。Zenn（技術）と note（物語）の二本立てが定番です。

### はてなブックマーク — 増幅装置として使う

自分で投稿する場所というより、**Zenn / note の記事がホットエントリに入ると一気に跳ねる**、という増幅装置です。狙って書くのではなく、良い記事の結果として付いてきます。

### X（日本語） — 具体的に書く

抽象的な宣伝文は読まれません。

> ❌ AIがあなた専用のTODOアプリを作ります。
>
> ✅ 30秒で営業管理アプリを生成できます。

投稿するもの: 開発ログ / Before・After / 面白い技術 / ユーザーの反応 / リリース。

### 日本向け：最初の 100〜1000 人を集める順番

1. 知り合い・想定ユーザーに直接見せる（スケールしないこと）
2. X で開発ログを毎日投稿する
3. Zenn に技術記事を書く（結果としてはてブ狙い）
4. note に「なぜ作ったか」を書く
5. 関連コミュニティ（Slack / Discord / connpass 等）で質問に答える
6. YouTube Shorts でデモ動画を出す

---

## グローバル向け

海外ユーザー・開発者が対象ならこちらです。**コミュニティごとの作法が厳しい**ので、ルールを読んでから投稿してください。

| チャネル | 向いているもの | 効果 |
| --- | --- | --- |
| **GitHub** | OSS・開発ツール | ★★★★★ |
| **Hacker News** | 開発ツール・技術的に面白いもの | ★★★★★ |
| **Reddit** | ニッチ・技術系 | ★★★★★ |
| **X (Twitter)** | 開発者・スタートアップ | ★★★★★ |
| **Product Hunt** | 新サービスのローンチ | ★★★★☆ |
| **LinkedIn** | B2B・経営者 | ★★★★☆ |
| **Bluesky** | 開発者（飽和していない） | ★★★☆☆ |
| **Indie Hackers** | 個人開発・SaaS | ★★★★☆ |
| **dev.to / daily.dev** | 開発者 | ★★★☆☆ |
| **ニュースレター** | 開発者 | ★★★★☆ |
| **ディレクトリ** | 初期流入・被リンク | ★★★☆☆ |
| **Discord / Slack** | コミュニティ | ★★★★☆ |

### GitHub — OSS なら最重要

- README を充実させる、**GIF を載せる**、Examples を増やす
- GitHub Topics を設定する、Star History を見せる
- **毎週少しずつ更新**するとおすすめに載りやすい

### Hacker News — 「Show HN」の作法を守る

- **Show HN は「実際に試せる／動かせるもの」専用**です。ブログ記事・サインアップページ・ニュースレター・まとめ記事は **対象外**（通常投稿を使う）
- 可能なら **サインアップ不要**で試せる形にする
- 投稿者は**スレッドに張り付いて質問に答える**
- タイトル例: `Show HN: GraphAI`
- マーケティング臭い文章は嫌われます
- **⚠️ 友人・同僚・ユーザーに upvote や応援コメントを依頼しないこと。** ドメインを永久に失います

### Reddit — サブレごとにルールが全く違う

- **⚠️ r/programming（600万人）は極めて厳格**。ユーザーの課題を直接解決する文脈以外でのツール言及は不可。**r/SaaS で許されることが r/programming では即BAN**になり得ます
- **r/SideProject（約18万人）は show-and-tell 目的**なので自己紹介OK。ただし「何を作ったか・なぜ作ったか・技術構成・欲しいフィードバック」を書くこと。**リンクだけの投稿は伸びません**
- 有名な「90/10ルール」は**既に廃止**され、今はモデレーターがアカウントの総合的な振る舞いで判断します。ただし「自分の製品リンクは投稿・コメントの 1/10 以下」という目安は生きています
- **投稿前に必ずサイドバーのルールを読む**
- 勝ち筋は「**1つのサブレを選び、3ヶ月は返信だけ、その後にローンチ投稿**」

宣伝ではなく「こういう課題ありませんか？」という問いかけが伸びます。コメントも重要。

### Product Hunt — 期待値を正しく持つ

- 投稿時刻は **12:01am PT** 固定
- 曜日はトレードオフ: **火〜木＝流入最大だが競争が激しい** / **金〜日＝#1バッジは取りやすいが上限は低い**
- **Featured されるのは約10ローンチに1つ**。非Featured は upvote がいくつでも PH 経由流入が**約70%減**
- **hunter の重要性は 2018〜22年より低下**したが消滅はしておらず、激戦カテゴリの初回ローンチでは最初の4時間の勢いに効きます。2回目以降は self-hunt で十分
- **30日前から準備**する（プロフィール育成、アイコン・ギャラリー・デモ・タグライン・maker comment、最初の6時間に動く応援リスト）
- 報酬は2つ: **36時間のスパイク**と、**後からじわじわ効く被リンク・AI引用**（こちらが複利で効く）

Launch 日には **Product Hunt / X / LinkedIn を同時に流します**。PH だけ出しても伸びません。

### X (Twitter) / Bluesky

X は依然として強い一方、**Bluesky は飽和しておらず**、ゼロから始めて X よりフォロワー・反応が伸びたという個人開発者の報告もあります。母数は X が大きいので、**併用**が現実的です。

### LinkedIn — 海外 B2B なら必須

X より文章は長めに。投稿するもの: 開発背景 / 技術選定 / KPI / 導入事例。

### Indie Hackers / dev.to / daily.dev / Stack Overflow

Reddit と Indie Hackers を合わせると **月間240万人以上**の SaaS 関心層に届きます。dev.to・daily.dev・Stack Overflow も開発者マーケの主要面です。

### ニュースレター

- **TLDR は13媒体で720万人超**。掲載・スポンサー枠が効きます
- ただし**開発者向けニュースレターへの広告出稿は ROI が低い**（良質なコンテンツを適切なコミュニティに置く方が速い）
- **自前のメールリストが最強の資産**です

### ディレクトリ — 数ではなく質

原則は **「厳選12件 ＞ 雑な100件」**。ディレクトリは**質が複利で効き、件数は効きません**。

- 一般: **Uneed**（ニュースレター露出・穏やかで息が長い）/ **Peerlist**（Launchpad）/ **Fazier**（技術寄りの高関心層）/ **BetaList**（プレローンチ・waitlist）/ **DevHunt**（無料・dofollow）
- B2B の信頼獲得: **G2 / Capterra**
- 収益直結: **AppSumo / PitchGround**

### AI ディレクトリ（AI サービスなら）

2026年時点でいずれも現役です。まずはこの3つに絞るのが安全。

- **Toolify** — 最大規模・更新が最も速い
- **There's An AI For That** — 巨大（数千万人規模の利用）
- **Futurepedia** — 小規模だが厳選

### YouTube Shorts / Discord・Slack

ショート動画は画面録画だけでも十分（30秒デモ / 1分説明 / 新機能紹介）。Discord・Slack コミュニティ（AI・TypeScript・Firebase・Vue・React など）では、**宣伝より質問に答えて信頼を作る**方が効果があります。

### ローンチの順序

1. **プレローンチ** — LP ＋ waitlist、BetaList
2. **ソフトローンチ** — コミュニティで少人数にフィードバックをもらう
3. **本番ローンチ** — Product Hunt ＋ Hacker News ＋ X ＋ LinkedIn を**同日に集中**
4. **ロングテール** — 技術ブログ・SEO・ディレクトリ掲載

### グローバル向け：最初の 100〜1000 人を集める順番

1. GitHub を整える（OSS なら）
2. X で開発ログを毎日投稿する
3. Reddit で課題ベースの投稿をする（3ヶ月は返信だけ）
4. Hacker News に「Show HN」を出す
5. Product Hunt で正式リリースする
6. AI ディレクトリ・一般ディレクトリに厳選登録する
7. 技術ブログを書く（複利型）
8. Discord・Slack コミュニティで活動する
9. YouTube Shorts でデモ動画を出す
10. LinkedIn で開発ストーリー・導入事例を発信する

---

## サービス種別のおすすめ

| 種別 | 特に効くチャネル |
| --- | --- |
| **開発者向け・OSS** | GitHub / Hacker News / Reddit / X / Product Hunt |
| **AI サービス** | 上記 ＋ AI ディレクトリ（Toolify / There's An AI For That / Futurepedia） |
| **B2B SaaS** | LinkedIn / SEO / 技術ブログ / 導入事例 / メールニュースレター / G2・Capterra |
| **B2C** | YouTube / TikTok / Instagram Reels |
| **国内向け** | X（日本語）/ Zenn / note / はてなブックマーク |

---

## 効果が薄いもの（初期フェーズ）

- **広告** — ユーザー像（ICP）が定まる前は低ROI。ただし **PMF・ICP が固まった後は有効**です
- **プレスリリースだけに頼ること**
- **Facebook ページ運営**
- **Instagram**（開発ツール系）
- **TikTok**（B2B・開発者向け）

---

## 参考リンク

- [Show HN Guidelines](https://news.ycombinator.com/showhn.html) / [Hacker News Guidelines](https://news.ycombinator.com/newsguidelines.html)
- [Reddit self-promotion rules (2026)](https://redship.io/blog/reddit-self-promotion-rules) / [r/SideProject Rules (2026)](https://www.mediafa.st/subreddit/sideproject)
- [How to Launch on Product Hunt in 2026](https://smollaunch.com/guides/launching-on-product-hunt) / [Product Hunt Launch Checklist 2026](https://getlaunchlist.com/blog/how-to-launch-on-product-hunt-2026)
- [Where to Launch Your Startup in 2026: 80+ Directories](https://bigideasdb.com/where-to-launch-your-startup-2026)
- [Best AI Tools Directories in 2026](https://stackbuilt.co/blog/best-ai-tools-directories-2026)
- [Note・Qiita・Zenn の使い分け](https://zenn.dev/soshi1234/articles/note-qiita-zenn-comparison) / [データで見る Qiita と Zenn の比較](https://qiita.com/mima_ita/items/5961d4d572c9e97e3f29)
- [7 Best Developer Marketing Channels (2026)](https://www.infrasity.com/blog/top-developer-marketing-channels) / [Indie Hacker Marketing Playbook 2026](https://prems.ai/blog/indie-hacker-marketing-playbook-2026)

---

## 合わせて読む

- [ブートキャンプ](../bootcamp.md)
- [デモを作る難しさ](./demo.md)
- [短期/長期プロジェクトアイデア](./short-term-projects.md)
