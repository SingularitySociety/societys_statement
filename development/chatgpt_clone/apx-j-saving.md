---
title: "付録J 会話の保存 — いつ・何を保存する？（ツール対応のデータ形式）"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 24
nav_exclude: true
---

# 付録J 会話の保存 — いつ・何を保存する？（ツール対応のデータ形式）

> 📖 このページのゴール：**user / assistant をいつ保存するか**を判断でき、**ツール（function calling）を含む会話**も壊さず保存・復元できるようになる。
> [← 目次・はじめにへもどる](README.md)

---

## 🤔 なぜ「保存」が難しいのか（🟢 基礎）

第4章で見たとおり、LLMは毎回まっさら。だから **DB が会話の“正本（しょうほん＝オリジナル）”** で、送るたびに DB から `messages` を組み立て直します（第5章）。

つまり保存は「ただ文字を残す」のではなく、**“次にLLMへ渡せる形”で過不足なく残す**必要があります。難しいのはこの2つ：

1. **タイミング**：user と assistant を、いつ書く？　途中で失敗・中断したら？
2. **ツールのとき**：1回のやりとりが**複数のメッセージ**にふくらむ。その特殊な形をどう持つ？

---

## ⏱ いつ保存する？（基本の user / assistant）（🟢 基礎）

鉄則はこれだけです。

> **user は「送られた瞬間」に保存。assistant は「返事が完成した瞬間」に保存。**

順番にすると：

```text
① ユーザーが送信
   → user メッセージを保存（status: done）        ← まず残す
② LLM を呼ぶ
③ 返事が完成
   → assistant メッセージを保存（status: done）
   失敗・中断したら → assistant は保存しない（or status:'error' で復元から除外）
```

**なぜこの順番？**

- **user を先に保存**：LLMが落ちても、ユーザーの発言と会話そのものは消えません。
- **assistant は“完成品”だけ**：途中で切れた半端な返事を正本に混ぜない（第7章「順番が崩れるとき」の対策）。

```ts
// 擬似コード：保存のタイミング
await saveMessage(conversationId, { role: "user", content: text });        // ①
const reply = await callLLM(await loadMessages(conversationId));           // ②
await saveMessage(conversationId, { role: "assistant", content: reply });  // ③
```

**1行ずつ読むと：**
- `saveMessage(... "user" ...)`：送信された瞬間に user を1行 insert する。
- `loadMessages(...)`：DB から全履歴を読み、`messages` を組み立てる（第5章）。
- `saveMessage(... "assistant" ...)`：**返事が返ってきてから** assistant を保存。②が落ちたらここには来ない＝半端な返事は残らない。

### ストリーミング（第11章）のとき

1文字ずつ来ても、**1トークン＝1行で保存しない**（行が爆発します）。やり方は2つ：

- かんたん：**完成したときに1行だけ**保存（上と同じ）。
- ていねい：開始時に `status:'pending'` の空の assistant 行を作り、**完了時に最終テキストへ UPDATE**（中断なら `'aborted'`、付録F）。再開時に pending は復元から外す。

具体的には、**ストリーム中はサーバー側で文字を貯めておき、終わったらまとめて1回だけ保存**します。

```ts
// サーバー：画面へ1文字ずつ流しつつ全文を貯め、最後に1回だけ保存
let full = "";
const stream = await openai.chat.completions.create({ model, messages, stream: true });
for await (const chunk of stream) {
  const piece = chunk.choices[0]?.delta?.content ?? "";
  full += piece;     // ① 自分用に全文を組み立てる
  res.write(piece);  // ② 同時に画面へ流す（第11章のSSE）
}
res.end();
await saveMessage(conversationId, { role: "assistant", content: full }); // ③ 完成してから保存
```

**1行ずつ読むと：**
- `let full = ""`：流れてくる断片（chunk）をつなげる入れ物。
- `chunk.choices[0]?.delta?.content`：ストリームで届く **今回ぶんの文字のかけら**。
- `full += piece`：画面へ流す**のと同時に**、サーバーでも全文を組み立てる。
- `res.write(piece)`：ユーザーには1文字ずつ見せる（これは“表示”の流れで、保存とは別）。
- `await saveMessage(... content: full)`：**ストリームが終わってから**、できあがった全文を1行だけ保存。

途中で切れたら（ユーザーが停止／回線断、付録F）、`full` は半端なので **保存しない**か、`status:'aborted'` で残して復元から外します。`pending` 行を先に作る方式なら、③を `update`（pending → done / aborted）に変えるだけです。

> 💡 **SDKのヘルパーを使うともっと楽**：OpenAI `client.chat.completions.stream(...).finalChatCompletion()`／Anthropic `client.messages.stream(...).finalMessage()`／Vercel AI SDK `await result.text`（＋`usage`）など、**ストリームの後に「REST と同じ形の完成レスポンス」を1発で取れる**SDKが多いです。これなら手で `full += piece` せずに、**`tool_calls`・`usage` 込み**でそのまま保存できます（→ 付録E）。

### 二重保存を防ぐ（第7章）

「送信が二重に飛ぶ」「リトライ」で同じ行が2つできないよう、**クライアントが作ったメッセージID**で `upsert`（あれば更新・無ければ作成）すると安全＝**冪等（べきとう）**。

---

## 🧰 ツール（function calling）のときの形（🔧 応用）

ツールを使うと（第12章）、**1回のやりとりが4つのメッセージ**になります（OpenAIの形）：

```text
1. user        … 「東京の天気は？」
2. assistant   … content は空、代わりに tool_calls=[getWeather("東京")]   ← 「この関数を呼んで」
3. tool        … tool_call_id 付きで関数の結果（"晴れ, 28℃"）
4. assistant   … 「東京は晴れ、28℃です」                                  ← 最終回答
```

**ここが肝**：この **2〜4を全部保存**しないと、次の呼び出しで履歴が壊れます（「呼んでと言ったのに結果が無い」＝APIエラー）。

だから `messages` テーブルに**列を足します**（付録Hの拡張）。

```sql
-- 付録H の messages に、ツール用の列を追加
alter table messages add column tool_calls jsonb;   -- assistantが「呼んで」と指示した中身
alter table messages add column tool_call_id text;  -- tool行が「どの呼び出しの結果か」
alter table messages add column name text;          -- 呼ばれたツール名（任意・あると読みやすい）
```

**1行ずつ読むと：**
- `tool_calls jsonb`：**assistantメッセージ**用。`[{ id, name, arguments }]` をそのまま入れる（JSONなので jsonb 型）。
- `tool_call_id text`：**toolメッセージ**用。どの呼び出しへの結果かをひも付ける（2の id と一致させる）。
- `name text`：呼ばれた関数名。

**保存タイミング**は、できた順に1行ずつ：

```text
user 保存 →（LLM）→ assistant(tool_calls) 保存 → ツール実行 → tool結果 保存 →（LLM再）→ 最終assistant 保存
```

**復元（次に送るとき）**は、行を `created_at` 順に読み、保存した `role` / `tool_calls` / `tool_call_id` を**そのままLLMの形に戻して** `messages` に並べるだけです。

> 🔁 **OpenAIとAnthropicで形が違う**（付録C）：OpenAIは上のように `tool_calls` ＋ `role:"tool"`＋`tool_call_id`。Anthropicは assistant の中に `type:"tool_use"`、結果は **user メッセージ**の `type:"tool_result"`＋`tool_use_id` で表します。**どちらか一方のネイティブ形で保存し、付録Cの `callLLM` で変換**するのが現実的です。

---

## ⚠️ ハマりどころ

- **tool_calls だけ保存して、tool結果を保存し忘れる** → 履歴が「呼んだのに答えが無い」状態で終わり、次回**APIエラー**。tool_calls と tool結果は**必ずペア**で保存。
- **assistant を“表示前”に空で保存** → 生成失敗で空の返事が正本に残る。**完成後に保存**、または pending 管理（第7章・付録F）。
- **ストリーミングを1トークン1行で保存** → 行が爆発。完成時に1行、または pending→UPDATE。
- **復元時に role を取り違える**（tool を user にする等）→ 会話が壊れる。保存した role をそのまま使う。
- **content に PII や秘密を生のまま** → ログ・メモリと同じ注意（第10章・付録G）。

---

## 📝 ことばメモ

- **正本（しょうほん）**：オリジナル＝ここが正しい、という大もと。会話ではDBが正本
- **tool_calls**：assistantが「この関数を呼んで」と指示する中身（id・名前・引数）
- **tool_call_id**：tool結果が「どの呼び出しへの返事か」を結ぶ札
- **pending（ペンディング）**：まだ完成していない・保留中。中断した返事の印
- **冪等（べきとう）**：同じ操作を何回しても結果が1回分になること（二重保存しない）

---

## ➡️ 関連

第5章 会話ログと複数セッション（保存の土台）／第7章 順番が崩れるとき（未完了・冪等性）／第12章 ツール（呼び出しの流れ）／付録H スキーマ（テーブル定義）／付録F 割り込み（中断の後始末）／付録C 2社の違い（保存形の変換）。

[← 目次・はじめにへもどる](README.md)
