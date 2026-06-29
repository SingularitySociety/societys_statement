---
title: "第6章　【守り②】使いすぎを防ぐ — 認証ユーザーごとの上限"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 6
---

# 第6章　【守り②】使いすぎを防ぐ — 認証ユーザーごとの上限

> 📖 この章のゴール：**なぜ公開時にログイン＋使用量制限が要るか**を自分の言葉で説明でき、**ユーザーごとの上限（回数／トークン／1日）をサーバー側で実装**できるようになる。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 ChatGPTではこう見える

ChatGPTをたくさん使っていると、ときどきこんな表示に出会います。

- 「**いまは混み合っています**。しばらくしてからお試しください」
- 「**この期間の上限に達しました**。○時間後にまた使えます」

これは故障ではありません。**わざと止めている**のです。なぜ、わざわざ止めるのでしょう？ その理由が分かると、自分のクローンにも同じ仕組みが必要だと腹落ちします。

第1章の「**たくさん使うと、ときどき制限がかかる**」の正体が、この章のテーマです。

---

## 🤔 なぜ上限が要る？やらないとどうなる（🟢 基礎）

LLMは **従量課金（じゅうりょうかきん）**＝**使った分だけお金がかかる**仕組みです。電気やガスと同じで、たくさん使えば請求も増えます。そして料金は、第2章で見たとおり **鍵の持ち主（あなた）に請求**されます。

手元（localhost）で自分だけが使っているうちは、これで問題ありません。使いすぎても、困るのは自分だけだからです。

問題は **公開して「自分以外」の人が使えるようになった瞬間**に起きます。

- 誰かが（あるいは自動プログラム＝**bot（ボット）**が）**休みなく無限に叩いたら**どうなる？
- そのたびにあなたのサーバーがLLMを呼び、**あなたの財布に請求が積み上がります**。
- 一晩で**数万円・数十万円**——という事故が、実際に起きています。

だから公開アプリには、次の2段構えが要ります。

> **① ログイン（認証）で「誰が使っているか」を確定する**
> → **② その人ごとに使った量を数えて、上限を超えたら止める**

これが第2章で予告した「**ローカルでは不要／公開して初めて要る**」の回収です。背骨②の **「財布（使用量）を守る」** に当たります。

> ⚠️ **認証 ≠ 認可（にんか）**——ここを混同しないこと。
> - **認証（Authentication）**＝「**あなたは誰か**」を確かめること（＝ログイン）。
> - **認可（Authorization）**＝「**その人に、何をどれだけ許すか**」を決めること（＝上限・権限）。
>
> 「ログイン済み＝無制限に使ってOK」**ではありません**。ログインで誰かが分かったうえで、**「1日◯回まで」という許可の線を引く**のが認可です。両方そろって初めて財布が守れます。

> 🔧 **応用：レート制限とクォータは別物**
> - **レート制限（Rate Limit）**＝**短時間の連打**を防ぐ（例：1分に20回まで）。一気に叩かれる事故への備え。
> - **クォータ（Quota）**＝**長い期間の総量**に上限（例：1日100回まで）。じわじわ使われる事故への備え。
>
> この章では分かりやすい **「1日の上限（クォータ）」** を作ります。連打対策のレート制限も大事ですが、**詳しくは付録G**にまとめます。

---

## 🛠 こう作る — ログイン必須＋1日の上限（🟢 基礎）

作戦は3ステップです。**① ログイン中の人だけ `/api/chat` を使える**ようにし、**② その人が今日どれだけ使ったかを数え**、**③ 上限を超えたら止める**。

### ステップ1：ログインしている人だけ通す（認証）

```ts
// auth.ts — リクエストに付いてきたトークンから「誰か」を確定する
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!);

async function requireUser(req, res, next) {
  const token = req.headers.authorization?.replace("Bearer ", "");
  const { data, error } = await supabase.auth.getUser(token);
  if (error || !data.user) {
    return res.status(401).json({ error: "ログインが必要です" });
  }
  req.userId = data.user.id;
  next();
}
```

**1行ずつ読むと：**
- `import { createClient } ...`：Supabase（保存とログインを担当する道具）を読み込む。
- `createClient(process.env.SUPABASE_URL!, ...)`：自分のSupabaseに接続する。URLや鍵は **第2章と同じく `.env` から**読む。
- `async function requireUser(req, res, next)`：本番の処理の**前に挟む見張り役**（これを「ミドルウェア」と呼びます）。`next()` を呼ぶと次へ進みます。
- `req.headers.authorization?.replace("Bearer ", "")`：ブラウザが付けてきた**ログインの証明書（トークン）**を取り出す。Googleログイン後にSupabaseが発行したものです。
- `await supabase.auth.getUser(token)`：そのトークンが本物か、**Supabaseに確認**してもらう。
- `if (error || !data.user)`：本物でなければ——
- `res.status(401).json(...)`：**HTTP 401**（＝「あなたが誰か分からない」）を返して**門前払い**。
- `req.userId = data.user.id`：本物なら、**この人のID**を後ろの処理へ渡す。
- `next()`：チェック通過。本番の `/api/chat` へ進ませる。

> 💡 トークン（証明書）は、ブラウザが `fetch` のときに `headers: { Authorization: "Bearer <トークン>" }` の形で付けます。Googleログインの詳しい設定は **付録B**（第1弾と共通）。

### ステップ2：使った量を数える台帳（usageテーブル）

「誰が・いつ・何回・何トークン使ったか」を覚える表を、Supabase（Postgres）に作ります。

```sql
-- usage: ユーザーごと・日付ごとの使用量
create table usage (
  user_id uuid not null,
  day date not null,
  count int not null default 0,
  tokens int not null default 0,
  primary key (user_id, day)
);
```

**1行ずつ読むと：**
- `create table usage (...)`：`usage`（使用量）という名前の表を作る。
- `user_id uuid not null`：**誰の**記録か（ステップ1の `req.userId` が入る）。
- `day date not null`：**いつの日**の記録か（1日ごとに1行）。
- `count int not null default 0`：その日の**呼び出し回数**。最初は0。
- `tokens int not null default 0`：その日の**使ったトークン量**（トークン＝料金の単位。第8章・付録Dで詳しく）。
- `primary key (user_id, day)`：**「人＋日」で1行**と決める（同じ人の同じ日は1行にまとまる）。

### ステップ3：上限チェック→加算（認可）

`/api/chat` の**いちばん最初**で今日の使用量を見て、上限オーバーなら止めます。通れば処理し、最後に**使った分を足します**。

```ts
const DAILY_LIMIT = 100; // 1日の上限回数（マジックナンバーにせず名前を付ける）

app.post("/api/chat", requireUser, async (req, res) => {
  const today = new Date().toISOString().slice(0, 10); // "2026-06-29"

  const { data: usage } = await supabase
    .from("usage").select("count")
    .eq("user_id", req.userId).eq("day", today).maybeSingle();

  if ((usage?.count ?? 0) >= DAILY_LIMIT) {
    return res.status(429).json({ error: "本日の上限に達しました。明日また使えます" });
  }

  const completion = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: req.body.message }],
  });

  await supabase.rpc("increment_usage", {
    p_user_id: req.userId,
    p_day: today,
    p_tokens: completion.usage?.total_tokens ?? 0,
  });

  res.json({ reply: completion.choices[0].message.content });
});
```

**1行ずつ読むと：**
- `const DAILY_LIMIT = 100`：上限回数を**名前付きの定数**にする（コードの途中に直接 `100` と書かない）。
- `app.post("/api/chat", requireUser, ...)`：窓口に**ステップ1の見張り役を挟む**。これだけで「ログイン必須」が成立。
- `new Date().toISOString().slice(0, 10)`：今日の日付を `"2026-06-29"` の形で取り出す（時刻は捨てて日付だけ）。
- `supabase.from("usage").select("count")`：`usage` 表から回数だけを読む。
- `.eq("user_id", req.userId).eq("day", today)`：**この人の・今日の**行に絞り込む。
- `.maybeSingle()`：1行（まだ無ければ無し）として受け取る。
- `if ((usage?.count ?? 0) >= DAILY_LIMIT)`：今日の回数が上限**以上**なら——（記録がまだ無ければ0として扱う）。
- `res.status(429).json(...)`：**HTTP 429**（＝「使いすぎ」）を返して**止める**。LLMはまだ呼んでいないので、ここで止めれば**お金はかからない**。
- `openai.chat.completions.create({ ... })`：通った人だけ、LLMに代理で頼む（第2章と同じ）。
- `await supabase.rpc("increment_usage", { ... })`：処理が済んだら、**今日の回数を+1し、使ったトークンを足す**（後述の関数を呼ぶ）。
- `p_tokens: completion.usage?.total_tokens ?? 0`：OpenAIが教えてくれる**実際に使ったトークン数**を渡す。
- `res.json({ reply: ... })`：返事の文章だけをブラウザへ返す。

> 💡 `increment_usage` は「**無ければ作り、あれば足す**」を1回でやるDB側の小さな関数（**UPSERT**＝update or insert）。`count` を `+1`、`tokens` を `+渡された値` するだけです。中身のSQLは付録Gに置きます。**数える処理はDB側で1回にまとめる**のがコツ（同時アクセスでも数え間違えにくい）。

> ✅ **正解のしるし**：ブラウザの送信ボタンをいくら連打しても、**サーバーが上限で429を返して止める**。フロントを細工して回避しようとしても、**サーバーで止まっているので無駄**——これが守れていれば合格です。

---

## ⚠️ ハマりどころ

- **フロントのボタンを無効化（disabled）しただけ＝無意味。** ボタンの状態はブラウザ側の都合。**サーバーを直接叩けば素通り**します。第2章と同じ鉄則——**止めるのは必ずサーバー側**。フロントの無効化は「親切な見た目」にすぎません。
- **匿名（ログイン無し）でも使える窓口を残す** → 数えようにも「誰か」が無いので、**上限がかけられません**。公開するなら匿名利用は**絞る**（IP単位でごく少回数のみ）か、**不可**にする。
- **数え漏れ／数えすぎ。** LLM呼び出しが**失敗したのに加算**すると不公平、逆に**成功したのに加算し忘れる**とすり抜けます。「**成功した分だけ確実に足す**」を意識する（失敗時の扱いを決めておく）。
- 🔧 **上限チェックを処理の後ろに置く** → 先にLLMを呼んでしまい、**お金が出てから**気づくことに。チェックは必ず**窓口の最初**で。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「TypeScript + Express の `/api/chat` に、**Supabaseの認証でログイン必須**を付けて。さらに **1日の上限回数（DAILY_LIMIT）をサーバー側でチェック**して、超えていたら **HTTP 429** を返して。使用量は `usage(user_id, day, count, tokens)` テーブルに、**成功時だけ加算**して。**フロントの無効化に頼らない**で」

出てきたコードの確認ポイント：

- [ ] **止めているのはサーバー側**か（フロントのボタン無効化だけに頼っていないか）
- [ ] `/api/chat` が**ログイン必須**か（トークン未検証で素通りしないか）
- [ ] 上限超で **429** を返しているか（しかも**LLMを呼ぶ前**に止めているか）
- [ ] 加算は **成功時だけ**か（失敗時に二重・過小カウントにならないか）
- [ ] ログに**会話本文や鍵などの秘密（PII）を残していない**か

---

## 📝 ことばメモ

- **従量課金**：使った分だけ料金がかかる仕組み。LLMはこれ
- **認証（Authentication）**：「あなたは誰か」を確かめること（＝ログイン）
- **認可（Authorization）**：「その人に何をどれだけ許すか」を決めること（＝上限・権限）。認証とは別物
- **クォータ（Quota）**：一定期間の**総量**の上限（例：1日100回）
- **レート制限（Rate Limit）**：**短時間の連打**への上限（例：1分20回）。詳しくは付録G
- **429（Too Many Requests）**：「使いすぎ／多すぎ」を表すHTTPの返事
- **PII**：個人を特定できる情報。ログに残さない

---

## ➡️ 次の章へ

財布を守れるようになりました。次の **第7章** では、**会話の「順番」**に注目します。ChatGPTで返事の途中に止めたり、エラーが出たりしたあと、**続きを送ると話がかみ合わなくなる**——あの現象の正体です。`user`（あなた）と `assistant`（AI）の発言が**交互**に並ぶべき、という記憶のルール（背骨①）を守る話に入ります。

[第7章　順番が崩れるとき（会話を途中で止めると順番が崩れる）→](07-message-order.md)

[← 目次・はじめにへもどる](README.md)
