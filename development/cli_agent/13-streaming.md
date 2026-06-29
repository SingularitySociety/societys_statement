---
title: "第13章　考えている様子を見せる — ストリーミング＋道具呼び出しの表示"
parent: "自作CLIエージェントで学ぶ AIエージェント開発入門"
grand_parent: "開発の心得"
nav_order: 13
---

# 第13章　考えている様子を見せる — ストリーミング＋道具呼び出しの表示

> 📖 この章のゴール：これまでの `create`（全部できてから一気に表示）を `stream`（できたそばから流す）に替えて、返事を**1文字ずつ**ターミナルに出せるようになる。さらに、AIが**道具を使う瞬間**も画面に出して、「いま何をしているか」が見えるエージェントにする。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 Claude Code ではこう見える

Claude Code に「このテスト直して」と頼むと、画面は**だまって固まったりしません**。考えている言葉が**スルスルと流れて**出てきて、途中で

```
Running: npm test
```

のように、**いま道具（コマンド）を実行している様子**まで見せてくれます。テストが終わると、また考えの続きが流れ出す——という具合です。

これは飾りではありません。エージェントは1回の依頼で**何往復もループ**します（第4章・第12章）。各往復に数秒〜十数秒かかることもある。その間ずっと真っ黒な画面だったら、「**止まったのかな？　もう一回押す？**」と不安になります。流しながら見せるだけで、その不安が消えるのです。

前作（ChatGPTクローン）の第11章で、ブラウザに「1文字ずつ」出す **SSE** をやりました。あれは「**サーバー → ブラウザ**」に断片を流す話。今回は舞台が**ターミナル**に変わるだけで、考え方はそっくりです。しかも——**今回のほうがずっと簡単**。ブラウザもサーバーも無いので、`process.stdout.write()`（画面に書き出す）で直接ターミナルに流せます。

---

## 🤔 なぜ「流す」だけで体感が変わるのか

### だまって待たされるのが一番つらい（🟢 基礎）

これまでの章では、こう書いてきました（第2章で確立した形）。

```ts
const res = await anthropic.messages.create({ model: MODEL, max_tokens: 1024, system: SYSTEM_PROMPT, messages, tools });
printText(res); // 全部できてから、一気に表示
```

`create` は「**AIが返事を全部書き終わるまで待って、完成品を一気に受け取る**」やり方です。短い返事ならいいのですが、長い説明や、何往復もするタスクだと——**書き終わるまで画面はまっさら**。利用者からは「固まった」ようにしか見えません。

> 🍽 **たとえ話（前作と同じ皿の話）：料理の運び方**
> - `create`＝コース料理を**全部できてから、まとめて1回で運ぶ**。最後の皿が焼けるまで、テーブルは空っぽ。
> - `stream`＝**できた皿から、順番にどんどん運ぶ**。待っている人は「お、来た来た」と楽しめる。
>
> 料理（返事）の総量も、料金も同じ。**運び方**を変えるだけで、待ち時間の体感がまるで違います。

LLMは、実は返事を**少し書くたびに、その断片（かけら）を出せます**。`stream` は、その断片を**届いたそばから**受け取れる呼び方。受け取るたびに `process.stdout.write()` で画面に足していけば、ターミナルに「1文字ずつ」流れて見えます。

> 📝 この「今回増えたぶんの小さな断片」のことを **デルタ（delta＝差分）** と呼びます。前作のSSEで `delta.content` を拾ったのと同じ言葉です。

### 「道具を使う瞬間」も見せる（🟢 基礎）

ストリーミングのもう一つの大事な仕事は、**道具を使うときに「いま○○をやってます」と知らせる**ことです。

第4章のループを思い出してください。AIは「考える → 道具を使う → 結果を見て、また考える」をぐるぐる回します。道具を使う往復（`stop_reason === "tool_use"`）では、AIは**まだ最終回答を書いていません**。だから流す言葉も少ない。ここで何も出さないと、ちょうど道具の実行中（ファイルを読む・コマンドを走らせる）に画面が止まって見えます。

そこで、道具を使うブロックを見つけたら

```
[道具を使います: read_file]
```

のように画面に出します。Claude Code の `Running: npm test` と同じ役割です。「黙って止まった」ではなく「**いま read_file を使っているんだな**」と分かる。これだけで安心感がまるで違います。

> 💡 **背骨の確認**：流して見せるのは UX（使い心地）の工夫であって、**ループそのものは第4章から1ミリも変わりません**。判定（`stop_reason` / `tool_use`）も、道具を実行して結果を返す流れも、全部そのまま。**変えるのは「受け取り方」だけ**です。

---

## 🛠 こう作る — `create` を `stream` に差し替える（🟢 基礎）

やることは1か所だけ。第4章のループの中の `messages.create({...})` を、**`messages.stream({...})` ＋ 断片を流す ＋ `finalMessage()` で完成品を受け取る**に置き換えます。

### ステップ1：1往復ぶんを「流す」関数にする

まず、LLMを1回呼んで**流しながら**完成した返事を受け取る部分を、小さな関数に切り出します。

```ts
async function streamTurn(): Promise<Anthropic.Message> {
  const stream = anthropic.messages.stream({
    model: MODEL,
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    messages,
    tools,
  });

  stream.on("text", (delta) => process.stdout.write(delta)); // 届いた文字を即・画面へ

  return await stream.finalMessage(); // 流し終わったら、完成した Message を受け取る
}
```

**1行ずつ読むと：**
- `anthropic.messages.stream({...})`：これまでの `create` を `stream` に替えただけ。渡す中身（`model` / `max_tokens` / `system` / `messages` / `tools`）は**まったく同じ**。`tools` も忘れず渡す（道具メニューは流しても必要）。
- `stream.on("text", (delta) => ...)`：「**テキストの断片が届くたびに、この処理を呼んで**」という登録。`delta`（デルタ）が**今回増えた文字**。
- `process.stdout.write(delta)`：その断片を**ターミナルに即・書き出す**。`console.log` と違って**改行を付けない**ので、断片がつながって「1文字ずつ」に見えます。これが前作の `res.write(...)`（ブラウザへ流す）にあたる、ターミナル版。
- `return await stream.finalMessage()`：流し終わったあと、**完成した `Message`**（`stop_reason` や `content` が全部入った、`create` と同じ形のもの）を受け取って返す。**ループの判定はこの完成品を使う**ので、これがキモ。

> ⚠️ ここで `.on()` を `new Promise()` で自分で包みたくなりますが、**やめてください**。SDKが用意した **`finalMessage()`** が「流し終わるまで待って、完成品を返す」を全部やってくれます（中で待つ・組み立てる・エラーを上げる、を面倒みてくれる）。自前の Promise は二重管理になってバグの元です。

### ステップ2：ループに組み込む（道具の表示も足す）

第4章の `runAgent` を、`streamTurn()` を使う形に書き換えます。**ループの骨格は第4章と同じ**。違いは「**流す**」と「**道具を使う前に一言出す**」の2点だけ。

```ts
const MAX_STEPS = 10; // 暴走防止の上限（第4章から）

async function runAgent(userInput: string) {
  messages.push({ role: "user", content: userInput });

  for (let step = 0; step < MAX_STEPS; step++) {
    const res = await streamTurn();          // ★ 流しながら1往復
    process.stdout.write("\n");              // 流し終わりに1回だけ改行
    messages.push({ role: "assistant", content: res.content });

    if (res.stop_reason !== "tool_use") return; // 最終回答だった → 終了（もう流して表示済み）

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const block of res.content) {
      if (block.type !== "tool_use") continue;
      console.log(`[道具を使います: ${block.name}]`); // ★ いま何をするかを見せる
      const result = await executeTool(block.name, block.input);
      toolResults.push({ type: "tool_result", tool_use_id: block.id, content: result });
    }
    messages.push({ role: "user", content: toolResults });
  }
}
```

**1行ずつ読むと：**
- `const res = await streamTurn();`：ステップ1の関数を呼ぶ。**画面にはもう言葉が流れている**。返ってくる `res` は完成した `Message`。
- `process.stdout.write("\n");`：断片には改行を付けていなかったので、1往復ぶん流し終わったら**ここで1回だけ改行**して見た目を整える。
- `messages.push({ role: "assistant", content: res.content });`：完成した返事を会話履歴に積む。**第4章とまったく同じ**。
- `if (res.stop_reason !== "tool_use") return;`：道具を使わない＝**最終回答**。`create` の頃は `printText(res)` で表示してから終わったが、**今回はもう流して表示し終えている**ので、表示せずそのまま `return`。
- `console.log(\`[道具を使います: ${block.name}]\`);`：道具ブロックを見つけたら、**実行する前に道具名を出す**。`block.name` は `read_file` などの道具名。Claude Code の `Running: ...` にあたる一言。
- `const result = await executeTool(block.name, block.input);`：実際に道具を動かす。**`block.input` はSDKがパース済みのオブジェクト**（第3章のことばメモのとおり、`JSON.parse` 不要）。ここも第4章のまま。
- `messages.push({ role: "user", content: toolResults });`：道具の結果を履歴に積んで、次の往復へ。**ループが続く＝また `streamTurn()` が呼ばれて、続きが流れ出す**。

> ✅ **動かし方**：第2章のCLI入力ループ（`rl.question("> ")`）はそのまま。`runAgent` の中身だけ上の形に替えて `npx tsx agent.ts` で起動し、「このフォルダのファイルを一覧して」などと頼んでみてください。返事が**スルスルと流れ**、途中で `[道具を使います: list_files]` が出て、また続きが流れれば成功です。

> 💡 **他の言語でも**：本質は「`create` を `stream` に替えて、断片を `on("text")` で受けて、`finalMessage()` で完成品を待つ」だけ。手元のLLMに「**このTypeScriptをPythonで書き換えて。ストリーミングは公式SDKの stream を使って**」と頼めば、同じ仕組みが他の言語でも作れます。

---

## ⚠️ ハマりどころ

- **`.on()` を `new Promise()` で包んでしまう（🟢）**：上でも書いたとおり、**`finalMessage()` を使えば自前のPromiseは不要**。包むと「いつ完成したか」を二重に管理することになり、`await` の取りこぼしや握りつぶしたエラーの温床になります。受け取りは `on("text", ...)`、完成待ちは `finalMessage()`、と役割を分けるのが正解。
- **道具の `input` を“途中で”parseしようとする（🟢→🔧）**：道具の入力（`tool_use` の `input`）は、流れている**最中は中途半端なJSONの断片**で届きます。途中の断片を拾って `JSON.parse` すると**必ず壊れます**（まだ閉じカッコが来ていない、など）。**途中では絶対にparseしない**。`finalMessage()` で受け取った完成品の `block.input` を使えば、SDKが組み立て終わった**完全なオブジェクト**になっています。だから上のコードは「流すのはテキストだけ、道具の中身は完成後に読む」という形にしてあります。
- **まとめてドンと出る／カクつく（出力のバッファリング）（🔧）**：`process.stdout.write` で出しているのに1文字ずつに見えないときは、**出力がどこかでためこまれている**サインです。`console.log` を断片ごとに使うと毎回改行が入って崩れるので、断片の表示は必ず **`process.stdout.write`**（改行なし）に。ログをファイルやパイプに流している場合はバッファリングが効いてまとめて出ることがあります（手元のターミナルで直接動かすと素直に流れます）。
- **途中でエラー／接続が切れたとき（🟢）**：通信は途中で切れることがあります。`runAgent` の呼び出しを `try/catch` で囲み、失敗したら**だまって固まらせず**「失敗しました」と一言出す（前作のSSEで `[ERROR]` を出したのと同じ心がけ）。`finalMessage()` は途中で失敗すると例外を投げるので、`catch` で受け止められます。
- **最終回答を二重に表示する（🟢）**：`stream` に替えたあとも、つい昔の `printText(res)` を残してしまうと、**流して出した同じ文章をもう一度ドンと表示**してしまいます。流すようにしたら、最後の `printText` は**消す**（上のコードでは `return` だけにしてあります）。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「**TypeScript + 公式 `@anthropic-ai/sdk`** で作ったエージェントの返事を、ターミナルに**1文字ずつ**流したい。いまの `anthropic.messages.create({...})` を **`anthropic.messages.stream({...})`** に替えて、**`stream.on(\"text\", d => process.stdout.write(d))`** で断片を流し、完成品は **`const res = await stream.finalMessage()`** で受け取って。**`.on()` を `new Promise()` で包まないで**（`finalMessage()` を使う）。ループ判定（`stop_reason` / `tool_use`）と道具の実行は**今のまま変えないで**。AIが道具を使う往復では、実行の前に **`[道具を使います: ${block.name}]`** を表示して。道具の `input` は**途中でparseせず**、`finalMessage()` の完成品から読むこと。最後の `printText` は**消して**（流して表示済みなので二重表示しない）。失敗したら `try/catch` で一言出して」

出てきたコードの確認ポイント：

- [ ] `create` ではなく **`messages.stream({...})`** を使っているか（`tools` も渡しているか）
- [ ] 断片を **`stream.on("text", d => process.stdout.write(d))`** で流しているか（`console.log` で毎回改行していないか）
- [ ] 完成品を **`await stream.finalMessage()`** で受け取り、`.on()` を**自前のPromiseで包んでいない**か
- [ ] **ループ判定（`stop_reason` / `tool_use`）と `executeTool` は第4章のまま**か
- [ ] 道具を使う前に **`[道具を使います: ${block.name}]`** を表示しているか
- [ ] 道具の `input` を**途中でparseしていない**か（`finalMessage()` の完成品から読んでいるか）
- [ ] 古い **`printText` を消して**、最終回答を二重表示していないか
- [ ] **ループ回数の上限**（暴走防止）が残っているか／途中エラーを `try/catch` で出しているか

---

## 📝 ことばメモ

- **ストリーミング**：返事を**完成まで待たず、できた断片から順に流す**やり方。前作はブラウザへ（SSE）、今回はターミナルへ（`process.stdout.write`）流す
- **デルタ（delta＝差分）**：1回の通知で**今回増えたぶん**の小さな断片。`stream.on("text", delta => ...)` の `delta` がこれ
- **`finalMessage()`**：流し終わったあと、**完成した `Message`（`stop_reason` / `content` 入り）**をまとめて返してくれるSDKの関数。これを使えば自前の `new Promise()` は不要
- **`process.stdout.write`**：ターミナルに**改行なしで**書き出す関数。`console.log`（毎回改行）と違い、断片をつなげて「1文字ずつ」に見せられる

---

## ➡️ 次の章へ

これで、考えている言葉も、道具を使う瞬間も、**流しながら見せる**エージェントになりました。Claude Code のあの「いま何をしているか分かる」体験を、自分の手で再現できたわけです。

でも、ここまで作ってきて、**ある不安**が頭をよぎりませんか——往復を重ねるほど、会話履歴（`messages`）は**どんどん長くなります**。第4章で見たように、毎ターン**これまでの全部をAIに送り直して**いるからです。長い作業を続けると、いつか**AIが一度に見られる量（コンテキスト）の限界**にぶつかり、料金も積み上がります。

次の第14章では、この「**長い作業の記憶**」の問題に向き合います。コンテキストとは何か、なぜ膨らむのか、**古い話を要約して圧縮する**にはどうするか、そしてコストとどう付き合うか——を見ていきます。

[→ 第14章 長い作業の記憶 — コンテキストが膨らむ問題・要約・コストへ](14-context.md)

[← 目次・はじめにへもどる](README.md)
