---
title: "第5章　会話ログと複数セッション — 会話を保存して、いくつも持つ"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 5
---

# 第5章　会話ログと複数セッション — 会話を保存して、いくつも持つ

> 📖 この章のゴール：**会話を保存して、一覧から選んで、続きから話せる**ようにする（ChatGPTのサイドバーのように、**独立した会話を何本も並行して**持つ）。そのために `conversations`（会話）と `messages`（発言）の **2つの表** を設計し、RLSで「**自分の会話だけ**」見えるようにする。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 ChatGPTではこう見える

ChatGPTの画面の **左側** を思い出してください。

- 過去の会話が **いくつも縦に並んで** います（「旅行の相談」「コードの質問」…）。
- どれか1つをクリックすると、その会話が開いて、**続きから** 話せます。
- 新しく「+ New chat」を押すと、**まっさらな別の会話** が始まります。

この「会話がいくつもあって、開けば続けられる」状態を **複数セッション**（session＝ひとまとまりの会話）と呼びます。第4章では会話を1本だけ、プログラムの中（メモリ）に持っていました。でもそれだと閉じれば消えるし、1本しか持てません。この章で **保存** して、**いくつも** 持てるようにします。

> 💡 **これは「会話の枝分かれ（fork）」ではありません。** 1つの会話を途中で分岐させる話ではなく、**最初から別々の・独立した会話**を何本も持つ話です。会話どうしは **記憶を共有しません**（「旅行の相談」の内容は「コードの質問」には出てきません）。ChatGPTのサイドバーのように **切り替えながら、並行して** 進められます。

---

## 🤔 なぜ「表が2つ」いるの？（🟢 基礎）

ポイントは、**「会話」と「発言」は数が釣り合わない** ことです。

- **会話（conversation）**＝1本のおしゃべりのまとまり。タイトルが付く単位。
- **発言（message）**＝その中の1つ1つのセリフ（あなたの発言・AIの返事）。

1本の会話の中には、発言が **たくさん** 入ります。この「**1つに対して多数**」という関係を **1対多**（いちたいた）と呼びます。

> 🍱 **たとえ話**：会話は「**お弁当箱**」、発言は「**中のおかず**」。箱（会話）は1つでも、おかず（発言）は何個も入ります。だから「箱の表」と「おかずの表」を分け、おかずには「**どの箱のものか**」の札を付けます。この札が、次に出てくる `conversation_id` です。

| 表（テーブル） | 何を1行に入れる | レストランでいうと |
|---|---|---|
| **conversations**（会話） | 会話1本ぶん（タイトル・持ち主など） | 注文票の **表紙** |
| **messages**（発言） | 発言1つぶん（誰の発言か・本文） | 注文票の **明細1行** |

そして「**どの発言が、どの会話のものか**」を結ぶのが、`messages` が持つ **`conversation_id`**（会話のID＝背番号）です。これで「この会話の発言だけ」を後から集められます。

### 「自分の会話だけ見える」をどう守る？（🟢 基礎）

複数の人が使うアプリでは、**他人の会話が見えてはいけません**。これは第1弾（Twitter編）でやった **データ分離** とまったく同じ話です。

- 各会話に「**持ち主は誰か**」を表す **`user_id`**（ログインした人のID）を付ける。
- データベース側のルール **RLS**（Row Level Security＝ぎょうレベルセキュリティ＝「行ごとの見える・見えないルール」）で、**「自分の行（＝自分が持ち主の会話）だけ」** に絞る。

> 💡 Supabaseの使い方（テーブル作成・RLS・Google認証）は **第1弾とまったく同じ**です。ここでは要点だけ示します。手順をいちから知りたい人は **付録B（Supabase＋Google認証セットアップ）** を見てください。

---

## 🛠 こう作る — 2つの表とRLS、そして再開（🟢 基礎）

### ステップ1：2つの表を作る

```sql
-- 会話（お弁当箱）
create table conversations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  title text not null default '新しい会話',
  created_at timestamptz not null default now()
);

-- 発言（中のおかず）。conversation_id でどの会話かを指す
create table messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references conversations(id),
  role text not null,                       -- 'system' | 'user' | 'assistant'
  content text not null,
  created_at timestamptz not null default now()
);
```

**1行ずつ読むと：**
- `id uuid primary key default gen_random_uuid()`：会話ごとの背番号。自動で重複しないID（UUID）が振られる。
- `user_id ... references auth.users(id)`：**持ち主**。Supabaseのログイン情報（`auth.users`）とつなぐ。これがRLSの判定に効く。
- `title ... default '新しい会話'`：一覧に出す名前。まず仮の名前を入れておく。
- `conversation_id ... references conversations(id)`：その発言が **どの会話のものか** を指す札（1対多のかなめ）。
- `role text not null`：発言の種類。第4章の `system` / `user` / `assistant` をそのまま入れる。
- `content text not null`：発言の本文。
- `created_at ... default now()`：作った／発言した時刻。**この順で並べる** と会話の順番が正しく戻る。

### ステップ2：RLSで「自分の会話だけ」にする

```sql
alter table conversations enable row level security;
alter table messages enable row level security;

-- 自分が持ち主の会話だけ、見る・作る・消すができる
create policy "own conversations" on conversations
  for all using (auth.uid() = user_id);

-- 自分の会話に属する発言だけ触れる
create policy "own messages" on messages
  for all using (
    auth.uid() = (select user_id from conversations
                  where conversations.id = messages.conversation_id)
  );
```

**1行ずつ読むと：**
- `enable row level security`：その表に **行ごとの見える・見えないルール** をONにする（ONにしないとRLSは効かない）。
- `create policy "own conversations" ...`：会話の表のルールに名前を付けて作る。
- `for all using (auth.uid() = user_id)`：**いま操作している人（`auth.uid()`）と、行の持ち主（`user_id`）が一致する行だけ** 触れる。これで他人の会話は丸ごと見えなくなる。
- `create policy "own messages" ...`：発言の表のルール。
- `auth.uid() = (select user_id from conversations where ... = conversation_id)`：発言自体は持ち主を持たないので、**親の会話をたどって** 持ち主を確かめる。自分の会話の発言だけ触れる。

### ステップ3：会話を一覧する／新しく作る

```ts
// 会話の一覧（新しい順）。RLSのおかげで「自分の会話」しか返らない
const { data: conversations } = await supabase
  .from("conversations")
  .select("id, title, created_at")
  .order("created_at", { ascending: false });

// 新しい会話を1本作って、その id を受け取る
const { data: created } = await supabase
  .from("conversations")
  .insert({ user_id: userId, title: "新しい会話" })
  .select("id")
  .single();
const conversationId = created.id;
```

**1行ずつ読むと：**
- `.from("conversations").select("id, title, created_at")`：会話の表から、一覧に必要な列だけ取り出す。
- `.order("created_at", { ascending: false })`：**新しい順** に並べる（ChatGPTの左側と同じ並び）。
- `// RLSのおかげで…`：`where user_id = …` を書かなくても、**RLSが自動で自分の行だけ** に絞ってくれる（書き忘れによる事故を防げる）。
- `.insert({ user_id: userId, title: "新しい会話" })`：新しい会話を1行追加。持ち主に **いまログインしている人** を入れる。
- `.select("id").single()`：作った行の **id を1件** 受け取る。この `conversationId` を、以降の発言保存にひも付ける。

### ステップ4：発言を保存する

```ts
// あなたの発言とAIの返事を、同じ会話に保存する
await supabase.from("messages").insert([
  { conversation_id: conversationId, role: "user", content: userText },
  { conversation_id: conversationId, role: "assistant", content: replyText },
]);
```

**1行ずつ読むと：**
- `.from("messages").insert([...])`：発言の表に、複数の行をまとめて追加する。
- `{ conversation_id: conversationId, role: "user", content: userText }`：**あなたの発言**。どの会話かを `conversation_id` で結びつける。
- `{ ..., role: "assistant", content: replyText }`：**AIの返事**。LLMから返ってきた文章をそのまま保存。
- （`created_at` は省略：DBが自動で「いまの時刻」を入れてくれるので、**保存した順＝会話の順** が保たれる）

### ステップ5：会話を「再開」する（DBから第4章の `messages` 配列を組み立てる）

ここが第4章とのつなぎ目です。LLMに渡す **`messages` 配列**（`[{ role, content }, …]`）を、**メモリではなくDBから組み立て直す**——これが「再開」の正体です。

```ts
// 1) 選んだ会話の発言を、古い順に全部とる
const { data: rows } = await supabase
  .from("messages")
  .select("role, content")
  .eq("conversation_id", conversationId)
  .order("created_at", { ascending: true });

// 2) 第4章と同じ messages 配列に組み立てる
const messages = rows.map((row) => ({ role: row.role, content: row.content }));

// 3) 新しい発言を足して、いつものように create に渡す
messages.push({ role: "user", content: userText });
const completion = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages,
});
```

**1行ずつ読むと：**
- `.eq("conversation_id", conversationId)`：**その会話の発言だけ** に絞る。
- `.order("created_at", { ascending: true })`：**古い順**（＝会話が進んだ順）に並べる。ここを間違えると会話がちぐはぐになる。
- `rows.map((row) => ({ role: row.role, content: row.content }))`：DBの行を、第4章とまったく同じ `{ role, content }` の形に変換する。**これがLLMの「記憶」の正体**——DBに溜めた発言を毎回そろえ直しているだけ。
- `messages.push({ role: "user", content: userText })`：今回の新しい発言を末尾に足す。
- `openai.chat.completions.create({ model, messages })`：第4章と同じ呼び方。**LLMは相変わらず毎回忘れている**ので、こうして全部渡す。返事は再びステップ4で保存する。

### 🪟 複数の会話を並行して動かす

画面では、ChatGPTのように **複数の会話を同時に開いて切り替え**られます。やることは2つだけ：

- 画面は「**いまどの会話を表示中か**」を `activeConversationId` として1つ持つ。
- 送信・保存・履歴の組み立ては、**すべてその id にひも付ける**（ステップ4・5をその会話に対して行う）。

```ts
// 画面の状態：開いている会話のidを1つ持つ
let activeConversationId = conversationId; // 一覧でクリックすると切り替わる

// 送信時は「表示中の会話」の履歴だけを組み立てて送る
async function send(text: string) {
  const messages = await loadMessages(activeConversationId); // ← その会話だけ
  messages.push({ role: "user", content: text });
  // …LLMへ送る → 返事を activeConversationId に保存（ステップ4）
}
```

**1行ずつ読むと：**
- `activeConversationId`：いま画面に出している会話のid。サイドバーで別の会話を選ぶと、ここが切り替わる。
- `loadMessages(activeConversationId)`：**その会話の発言だけ**をDBから組み立てる（他の会話は混ざらない）。
- 送信も保存も、この id にひも付ける。これだけで会話は独立して並行に動く。

> 🧩 **なぜ並行しても混ざらない？**：サーバーは **ステートレス**（第4章）で、リクエストごとに **`conversation_id` を見て、DBからその会話の履歴だけ**を組み立て直します。だから会話Aと会話Bが**同時に走っても互いに干渉しません**——サーバーは毎回「どの会話の続きか」をリクエストの `conversation_id` だけで判断するからです。会話の状態をサーバーが覚えておく必要はありません。

> 🧠 **背骨①につながる**：会話が「続いて見える」のは、LLMが覚えているからではありません。**DBに保存した発言を、毎回そろえて渡し直している**から。記憶を作るのは、やはり開発者（あなた）の仕事です。

---

## ⚠️ ハマりどころ

- **`user_id` を付け忘れる／RLSをONにし忘れる** → **全員の会話が混ざって見えます**（第1弾の最大の教訓）。表を作ったら、まずRLSをONにして「自分の行だけ」を確認する。
- **発言の並びを `created_at` でそろえない** → 会話の順番が崩れ、LLMへの渡し方もちぐはぐに。再開のときは **必ず古い順** に並べる。
- **`conversation` と `message` を取り違える** → 「会話＝箱、発言＝中身」。一覧やタイトルは `conversations`、本文は `messages`。混ぜると設計が崩れます。
- **どの会話の発言かを取り違える**（複数を並行して開いているとき）→ 送信・保存・履歴の組み立ては必ず **表示中の `conversation_id`** にひも付ける。画面の「いま開いている会話」と、送る履歴・保存先の `conversation_id` がズレると、片方の返事がもう片方に紛れ込みます。
- 🔧 **タイトルが全部「新しい会話」のまま** → 最初の発言の先頭などから自動で付けると見分けやすい（応用。後の章で扱います）。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「Supabase で **会話（conversations）** と **発言（messages）** の2テーブルを作って。`messages` は `conversation_id` で会話にひも付け、`role`／`content`／`created_at` を持たせて。**RLSを有効化し、`auth.uid() = user_id` で自分の会話だけ** 見える・作れるようにして。発言は親の会話の持ち主をたどって絞って」

出てきたものの確認チェックリスト：

- [ ] **RLSが本当に効くか**（別アカウントでログインして、他人の会話が **見えない** ことを実際に確認）
- [ ] `messages` に **`conversation_id`** があり、会話に正しくひも付くか
- [ ] 再開のとき、発言を **`created_at` の古い順** に並べて `messages` 配列にしているか
- [ ] 一覧の取得に `where user_id = …` を書かなくても **RLS任せ** で自分のぶんだけ返るか

---

## 📝 ことばメモ

- **conversation（会話）**：1本のおしゃべりのまとまり。タイトルが付く単位（お弁当箱）
- **message（発言・メッセージ）**：会話の中の1つ1つのセリフ（中のおかず）
- **1対多（いちたいた）**：1つに対して多数がぶら下がる関係（1つの会話 ＝ 多数の発言）
- **conversation_id**：発言が「どの会話のものか」を指す札（背番号）
- **RLS（Row Level Security）**：行ごとに見える・見えないを決めるDBのルール。ここでは「自分の会話だけ」
- **セッション（session）**：ひとまとまりの会話。これを何個も持てるのが複数セッション

---

## ➡️ 次の章へ

会話を保存して、いくつも持てるようになりました。でも、このアプリを **公開して「自分以外の人」が使い始める** と、新しい問題が出ます——**使いすぎ** です。LLMは使うたびにお金がかかるので、放っておくと請求が膨らみます。

第6章では **【守り②】使いすぎを防ぐ** を扱います。ログインした人ごとに「今日はここまで」を数える、**背骨②（守り）** の後半です。

[第6章　【守り②】使いすぎを防ぐ →](06-usage-limit.md)

[← 目次・はじめにへもどる](README.md)
