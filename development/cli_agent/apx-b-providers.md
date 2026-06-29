---
title: "付録B　別のLLMで動かす（OpenAI・ローカルへの書き換え）＝ポータビリティ"
parent: "自作CLIエージェントで学ぶ AIエージェント開発入門"
grand_parent: "開発の心得"
nav_order: 17
nav_exclude: true
---

# 付録B　別のLLMで動かす（OpenAI・ローカルへの書き換え）＝ポータビリティ

> 📖 このページのゴール：エージェントの“頭脳（LLM）”を Anthropic から OpenAI やローカルLLMに**1か所差し替える**だけで動かせる、と体感する。**仕組み（道具・ループ・確認）は頭脳を替えても変わらない**。
> [← 目次・はじめにへもどる](README.md)

---

本編はずっと **Anthropic（Claude）** を頭脳に使ってきました。でも「こっちのモデルが安い」「会社の都合で OpenAI（GPT）を使う」「ネットに出さず**自分のPCの中（ローカル）**で動かしたい」——そんな日が来ます。

うれしいことに、ここまで作ってきたエージェントの**“仕組み”**——🔁 **考える→道具を使う→結果を返して、また考える（ループ）** と、🛡 **危ない操作の前に確認する**——は、**頭脳が誰でも変わりません**。料理にたとえると、**電話の向こうのシェフ（LLM）を別の人に替えても、厨房の段取り（あなたのコード）はそのまま**でいい、ということです。

変わるのは **「シェフへの電話のかけ方」だけ**。このページでは、その電話のかけ方を **1つの小さな窓口 `callLLM()` にまとめておけば、差し替えが1か所で済む** ことを実演します。

---

## 🟢 まず設計：LLM呼び出しを「窓口」1つにまとめる

本編のループ（4章で作った `runAgent`）は、中で何度も `anthropic.messages.create(...)` を直接呼んでいました。ここを**そのまま**にすると、頭脳を替えるとき**あちこち書き換え**になって大変です。

そこで、**「LLMに電話する場所」をたった1つの関数 `callLLM()` に閉じ込めます**。考え方はこうです。

> **本編のループは `callLLM(messages)` だけを呼ぶ。OpenAIを使うか・Claudeを使うか・ローカルを使うかは、この関数の中だけで切り替える。**

こうすると、ループ本体は **1文字も触らず**、`callLLM()` の中身を差し替えるだけで頭脳が丸ごと入れ替わります。これを **抽象化（ちゅうしょうか＝違いを隠して共通の窓口にまとめること）** と呼びます。

まずは本編（Anthropic）を、この窓口の形に整理してみます。

```ts
// llm.ts ── LLMに電話する「共通の窓口」。本編のループは callLLM だけを呼ぶ
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic(); // ANTHROPIC_API_KEY を自動で読む
const MODEL = "claude-opus-4-7";

async function callLLM(
  messages: Anthropic.MessageParam[],
  tools: Anthropic.Tool[],
  system: string,
) {
  return await anthropic.messages.create({
    model: MODEL,
    max_tokens: 1024,
    system,
    messages,
    tools,
  });
}
```

**1行ずつ読むと：**
- `import Anthropic from "@anthropic-ai/sdk"`：Anthropic を呼ぶための **道具箱（SDK）** を読み込む。
- `const anthropic = new Anthropic()`：電話機を用意。鍵（`ANTHROPIC_API_KEY`）は `.env` から自動で読む（付録A・第2章と同じ）。
- `const MODEL = "claude-opus-4-7"`：頭脳の名前（モデルID）。**ここを1か所変えるだけ**で別のClaudeにも替えられる（コストは付録F）。
- `async function callLLM(messages, tools, system)`：**唯一の窓口**。会話・道具メニュー・性格（システムプロンプト）を受け取る。
- `anthropic.messages.create({...})`：実際の電話。返事（`res`）をそのまま返す。本編のループは、この返事の `stop_reason` や `content` を**今まで通り**見ればよい。

> 💡 ポイントは、**本編のループが「`callLLM` に投げて返事をもらう」としか知らない**こと。中で誰に電話しているかは気にしません。だから次に窓口の中身を OpenAI に差し替えても、ループは無傷です。

---

## 🤔 同じ「会話→道具→返事」でも、OpenAIはここが違う

頭脳を OpenAI に替えるとき、**やること（会話を送り、道具を使わせ、結果を返す）は同じ**なのに、**書き方（電話のかけ方）が少しずつ違います**。本編のエージェントで効いてくる違いは、大きく **3つ** です。レストランでたとえると、**注文用紙の様式**・**呼び出しベルの押し方**・**料理の受け取り口**が、お店ごとに違う、という感じです。

### ① 道具メニューの“様式”が違う 🟢

道具（tools）をAIに渡すときの**形**が違います。

- **Anthropic**：道具そのものを**べたっと**並べる。
  ```ts
  { name, description, input_schema: { type: "object", properties, required } }
  ```
- **OpenAI**：道具を **`{ type: "function", function: { ... } }` で1枚包む**。中の鍵の名前も少し違う（`input_schema` ではなく **`parameters`**）。
  ```ts
  { type: "function", function: { name, description, parameters: { type: "object", properties, required } } }
  ```

中身（`name` / `description` / JSON Schema 本体）は**ほぼ同じ**で、**包み紙と一部の鍵名が違うだけ**です。

### ② 「道具を呼びたい」の伝わり方が違う 🟢（ここが最大のつまずき）

AIが「この道具を使って」と言ってくるときの**入れ物**と、**引数の渡され方**が違います。

- **Anthropic**：`res.content` の中の `block.type === "tool_use"` ブロックに `{ id, name, input }`。**`input` は最初からオブジェクト**（例：`{ path: "memo.txt" }`）。**そのまま使える**。
- **OpenAI**：`message.tool_calls` という配列に入ってくる。各要素は `{ id, function: { name, arguments } }`。そして——**`arguments` は“文字列”**（例：`'{"path":"memo.txt"}'`）。**つまり `JSON.parse()` で自分でオブジェクトに戻す必要**があります。

> ⚠️ **これがいちばん多いハマりどころ**です。Anthropic の感覚（`input` はもう物）のまま OpenAI の `arguments` を `executeTool` に渡すと、**ただの文字列**が来て道具が壊れます。OpenAI では **必ず `JSON.parse(arguments)` してから**道具に渡します。

### ③ 道具の結果の“返し方”が違う 🟢

道具を実行した結果を、次の往復でAIに返すときの**役割名**が違います。

- **Anthropic**：`role: "user"` のメッセージに `{ type: "tool_result", tool_use_id, content }` を入れて返す（本編4章のやり方）。
- **OpenAI**：**`role: "tool"`** という専用の役割で、`{ role: "tool", tool_call_id, content }` として返す。

---

## 🔁 Anthropic ↔ OpenAI 対応表

ひと目で見比べられるようにまとめます。**左がAnthropic（本編）、右がOpenAI**です。**やることは全部同じで、名前と形だけが違う**のが分かります。

| 項目 | Anthropic（Claude／本編） | OpenAI（GPT） |
|---|---|---|
| **SDK（道具箱）** | `@anthropic-ai/sdk` | `openai` |
| **電話機を作る** | `new Anthropic()` | `new OpenAI()` |
| **呼び出すメソッド** | `anthropic.messages.create` | `openai.chat.completions.create` |
| **systemの置き場所** | `system` パラメータ（別枠） | `messages` の先頭に `role: "system"` で入れる |
| **`max_tokens`** | **必須** | 任意（省略可） |
| **道具メニューの形** | `{ name, description, input_schema }` | `{ type: "function", function: { name, description, **parameters** } }` |
| **「道具を使って」の入れ物** | `res.content` の `type:"tool_use"` ブロック | `message.tool_calls` 配列 |
| **道具の引数** | `block.input`（**もうオブジェクト**） | `call.function.arguments`（**JSON文字列→`JSON.parse`が要る**） |
| **道具の結果の返し方** | `role:"user"` に `tool_result` ブロック | **`role:"tool"`** に `tool_call_id`＋`content` |
| **「道具を呼びたい」の合図** | `res.stop_reason === "tool_use"` | `message.tool_calls` が存在するか |
| **返事の本文の取り出し** | `res.content` の `text` ブロック | `choices[0].message.content` |
| **APIキー** | `.env`（`ANTHROPIC_API_KEY`） | `.env`（`OPENAI_API_KEY`） |

> 💡 表の右と左を見比べると、**「道具を渡して、呼びたいと言われたら実行して、結果を返す」という流れ（＝エージェントの仕組み）はまったく同じ**で、**包み紙と鍵の名前が違うだけ**だと分かります。だから次の「OpenAI版の窓口」も、形は本編とそっくりになります。

---

## 🛠 OpenAIへの差し替え：窓口の中身を書き換える

では `callLLM()` を OpenAI 版に書き換えます。**本編のループ（`runAgent`）は触りません**。違い①②③を、この窓口の中で吸収します。

> 🔧 本編のループは Anthropic の返事の形（`res.content` のブロック）を前提に書かれています。頭脳を OpenAI に替えるなら、**返事の形を本編が期待する形に“通訳”する**か、**ループ側も OpenAI の形で読む**かのどちらかが要ります。ここでは分かりやすさ優先で、**OpenAI版のループ部分も合わせて**示します（仕組みは4章と同じ、名前だけ違う）。

### まず道具メニューを OpenAI の様式に包む

```ts
// OpenAIの道具メニュー：本編の tools を {type:"function", function:{...}} で包む
import OpenAI from "openai";
const openai = new OpenAI(); // OPENAI_API_KEY を自動で読む

const openaiTools = [{
  type: "function" as const,
  function: {
    name: "read_file",
    description: "ファイルの中身を読んで返す。",
    parameters: {
      type: "object",
      properties: { path: { type: "string", description: "読むファイルのパス" } },
      required: ["path"],
    },
  },
}];
```

**1行ずつ読むと：**
- `import OpenAI from "openai"`：OpenAI の **道具箱（SDK）** を読み込む（Anthropic とは別物＝違いの一部）。
- `const openai = new OpenAI()`：OpenAI の電話機。鍵は `.env` の `OPENAI_API_KEY` から自動で読む（鍵を `.env` に隠す鉄則は両社共通）。
- `type: "function" as const`：OpenAI 様式の**包み紙**。「これは関数（道具）です」の宣言。
- `function: { name, description, parameters }`：中身は本編とほぼ同じ。ただし JSON Schema 本体の鍵は **`input_schema` ではなく `parameters`**（違い①）。
- `properties` / `required`：道具の引数の形。**ここは Anthropic とまったく同じ書き方**（JSON Schema の書き方は付録C）。

### 窓口とループを OpenAI 版にする

```ts
const OPENAI_MODEL = "gpt-4o";

// OpenAI版の窓口：会話を送り、返事（message）を返す
async function callLLM(messages: any[]) {
  const res = await openai.chat.completions.create({
    model: OPENAI_MODEL,
    messages,
    tools: openaiTools,
  });
  return res.choices[0].message; // content と tool_calls が入っている
}

// OpenAI版のループ：仕組みは本編4章と同じ。名前と引数の扱いだけ違う
async function runAgent(userInput: string) {
  messages.push({ role: "system", content: SYSTEM_PROMPT }); // systemは messages の中へ（違い）
  messages.push({ role: "user", content: userInput });

  for (let step = 0; step < 10; step++) { // ループ上限＝暴走防止は同じ
    const message = await callLLM(messages);
    messages.push(message); // AIの発言を履歴に積む

    if (!message.tool_calls) {       // 道具を使わないなら最終回答
      console.log(message.content);
      return;
    }

    for (const call of message.tool_calls) {
      const args = JSON.parse(call.function.arguments); // ★文字列→オブジェクト（違い②）
      const result = await executeTool(call.function.name, args); // 道具の実装は本編と同じ
      messages.push({
        role: "tool",                 // 結果は role:"tool" で返す（違い③）
        tool_call_id: call.id,
        content: result,
      });
    }
  }
}
```

**1行ずつ読むと：**
- `const OPENAI_MODEL = "gpt-4o"`：OpenAI の頭脳の名前。`claude-opus-4-7` は OpenAI には通じない（モデル名は各社別物）。
- `openai.chat.completions.create({...})`：OpenAI への電話。メソッド名が `messages.create` ではなく **`chat.completions.create`**（違い）。
- `res.choices[0].message`：返事の本体。中に `content`（文章）と `tool_calls`（道具のお願い）が入る。**ここから取り出すのが OpenAI 流**。
- `messages.push({ role: "system", content: SYSTEM_PROMPT })`：OpenAI は **system を `messages` の中（先頭）に入れる**（Anthropic は別パラメータ）。
- `for (let step = 0; step < 10; step++)`：**ループ上限10回**。無限ループ防止は頭脳が替わっても必須（背骨🛡）。
- `if (!message.tool_calls)`：**道具のお願いが無ければ最終回答**。Anthropic の `stop_reason === "tool_use"` 判定に当たる（違い）。
- `JSON.parse(call.function.arguments)`：**ここが OpenAI 最大の山場**。`arguments` は**文字列**なので、`JSON.parse` でオブジェクトに戻してから道具へ渡す（違い②）。Anthropic の `block.input` は最初から物なので、この行が要らなかった。
- `executeTool(call.function.name, args)`：**道具の実装（`read_file` 等）は本編とまったく同じ**。頭脳を替えても、手（道具）は使い回せる。
- `messages.push({ role: "tool", tool_call_id: call.id, content: result })`：結果を **`role:"tool"`** で返す（違い③）。`tool_call_id` で「どのお願いへの返事か」を結びつける。

> ✅ こうして見ると、**変わったのは `callLLM` の中身と、引数を `JSON.parse` する1行、役割名（`tool`）だけ**。**「道具を渡す→呼びたいと言われたら実行→結果を返す→また考える」というループの骨格は、本編4章と1ミリも変わっていません**。これが「**頭脳を差し替えても仕組みは同じ＝ポータビリティ（持ち運びやすさ）**」の正体です。

> 🔧 **環境変数で切り替える**：`.env` に `LLM_PROVIDER=openai`（または `anthropic`）と書いておき、`callLLM` の入口で `process.env.LLM_PROVIDER` を見て分岐すれば、**コードを書き換えずに設定だけで頭脳を交換**できます。

---

## 🛠 ローカルLLMで動かす（Ollama / LM Studio）— `baseURL` を差し替えるだけ

「鍵も要らない・無料で・ネットに出さず**自分のPCの中だけ**で動かしたい」——そんなときは **ローカルLLM** を使います。**Ollama** や **LM Studio** といった道具は、自分のPCでモデルを動かしつつ、**OpenAIと“同じ電話のかけ方”ができる窓口（OpenAI互換API）** を用意してくれます。

うれしいのは、**OpenAI互換**なので、さっきの OpenAI 版コードが**ほぼそのまま使える**こと。違いは基本、**電話のかけ先（`baseURL`）を自分のPC内に向ける**ことだけです。

```ts
import OpenAI from "openai";

// 電話のかけ先を「自分のPCの中」に向けるだけ。中身は OpenAI 版コードのまま
const openai = new OpenAI({
  baseURL: "http://localhost:11434/v1", // Ollama の既定。LM Studio は :1234/v1 など
  apiKey: "ollama",                     // ローカルは鍵不要。形だけ何か入れておく
});

const LOCAL_MODEL = "llama3.1"; // 自分のPCに入れたモデル名（例）
```

**1行ずつ読むと：**
- `import OpenAI from "openai"`：**OpenAI の道具箱をそのまま使う**。ローカルLLMが「OpenAIのふり」をしてくれるので、専用SDKは要らない。
- `baseURL: "http://localhost:11434/v1"`：**電話のかけ先**。`localhost`（自分のPC）の中で動いている Ollama に向ける。**ここを差し替えるだけ**で、クラウドの OpenAI ではなく手元のモデルに繋がる（LM Studio など他のツールは番号が違う＝そのツールの説明で確認）。
- `apiKey: "ollama"`：ローカルは**鍵が要らない**。でも SDK が空だと怒るので、**形だけ**何か入れておく。
- `const LOCAL_MODEL = "llama3.1"`：自分のPCに入れたモデルの名前。あとは OpenAI 版の `callLLM` の `model` をこれに替えるだけ。

> 💡 つまりローカルLLMは、**「OpenAI版コードの `baseURL` を自分のPCに向け、鍵をダミーにする」だけ**。ループも道具も `JSON.parse` も、**OpenAIのときと同じ**です。

> ⚠️ **ローカルLLMの落とし穴（正直に）**：鍵不要・無料・データが外に出ない、と良いことずくめに見えますが、**非力なモデルは“道具呼び出し”が下手**なことがあります。具体的には——
> - 道具を**呼ぶべき場面で呼ばない**（自分で適当に答えてしまう）／**呼ばなくていい場面で呼ぶ**
> - `arguments`（引数）の **JSONが壊れて返る**（`JSON.parse` が失敗する）→ try/catch で受け止め、エラーを文字列で道具結果として返す設計（本編の流儀）が効く
> - そもそも **tool_calls の様式に対応していない**小さなモデルもある
>
> なので「練習・オフライン・機密データ用」には最高ですが、**複雑なエージェント挙動は、まだクラウドの大きいモデル（Claude / GPT）の方が安定**です。**まず Claude で仕組みを完成させ、ローカルは“替えてみる”** くらいの順がおすすめです。

---

## 🤖 AIに頼むなら（書き換えそのものをAIに任せる）

ここがいちばん言いたいことです。**この付録の書き換え作業すら、手元のAIに頼める**。エージェントの本質は**特定のSDKでなく“考え方（ループと安全）”**なので、コードは**別の頭脳・別の言語へ翻訳が効きます**。

> 🗣 **プロンプト例**
> 「この **Anthropic（`@anthropic-ai/sdk`）版のCLIエージェント**を、**OpenAI（`openai`）版**に書き換えて。守ってほしい点：
> ① LLM呼び出しは `callLLM()` という1つの関数にまとめる（差し替えが1か所で済むように）
> ② 道具メニューは `{type:'function', function:{name, description, parameters}}` の形にする
> ③ 道具のお願いは `message.tool_calls` から読み、**`arguments` は `JSON.parse` してから**道具に渡す
> ④ 道具の結果は `role:'tool'` で返す
> ⑤ ループ上限（暴走防止）と、危険な道具の前の確認はそのまま残す」

> 🗣 **言語ごと替えたいとき**
> 「この TypeScript 版エージェントを **Python** で書き直して。OpenAI の Python SDK（`openai`）を使い、上の①〜⑤の設計はそのまま守って」

AIが書き換えてくれたら、**鵜呑みにせず**、下のチェックリストで**自分の目で確かめます**（このチェックこそ、本編を通して身につけた“安全の目”です）。

**出力チェックリスト**
- [ ] **窓口は1つ**になっているか（`callLLM()` 等にLLM呼び出しが集約され、ループ本体に `create` が散らばっていない）
- [ ] OpenAI版で **`arguments` を `JSON.parse`** しているか（**文字列のまま道具に渡していない**＝違い②の最頻バグ）
- [ ] 道具の結果を **`role:"tool"`（＋`tool_call_id`）** で返しているか（違い③）
- [ ] 道具メニューが **`{type:"function", function:{…, parameters}}`** の形か（`input_schema` のままになっていない＝違い①）
- [ ] **ループ上限**（例：10回）が残っているか（頭脳を替えても**暴走防止**は必須）
- [ ] **危険な道具（書き込み・コマンド）の前の確認**（第8章）が**消えていない**か
- [ ] 引数を**鵜呑みにせず検証**しているか（型・必須・作業フォルダ外へ出さない＝第9章）。ローカルLLMは特に**壊れたJSON**を返しがちなので **try/catch** で受け止めているか
- [ ] **鍵を `.env` に隠し**、コードに直書きしていないか（ローカルはダミー鍵でOK／クラウドは本物を `.env` に）

---

## 📝 ことばメモ

- **プロバイダ**：LLM（頭脳）を提供する会社・サービス。ここでは Anthropic / OpenAI と、手元で動かすローカルLLM
- **ポータビリティ（持ち運びやすさ）**：中身（頭脳）を替えても、外側の仕組み（ループ・道具・確認）がそのまま使えること。窓口 `callLLM()` がそれを支える
- **抽象化（ちゅうしょうか）**：違いを隠して**共通の窓口**にまとめること。ここでは LLM 呼び出しを `callLLM()` に集約すること
- **OpenAI互換API**：OpenAI と**同じ電話のかけ方**ができる窓口。Ollama / LM Studio などのローカルLLMが用意してくれる。だから OpenAI 版コードが `baseURL` 差し替えでほぼ動く
- **`baseURL`（ベースユーアールエル）**：電話のかけ先。クラウド（OpenAI）か、自分のPCの中（`localhost`）かを切り替える設定
- **tool_calls / arguments**：OpenAI が「この道具を使って」と返す入れ物（`tool_calls`）と、その引数（`arguments`）。**`arguments` は文字列なので `JSON.parse` が要る**（Anthropic の `input` はオブジェクト＝そのまま使える）

---

[← 目次・はじめにへもどる](README.md)
