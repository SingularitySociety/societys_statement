---
title: "Git/GitHub 入門（非エンジニア向け）"
parent: "Gitを使った開発について"
grand_parent: "開発の心得"
nav_order: 1
---

# Git/GitHub 入門（非エンジニア向け）

はじめて Git と GitHub を使う人のための入門です。
覚えるコマンドは **10個くらいだけ**。この範囲で「1人での開発」から「チーム開発（コンフリクト対応まで）」までひと通りできます。

このドキュメントは、チームのルール [Gitを使った開発について](./git.md) に沿っています。
とくに次の2つは、そのまま本文の前提になっています。

- **rebase ではなく merge**（履歴を書き換える高度な操作は使わない）
- **コミットも Pull Request も小さく**（小さいほど安全で、あとで出てくる「コンフリクト」も減る）

---

## Git と GitHub ってなに？

| なまえ | ひとことで |
| --- | --- |
| **Git（ギット）** | ファイルの**変更履歴を記録する道具**。自分のパソコンの中で動く |
| **GitHub（ギットハブ）** | その記録を**インターネット上で共有する場所**。チームの共有本棚 |

### なぜ必要？

Git を使わないと、こうなりがちです。

```text
企画書_最終版.docx
企画書_最終版_v2.docx
企画書_最終版_v2_修正済み.docx
企画書_本当に最終.docx   ← どれが最新…？
```

Git があると：

- **いつでも過去の状態に戻れる**（記録した時点＝セーブポイントが残る）
- **誰が・いつ・何を・なぜ変えたか**が全部残る
- **複数人で同時に作業しても**、あとで安全に合体できる

プログラムはファイル同士が連動して動くので、「1ファイルだけ古い」と簡単に壊れます。Dropbox や Google Drive のような「ファイル置き場」ではなく、**変更の歴史ごと管理する** Git が使われるのはこのためです。

---

## ことばのミニ辞典

先にざっと眺めておくと、あとが楽です。

| ことば | かんたんな意味 |
| --- | --- |
| リポジトリ | プロジェクト一式（ファイル＋変更の記録帳）が入った箱 |
| コミット | 「ここまでの変更」をひとまとまりで記録すること。**ゲームのセーブ** |
| ブランチ | 完成版には手を付けず、**下書きのコピー**で作業するしくみ |
| main（メイン） | いちばん大事なブランチ。**みんなが見る完成版** |
| プルリクエスト（PR） | 「この下書き、完成版に反映してください」という**提案ページ** |
| マージ | 下書きの内容を完成版に**合体**させること |
| コンフリクト | 2人が同じ場所を別々に変えて、Git が「**どちらにしますか？**」と聞いてくる状態 |
| push（プッシュ） | 自分の記録を GitHub に**送る** |
| pull（プル） | GitHub の最新を手元に**取り込む** |
| clone（クローン） | GitHub にあるリポジトリ一式を手元に**まるごとコピー**する（最初の1回だけ） |

## 使うコマンドはこれだけ

| コマンド | ひとことで | 使う場面 |
| --- | --- | --- |
| `git init` | このフォルダの記録を始める | 新規プロジェクトで最初に1回 |
| `git status` | いまの状態を見る | **迷ったらまずこれ** |
| `git diff` | 何をどう変えたかを見る | コミットする前の確認 |
| `git add ファイル名` | 記録する変更を選ぶ | コミットの直前 |
| `git commit -m "メモ"` | 選んだ変更を記録（セーブ） | 小さな区切りごとに |
| `git show` | 直前のコミットの中身を見る | 記録できたかの確認 |
| `git switch -c 名前` | 下書きブランチを作って移動 | 作業を始めるとき |
| `git switch 名前` | ブランチを移動する | main に戻るときなど |
| `git push` | 自分の記録を GitHub へ送る | コミットのあと |
| `git pull` | GitHub の最新を取り込む | 作業の前、マージのあと |
| `git clone URL` | リポジトリ一式を手元へコピー | 参加するとき最初に1回 |

---

## 第1部　ひとりで使う — 記録の基本

### 準備（最初に1回だけ）

Git に「あなたが誰か」を教えます。コミットの記録に名前が残ります。

```bash
git config --global user.name "自分の名前"
git config --global user.email "自分のメールアドレス"
git config --global pull.rebase false
```

3行目は、あとで出てくる `git pull` を**チームのルール通り merge 方式で動かす**ための設定です（設定しないままだと、pull のときに「どの方式で取り込むか決めてください」という英語の長いメッセージが出て止まることがあります）。

3つとも、実行しても**何も表示されません**。Git のコマンドは「**何も出ない＝成功**」のことが多いので、覚えておいてください。

### 1. 記録を始める — `git init`

プロジェクトのフォルダの中で1回だけ実行します。

```bash
cd my-project
git init
```

実行結果の例：

```text
Initialized empty Git repository in /Users/hanako/my-project/.git/
```

「空のリポジトリを用意しました」という意味です。これでこのフォルダは「Git が変更を記録するフォルダ（リポジトリ）」になりました。見た目は何も変わりませんが、記録帳が用意された状態です。

### 2. 変更を確認する — `git status` と `git diff`

ファイルを直したら、まず状態を見ます。例として、すでに一度コミット（記録）してある `menu.txt` の営業時間を「9時から」→「10時から」に直したとします。

```bash
git status   # どのファイルが変わった？（一覧）
```

実行結果の例：

```text
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   menu.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

英語ですが、見るのは真ん中の1行だけで大丈夫です。「`modified: menu.txt`（menu.txt が変更されています）」と教えてくれています。

```bash
git diff     # 中身がどう変わった？（行ごとの差分）
```

実行結果の例：

```text
diff --git a/menu.txt b/menu.txt
index 192a06d..d491afb 100644
--- a/menu.txt
+++ b/menu.txt
@@ -1,4 +1,4 @@
 今日のメニュー
 ・コーヒー 400円
 ・紅茶 350円
-営業時間は 9時から17時です。
+営業時間は 10時から17時です。
```

`-` の行が**変更前**、`+` の行が**変更後**です（表示が長いときは `q` キーで閉じます）。
**「迷ったら `git status`」** — これはこの先ずっと使える合言葉です。

なお、**作ったばかりの新しいファイル**は、`git status` では「Untracked files（まだ一度も記録していないファイル）」という欄に表示されます。それも次の add → commit で同じように記録できます。

### 3. 記録する — `git add` と `git commit`

記録は2ステップです。**「選んでから、セーブする」**と覚えてください。

```text
✍️ ファイルを直す
 │    （git status / git diff で確認）
 ▼
📦 git add ファイル名
 │    「この変更を次の記録に含めます」と選ぶ
 ▼
📸 git commit -m "何をしたか"
      選んだ変更をセーブ（コミット）
 │
 └→ また直す → add → commit → …（小さく何度でも）
```

```bash
git add menu.txt
```

`git add` は成功しても**何も表示されません**。ちゃんと選べたかは `git status` で確認できます：

```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   menu.txt
```

さっきと表示が変わり、「Changes to be committed（次のコミットに含まれる変更）」の欄に移りました。続けてセーブします。

```bash
git commit -m "営業時間の誤りを修正"
```

実行結果の例：

```text
[main 78c5351] 営業時間の誤りを修正
 1 file changed, 1 insertion(+), 1 deletion(-)
```

「main ブランチに、1ファイル・1行増えて1行減った変更を記録しました」という意味です。`78c5351` はこのコミットに自動で付いた**整理番号（の先頭7文字）**で、実行するたびに違う値になります。

コミットしたあとに `git status` を実行すると：

```text
On branch main
nothing to commit, working tree clean
```

「記録していない変更はありません」＝**全部セーブ済み**のしるしです。

- メッセージは**日本語でOK**。「何をしたか」をひとことで書きます。
- **コミットは小さく、こまめに**。「営業時間を直した」「写真を差し替えた」のように、1つの用事ごとにセーブします。[チームのルール](./git.md)にもある通り、**コミット＝進捗**です。小さければ、間違えたときも小さく戻れます。

### 4. 記録を確かめる — `git show`

```bash
git show
```

実行結果の例：

```text
commit 78c5351d14bb344810c3393c9ed81715c0c6b22a
Author: 山田花子 <hanako@example.com>
Date:   Wed Jul 15 09:28:44 2026 +0900

    営業時間の誤りを修正

diff --git a/menu.txt b/menu.txt
index 192a06d..d491afb 100644
--- a/menu.txt
+++ b/menu.txt
@@ -1,4 +1,4 @@
 今日のメニュー
 ・コーヒー 400円
 ・紅茶 350円
-営業時間は 9時から17時です。
+営業時間は 10時から17時です。
```

直前のコミットの「誰が・いつ・何を変えたか」が表示されます（長いときは `q` キーで閉じます）。

過去のコミットの一覧を見たいときは `git log` です（これも `q` で閉じます）。実行結果の例：

```text
commit 78c5351d14bb344810c3393c9ed81715c0c6b22a
Author: 山田花子 <hanako@example.com>
Date:   Wed Jul 15 09:28:44 2026 +0900

    営業時間の誤りを修正

commit 3d5968e151dbd4fa62c11c02d73c58f98d0ad6d7
Author: 山田花子 <hanako@example.com>
Date:   Wed Jul 15 09:28:44 2026 +0900

    メニューの最初の版を作成
```

新しい記録が上に来ます。日記帳を最後のページからさかのぼって読むイメージです。

ここまでが記録の基本サイクルです。**直す → status/diff で確認 → add → commit** を体が覚えるまで繰り返してください。

---

## 第2部　ひとりでも「ブランチ → PR → マージ」

ここからが本題です。GitHub と組み合わせて、実際の開発と同じ流れで作業します。

### なぜ main に直接書かないの？

main は「**みんなが見る完成版ノート**」です。完成版に直接ペンを入れるのではなく、**コピー（下書き）を作ってそちらに書く**のが Git の作法です。この下書きが**ブランチ**です。

```text
📗 main（完成版ノート）
 │    いつでも人に見せられる、正式な版
 │
 │  コピーをとって…（ブランチを作る）
 │
 └──→ 📝 作業ブランチ（自分の下書き）
        ここでは自由に試してOK。
        書き損じても、完成版ノートは無傷のまま
```

下書きが完成したら、「**完成版に反映してください**」とお願いします。これが**プルリクエスト（PR）**で、反映することを**マージ**と呼びます。

```text
📗 main（完成版）
 │
 │                 📝 作業ブランチ（下書き）
 │                  ├─ 直し① をコミット
 │                  ├─ 直し② をコミット
 │                  │
 │ 「反映してください！」＝ プルリクエスト（PR）
 │ ←────────────────┘
 │
 ▼  内容を確認して、ボタンを押すと…
📗 main（新しい完成版） ← 下書きの内容が合体した ＝ マージ ✅
```

1人で作業していてもこの流れを使います。理由は：

- 「何をどう変えたか」が PR という**まとまった記録**で残る
- 完成版がいつも無傷なので、**失敗を恐れず試せる**
- あとでチームに人が増えても、**同じやり方のまま**でいい

### ブランチは怖くない — 5つの安心ポイント

ブランチは「慎重に扱う特別なもの」ではなく、**メモ用紙のように気軽に使い捨てるもの**です。

1. **いくつでも作れる**
   「コピー」といっても、実際はメモ1枚ぶんくらいの軽いしくみです。一瞬で作れて、何十個作ってもかまいません。

2. **失敗してもいい**
   下書きで何をしても、完成版（main）は無傷です。うまくいかなかったら、そのブランチは捨てて（放っておいて）、新しいブランチで最初からやり直せばOK。誰にも迷惑はかかりません。

3. **作業している間に main が進んでも大丈夫**
   下書きしている間に、ほかの人の変更が main に入っても問題ありません。あとから `git pull origin main` で**最新の main を自分の下書きに取り込めます**（やり方は第4部で説明します）。

4. **途中でもコミットしておけば、いつでも別の作業に移れる**
   作業が中途半端でも、「作業中」のままコミットして大丈夫です。コミットさえしておけば、ブランチを切り替えても変更は消えず、戻ってくれば続きがそのまま残っています。
   例：作業の途中で、急ぎの修正を頼まれたら —

   ```bash
   git add menu.txt
   git commit -m "作業中"      # 途中のままセーブ（汚くてOK）
   git switch main             # いったん main に戻って
   git switch -c fix-urgent    # 急ぎ用のブランチで対応
   ```

   急ぎの用事が済んだら、`git switch もとのブランチ名` で続きに戻れます。

5. **コミットは汚くてもいい**
   「作業中」「とりあえず動いた」のようなコミットが並んでも、気にしないでください。過去のコミットに戻したり取り消したりする操作は、**実際の開発ではほとんど使いません**。きれいであるべきなのは、最後に main に合体する**結果**（コードと PR の説明）であって、途中経過ではありません。

> 💡 **stash / worktree は、まだ使わない**
> 調べものをすると `git stash`（変更の一時退避）や `git worktree`（作業フォルダの複製）という道具が出てきますが、どちらも「いま自分がどこで何を編集しているのか」が分からなくなりやすく、**最初は混乱のもと**です。「**途中でもブランチにコミットして進める**」（ポイント4）だけで、同じことが全部できます。

### GitHub でリポジトリを用意する

GitHub と組み合わせるときは、**先に GitHub 上でリポジトリを作って、手元にコピー（clone）する**のがいちばん簡単です。

1. GitHub にログイン →「New repository」で新規作成
2. 表示された URL をコピーして、手元に持ってくる：

```bash
git clone https://github.com/自分のアカウント名/my-project.git
cd my-project
```

（第1部のように手元で `git init` したフォルダを後から GitHub に載せることもできます。その場合は、GitHub でリポジトリを作ったときに画面に表示されるコマンド数行をそのまま貼り付ければOKです。）

### 作業の流れ（全体図）

```text
① git switch -c 下書き名     … 下書きブランチを作る
② 直して add / commit        … 小さくセーブ（何回でも）
③ git push                   … GitHub に送る
④ GitHub の画面で PR を作る   … 「完成版に反映して」と提案
⑤ 内容を確認してマージ        … 完成版に合体（ボタンを押す）
⑥ git switch main → git pull … 手元の完成版も最新にする
```

順番にやってみましょう。

#### ① ブランチを作る

```bash
git switch -c fix-hours
```

実行結果の例：

```text
Switched to a new branch 'fix-hours'
```

`-c` は「新しく作る」の意味です。名前は `fix-hours`（営業時間を直す）、`add-menu-photo`（メニュー写真を足す）のように、**やることが分かる英単語**を付けます。

#### ② 直して、小さくコミット

第1部と同じです。

```bash
git status
git diff
git add menu.txt
git commit -m "営業時間の誤りを修正"
```

#### ③ GitHub に送る（push）

```bash
git push -u origin fix-hours
```

実行結果の例：

```text
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 20 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 315 bytes | 315.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
remote:
remote: Create a pull request for 'fix-hours' on GitHub by visiting:
remote:      https://github.com/自分のアカウント名/my-project/pull/new/fix-hours
remote:
To https://github.com/自分のアカウント名/my-project.git
 * [new branch]      fix-hours -> fix-hours
branch 'fix-hours' set up to track 'origin/fix-hours'.
```

上半分の `Enumerating...` はデータを送っている進捗表示なので、読まなくて大丈夫です。注目してほしいのは `remote:` の行 — **GitHub が「PR を作るならこの URL からどうぞ」と案内してくれています**。次の④でそのまま使えます。

新しいブランチを**最初に送るときだけ** `-u origin ブランチ名` を付けます（「GitHub 側にもこの下書きの置き場を作ってね」という意味です）。2回目からは `git push` だけで送れます。

#### ④ プルリクエストを作る

push の実行結果に表示された URL をそのまま開くか、ブラウザで GitHub のリポジトリを開くと出ている**「Compare & pull request」という緑のボタン**を押して：

- **タイトル**：何をしたか（例：営業時間の誤りを修正）
- **説明**：なぜ直したか、確認してほしいこと

を書いて「Create pull request」を押します。

#### ⑤ 確認してマージ

PR の画面の「**Files changed**」タブで、変更内容を自分の目でもう一度確認します。よければ「**Merge pull request**」ボタンを押します。

> ⚠️ マージボタンの横の「▼」で方式を選べますが、必ず「**Create a merge commit**」を選んでください。「Squash」や「Rebase」は[チームのルール](./git.md)で使いません（履歴をそのまま残す merge 方式に統一しています）。

マージ後に「Delete branch」ボタンが出たら押してOKです。役目を終えた下書きは消して構いません。

#### ⑥ 手元の main を最新にする

マージは GitHub 上で起きたので、手元のパソコンの main はまだ古いままです。取り込みましょう。

```bash
git switch main
git pull
```

`git switch main` の実行結果の例：

```text
Switched to branch 'main'
Your branch is up to date with 'origin/main'.
```

`git pull` の実行結果の例：

```text
remote: Enumerating objects: 1, done.
remote: Counting objects: 100% (1/1), done.
remote: Total 1 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
From https://github.com/自分のアカウント名/my-project
   8edad98..131095e  main       -> origin/main
Updating 8edad98..131095e
Fast-forward
 menu.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

switch の時点で「up to date（最新です）」と表示されていますが、これは手元がまだ GitHub に確認しに行っていないだけです。気にせず `git pull` すると、最後の2行の通り、マージされた変更（menu.txt）がちゃんと届きます。

これで1周です。**次の作業も、必ず①のブランチ作成から**始めます。

---

## 第3部　チームで使う

チームになっても、やることは第2部と**ほぼ同じ**です。GitHub が「共有の本棚」になり、全員がそこを経由してやりとりします。

```text
        ☁️ GitHub（共有の本棚）
        📗 みんなの完成版（main）
          ↑ push …… 自分の変更を送る
          ↓ pull …… みんなの変更を受け取る

 💻 Aさんの手元   💻 Bさんの手元   💻 あなたの手元
   （それぞれがコピー一式を持ち、自分の下書きで作業する）
```

### 参加するとき（最初の1回）

```bash
git clone https://github.com/チーム名/プロジェクト名.git
cd プロジェクト名
```

実行結果の例（`my-project` というリポジトリの場合）：

```text
Cloning into 'my-project'...
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 7 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (7/7), done.
Resolving deltas: 100% (2/2), done.
```

`my-project` というフォルダができて、中に**ファイル一式と、これまでの変更履歴のすべて**が入っています。

### チームの鉄則3つ

1. **作業を始める前に、main で `git pull`**
   ほかの人の変更が毎日 main に入ってきます。**古い完成版から下書きを作らない**こと。

   ```bash
   git switch main
   git pull
   git switch -c 新しいブランチ名
   ```

   すでに最新のときは、`git pull` はこう表示します：

   ```text
   Already up to date.
   ```

2. **2人以上なら、PR には必ずレビューをもらう**
   自分でマージボタンを押す前に、チームメイトに PR を見てもらいます（[チームのルール](./git.md)）。レビューする側は「Files changed」タブで変更を読み、気になる行にコメントし、よければ Approve します。

3. **PR は小さく、早くマージする**
   数分〜長くても数時間でレビューできる大きさが目安です（[チームのルール](./git.md)）。大きい PR は読んでもらえず、マージが遅れ、次に説明する**コンフリクトの原因**にもなります。

### 1日のリズム

```text
朝：  git switch main → git pull   （最新の完成版から始める）
　　  git switch -c 今日の作業名     （下書きを作る）
日中：直す → add → commit を小さく繰り返す → git push
夕方：PR を作る → レビューをもらう → マージ
　　  git switch main → git pull    （また最新にして、次へ）
```

---

## 第4部　コンフリクト — 怖くない

チーム開発でいちばん最初につまずくのが**コンフリクト（衝突）**です。先に結論を言うと：

- コンフリクトは**故障でもエラーでもありません**
- Git からの「**どちらが正しいか、人間が選んでください**」という**質問**です
- 落ち着いて答えれば、必ず解決できます

### どういうときに起きる？

**2人が、同じファイルの同じあたりを、別々に変えたとき**だけ起きます。

```text
もとの文：      「営業時間は 9時から17時です。」

Aさんの下書き： 「営業時間は 10時から17時です。」 （開店を遅らせた）
Bさんの下書き： 「営業時間は 9時から18時です。」  （閉店を延ばした）

先に A さんの下書きが main にマージされました。
続いて B さんの下書きをマージしようとすると……

Git 「同じ行が両方で変わっています。
      どちらが正しいか、私には決められません。
      人間さん、選んでください」 ＝ これがコンフリクト
```

逆に言うと、**別のファイル**や**同じファイルの離れた場所**なら、Git が自動で上手に合体してくれます。コンフリクトは毎回起きるものではありません。

### 起きたらどうする？

PR の画面に「**This branch has conflicts that must be resolved**」と表示されたときの対処です。2つの方法があります。

#### 方法1：GitHub の画面上で直す（簡単なとき）

表示のそばにある「**Resolve conflicts**」ボタンを押すと、ブラウザ上のエディタが開きます。次の「マークの読み方」の要領で直して、「Mark as resolved」→「Commit merge」を押せば完了です。数行程度のコンフリクトならこれが一番手軽です。

#### 方法2：手元のパソコンで直す（基本形）

自分のブランチに、main の最新を**合流**させてから直します。先ほどの図の例で言うと、あなたは B さん。`fix-closing-time` というブランチで、閉店時間を 18時 に延ばしたところだとしましょう。

```bash
git switch fix-closing-time
git pull origin main
```

`git pull origin main` は「main の最新を、いまのブランチに取り込んで合体して」という意味です（この合体も merge です。rebase は使いません）。コンフリクトがあると、実行結果はこうなります：

```text
remote: Enumerating objects: 6, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 4 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
From https://github.com/チーム名/プロジェクト名
 * branch            main       -> FETCH_HEAD
   8edad98..131095e  main       -> origin/main
Auto-merging menu.txt
CONFLICT (content): Merge conflict in menu.txt
Automatic merge failed; fix conflicts and then commit the result.
```

見るのは最後の2行だけです。「**menu.txt でコンフリクトしました。直してから、コミットしてください**」と言っています。

ここで慌てず、合言葉の `git status`：

```text
On branch fix-closing-time
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   menu.txt
```

`both modified: menu.txt` ＝「menu.txt を**両方が変更しています**」。つまり、質問が出ているファイルの一覧です。
（ちなみに2行目のヒントの通り、いったんやり直したくなったら `git merge --abort` で合体を中止して、pull する前の状態に戻ることもできます。）

### マークの読み方

コンフリクトしたファイル（menu.txt）を開くと、問題の場所に Git が**質問のマーク**を書き込んでいます。

```text
今日のメニュー
・コーヒー 400円
・紅茶 350円
<<<<<<< HEAD
営業時間は 9時から18時です。
=======
営業時間は 10時から17時です。
>>>>>>> 131095e33da1c5326c5b9a0eef73ce01b848f87e
```

- `<<<<<<< HEAD` から `=======` まで：**自分のブランチ**の内容（閉店を18時に延ばした）
- `=======` から `>>>>>>>` まで：**取り込もうとした相手（main）**の内容（開店を10時に遅らせた）
- `>>>>>>>` の後ろの長い英数字は、相手側のコミットの整理番号です。読まなくてOK
- ぶつかっていない行（メニューの部分）は、**そのまま無事に残っています**。質問されているのはマークで挟まれた部分だけ

直し方は、**マークの行ごと全部消して、正しい1つの内容に書き直す**だけです。どちらか片方を採用してもいいし、両方を活かして書き直してもかまいません。直したあとのファイルはこうなります：

```text
今日のメニュー
・コーヒー 400円
・紅茶 350円
営業時間は 10時から18時です。
```

（この例では「開店を遅らせた」「閉店を延ばした」の両方の意図を活かしました。迷ったら、**変更した本人に聞く**のがいちばん確実です。）

直したら、いつもの記録と同じです：

```bash
git add menu.txt
git commit -m "営業時間のコンフリクトを解消"
git push
```

`git commit` の実行結果の例：

```text
[fix-closing-time f204cb6] 営業時間のコンフリクトを解消
```

push すると、PR の画面からコンフリクトの警告が消えて、マージできるようになります。

### コンフリクトを減らすコツ

コンフリクトは「別々の下書きが長く離れているほど」起きやすくなります。つまり、これまでのルールがそのまま予防策です。

- **作業前に必ず `git pull`**（最新から始める）
- **PR を小さくして、早くマージする**（下書きを長生きさせない）
- **同じファイルを同時にいじりそうなら、先にひとこと相談する**
- 作業が数日にまたがるときは、**途中でも `git pull origin main`** して差を小さくしておく

---

## 困ったときは

| 症状 | 対処 |
| --- | --- |
| なんだか分からなくなった | まず `git status`。いまの状態を教えてくれる上に、次にやるべきコマンドのヒントも表示してくれます |
| 間違えて main のまま作業してしまった（コミット前） | そのまま `git switch -c 新しいブランチ名`。変更ごと新しいブランチに移動できます |
| `git push` が「rejected」と言われた | 誰かが先に push しています。`git pull` で取り込んでから、もう一度 `git push` |
| 過去のコミットに戻したい・消したい | まず一呼吸。**過去に戻す・取り消す操作は、実際の開発ではほとんど使いません**（コミットしてあれば消えないので、新しいコミットで直せば済みます）。どうしても必要なら、詳しい人や AI に `git status` の出力を見せて相談を |

コマンドの出力をそのままコピーして、チームの詳しい人や AI（Claude や ChatGPT）に貼り付けて聞くのは、まったく恥ずかしいことではありません。全員そうやって覚えました。

---

## 合わせて読む

- [Gitを使った開発について](./git.md) — チームのルール（merge 方式・コミットと PR の粒度）。本ドキュメントの前提
- [Vibe Coding 入門（非エンジニア向け）](./vibe_coding_simple.md) — AI と一緒に開発するときの心得
- [GitHub ActionsではじめるCI/CD](./github_actions/README.md) — push や PR をきっかけに、テストやチェックを自動で走らせるしくみ
