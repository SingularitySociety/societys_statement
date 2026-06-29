---
title: "付録C　OpenAI と Anthropic の違い — messages/system の差と「切り替えできる設計」"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 17
nav_exclude: true
---

# 付録C　OpenAI と Anthropic の違い — messages/system の差と「切り替えできる設計」

> 📖 このページのゴール：2社のAPIの違いを知り、1か所差し替えで切り替えられる作りにする。
> [← 目次・はじめにへもどる](README.md)

---

本編は **OpenAI（GPT）** を主に使ってきました。でも、頭脳（LLM）を **Anthropic（Claude）** に替えたくなることがあります。たとえば「こっちのモデルの方が安い」「会社の都合でこっちを使う」など。

うれしいことに、**「ChatGPTクローン」はアプリの“見た目・体験”の話**で、**中の頭脳は差し替え可能**です。ただし、2社のAPIは「会話を送って返事をもらう」という**やることは同じ**なのに、**書き方が少しずつ違います**。このページで、その違いと「うまく切り替えられる作り方」を学びます。

---

## 🤔 同じ「会話→返事」でも、ここが違う

レストランでたとえると、**注文して料理が出てくる**のはどちらの店も同じ。でも、**注文用紙の書き方**や**料理の受け取り口**が、お店ごとに少しずつ違う——そんなイメージです。

ちがいは大きく5つあります。

### ① systemの置き場所（いちばん大事なちがい）

**system（システムめいれい）**＝「あなたは親切なアシスタントです」のような、AIへの**役割の指示**。これの置き場所が2社で違います。

- **OpenAI**：`messages` の中に **`role: "system"` の発言として**入れる（user / assistant と同じ配列の中に混ぜる）。
- **Anthropic**：`messages` には入れず、**`system` という別パラメータ**として渡す。`messages` には **user / assistant だけ**を入れる。

> ⚠️ ここが最大のつまずきポイント。OpenAIの感覚のまま、Anthropicの `messages` に `role: "system"` を入れてしまう間違いが本当に多いです（後半の「ハマりどころ」で再掲）。

### ② SDK（呼ぶための道具）がちがう

**SDK（えすでぃーけー）**＝そのサービスを呼ぶための**公式の道具箱**（部品の集まり）。

- **OpenAI**：`openai`
- **Anthropic**：`@anthropic-ai/sdk`

### ③ 呼び出すメソッド（注文する関数）がちがう

- **OpenAI**：`openai.chat.completions.create(...)`
- **Anthropic**：`anthropic.messages.create(...)`

### ④ 返事の取り出し方（料理の受け取り口）がちがう

返ってきた中から、**返事の文章だけ**を取り出す書き方が違います。

- **OpenAI**：`choices[0].message.content`
- **Anthropic**：`content[0].text`

### ⑤ プロンプトキャッシュ（同じ前文を安く速く）のやり方がちがう

**プロンプトキャッシュ**＝毎回おなじ長い前置き（システムめいれいなど）を送るとき、**使い回して安く速くする**しくみ（第13章）。

- **OpenAI**：基本的に **自動**（こちらが何もしなくても効くことがある）。
- **Anthropic**：**`cache_control` を付けて明示的に**「ここを使い回して」と指定する。

> 💡 **鍵（APIキー）の置き場所は、どちらも同じ**です。`.env` に入れて `process.env` から読み、**サーバーだけが知っている**——この鉄則（背骨②・第2章）は2社共通で変わりません。

---

## 🔁 対応表

ひと目で見比べられるようにまとめます。**左がOpenAI、右がAnthropic**です。

| 項目 | OpenAI（GPT） | Anthropic（Claude） |
|---|---|---|
| **system の置き場所** | `messages` の中（`role: "system"`） | **`system` パラメータ（別枠）** |
| **messages の中身** | system / user / assistant | **user / assistant のみ** |
| **SDK（道具箱）** | `openai` | `@anthropic-ai/sdk` |
| **呼び出すメソッド** | `openai.chat.completions.create` | `anthropic.messages.create` |
| **返事の取り出し** | `choices[0].message.content` | `content[0].text` |
| **プロンプトキャッシュ** | 自動（こちらは指定不要なことが多い） | `cache_control` で明示 |
| **APIキー** | `.env` ＋ `process.env`（サーバーのみ） | **同じ**（`.env` ＋ `process.env`） |

> 💡 表を見ると、**「やること（会話を送る・返事をもらう）」は同じ**で、**「名前と置き場所」だけが違う**のが分かります。だからこそ、次の「切り替えできる設計」がうまくいきます。

---

## 🛠 切り替えできる設計（callLLM にまとめる）

違いを毎回コードのあちこちに書くと、切り替えが大変です。そこで、**「LLMを呼ぶ窓口」をたった1つの関数 `callLLM` にまとめます**。

考え方はこうです。

> **本編のコードは `callLLM(messages)` だけを呼ぶ。中で OpenAI を使うか Anthropic を使うかは、この関数の中だけで切り替える。**

こうすれば、**本編のコードは1文字も触らず**、`callLLM` の中身を差し替えるだけで頭脳を交換できます。これを **抽象化（ちゅうしょうか＝共通の窓口にまとめること）** と呼びます。

```ts
// llm.ts  ── LLMを呼ぶ「共通の窓口」。本編はこの callLLM だけを使う
import OpenAI from "openai";
import Anthropic from "@anthropic-ai/sdk";

type Message = { role: "user" | "assistant"; content: string };

const SYSTEM_PROMPT = "あなたは親切なアシスタントです。";

// ── OpenAI 版 ──────────────────────────────
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function callOpenAI(messages: Message[]): Promise<string> {
  const completion = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "system", content: SYSTEM_PROMPT }, ...messages],
  });
  return completion.choices[0].message.content ?? "";
}

// ── Anthropic 版 ───────────────────────────
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

async function callAnthropic(messages: Message[]): Promise<string> {
  const response = await anthropic.messages.create({
    model: "claude-3-5-haiku-latest",
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    messages,
  });
  const first = response.content[0];
  return first.type === "text" ? first.text : "";
}

// ── 共通の窓口（ここで切り替える）─────────────
export async function callLLM(messages: Message[]): Promise<string> {
  return callOpenAI(messages); // ← ここを callAnthropic に変えるだけ
}
```

**1行ずつ読むと：**
- `import OpenAI from "openai"` / `import Anthropic from "@anthropic-ai/sdk"`：2社の **道具箱**をそれぞれ読み込む（SDKが違う＝ちがい②）。
- `type Message = ...`：会話1件の形。**user か assistant** と、その文章。**system はここに入れない**（後で店ごとに置き場所が違うから）。
- `const SYSTEM_PROMPT = ...`：AIへの役割めいれい。**文章は1つだけ持っておき**、店ごとに置き場所を変える。
- `new OpenAI({ apiKey: process.env.OPENAI_API_KEY })`：OpenAIの鍵は **`.env` から**読む（背骨②）。
- `callOpenAI(...)`：**OpenAI版の窓口**。`messages` の **先頭に `role: "system"` を足してから**渡す（ちがい①）。`...messages` で会話本体をつなげる。
- `model: "gpt-4o-mini"`：OpenAIのモデル名。
- `completion.choices[0].message.content ?? ""`：**OpenAIの受け取り口**から返事を取り出す（ちがい④）。`?? ""` は「もし空なら空文字」の保険。
- `new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })`：Anthropicの鍵も **同じく `.env` から**読む（置き場所は2社共通）。
- `callAnthropic(...)`：**Anthropic版の窓口**。
- `max_tokens: 1024`：Anthropicは **返事の最大の長さ（トークン）を必ず指定**する（OpenAIと違って省略できない）。
- `system: SYSTEM_PROMPT`：systemは **別パラメータ**として渡す（ちがい①）。`messages` には混ぜない。
- `messages`：**user / assistant だけ**をそのまま渡す。
- `response.content[0]` / `first.text`：**Anthropicの受け取り口**から返事を取り出す（ちがい④）。`first.type === "text"` で文章ブロックか確かめてから読む。
- `export async function callLLM(...)`：**本編が唯一さわる窓口**。中で `callOpenAI` を呼んでいる。
- `return callOpenAI(messages)`：**この1行を `return callAnthropic(messages)` に変えるだけ**で、頭脳が丸ごと入れ替わる。本編のコードは無傷。

> ✅ **これが「1か所差し替え」の正体**です。本編（第3章〜）のコードは「`callLLM` に会話を渡して返事をもらう」とだけ知っていればよく、**OpenAIかAnthropicかを気にしません**。お店を替えても、注文係（`callLLM`）が違いを吸収してくれます。

> 🔧 **応用：環境変数で切り替える**
> コードを書き換えずに切り替えたいなら、`.env` に `LLM_PROVIDER=openai`（または `anthropic`）と書いておき、`callLLM` の中で `process.env.LLM_PROVIDER === "anthropic" ? callAnthropic(messages) : callOpenAI(messages)` のように分岐します。**再デプロイ不要・設定だけで切り替え**られます。

---

## ⚠️ ハマりどころ

- **system を messages に入れたまま Anthropic へ送る**：いちばん多い間違い。OpenAIの感覚で `messages` に `role: "system"` を残すと、Anthropicでは**エラー**か**無視**になります。Anthropicは **`system` パラメータ**へ。`messages` は user / assistant だけ。
- **返事の取り出しを取り違える**：OpenAIは `choices[0].message.content`、Anthropicは `content[0].text`。**コピペ流用**で `choices` のままAnthropicに使うと `undefined`（中身なし）になります。
- **モデル名・料金・上限が各社で違う**：`gpt-4o-mini` はOpenAIだけの名前。Anthropicには通じません（例：`claude-3-5-haiku-latest`）。**料金体系も使えるトークン上限も別物**なので、片方の感覚で見積もらない（数え方は付録D）。
- **`max_tokens` の付け忘れ**：Anthropicは返事の最大長を**必ず指定**します。OpenAIの感覚で省くとエラーになります。
- **どちらも鍵はサーバーのみ**：切り替えに気を取られて、鍵をフロントに出さないこと（第2章）。`ANTHROPIC_API_KEY` も `.env` ＋ `process.env`。

> 💡 これらは全部、**`callLLM` の中だけで起きる**話です。窓口を1つにまとめておけば、間違いも **1か所を直すだけ**で済みます。

---

## 📝 ことばメモ

- **プロバイダ**：LLMを提供している会社・サービス。ここでは OpenAI と Anthropic
- **SDK**：そのサービスを呼ぶための公式の道具箱（部品の集まり）。`openai` / `@anthropic-ai/sdk`
- **system（システムめいれい）**：AIへの役割・口調の指示。OpenAIは messages の中、Anthropicは別パラメータ
- **抽象化（ちゅうしょうか）**：違いを隠して**共通の窓口**にまとめること。ここでは `callLLM` が窓口

---

[← 目次・はじめにへもどる](README.md)
