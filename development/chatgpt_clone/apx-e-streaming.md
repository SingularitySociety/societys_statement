---
title: "付録E　ストリーミングの中身 — SSE と WebSocket の実装詳細"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 19
nav_exclude: true
---

# 付録E　ストリーミングの中身 — SSE と WebSocket の実装詳細

> 📖 このページのゴール：SSEとWebSocketを実際に書けるようになる。
> [← 目次・はじめにへもどる](README.md)

---

第11章（SSE）で「1文字ずつ流す」感覚をつかみました。この付録では、その**中身を手を動かして書ける**ところまで踏み込みます。**双方向の WebSocket** は本編からは外していますが、必要になったとき用にここで参考としてまとめます。鍵を知っているのは**サーバーだけ**、という背骨はここでも変わりません。

## 📡 SSE の中身

SSEの正体は、ふつうのHTTPレスポンスを**閉じずに開いたまま**にして、そこへ少しずつ書き足し続けることです。守るべき形は3つだけ。

- `Content-Type: text/event-stream` を返す（「これは流すよ」の宣言）
- 1つの断片を `data: <断片>\n\n` の形で送る（**末尾は必ず空行＝`\n\n`**）
- 全部送り終えたら `data: [DONE]\n\n` を送る（受け取る側の止め時）

OpenAIの `stream: true` で届くチャンクを取り出して、そのまま流すサーバー例です。

```ts
// server.ts （鍵を知っているのはこのサーバーだけ）
app.post("/api/chat", async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.setHeader("X-Accel-Buffering", "no"); // nginxにためこませない
  res.flushHeaders(); // ヘッダーを今すぐ送り出す

  const stream = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: req.body.message }],
    stream: true,
  });

  for await (const chunk of stream) {
    const piece = chunk.choices[0]?.delta?.content ?? "";
    if (piece) res.write(`data: ${piece}\n\n`); // 断片を1つ流す
  }
  res.write("data: [DONE]\n\n"); // おしまいの合図
  res.end();
});
```

**1行ずつ読むと：**
- `Content-Type` を `text/event-stream` に：ブラウザに「**これからSSEで少しずつ流す**」と伝える、SSEの目印。
- `Cache-Control: no-cache`：途中の断片を**キャッシュにためこませない**（古いものを返させない）。
- `Connection: keep-alive`：返事の最中も**接続を開いたまま**にしておく。
- `X-Accel-Buffering: no`：間に立つ **nginx に「ためこまず素通しして」** と頼むヘッダー（後述のハマりどころ対策）。
- `res.flushHeaders()`：ヘッダーを**今すぐ**押し出す。これを忘れると最初の断片が遅れて「1文字ずつ」に見えない。
- `stream: true`：OpenAIに「**完成を待たず断片で渡して**」と頼むスイッチ。
- `for await (const chunk of stream)`：流れてくる断片を**届いた順に1つずつ**受け取るループ。
- `chunk.choices[0]?.delta?.content ?? ""`：チャンクの中の**今回増えた文字だけ**を取り出す（`delta`＝差分）。中身が空の断片もあるので `?? ""`。
- `res.write(\`data: ${piece}\n\n\`)`：断片を**SSEの形**で送る。`data:` で始め、**末尾は空行（`\n\n`）** が決まり。
- `res.write("data: [DONE]\n\n")`：全部送り終えたら、自分で**おしまいの合図**を流す。
- `res.end()`：接続を**きちんと閉じる**（開きっぱなしを防ぐ）。

> 💡 `res.flushHeaders()` とバッファ無効化（`Cache-Control: no-cache` ／ 圧縮gzipを切る）が、SSEを「ためずに流す」ための2大ポイントです。

---

## 🖥 画面側で受ける

受け取り方は2通り。手軽な **`EventSource`** と、POSTもできる **`fetch` + reader** です。

`EventSource` は標準部品で、断片の受信を自動でやってくれます。ただし**GET専用**なので、本文をクエリで渡します。

```ts
const source = new EventSource(`/api/chat?message=${encodeURIComponent(text)}`);
const replyArea = document.getElementById("reply")!;

source.onmessage = (e) => {
  if (e.data === "[DONE]") { source.close(); return; } // 合図なら閉じる
  replyArea.textContent += e.data; // 届いたぶんを足す
};
source.onerror = () => source.close(); // 失敗したら閉じる
```

**1行ずつ読むと：**
- `new EventSource(...)`：SSEを**自動で受信する標準部品**を作る。URLに本文をのせる（GET専用なので）。
- `encodeURIComponent(text)`：URLに入れて壊れないよう**文字をエスケープ**する。
- `source.onmessage = (e) => ...`：断片が届くたびに呼ばれる。中身は `e.data`（`data:` の後ろが入っている）。
- `if (e.data === "[DONE]")`：**おしまいの合図**なら、`source.close()` で受信をやめる。
- `replyArea.textContent += e.data`：**`+=` で足す**のがキモ。文字が増えて「1文字ずつ」に見える。安全な `textContent` を使う。
- `source.onerror`：通信が切れたら**閉じて後始末**（無言で放置しない）。

POSTで本文を送りたいときは、`fetch` の `response.body.getReader()` で同じことができます。

```ts
const res = await fetch("/api/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: text }),
});
const reader = res.body!.getReader();
const decoder = new TextDecoder();

while (true) {
  const { value, done } = await reader.read(); // 次の断片を待つ
  if (done) break;
  for (const line of decoder.decode(value).split("\n")) {
    if (!line.startsWith("data: ")) continue;
    const piece = line.slice(6); // "data: " の後ろが中身
    if (piece === "[DONE]") break;
    replyArea.textContent += piece; // 届いたぶんを足す
  }
}
```

**1行ずつ読むと：**
- `res.body.getReader()`：返事の本体を**少しずつ読む読み取り口**を作る（全部待たない）。
- `new TextDecoder()`：ネットワークを流れる**バイト（数字の列）を文字にもどす**道具。
- `await reader.read()`：**次の断片が届くまで待って**受け取る。`done` が `true` ならもう来ない合図。
- `decoder.decode(value).split("\n")`：届いたかたまりを文字にもどし、**行ごと**に分ける（複数行まとめて届くこともある）。
- `line.slice(6)`：`"data: "`（6文字）の**後ろが本当の中身**。
- `replyArea.textContent += piece`：**足していく**。`innerHTML` は使わず `textContent` で安全に。

---

## 🎁 ストリームの後に「REST と同じ形」の最終レスポンスを取る

上の `stream: true` の生ループは、**届く断片（delta）を自分でつなぐ**やり方でした。表示にはこれで十分ですが、**保存**には少し不便です——`tool_calls`（ツール呼び出し）や `usage`（使ったトークン数）、`finish_reason`（終わり方）は断片に散らばっていて、手で組み立て直すのが面倒だからです。

そこで多くのSDKは、**「ストリームで流しつつ、終わったら REST（非ストリーミング）と同じ形の“完成レスポンス”を1発で受け取る」ヘルパー**を用意しています。**表示はストリーム、保存は完成レスポンス**と役割を分けられて、`tool_calls` も `usage` も取りこぼしません（→ 保存は付録J）。

```ts
// OpenAI（openai）: client.chat.completions.stream(...)
const stream = openai.chat.completions.stream({ model: "gpt-4o-mini", messages });
stream.on("content", (delta) => res.write(`data: ${delta}\n\n`)); // 表示はストリーム
const final = await stream.finalChatCompletion();                  // ← REST と同じ ChatCompletion
// final.choices[0].message（content / tool_calls）・final.usage・final.choices[0].finish_reason
await saveMessage(conversationId, final.choices[0].message);        // 保存はこれ1発でOK
```

```ts
// Anthropic（@anthropic-ai/sdk）: client.messages.stream(...)
const stream = anthropic.messages.stream({ model, max_tokens: 1024, messages });
stream.on("text", (t) => res.write(`data: ${t}\n\n`));   // 表示はストリーム
const final = await stream.finalMessage();                // ← messages.create と同じ Message
// final.content（text / tool_use ブロック）・final.stop_reason・final.usage
```

```ts
// Vercel AI SDK（ai）: streamText(...)
const result = streamText({ model, messages });
// 表示：result.textStream を画面へ流す（フレームに合わせて pipe / Response 化）
const text   = await result.text;          // 完成テキスト
const usage  = await result.usage;         // トークン使用量
const finish = await result.finishReason;  // 終わり方
// もしくは onFinish: ({ text, usage, finishReason, response }) => save(...) コールバックで受ける
```

**1行ずつ読むと（共通の考え方）：**
- `stream.on("content" / "text", ...)`：**届いた断片を画面へ流す**（ユーザー体験＝1文字ずつ）。
- `await stream.finalChatCompletion()` ／ `finalMessage()` ／ `await result.text`：**ストリームが終わるのを待って、完成した“まるごと”を受け取る**。中身は非ストリーミングの戻り値と**同じ形**。
- だから**保存**は、手で `full += piece` するより、この**完成レスポンスをそのまま使う**ほうが安全（`tool_calls`・`usage`・`finish_reason` 込み）。

> 💡 生ループ（`for await`）で自前管理してもOKですが、その場合 `usage` を取るには OpenAI なら `stream_options: { include_usage: true }` を足す、`tool_calls` は `delta.tool_calls` を id ごとに継ぎ足す、といった**手作業**が増えます。**ヘルパーが使えるなら使う**のが楽で安全です。

---

## 🔌 WebSocket の実装

WebSocketは**つなぎっぱなしの1本の線**で、ブラウザとサーバーがいつでも送り合えます。サーバー側は **`ws`** パッケージ（`yarn add ws`）で、「**接続・message・close**」の3つを扱います。

```ts
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (socket) => {        // ① ブラウザが1本つないできた
  const stop = new AbortController();      //    中断スイッチを用意

  socket.on("message", (raw) => {          // ② ブラウザから合図が届いた
    const msg = JSON.parse(raw.toString());
    if (msg.type === "chat") streamReply(socket, msg.text, stop.signal);
    if (msg.type === "stop") stop.abort(); // ③ "stop"なら生成を打ち切る
  });

  socket.on("close", () => stop.abort());  // ④ 線が切れたら後始末
});
```

**1行ずつ読むと：**
- `new WebSocketServer({ port: 8080 })`：双方向の**待ち受け口**を1つ開く（ポート＝入り口の番号）。
- `wss.on("connection", (socket) => ...)`：ブラウザが**つないでくる**たび、その人専用の `socket`（話し相手1本）が手に入る。
- `const stop = new AbortController()`：**中断スイッチ**を用意（`abort()` で「やめて」を伝える標準の仕組み。詳しくは付録F）。
- `socket.on("message", ...)`：その線から**合図が届く**たびに呼ばれる。SSEと違い、ここで**ブラウザ側の声を受け取れる**。
- `if (msg.type === "stop") stop.abort()`：合図が `"stop"` なら、スイッチを入れて**生成を打ち切るきっかけ**にする。
- `socket.on("close", ...)`：線が**切れたとき**に呼ばれる。動いていた生成を止めて**後片付け**する（やりっぱなしを防ぐ）。

画面側は標準部品 `new WebSocket(...)` でつなぎ、送るのも受けるのも同じ1本の線です。

```ts
const ws = new WebSocket("ws://localhost:8080");

ws.onopen = () => ws.send(JSON.stringify({ type: "chat", text })); // 質問を送る
ws.onmessage = (e) => { replyArea.textContent += e.data; };         // 断片を足す

stopBtn.onclick = () => ws.send(JSON.stringify({ type: "stop" }));  // 止めてと送る
```

**1行ずつ読むと：**
- `new WebSocket("ws://localhost:8080")`：サーバーへ**つなぎっぱなしの線**を1本張る（`ws://` で始まる）。
- `ws.onopen`：つながった瞬間に呼ばれる。ここで `type: "chat"` の合図に**質問をのせて送る**。
- `ws.onmessage`：サーバーから断片が届くたびに呼ばれる。`textContent += e.data` で**足していく**。
- `stopBtn.onclick`：停止ボタンを押したら `type: "stop"` を送る。**返事の最中でも逆向きに割り込める**のが双方向の強み。

---

## 🔁 SSE と WebSocket の使い分け

| | SSE | WebSocket |
|---|---|---|
| 向き | サーバー → ブラウザの**片方向** | ブラウザ⇄サーバーの**双方向** |
| 得意な用途 | 返事を1文字ずつ流す（放送） | 途中で止める・入力中の共有（会話） |
| 難しさ | やさしい（ふつうのHTTPの延長） | やや難しい（接続管理・再接続が要る） |

**多くのチャットはSSEで十分**です。「双方向＝えらい」ではありません。リアルタイムに止める・複数人の入力中を共有する、といった**本当に双方向が要るとき**だけWebSocketを足します。合言葉は「**まずSSE、必要になったらWebSocket**」。

---

## ⚠️ ハマりどころ

- **プロキシ／nginx がSSEを途中で溜める**：手元では流れるのに本番でだけ固まる典型。`X-Accel-Buffering: no` を返す／nginx側で `proxy_buffering off;` にする、で素通しさせます。
- **flush忘れ**：`res.flushHeaders()` や圧縮(gzip)が原因で断片が**まとめてドンと届く**。途中のどこかが**ためこんでいる**サインです。
- **WSの再接続・切断管理**：線は勝手に切れます。`onclose` で**後片付け**し、画面側は**つなぎ直す**処理を用意する。閉じたソケットを放置しないこと。
- **認証をストリームでも通す**：背骨②（鍵と財布を守る）はここでも生きます。SSEもWebSocketも、**最初の接続時に「誰？」を必ず確認**する（つなぎっぱなしだからこそ最初が肝心）。

---

## 📝 ことばメモ

- **SSE（Server-Sent Events）**：サーバー → ブラウザの**一方向**に、`text/event-stream` でデータを流し続けるしくみ。
- **EventSource**：ブラウザ標準のSSE受信部品。手軽だが**GET専用**（POSTなら `fetch`＋reader）。
- **チャンク（chunk）**：流れてくる**断片**のひとかたまり。中の `delta` が「今回増えた文字」。
- **WebSocket**：ブラウザとサーバーをつなぎっぱなしにし、**双方向**で送り合える通信のしくみ。
- **flush（フラッシュ）**：ためこんだデータを**今すぐ押し出す**こと。SSEを「1文字ずつ」にする鍵。

---

[← 目次・はじめにへもどる](README.md)
