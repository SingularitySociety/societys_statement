---
title: "付録H　会話データのスキーマ例（conversations / messages / usage / memories）"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 22
nav_exclude: true
---

# 付録H　会話データのスキーマ例（conversations / messages / usage / memories）

> 📖 このページのゴール：この教材で使うテーブルを一望し、そのままコピーして使える。
> [← 目次・はじめにへもどる](README.md)

---

このページは、第5章（会話ログ）・第6章（usage）・第9章（要約）・第10章（メモリ）でバラバラに出てきた表（テーブル）を、**1ページにまとめた“地図”**です。各章でその都度くわしく説明しているので、ここは「**全体を見渡す**」「**そのままコピーして使う**」ための索引として使ってください。保存先は前提どおり **Supabase（中身は Postgres というデータベース）** です。

---

## 🗺 全体像

4つの表が、どうつながっているかを先に見ます（矢印は「→ の先を指している」の意味）。

```text
  ┌─────────────────────────┐
  │  auth.users (Supabaseが用意) │  ← ログインした人。ここは自分で作らない
  └─────────────┬───────────┘
                │ user_id（持ち主）
   ┌────────────┼───────────────┬───────────────┐
   ▼            ▼               ▼               ▼
conversations  usage          memories      （どれも user_id で
（会話＝箱）   （使った量）   （覚えた事実）   本人にひも付く）
   │ id
   │
   ▼ conversation_id（どの会話か）
messages（発言＝中身）
```

**1行ずつ読むと：**
- `auth.users` … **Supabaseが最初から持っている**ログイン台帳。自分では作らない。すべての表の `user_id` は、ここの人を指す。
- `conversations` … 会話1本＝1行（お弁当箱）。`user_id` で持ち主に、`id` で発言につながる。
- `messages` … 発言1つ＝1行（中のおかず）。`conversation_id` で「どの会話のものか」を指す（1対多）。
- `usage` … 「誰が・どの日・何回・何トークン」使ったかの台帳（第6章）。会話とは直接つながらず、人と日付でまとまる。
- `memories` … 会話をまたいで覚える「事実」（第10章）。会話に関係なく、人ごとにぶら下がる。

---

## 🧱 テーブル定義

そのままコピーして、SupabaseのSQLエディタに貼れば作れます。**上から順に**（conversations → messages → usage → memories）実行してください（messagesは conversations を参照するので順番が大事）。

### conversations（会話＝箱）

```sql
create table conversations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  title text not null default '新しい会話',
  summary text not null default '',
  created_at timestamptz not null default now()
);
```

**1行ずつ読むと：**
- `id ... default gen_random_uuid()` … 会話の背番号。自動で重複しないID（UUID）が振られる。
- `user_id ... references auth.users(id)` … 持ち主。ログイン台帳とつなぐ。RLSの判定はここを見る。
- `title ... default '新しい会話'` … 一覧に出す名前。まず仮の名前を入れておく。
- `summary ... default ''` … **これまでの会話の要約**（第9章で追加）。最初は空っぽで、会話が伸びると中身が入る。
- `created_at ... default now()` … 作った時刻。会話一覧を新しい順に並べるのに使う。

### messages（発言＝中身）

```sql
create table messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references conversations(id),
  user_id uuid not null references auth.users(id),
  role text not null,                        -- 'system' | 'user' | 'assistant'
  content text not null,
  status text not null default 'done',       -- 'done' | 'pending' | 'error'
  created_at timestamptz not null default now()
);
```

**1行ずつ読むと：**
- `conversation_id ... references conversations(id)` … その発言が **どの会話のものか**を指す札（1対多のかなめ）。
- `user_id ... references auth.users(id)` … 持ち主。発言からも本人をたどれるよう持たせる（RLSが書きやすくなる。後述）。
- `role text not null` … 発言の種類。`system`（最初の設定）／`user`（あなた）／`assistant`（AI）のどれか。
- `content text not null` … 発言の本文。
- `status text not null default 'done'` … 発言の状態（第7章）。ふだんは `done`（完了）。返事を生成中なら `pending`、途中で失敗したら `error` を入れて、**順番が崩れた発言を見分ける**のに使う。
- `created_at ... default now()` … 発言した時刻。**この順で並べる**と会話の順番が正しく戻る。

### usage（使った量＝台帳）

```sql
create table usage (
  user_id uuid not null references auth.users(id),
  day date not null,
  count int not null default 0,
  tokens int not null default 0,
  primary key (user_id, day)
);
```

**1行ずつ読むと：**
- `user_id ...` … 誰の記録か（第6章の `req.userId` が入る）。
- `day date not null` … いつの日の記録か（1日ごとに1行）。
- `count int not null default 0` … その日の呼び出し回数。最初は0。
- `tokens int not null default 0` … その日に使ったトークン量（料金の単位。付録D参照）。
- `primary key (user_id, day)` … **「人＋日」で1行**と決める。同じ人の同じ日は必ず1行にまとまる。

### memories（覚えた事実）

```sql
create table memories (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  fact text not null,
  created_at timestamptz not null default now()
);
```

**1行ずつ読むと：**
- `id ... default gen_random_uuid()` … 事実1つごとの背番号。あとで「古い事実を消す・上書きする」とき、この背番号で1件をねらえる。
- `user_id ...` … 誰のメモリか（持ち主）。
- `fact text not null` … 覚える事実そのもの（例「名前は太郎」）。空はダメ。
- `created_at ... default now()` … いつ覚えたかの時刻。

---

## 🔒 RLS

公開アプリでは「**他人の会話・他人のメモリが見えてはいけない**」。これを守るのがRLS（Row Level Security＝行ごとの見える・見えないルール）です。**全部の表でONにし、「自分の行だけ」**に絞ります。

```sql
-- まず4つの表すべてでRLSをONにする（ONにするまでは全公開なので注意）
alter table conversations enable row level security;
alter table messages      enable row level security;
alter table usage         enable row level security;
alter table memories      enable row level security;

-- どの表も「自分が持ち主の行だけ」さわれる
create policy "own conversations" on conversations
  for all using (auth.uid() = user_id);

create policy "own messages" on messages
  for all using (auth.uid() = user_id);

create policy "own usage" on usage
  for all using (auth.uid() = user_id);

create policy "own memories" on memories
  for all using (auth.uid() = user_id);
```

**1行ずつ読むと：**
- `enable row level security` … その表で「行ごとの見える・見えないルール」をONにする。**4つ全部に必要**（1つでも忘れると、その表だけ全公開になる）。
- `for all using (auth.uid() = user_id)` … 読み書き全部について、「**いま操作している人（`auth.uid()`）＝その行の持ち主（`user_id`）**」のときだけ許可する。これで他人の行は丸ごと見えなくなる。
- `messages` も **`user_id` を持たせている**ので、第5章のように親の会話をたどらず、`auth.uid() = user_id` の**同じ一文**で書ける（ポリシーがシンプルで間違えにくい）。

---

## ⚡ インデックス

インデックス（索引）は、本の巻末さくいんと同じ。**よく「この条件でしぼり込む」列**にあらかじめ張っておくと、表が大きくなっても検索が速いままです。この教材でよく使う絞り込みに合わせて、次を張ります。

```sql
-- 「ある会話の発言を、古い順に取る」が速くなる（第5章の再開・第9章の直近N件）
create index messages_conv_time on messages (conversation_id, created_at);

-- 「この人の・この日の使用量」を一発で引ける（第6章の上限チェック）
-- usage は primary key (user_id, day) が索引も兼ねるので、追加は不要
```

**1行ずつ読むと：**
- `messages (conversation_id, created_at)` … `conversation_id` でしぼり、`created_at` で並べる——という**いつもの取り方とぴったり同じ並び**の索引。会話を開くたびの検索がぐっと速くなる。
- `usage` のコメント … `primary key (user_id, day)` を作った時点で、その2列の索引も**自動で**できている。だから「この人の・この日」を引くための追加インデックスは要らない。

> 💡 「なぜUUIDを主キーにするのか」「どの列に索引を張るべきか」をもっと基礎から知りたい人は、第1弾 Twitterクローンで学ぶWeb開発入門 — はじめに・目次 の **付録G「ID・インデックス」** が詳しいです（考え方はそのまま同じ）。

---

## ⚠️ ハマりどころ

- **RLSの有効化を忘れる** … 表を作っただけで `enable row level security` を実行しないと、**その表は全公開**（誰でも全行を読める）。**4つ全部**でONにしたか、作った直後に確認する。
- **`user_id` を入れ忘れる** … 行に持ち主が入っていないと、RLSの `auth.uid() = user_id` に合わず、**自分のなのに見えない**／逆に**他人のが見える**事故に。insert のたびに `user_id` を必ず入れる（DB側で `default auth.uid()` を付けておくと入れ忘れにくい）。
- **`created_at` で並べ替えない** … 発言を取り出すとき順番を指定しないと、**会話がちぐはぐ**に。再開・要約のときは必ず `created_at` の昇順（古い順）で並べる。

---

## 📝 ことばメモ

- **主キー（primary key）**：その表の中で1行を**ただ1つに決める**目印。`conversations` なら `id`、`usage` なら「`user_id`＋`day`」の組
- **外部キー（foreign key／references）**：別の表の行を**指し示す札**。`messages.conversation_id` が `conversations.id` を指す、など
- **RLS（Row Level Security）**：行ごとに見える・見えないを決めるDBのルール。ここでは「自分の行だけ」（`auth.uid() = user_id`）
- **インデックス（索引）**：よく絞り込む列に張る“さくいん”。表が大きくても検索が速いまま
- **status**：発言の状態（`done`／`pending`／`error`）。途中で止まった・失敗した発言を見分ける（第7章）

---

[← 目次・はじめにへもどる](README.md)
