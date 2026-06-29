# 付録G　使用量制限の実装パターン（カウンタ・クォータ・レート制限）

> 📖 このページのゴール：使いすぎを止める仕組みを実際に作れる。
> [← 目次・はじめにへもどる](README.md)

---

第6章では「ログイン必須＋1日の上限」を作り、数える処理は **DB側の `increment_usage` にまとめる**とだけ言いました。このページは、その **中身のくわしい版** です。前提は第6章と同じで、サーバーは **Express**、保存は **Supabase（Postgres）**。数える単位は **認証ユーザーごと**（`user_id`）です。

---

## 🔢 DBカウンタ（クォータ）

第6章の `usage(user_id, day, count, tokens)` 表に、リクエストのたびに **1回足す**だけ——に見えて、ここに落とし穴があります。「**読んでから書く**」を素直に書くと、同じ人がほぼ同時に2回叩いたとき、**両方が同じ値を読んで両方が同じ値を書き**、+1のはずが**1回しか増えません**。これでは上限がすり抜けます。

正解は、**読まずに DB へ「足して」と命じる**こと。しかも「**行が無ければ作り、あれば足す**」を1回でやります（＝**UPSERT**）。Postgres の `on conflict ... do update` がちょうどこれです。

```sql
-- increment_usage: 無ければ作る／あれば足す を1回で（原子的）
create or replace function increment_usage(
  p_user_id uuid, p_day date, p_tokens int
) returns void as $$
  insert into usage (user_id, day, count, tokens)
  values (p_user_id, p_day, 1, p_tokens)
  on conflict (user_id, day) do update
    set count  = usage.count  + 1,
        tokens = usage.tokens + excluded.tokens;
$$ language sql;
```

**1行ずつ読むと：**
- `create or replace function increment_usage(...)`：DB の中に置く小さな関数を作る（サーバーから `rpc("increment_usage", ...)` で呼ぶ）。
- `p_user_id uuid, p_day date, p_tokens int`：受け取る材料は **誰の・いつの・何トークン**の3つ。
- `insert into usage (...) values (..., 1, p_tokens)`：まだその人のその日の行が無ければ、**count=1** で新しく作る。
- `on conflict (user_id, day) do update`：すでに行があった（`(user_id, day)` がぶつかった）ときは、作るのをやめて**更新へ切り替える**。
- `set count = usage.count + 1`：**いまDBにある値**に+1する（自分では読まない＝競合しない）。
- `tokens = usage.tokens + excluded.tokens`：トークンも、いまの値に**今回ぶん**を足す。`excluded` は「入れようとした値」を指す。
- `language sql`：中身はただのSQLですよ、という宣言。

サーバー側はこの関数を呼ぶだけ。**処理が成功した後**に1回呼びます。

```ts
await supabase.rpc("increment_usage", {
  p_user_id: req.userId,
  p_day: today,                                   // "2026-06-29"
  p_tokens: completion.usage?.total_tokens ?? 0,  // 実際に使った量
});
```

**1行ずつ読むと：**
- `supabase.rpc("increment_usage", { ... })`：さっきのDB関数を名前で呼び出す。`rpc` は「DBの関数を実行して」の意味。
- `p_user_id: req.userId`：第6章の見張り役（`requireUser`）が確定した**ログイン本人のID**を渡す。
- `p_day: today`：今日の日付（時刻は捨てて日付だけ）。これが**リセットの鍵**になる（次節）。
- `p_tokens: completion.usage?.total_tokens ?? 0`：OpenAIが返す**実際のトークン数**。取れなければ0。

---

## 📅 クォータの種類

クォータ（一定期間の総量の上限）は、**キーの作り方**を変えるだけで種類を増やせます。`usage` の `day` 列に「**期間を表す文字**」を入れておくのがコツです。

- **1日上限**：キー＝日付（`"2026-06-29"`）。`count >= 100` なら止める、など。
- **月上限**：キー＝月（`"2026-06"`）。月またぎで自動的に新しい行になる。
- **トークン上限**：`count`（回数）ではなく `tokens`（量）を見る。「**回数は少ないが長文ばかり**」の人を止められる。料金は回数より**トークン量**に効くので、本気の財布防御はこちら。

```ts
const DAILY_LIMIT_COUNT  = 100;     // 1日の回数上限
const DAILY_LIMIT_TOKENS = 200_000; // 1日のトークン上限

const overQuota =
  (usage?.count  ?? 0) >= DAILY_LIMIT_COUNT ||
  (usage?.tokens ?? 0) >= DAILY_LIMIT_TOKENS;
```

**1行ずつ読むと：**
- `const DAILY_LIMIT_COUNT = 100`：回数の上限を**名前付き定数**に（コード中に直接 `100` と書かない）。
- `const DAILY_LIMIT_TOKENS = 200_000`：トークンの上限。`_` は桁区切りで、ただの読みやすさ（値は20万）。
- `(usage?.count ?? 0) >= DAILY_LIMIT_COUNT`：今日の回数が上限以上か（記録がまだ無ければ0扱い）。
- `(usage?.tokens ?? 0) >= DAILY_LIMIT_TOKENS`：今日のトークン量が上限以上か。
- `||`：**どちらか一方**でも超えていれば `overQuota` は真→止める。

> 💡 「日が変わったらリセット」を**わざわざ消す処理は要りません**。日付が変われば `day` が変わり、**別の行（count=0から）**になるだけ。古い行は履歴として残るので、後で集計にも使えます。

---

## ⏱ レート制限（連打を防ぐ）

クォータは「1日の総量」。これとは別に、**短い時間の連打**を止めるのが**レート制限**です。たとえば「1分に20回まで」。一気に叩かれて**数分で大金**——という事故は、日次クォータだけでは間に合いません。

いちばん簡単なのは**固定窓（fixed window）**：時間を1分ごとの窓に区切り、**窓の中の回数**を数えて上限で止める。`usage` と同じ発想で、**キーを「分」にした表**を1つ足すだけです。

```ts
const RATE_LIMIT_PER_MINUTE = 20;

async function checkRateLimit(userId: string): Promise<boolean> {
  const minuteKey = new Date().toISOString().slice(0, 16); // "2026-06-29T14:03"
  const { data } = await supabase.rpc("increment_rate", {
    p_user_id: userId, p_minute: minuteKey,
  });
  return (data ?? 0) <= RATE_LIMIT_PER_MINUTE;
}
```

**1行ずつ読むと：**
- `const RATE_LIMIT_PER_MINUTE = 20`：1分あたりの上限回数を定数に。
- `new Date().toISOString().slice(0, 16)`：今の時刻を**分まで**で切る（`"...T14:03"`）。これが「今の窓」の名札。
- `supabase.rpc("increment_rate", { ... })`：さっきの `increment_usage` と同じ作りのDB関数で、**この窓の回数を+1して、足した後の値を返す**。
- `return (data ?? 0) <= RATE_LIMIT_PER_MINUTE`：足した後の回数が上限以内なら `true`（通す）、超えたら `false`（止める）。

> 🔧 **応用：トークンバケットの考え方**
> 固定窓は作りやすい反面、窓の境目（`14:02:59` と `14:03:00`）でドッと来ると**一瞬だけ2倍**通ってしまいます。これを滑らかにするのが**トークンバケット**。バケツに一定速度でコインを足し続け（例：3秒で1枚）、リクエストごとにコインを1枚払う。**コインが無ければ待ってもらう**。「普段は連打を抑えつつ、たまの一気押しは許す」が表現できます。最初は固定窓で十分。困ってから載せ替えればOKです。

---

## 🚦 超えたときの返し方

クォータでもレート制限でも、超えたら返すのは **HTTP 429（Too Many Requests）**。さらに、いつ回復するかを **`Retry-After` ヘッダ**（秒数）で伝えると、画面に「あと◯分」を出せます。

```ts
if (!(await checkRateLimit(req.userId))) {
  const secondsLeft = 60 - new Date().getSeconds(); // 今の分が終わるまで
  res.set("Retry-After", String(secondsLeft));
  return res.status(429).json({
    error: "短時間に送りすぎです",
    retryAfterSec: secondsLeft,
  });
}
```

**1行ずつ読むと：**
- `if (!(await checkRateLimit(req.userId)))`：レート制限に**ひっかかった**ら（`false` なら）——
- `const secondsLeft = 60 - new Date().getSeconds()`：今の窓（1分）が終わるまでの**残り秒**を出す。次の窓で回復するから。
- `res.set("Retry-After", String(secondsLeft))`：標準の `Retry-After` ヘッダに残り秒を入れる（道具や他のサーバーも読める共通語）。
- `res.status(429).json({ ... })`：本文にも `retryAfterSec` を入れて、**画面で「あと◯秒で回復」**と出せるようにする。

> 💡 画面側は `retryAfterSec` を 60 で割って「あと◯分で回復します」と出し、その間は送信ボタンを**一時的にだけ**無効化すると親切です。ただし——次節のとおり、**止めているのはあくまでサーバー**。ボタン無効化は見た目の親切にすぎません。

---

## ⚠️ ハマりどころ

- **フロントだけで止める＝無意味。** ボタンの `disabled` や「あと◯分」表示は**親切な見た目**であって、防御ではありません。ブラウザを使わず**サーバーを直接叩けば素通り**します。**止めるのは必ずサーバー側**（第2章・第6章と同じ鉄則）。
- **同時アクセスでカウントが競合する。** 「読んでから+1して書く」は、ほぼ同時の2回で**数え落とし**ます。必ず**DB側で足す**（`on conflict ... set count = usage.count + 1`）。アプリ側で読んだ値に足してはいけません。
- **失敗時にカウントを戻すか、決めておく。** LLM呼び出しが失敗したのに加算すると不公平、成功したのに加算し忘れるとすり抜けます。おすすめは「**成功した後にだけ足す**」。先に足して失敗したら戻す方式は、戻し忘れの事故が起きがちです。
- **ログにメッセージ本文（PII）を残さない。** 数を数えるのに**会話の中身は不要**です。`user_id`・回数・トークン数だけを記録し、**本文や鍵などの秘密**はログにもDBの使用量表にも残さないこと。

---

## 📝 ことばメモ

- **クォータ（Quota）**：一定期間の**総量**の上限（例：1日100回／月20万トークン）。キーを日・月に変えて作り分ける
- **レート制限（Rate Limit）**：**短時間の連打**への上限（例：1分20回）。固定窓やトークンバケットで作る
- **429（Too Many Requests）**：「使いすぎ／多すぎ」を表すHTTPの返事
- **Retry-After**：429と一緒に返す「**◯秒後にまた来て**」を伝える標準ヘッダ
- **原子的（アトミック）**：「読む→足す→書く」が**割り込まれず1回でまとまる**こと。`on conflict do update` で実現し、同時アクセスでも数え間違えない

---

[← 目次・はじめにへもどる](README.md)
