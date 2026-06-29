---
title: "付録A　Supabaseセットアップ手順"
parent: "Twitterクローンで学ぶWeb開発入門"
grand_parent: "開発の心得"
nav_order: 13
nav_exclude: true
---

# 付録A　Supabaseセットアップ手順

> 📖 このページのゴール：**Supabaseのプロジェクトを作り、ブラウザのコードから使える状態にする。**
> [← 目次・はじめにへもどる](README.md)

---

この付録は **手順中心** です。本編（第3章・第4章など）で出てくるSQLやコードを動かす「土台」を、ここで一度だけ用意します。上から順に番号どおりに進めれば大丈夫です。

## 🪜 手順

1. **無料アカウントを作る。** ブラウザで [https://supabase.com](https://supabase.com) を開き、右上の「Sign in」または「Start your project」を押します。**GitHubアカウントでサインイン**するのが一番かんたんです（持っていなければメールでも作れます）。

2. **新しいプロジェクトを作る。** ダッシュボードの「**New project**」を押し、次の3つを入力します。
   - **プロジェクト名（Name）**：好きな名前でOK（例：`twitter-clone`）。
   - **データベースのパスワード（Database Password）**：自分で決めます。**かならずメモして控えてください**（あとで必要になることがあり、再表示できません）。
   - **リージョン（Region）**：自分に近い場所を選びます。日本なら **Tokyo (ap-northeast-1)** がおすすめ（近いほど速い）。
   入力したら「Create new project」を押し、**準備が終わるまで1〜2分ほど待ちます**。

3. **URLと鍵（キー）を控える。** 左メニューの「**Project Settings**（歯車アイコン）」→「**API**」を開きます。次の2つを、あとでコードに貼るためにコピーしておきます。
   - **Project URL**（`https://xxxx.supabase.co` の形）
   - **anon public** key（`anon` `public` と書かれた長い文字列）

4. **作業する2つの場所を確認する。** 左メニューに、これからよく使う場所が2つあります。
   - **Table Editor** … 表（テーブル）を**画面（GUI）で作ったり中身を見たりする**場所。
   - **SQL Editor** … **SQLを実行する**場所。本編に出てくる `create table ...` などのSQLは、ここに貼って「Run」で実行します。
   いまは「どこにあるか」を見つけられればOKです。

5. **ブラウザのコードからSupabaseを使えるようにする。** HTMLの `<script type="module">` の中に、次のように書きます（フレームワークなしで動く最小の形です）。

   ```js
   import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
   const supabase = createClient('https://xxxx.supabase.co', 'ここに anon public key');
   ```

   **1行ずつ読むと：**
   - `import { createClient } from '...'` … Supabaseを使うための道具 `createClient` を、CDN（ネット上の配布先）から読み込む。
   - `createClient( ... )` … その道具で、Supabaseとやり取りする**クライアント（窓口）を作る**。
   - 第1引数 `'https://xxxx.supabase.co'` … 手順3の **Project URL**（あなたの値に置きかえる）。
   - 第2引数 `'ここに anon public key'` … 手順3の **anon public key**（あなたの値に置きかえる）。
   - 以後は、本編のコードで出てくる **この `supabase` を使って**保存・取得を行います。

---

## 🔑 鍵の注意（🟢 とても大事）

鍵（キー）には2種類あり、**扱い方がまったく違います**。

- **anon public key** … **フロント（ブラウザのコード）に置いてOK**です。名前のとおり「公開してよい鍵」。ただし「誰でも何でもできる」という意味ではなく、**RLS（行レベルセキュリティ）で守る前提**だから安全に置けます。RLSの設定は第4章で行います。
- **service_role key** … **全権限を持つ「マスターキー」**です。RLSも素通りします。**ぜったいにフロントや、GitHubなどの公開リポジトリに出さないでください**。これは**サーバー専用**で、本編の範囲では使いません。

> ⚠️ `service_role` key をブラウザのコードに書いて公開すると、**他人にデータを全部読まれる・消される**おそれがあります。第10章「ありがちな失敗」とも関わる、重要な落とし穴です。

---

## ⚠️ よくあるつまずき

- **プロジェクトが「Paused（一時停止）」になっている** … 無料枠では、しばらく使わないとプロジェクトが自動で休止します。ダッシュボードに「Restore」や「Resume」が出ていたら押して、**再開**してから使いましょう。
- **URLや鍵の貼り間違い** … `createClient(...)` でエラーが出るときは、**Project URL** と **anon key** を**もう一度コピーし直して**貼り付けます。よくあるのは、前後の空白が混じる・別の値を貼る、です。
- **リージョンを遠くにして遅い** … 日本から使うなら **Tokyo** など近い場所を選びます。遠いリージョンだと、保存や取得のたびに**待ち時間が長く**なります。

---

## 📝 ことばメモ

- **Project URL**：あなたのSupabaseプロジェクトの住所（`https://xxxx.supabase.co`）。コードの第1引数に使う
- **anon key（anon public key）**：**ブラウザに置いてよい鍵**。RLSで守る前提
- **service_role key**：**全権限のマスターキー**。サーバー専用で、フロントや公開リポジトリに出さない
- **Table Editor**：表（テーブル）を画面で作る・見る場所
- **SQL Editor**：SQLを実行する場所（本編のSQLはここで実行）

---

> 🔗 ログイン（認証）の設定は、続けて **[付録B　Google認証 設定手順](apx-b-google-auth.md)** で行います。

[← 目次・はじめにへもどる](README.md)
