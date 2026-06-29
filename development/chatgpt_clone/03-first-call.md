# 第3章　はじめての会話（REST編）— 送って待つ、最初の1往復

> 📖 この章のゴール：**messages の形（system / user / assistant）** を理解し、Express の `/api/chat` と素のTS/JSの画面で「送って待つ」1往復のチャットを作れるようになる。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 ChatGPTではこう見える

ChatGPTで何か入力して送ると、少し待って返事が返ってきます。今回はこの **いちばん基本の1往復**——「**人が1つ送る → AIが1つ返す**」——を自分で作ります。

第2章では「鍵を隠して、1発言を送る」ところまで作りました。この章では、その送り方を **ChatGPTと同じ「会話の形」** に整えます。

> **送って、待って、返事が全部いっぺんに届く。** まずはこの素直なやりとりから。

---

## 🤔 会話は「messages 配列」で渡す（🟢 基礎）

LLM（大規模言語モデル＝文章を作るAI）に会話を伝えるときは、**messages（メッセージズ）** という **配列**（＝順番付きのリスト）で渡します。中身の1つ1つは、こんな形です。

```ts
{ role: "user", content: "こんにちは" }
```

**1行ずつ読むと：**
- `role`（ロール＝役割）：**誰の発言か**を表す札。次の3種類があります。
- `content`（コンテント＝中身）：その発言の **本文**（文章）。

役割（role）は3つ。たとえ話でいうと **舞台の配役**です。

- **system**（システム）＝ **AIへの“設定・前提”**。「あなたは親切な先生です」のような **キャラや約束ごと**。お客さんには見えない、舞台の **台本のト書き**。
- **user**（ユーザー）＝ **人間（あなた）の発言**。
- **assistant**（アシスタント）＝ **AI（ChatGPTクローン）の返事**。

この章で作るのは **system ＋ user の1往復**だけ。`assistant`（AIの過去の返事）を積み上げて“続き”にする話は、次の第4章からです。

> 🎭 **system プロンプト（ト書き）をもう少し**
> `system` は、この **アプリの“性格・ルール”を決める、いちばん大事な指示**です。ここに書いた内容は、その後の会話ぜんぶに効きます。よく入れるのは——
> - **キャラ・口調**：「親切で、むずかしい言葉を避ける先生」
> - **やること／やらないこと**：「料理の質問だけ答える」「わからなければ正直に言う」
> - **出力の形**：「結論を先に、3行以内で」
>
> 大事な性質が2つ。**①ユーザーには見えません**（裏方のト書き）。だからアプリの方針はここに集約します。**②とても強く効く**ぶん、ユーザーが「さっきの指示は無視して」と **上書きを狙ってくる**ことがあります（＝**プロンプトインジェクション**）。だから **system を過信せず、大事な判断はコード側でも守る**——この守りは第13章（ツール）・付録Gで扱います。

> 💡 **REST（レスト）って？**
> 今回の送り方は **REST 方式**——「**お願いを送る → できあがるまで待つ → 全部いっぺんに届く**」という、いちばん素直なやりとりです（宅配ピザを注文して、焼き上がりを待って、1箱で受け取るイメージ）。第2章の `/api/chat` も、実はこの REST でした。
> ※「1文字ずつパラパラ届く」あの演出は別のしくみ（SSE）で、第11章で扱います。

---

## 🛠 こう作る — system付きの1往復チャット（🟢 基礎）

### ステップ1：サーバー（server.ts）に system を足す

第2章の鍵の読み方はそのまま。`messages` に **system を1枚追加**します。

```ts
// server.ts  （鍵を知っているのはこのサーバーだけ）
import express from "express";
import OpenAI from "openai";

const app = express();
app.use(express.json());
app.use(express.static("public")); // 画面（public/index.html）も同じサーバーから配る

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const SYSTEM_PROMPT = "あなたは親切で、やさしい日本語で答えるアシスタントです。";

app.post("/api/chat", async (req, res) => {
  const { message } = req.body;
  try {
    const completion = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [
        { role: "system", content: SYSTEM_PROMPT },
        { role: "user", content: message },
      ],
    });
    res.json({ reply: completion.choices[0].message.content });
  } catch (err) {
    console.error("OpenAI呼び出しに失敗:", err);
    res.status(500).json({ error: "返事の生成に失敗しました" });
  }
});

app.listen(3000, () => console.log("起動 → http://localhost:3000"));
```

**1行ずつ読むと：**
- `app.use(express.json())`：ブラウザから届くJSON（`{ message: ... }`）を読めるようにする。
- `app.use(express.static("public"))`：`public` フォルダの中身（`index.html` など）を **そのまま配る**。これで画面（`/`）も窓口（`/api/chat`）も同じサーバーから出ます（1ポート配信）。
- `new OpenAI({ apiKey: process.env.OPENAI_API_KEY })`：**鍵は `.env` から読む**（第2章のとおり、直書きしない）。
- `const SYSTEM_PROMPT = "..."`：AIのキャラ・前提を **名前付きの定数**にまとめる（後で直しやすい）。
- `app.post("/api/chat", ...)`：ブラウザが叩く **自分の窓口**。
- `const { message } = req.body`：ブラウザが送ってきた文章を取り出す。
- `messages: [ {system}, {user} ]`：**台本（system）を先頭に、人の発言（user）を次に**並べて渡す。順番に意味があり、system は **いちばん最初**。
- `model: "gpt-4o-mini"`：使うモデル（軽くて安い。詳しくは付録D）。
- `res.json({ reply: ... })`：返事の **文章だけ** を返す（**鍵は返さない**）。
- `try { ... } catch (err)`：通信やAPIは **失敗しうる**ので、囲んで備える。
- `res.status(500).json({ error: ... })`：失敗したら、**500（サーバー側エラー）** という札を付けて、わけを返す。
- `console.error(...)`：原因はサーバーのログに残す（※会話本文など個人情報は出しすぎない。後述）。

### ステップ2：画面（public/index.html）を素のTS/JSで作る

画面は **特別な道具なし**（素のHTML＋JS）で作れます。Reactなどを使っても考え方は同じです。

```html
<!-- public/index.html -->
<!doctype html>
<html lang="ja">
  <body>
    <input id="msg" placeholder="メッセージを入力" />
    <button id="send">送信</button>
    <p id="reply"></p>

    <script>
      const sendButton = document.getElementById("send");
      const input = document.getElementById("msg");
      const replyArea = document.getElementById("reply");

      sendButton.addEventListener("click", async () => {
        replyArea.textContent = "考え中…";
        const res = await fetch("/api/chat", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ message: input.value }),
        });
        const data = await res.json();
        replyArea.textContent = data.reply;
      });
    </script>
  </body>
</html>
```

**1行ずつ読むと：**
- `<input id="msg" />`：文章を打ち込む入力欄。
- `<button id="send">`：押すと送信するボタン。
- `<p id="reply">`：返事を表示する場所。
- `document.getElementById("send")`：HTMLの部品を **idで取ってくる**（JSから触れるようにする）。
- `addEventListener("click", ...)`：ボタンが **押されたとき** に動く処理を登録。
- `replyArea.textContent = "考え中…"`：返事を待つあいだ、**待っている合図**を出す（固まって見えない工夫）。
- `fetch("/api/chat", ...)`：**自分のサーバー**の窓口へ送る（`api.openai.com` ではない＝ブラウザは鍵を持たない）。
- `body: JSON.stringify({ message: input.value })`：入力欄の文章を **JSONにして** 送る。
- `const data = await res.json()`：返ってきた `{ reply: ... }` を受け取る。
- `replyArea.textContent = data.reply`：返事を画面に表示。**`textContent`** を使うのがポイント（後述のとおり安全）。

> ✅ **動かし方**：`.env` に鍵を置いた状態でサーバーを起動し、ブラウザで `http://localhost:3000` を開く。入力して「送信」→ 少し待って返事が出れば成功です。

> 💡 **CORSは出る？ → 出ません（同一オリジン）**
> 画面（`/`）も窓口（`/api/chat`）も **同じ `http://localhost:3000` から配っている**ので、`fetch("/api/chat")` は **同一オリジン＝CORSの制限対象外**です。CORSが問題になるのは、画面を**別のオリジン**から出すとき——たとえば開発で **Vite等の開発サーバー（`localhost:5173`）から `localhost:3000` のAPIを叩く**場合。そのときは ①Vite側の**プロキシ設定**で `/api` をサーバーへ転送して同一オリジンに見せるか、②サーバーに **`cors` ミドルウェア**を入れて**自分のオリジンだけ許可**します（`*` で全許可にしない）。本番でドメインが分かれるときも同じ考え方です。

---

## ⚠️ ハマりどころ

- **返事が来るまで画面が固まって見える**：REST は「全部できてから届く」ので、長い返事ほど待ち時間が目立ちます（RESTの弱点）。今回は「考え中…」表示でしのぎます。**1文字ずつパラパラ出す**演出は第11章（SSE）で。
- **model 名のミス**：`gpt-4o-mini` のつづり間違いや、存在しないモデル名を書くと、APIがエラーを返します。まず正しいモデル名か確認を（付録D）。
- **エラー処理を入れていない**：通信・API は失敗します。`try/catch` で囲み、画面にも「失敗しました」と出す（無言で固まらせない）。
- **system を入れ忘れる**：AIの **キャラがぶれます**（毎回ちがう口調・前提で答える）。困ったら、まず system を見直す。
- 🛡 **`textContent` を使う（`innerHTML` を避ける）**：AIの返事をそのまま `innerHTML` に入れると、変な文字列が **そのままページの一部として動いてしまう**危険（XSS）があります。文字として表示する `textContent` が安全です。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「**TypeScript + Express** で `/api/chat` を作って。`messages` は **`{ role:"system", content:... }` と `{ role:"user", content:... }`** の1往復にして、**それぞれの役割をコメントで説明**して。**OpenAIのAPIキーは `process.env` から読み、フロントには出さない**で。ブラウザ側は **素のTS/JS** で `fetch("/api/chat")` を叩き、返事は **`textContent`** で表示して」

出てきたコードの確認ポイント：

- [ ] 鍵が **`process.env` 経由**か（コード直書き・フロント埋め込みでないか）
- [ ] ブラウザは **`/api/chat`（自分のサーバー）** を叩いているか（`api.openai.com` を直接でないか）
- [ ] `messages` の **先頭が system**、次が user になっているか
- [ ] 返事の表示が **`textContent`**（`innerHTML` でない）か
- [ ] **`try/catch`** とエラー時の返事（500など）があるか

---

## 📝 ことばメモ

- **messages**：LLMに渡す会話の配列（順番付きリスト）。中身は `{ role, content }`
- **role（役割）**：発言の主を表す札。`system`（AIの前提・キャラ）/ `user`（人間）/ `assistant`（AIの返事）
- **content**：その発言の本文（文章）
- **REST**：「送る→できるまで待つ→全部いっぺんに届く」方式の通信。今回の `/api/chat` がこれ
- **XSS**：他人が仕込んだ文字列がページ上で勝手に動く攻撃。`textContent` で表示すると防ぎやすい

---

## ➡️ 次の章へ

これで「送って待つ」1往復ができました。でも今のままだと——**「さっきの話」が通じません**。続けて「で？」と送っても、AIは前の発言を覚えていないのです。

次の第4章では、その正体 **「LLMはステートレス＝毎回忘れる」** を知り、**会話の記憶を作るのは開発者の仕事** だと学びます。ここから本格的に **背骨①（🧠記憶）** の話が始まります。

[→ 第4章 LLMは毎回忘れる（ステートレス）へ](04-stateless.md)

[← 目次・はじめにへもどる](README.md)
