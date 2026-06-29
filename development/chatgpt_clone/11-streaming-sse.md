---
title: "第11章　ストリーミング（SSE編）— ChatGPTの「1文字ずつ」"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 11
---

# 第11章　ストリーミング（SSE編）— ChatGPTの「1文字ずつ」

> 📖 この章のゴール：**なぜChatGPTは1文字ずつ出るのか**を理解し、**SSE**で「できたそばから流す」最小の仕組みを作れるようになる。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 ChatGPTではこう見える

ChatGPTに何か聞くと、返事は **一気にドン**とは出ません。**1文字ずつ、スルスルッと**流れるように出てきます。タイプライターみたいに。

実はこの演出、ただのオシャレではありません。**待ち時間の体感を短くする**大事な工夫です。長い返事でも「もう書き始めてくれてる」と分かるので、固まって見えないのです。

第3章で作った「送って待つ」チャット（REST）は、これと正反対でした。この章では、その差を埋めにいきます。

---

## 🤔 RESTは「全部できてから」、SSEは「できた皿から」（🟢 基礎）

第3章の **REST（レスト）** は、「**お願いを送る → AIが全部書き終わるまで待つ → 完成品が一気に届く**」方式でした。だから、長い返事ほど待ち時間が目立ちます（その間、画面は「考え中…」のまま）。

そこで登場するのが **SSE（エスエスイー）** です。

> **SSE = Server-Sent Events（サーバー・セント・イベンツ）**
> 日本語にすると「**サーバーから送られてくる出来事**」。**サーバー → ブラウザの一方向**に、データを**少しずつ流し込み続ける**しくみです。

LLMは、実は返事を **少し書くたびに、その断片（かけら）を出せます**。SSEを使うと、書けたそばから**その断片をブラウザに送り**、画面はそれを**順に足して**いきます。これが「1文字ずつ」の正体です。

> 🍽 **たとえ話：料理の運び方**
> - **REST**＝コース料理を **全部できてから、まとめて1回で運ぶ**。最後の皿が焼けるまで、テーブルは空っぽ。
> - **SSE**＝**できた皿から、順番にどんどん運ぶ**。待っている人は「お、来た来た」と楽しめる。
>
> 料理（返事）の総量は同じでも、**運び方**を変えるだけで体感がまるで違います。

> 💡 SSEは「**サーバー → ブラウザ**」の**一方通行**です。ブラウザから「やっぱり止めて！」と**逆向きに割り込む**（生成を途中で止める）話は、**付録F**で扱います（双方向通信が本当に要るときの WebSocket は **付録E** に参考としてまとめています）。

---

## 🛠 こう作る — `/api/chat` を「流す」形にする（🟢 基礎）

やることは2つだけ。**①サーバーで OpenAI を“流す”呼び方にして断片を送る**、**②画面で断片を受けて足していく**。

### ステップ1：サーバー（server.ts）を「流す」呼び方にする

第3章の `/api/chat` を、**ストリーム対応**に書き換えます。

```ts
// server.ts  （鍵を知っているのはこのサーバーだけ）
app.post("/api/chat", async (req, res) => {
  const { message } = req.body;

  res.setHeader("Content-Type", "text/event-stream"); // SSEで流す宣言
  res.setHeader("Cache-Control", "no-cache");
  res.flushHeaders(); // ヘッダーをすぐ送り出す（ためこまない）

  try {
    const stream = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: message }],
      stream: true, // ★ここがポイント：断片で受け取る
    });

    for await (const chunk of stream) {
      const piece = chunk.choices[0]?.delta?.content ?? "";
      if (piece) res.write(`data: ${piece}\n\n`); // 断片を1つ送る
    }
    res.write("data: [DONE]\n\n"); // 終わりの合図
  } catch (err) {
    console.error("ストリーム中に失敗:", err);
    res.write("data: [ERROR]\n\n"); // 画面に「失敗」を伝える
  } finally {
    res.end(); // 流しおわり。接続を閉じる
  }
});
```

**1行ずつ読むと：**
- `res.setHeader("Content-Type", "text/event-stream")`：ブラウザに「これから **SSEで少しずつ流すよ**」と伝える宣言。これがSSEの目印。
- `res.setHeader("Cache-Control", "no-cache")`：途中の断片を **キャッシュにためこませない**（古いものを返させない）。
- `res.flushHeaders()`：ヘッダーを **すぐに送り出す**。ためこむ設定だと、断片がまとめて届いて「1文字ずつ」にならない（後述）。
- `stream: true`：OpenAIに「**完成を待たず、断片で渡して**」と頼むスイッチ。これが無いと第3章と同じ「全部いっぺん」。
- `for await (const chunk of stream)`：流れてくる断片を **届いた順に1つずつ**受け取るループ。
- `chunk.choices[0]?.delta?.content ?? ""`：断片の中の **増えた文字だけ**を取り出す（`delta`＝差分＝今回増えたぶん）。中身が無い断片もあるので `?? ""` で空にしておく。
- `res.write(\`data: ${piece}\n\n\`)`：取り出した断片を **SSEの形**でブラウザへ送る。`data:` で始め、**末尾は空行（`\n\n`）** が決まりごと（くわしい形式は付録E）。
- `res.write("data: [DONE]\n\n")`：全部送り終わったら、**おしまいの合図**を自分で送る（受け取る側の止め時になる）。
- `catch (err)` → `data: [ERROR]`：途中で失敗しても、**だまって固まらせず**「失敗」を画面に伝える。
- `finally { res.end() }`：成功でも失敗でも、最後に **接続をきちんと閉じる**（開きっぱなしを防ぐ）。

### ステップ2：画面（public/index.html）で断片を受けて足す

画面は **素のTS/JS**のまま。`fetch` で受け取り、**届いた断片を表示欄に足していく**だけです。

```html
<script>
  const send = document.getElementById("send");
  const input = document.getElementById("msg");
  const replyArea = document.getElementById("reply");

  send.addEventListener("click", async () => {
    replyArea.textContent = ""; // 表示欄をまっさらに
    const res = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ message: input.value }),
    });

    const reader = res.body.getReader(); // 流れを少しずつ読む読み取り口
    const decoder = new TextDecoder(); // バイト列を文字にもどす道具
    while (true) {
      const { value, done } = await reader.read(); // 次の断片を待つ
      if (done) break; // もう来ない＝終わり
      const text = decoder.decode(value); // 文字にもどす
      for (const line of text.split("\n")) {
        if (line.startsWith("data: ")) {
          const piece = line.slice(6); // "data: " の後ろが中身
          if (piece === "[DONE]") return; // 終わりの合図なら抜ける
          replyArea.textContent += piece; // ★届いたぶんを足す
        }
      }
    }
  });
</script>
```

**1行ずつ読むと：**
- `replyArea.textContent = ""`：送るたびに表示欄を **空にしてから**始める（前の返事が残らないように）。
- `res.body.getReader()`：返事の本体を **少しずつ読む読み取り口**を作る（全部待たずに、来たぶんから読む）。
- `new TextDecoder()`：ネットワークを流れる **バイト（数字の列）を、人が読める文字にもどす**道具。
- `await reader.read()`：**次の断片が届くまで待って**受け取る。`done` が `true` なら、もう来ない合図。
- `text.split("\n")`：届いたかたまりを **行ごと**に分ける（1回に複数行まとめて届くこともある）。
- `line.startsWith("data: ")`：SSEの中身は **`data:` で始まる行**。それ以外は無視。
- `line.slice(6)`：`"data: "`（6文字）の **後ろが本当の中身**。そこを取り出す。
- `if (piece === "[DONE]") return`：サーバーが送った **おしまいの合図**なら、読み取りを終える。
- `replyArea.textContent += piece`：**`+=` で足していく**のがキモ。断片が届くたびに文字が増え、「1文字ずつ」に見える。表示は安全な **`textContent`**（`innerHTML` は使わない）。

> 💡 もっと手軽な **`EventSource`（イベントソース）** という標準部品もあります（断片の受信を自動でやってくれる）。ただし `EventSource` は **GET専用**で、こちらのように **POSTで本文を送る**場合は、今回の `fetch` ＋ reader の形が向いています。使い分けは付録Eへ。

> ✅ **動かし方**：第3章のサーバーを上のように書き換えて起動し、`http://localhost:3000` を開く。送信して、返事が**スルスルと1文字ずつ**出れば成功です。

---

## ⚠️ ハマりどころ

- **まとめてドンと届く（1文字ずつにならない）**：途中のどこかが断片を**ためこんでいる**サインです。サーバー側で `res.flushHeaders()` を入れる／圧縮（gzip）を切る、などで改善します。原因は**バッファリング**（小出しをまとめてしまう仕組み）。
- **本番に出したら流れない**：間に立つ **プロキシや nginx（エヌジンエックス）** が、SSEを**ためこんで潰す**ことがあります。`X-Accel-Buffering: no` などの設定が必要な場合も（詳しくは付録E）。手元では動くのに本番でだけ固まる、の典型。
- **途中でエラー／接続が切れたとき**：通信は途中で切れます。サーバーは `try/catch/finally` で **必ず `res.end()`** して後始末を。画面側は `[ERROR]` を受けたら「失敗しました」と出す（無言で止めない）。
- **断片の結合ミス**：断片は **きれいな1文字ずつとは限らず**、行や文字の途中で割れて届くこともあります。`data:` 行だけ拾って `+=` で足す形なら大きく崩れませんが、厳密な復元は付録Eで。
- **`innerHTML` で表示してしまう**：流れてくる文字をそのまま `innerHTML` に入れると危険（XSS）。**`textContent`** で安全に。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「**TypeScript + Express** の `/api/chat` を、**SSE（`Content-Type: text/event-stream`）で1文字ずつ返す**形にして。OpenAIは **`stream: true`** で呼び、届いた **`delta.content` を `res.write(\"data: ...\\n\\n\")`** で流して。終わりに **`[DONE]`** を送り、`try/catch/finally` で **必ず `res.end()`** すること。**APIキーはサーバー側の `process.env` のまま**、フロントには出さないで。画面は **素のTS/JS** で `fetch` の reader を使い、**届いた断片を `textContent` に足して**表示して」

出てきたコードの確認ポイント：

- [ ] サーバーが **`Content-Type: text/event-stream`** を返しているか
- [ ] OpenAI呼び出しが **`stream: true`** になっているか
- [ ] 鍵が **サーバー側（`process.env`）のまま**か（フロントに漏れていないか）
- [ ] 断片を **`res.write("data: ...\n\n")`** の形（末尾の空行）で送っているか
- [ ] **`[DONE]`** の合図と、`finally` での **`res.end()`** があるか
- [ ] 画面が **`textContent` に足して**いるか（`innerHTML` でないか）

---

## 📝 ことばメモ

- **ストリーミング**：返事を**完成まで待たず、できた断片から順に流す**やり方。ChatGPTの「1文字ずつ」がこれ
- **SSE（Server-Sent Events）**：**サーバー → ブラウザの一方向**にデータを流し込み続けるしくみ。`text/event-stream` で送る
- **EventSource**：ブラウザ標準のSSE受信部品。手軽だが**GET専用**（POSTで本文を送るなら `fetch`＋reader）
- **チャンク（chunk）**：流れてくる**断片**のひとかたまり。中の `delta` が「今回増えた文字」

---

## ➡️ 次の章へ

これで、ChatGPTらしい「1文字ずつ」が作れました。でもSSEは **サーバー → ブラウザの一方通行**。だから「**やっぱり止めて！**」のように、**ブラウザから逆向きに割り込む**のは苦手です。

次の第12章では、LLMに **道具（ツール）を持たせる** 話に進みます。文章を作るのは得意でも、**今日の天気を調べる・計算する**といった“外の作業”は自分ではできません。そこで **function calling（関数呼び出し／tool use）** の出番です。（※返事の途中で「**止める**」割り込みは付録F、双方向通信が要るときの WebSocket は付録E に参考があります）

[→ 第12章 ツールを持たせる — function calling / tool use へ](12-tools.md)

[← 目次・はじめにへもどる](README.md)
