---
title: "付録F　会話中の割り込み・中断の作り込み（AbortController / stop）"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 20
nav_exclude: true
---

# 付録F　会話中の割り込み・中断の作り込み（AbortController / stop）

> 📖 このページのゴール：生成を途中で「止める」を安全に作る。
> [← 目次・はじめにへもどる](README.md)

---

## 🛑 何が難しい？

ChatGPTには **「⏹ 停止」ボタン** があります。返事が長すぎたとき、押すと **途中でピタッと止まる**。第11章（[SSE](11-streaming-sse.md)）で「1文字ずつ流す」を作りましたが、**流している最中に止める**には、もうひと工夫いります。

難しいのは、止めた **そのあと** です。途中で打ち切ると、画面には **書きかけのassistant返事** が残ります。これをうっかり保存して、次の送信でAIに渡すと——第7章（[順番が崩れるとき](07-message-order.md)）で見た **壊れた履歴** と同じ問題が起きます。

> **止めることより、止めたあとの後始末のほうが大事。** 「表示・保存・次の送信」の3つを壊さないように作ります。

このページは第7章の続きです。第7章は「止めたあとの履歴を整える」話、ここは「**実際に止めるしくみ**」の話です。

---

## 🖥 画面側：止めるボタン

ブラウザには **`AbortController`（アボート・コントローラー＝中断スイッチ）** という標準部品があります。これを `fetch` に渡しておくと、あとから **こちらの都合で通信を打ち切れます**。

```ts
let controller: AbortController | null = null; // いまの“中断スイッチ”

async function send(text: string) {
  controller = new AbortController(); // ① 新しいスイッチを用意
  try {
    const res = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ message: text }),
      signal: controller.signal, // ② このスイッチと通信をつなぐ
    });
    await readStream(res); // 第11章の「断片を足す」読み取り
  } catch (err) {
    if (err.name === "AbortError") return; // ③ 止めただけ＝エラー扱いしない
    throw err; // それ以外は本当の失敗
  } finally {
    controller = null; // ④ 役目を終えたスイッチは片づける
  }
}

stopButton.addEventListener("click", () => {
  controller?.abort(); // ⑤ 押されたら通信を打ち切る
});
```

**1行ずつ読むと：**
- `let controller`：いま動いている送信の **中断スイッチ** を1つだけ覚えておく入れもの。
- `new AbortController()`：送信のたびに **新しいスイッチ** を作る（前のを使い回さない）。
- `signal: controller.signal`：スイッチの **信号線（signal）** を `fetch` に渡す。これで「あとで止められる通信」になる。
- `err.name === "AbortError"`：`abort()` で止めると、`fetch` は **`AbortError`** で終わる。これは **失敗ではなく中断** なので、画面に「失敗」と出さずに静かに抜ける。
- `finally { controller = null }`：成功・失敗・中断の **どれでも** スイッチを片づける（古いスイッチが残ると、次の停止が誤爆する）。
- `controller?.abort()`：停止ボタンが押されたら **いまの通信を打ち切る**。`?.` は「スイッチが無ければ何もしない」の保険。

> 💡 第11章のように **`EventSource`** を使っている場合は、`AbortController` のかわりに **`eventSource.close()`** で止めます。`fetch`＋reader なら、上の `signal` 方式が素直です。

---

## 🧩 サーバー側：LLM呼び出しも止める

ここが **いちばん見落としがち** なところ。画面側で `abort()` しても、**サーバーがOpenAIを呼び続けていたら、生成は止まりません**（＝料金も発生し続ける）。だからサーバーでも、ちゃんと止めます。

合図は **「クライアントが切断した」** こと。`fetch` の `abort()` は通信を切るので、サーバーは `req.on("close")` でそれを **検知** できます。

```ts
app.post("/api/chat", async (req, res) => {
  const { message } = req.body;
  res.setHeader("Content-Type", "text/event-stream");
  res.flushHeaders();

  const upstream = new AbortController(); // OpenAI呼び出し用の中断スイッチ
  req.on("close", () => upstream.abort()); // ① 切断を検知したら止める

  try {
    const stream = await openai.chat.completions.create(
      { model: "gpt-4o-mini", messages: [{ role: "user", content: message }], stream: true },
      { signal: upstream.signal }, // ② OpenAI呼び出しにもスイッチを渡す
    );
    for await (const chunk of stream) {
      const piece = chunk.choices[0]?.delta?.content ?? "";
      if (piece) res.write(`data: ${piece}\n\n`);
    }
    res.write("data: [DONE]\n\n");
  } catch (err) {
    if (err.name !== "AbortError") res.write("data: [ERROR]\n\n"); // ③ 中断は静かに
  } finally {
    res.end(); // ④ どんな終わり方でも接続を閉じる
  }
});
```

**1行ずつ読むと：**
- `const upstream = new AbortController()`：**サーバー → OpenAI** の呼び出しを止めるための、サーバー側のスイッチ。画面側のものとは別物。
- `req.on("close", () => upstream.abort())`：ブラウザが通信を切ったら（停止ボタン or タブ閉じ）、**その瞬間にOpenAIへの呼び出しも止める**。これが課金を止めるカギ。
- `{ signal: upstream.signal }`：`create()` の **第2引数** にスイッチの信号線を渡す。これで「途中でやめられるAPI呼び出し」になる。
- `for await (... of stream)`：断片を流すループ。`abort()` されると、このループは **`AbortError` を投げて止まる**。
- `if (err.name !== "AbortError")`：中断はエラーではないので **`[ERROR]` を送らない**。本当の失敗のときだけ画面に伝える。
- `finally { res.end() }`：成功・失敗・中断、**どれでも必ず接続を閉じる**（開きっぱなしを防ぐ）。

> ⚠️ `req.on("close")` は「正常に終わって閉じた」ときも呼ばれます。`upstream.abort()` を **すでに終わったストリームに対して呼んでも無害** なので、そのままで大丈夫です。

---

## 🧹 後始末

止めたあとに残る **書きかけのassistant返事** を、どう扱うか決めます。やり方は2つ。

1. **破棄する**：途中のテキストは保存せず、捨てる。いちばん簡単で安全。
2. **「（中断）」付きで保存する**：途中までの文に印を足して残す。あとで読み返せるが、**続きを送るときは履歴から除く** 配慮がいる。

```ts
function finalizeReply(partial: string, aborted: boolean): ChatMessage {
  if (aborted) {
    return { role: "assistant", content: partial + "（中断）", status: "done" };
  }
  return { role: "assistant", content: partial, status: "done" };
}
```

**1行ずつ読むと：**
- `partial`：止まるまでに **届いていた途中のテキスト**。
- `aborted`：**中断で終わったか** のしるし（`AbortError` を受けたら `true`）。
- `partial + "（中断）"`：途中保存を選ぶなら、**人にもAIにも「ここで切れた」と分かる印** を足す。
- `status: "done"`：第7章の札。**`pending`（書きかけ）のまま残さない** のが肝心。中断でも「もう確定」として `done` にし、宙ぶらりんをなくす。

後始末で守ることは3つ。

- **未完了を放置しない**：止めた発言を `pending` のまま残すと、次に開いたとき履歴がゴミだらけに。**必ず `done` に確定** させる（破棄でも、中断印付き保存でも）。
- **次の送信に混ぜない**：中断印付きで残した場合、AIへ渡す前に第7章の `cleanHistory` で **整える**（壊れた途中文をそのまま渡さない）。
- **二重送信を防ぐ＝冪等性**：止めた直後に再送して、**同じお願いが2回飛ぶ** ことがある。第7章の送信ロックと「同じ内容が直前にないか」チェックで、**2回押しても1回分** にする。

---

## ⚠️ ハマりどころ

- **画面では止めたのに、サーバーでAPIが回り続ける**：`req.on("close")` を入れ忘れ／`create()` に `signal` を渡し忘れると、**裏でOpenAIが最後まで生成し、料金が発生** します。止める=「画面」と「サーバー呼び出し」の **両方** を止めること。
- **中断を「失敗」として扱う**：`AbortError` を普通のエラーと混同すると、止めるたびに「失敗しました」と出てしまう。**`err.name === "AbortError"` を分けて** 静かに抜ける。
- **壊れた履歴を次のmessagesに混ぜる**：途中で切れたassistant文を整えずに渡すと、APIエラーや噛み合わない応答に。送信前に必ず `cleanHistory`（第7章）を通す。
- **二重abort／スイッチの使い回し**：1つの `AbortController` を **複数の送信で使い回す** と、新しい送信まで巻き添えで止まる。**送信のたびに新しく作り**、終わったら `null` で片づける。
- **`pending` の残骸**：止めた発言を確定し忘れると、次に開くたびに混乱のもと。**開くときに捨てる**（第7章ステップ2）を徹底。

---

## 📝 ことばメモ

- **AbortController**：通信や処理を **あとから打ち切る** ための標準スイッチ。送信ごとに新しく作る
- **signal**：`AbortController` の信号線。`fetch` や OpenAI の `create()` に渡すと「止められる」ようになる
- **切断検知（req.on("close")）**：ブラウザが通信を切ったのをサーバーが気づくしくみ。これでAPI呼び出しも止める
- **後始末**：止めたあとの片づけ。書きかけを破棄/中断印付き保存し、`pending` を残さず、二重送信を防ぐこと

---

[← 目次・はじめにへもどる](README.md)
