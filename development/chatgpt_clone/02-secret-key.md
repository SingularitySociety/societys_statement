---
title: "第2章　【守り①】秘密のAPIキーを守る"
parent: "ChatGPTクローンで学ぶ LLMアプリ開発入門"
grand_parent: "開発の心得"
nav_order: 2
nav_exclude: true
---

# 第2章　【守り①】秘密のAPIキーを守る

> 📖 この章のゴール：**APIキーをブラウザに出さず、自分のサーバーに隠して使える**ようになる。なぜ「直書き」が危険かを自分の言葉で説明できる。
> [← 目次・はじめにへもどる](README.md)

---

## 📱 ChatGPTではこう見える

ChatGPTを使うとき、あなたは **OpenAIのAPIキーなんて入力しません**。ただログインするだけ。鍵は **OpenAI側のサーバーが持っていて**、あなたは「使う権利があるか」をログインで確認されているだけです。

自分でChatGPTクローンを作るときも、考え方は同じ。

> **鍵は「お店（自分のサーバー）」が預かり、お客さん（ブラウザ）には絶対に渡さない。**

第1章で出てきた「なぜ自分のサーバーを挟むのか」の **理由①（鍵を隠す）** を、この章で実際に作ります。

---

## 🤔 なぜ鍵を隠す？やらないとどうなる（🟢 基礎）

- **APIキー**＝LLMを使うための**秘密の合鍵**。料金は **鍵の持ち主（あなた）に請求**されます。
- **ブラウザのコードは、誰でも中身を見られます。** ページを右クリック→「ソースを表示」や開発者ツールで丸見え。だから **ブラウザのコードに鍵を書く＝全世界に鍵を配る** のと同じです。
- 公開された鍵は、**数分で自動収集のbotに拾われ**、悪用され、**高額請求**につながります（実話がたくさんあります）。

> ⚠️ **やってはいけない例**（鍵がページに含まれる＝アウト）
> ```ts
> // ❌ ブラウザから直接OpenAIを叩く。sk-... がページに丸見え
> fetch("https://api.openai.com/v1/chat/completions", {
>   headers: { Authorization: "Bearer sk-xxxxxxxx" },  // ← 漏れる！
> });
> ```

> 🔧 **応用：「環境変数にすれば安全」だけでは不十分**
> フロントのビルド道具（Vite / Next.js など）の環境変数には、ビルド時に値が**埋め込まれて公開**されるものがあります（`VITE_` や `NEXT_PUBLIC_` で始まる名前）。鍵は **そういう接頭辞を付けず、サーバーのプロセスだけが読む** のが鉄則です。

---

## 🛠 こう作る — 鍵をサーバーに隠す3ステップ（🟢 基礎）

### ステップ1：鍵を `.env` に置き、gitに上げない

```bash
# .env  ← このファイルは絶対にgitに入れない
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
```

**1行ずつ読むと：**
- `OPENAI_API_KEY=sk-...`：OpenAIの管理画面で発行した鍵を、`OPENAI_API_KEY` という名前で置く。`sk-` で始まる長い文字列です（取り方は付録A）。

```bash
# .gitignore
.env
node_modules
```

**1行ずつ読むと：**
- `.env`：鍵入りファイルを **git管理から外す**（公開リポジトリに鍵を上げる事故を防ぐ）。
- `node_modules`：これはついで（部品フォルダはgitに入れない）。

### ステップ2：サーバー（Express）が鍵を読み、LLMを代理で呼ぶ

```ts
// server.ts  （自分のサーバー＝鍵を知っているのはここだけ）
import express from "express";
import OpenAI from "openai";

const app = express();
app.use(express.json());

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

app.post("/api/chat", async (req, res) => {
  const { message } = req.body;
  const completion = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: message }],
  });
  res.json({ reply: completion.choices[0].message.content });
});

app.listen(3000);
```

**1行ずつ読むと：**
- `import express from "express"`：自分のサーバーを作る道具を読み込む。
- `import OpenAI from "openai"`：OpenAIを呼ぶ公式の道具を読み込む。
- `app.use(express.json())`：ブラウザから届くJSON（`{ message: ... }`）を読めるようにする。
- `new OpenAI({ apiKey: process.env.OPENAI_API_KEY })`：**鍵は `.env` から読む**。コードに直書きしない。`process.env` は **サーバーのプロセスだけ**が見られる場所。
- `app.post("/api/chat", ...)`：ブラウザはこの **“自分の窓口”** を叩く（OpenAIを直接ではなく）。
- `const { message } = req.body`：ブラウザが送ってきた文章を取り出す。
- `openai.chat.completions.create({ ... })`：サーバーが **代理で** OpenAIに頼む。鍵はこの内部で自動的に付く。
- `model: "gpt-4o-mini"`：使うモデル（賢さと値段の選択。詳しくは付録D）。
- `messages: [{ role: "user", content: message }]`：LLMへ渡す会話。いまは1発言だけ（会話の積み上げは第4章）。
- `res.json({ reply: completion.choices[0].message.content })`：返事の **文章だけ** をブラウザに返す。**鍵は返さない**。

### ステップ3：ブラウザ（画面）は「自分の窓口」を叩く

```ts
// 送信時：OpenAIではなく、自分のサーバーの窓口へ送る（画面ライブラリは何でもOK）
async function send(text: string): Promise<string> {
  const res = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: text }),
  });
  const data = await res.json();
  return data.reply; // この文字列を画面に表示するだけ（React / Vue / 素のDOM どれでも）
}
```

**1行ずつ読むと：**
- `fetch("/api/chat", ...)`：**自分のサーバー**の窓口へ送る。`api.openai.com` ではない＝ブラウザは鍵を持たない。
- `body: JSON.stringify({ message: text })`：入力した文章を送る。
- `const data = await res.json()`：返ってきた `{ reply: ... }` を受け取る。
- `return data.reply`：返事の文章を返す。あとは画面に表示するだけ（React なら state、素の JS なら `el.textContent` に入れる）。

> ✅ **正解のしるし**：ブラウザ側（画面）のコードに、**`sk-` で始まる鍵が一切登場しない**。これが守れていれば合格です。

> 🖥 **まずは手元（localhost）で動かそう — そしてログインは後回しでOK**
> 仕組みが十分に分かるまでは、Express を **自分のパソコンの中（localhost）** で動かすのがおすすめです。**デプロイ（公開）はまだ不要**。
>
> ⚠️ ただし初心者がつまずくところ：**サーバー（Express）は“見て楽しむページ”ではありません。** これは第1章の「キッチン（注文を処理する裏方）」。だからブラウザで `http://localhost:3000` を **そのまま開くと、`Cannot GET /` のような素っ気ない表示**になることがあります——**これは故障ではなく正常**です（窓口 `POST /api/chat` は注文を待っているだけで、表に見せるトップページを持っていないから）。
>
> あなたが実際に「開いて触る」のは **画面（チャット画面＝お客さんの席）** の方。いちばん簡単なのは、**その画面（HTML 1枚）も Express から一緒に配ってしまう**ことです。`server.ts` に **`app.use(express.static("public"));`** を1行足すと、`public` フォルダの `index.html` がそのまま配られ、**画面（`/`）も窓口（`/api/chat`）も同じサーバーから出ます**。これで `http://localhost:3000` を開けばチャット画面が出て、その画面が裏で `/api/chat` を叩きます。
>
> 「ちゃんと起動したか」は、**ターミナル（黒い画面）に出るログ**で確かめます（`app.listen` の所に `console.log("起動 → http://localhost:3000")` を置くと安心）。
>
> そして大事なポイント：**手元で動かしているうちは、外から誰も触れないので、ログイン（認証）も使用量制限も、まだ要りません。** まずは「**鍵をサーバーに隠して、LLMを呼べる**」——この一点だけに集中しましょう。
>
> 🔑 **ログインや使用量制限が必要になるのは、インターネットに公開して“自分以外の人”が使えるようになってから**（第6章／本番）。つまり背骨②のうち、**ローカルでは「鍵」だけ**を押さえればよく、**「財布（使用量制限）」は公開するときに**足せばOK、という順番です。

---

## ⚠️ ハマりどころ

- **フロントの環境変数（`VITE_` / `NEXT_PUBLIC_`）に鍵を入れる** → ビルドに埋め込まれて公開されます。鍵は **接頭辞なし**で、サーバーだけが読む。
- **`.env` をうっかりコミット** → 公開リポジトリなら即アウト。先に `.gitignore`。万一上げてしまったら、**履歴から消すだけでなく、鍵を無効化して再発行**（一度ネットに出た鍵は「漏れた」とみなす）。
- **CORSやプロキシを「とりあえず全部許可」** → 別の穴になります。必要な範囲だけに絞る。
- 🔧 **サーバーのログにリクエストを丸ごと出す** → 会話や秘密が残ることがあります。ログに秘密を出さない（第6章・付録G）。

---

## 🎁 おまけ：本番ではどこに鍵を置く？（🔧 応用）

開発中は `.env` でOK。でも **本番（公開）にデプロイするとき**は、`.env` ファイルごとアップロードするのではなく、**使うサービスの「秘密の置き場（Secrets／環境変数）」に登録**します。考え方はどこでも同じ——**鍵はサービスのenvに置き、サーバー側のコードが実行時に `process.env`（相当）で読む。クライアントには出さない。**

| サービス | 鍵の置き方 | コードから読む |
|---|---|---|
| **Vercel** | ダッシュボード → Settings → **Environment Variables** に `OPENAI_API_KEY` を追加（`vercel env add` でも可）。**`NEXT_PUBLIC_` は付けない** | サーバー関数内で `process.env.OPENAI_API_KEY` |
| **Supabase**（Edge Functions） | `supabase secrets set OPENAI_API_KEY=sk-...` | `Deno.env.get("OPENAI_API_KEY")` |
| **Firebase**（Cloud Functions） | `firebase functions:secrets:set OPENAI_API_KEY`（Secret Manager に保存） | 関数に `secrets: ["OPENAI_API_KEY"]` を紐づけて `process.env.OPENAI_API_KEY` |

> 💡 **サーバーレスでも同じ**：Vercel / Supabase / Firebase では、Express の代わりに **「サーバーレス関数」**（リクエストが来たときだけ動く小さな関数）で `/api/chat` を作ることが多いです。器（関数）は変わっても、**「鍵は env から読む・クライアントには出さない」** は1ミリも変わりません。

> ⚠️ **超重要：その `apiKey`、実は“公開してよい鍵”かも**（混同に注意）
> サービスによっては、**ブラウザに置くのが前提の `apiKey`** があります。これは **OpenAIの秘密キー（`sk-...`）とは別物** です。
> - **Firebase**：`firebaseConfig` の中の `apiKey` は **公開してOK**（ただのプロジェクト識別子。守りは Security Rules が担当）。
> - **Supabase**：`anon`（公開）キーはブラウザに置いてよい／`service_role`（全権限）キーは **秘密**（第1弾でも登場）。
>
> 見分け方はシンプル：**「これがバレたら、自分の財布で勝手に使われる・全データを触られる」鍵だけが“秘密”**。OpenAIの `sk-...` は完全に秘密側なので、**必ずサーバーの env に隠します**。

---

## 🤖 AIに頼むなら（Vibe codingのコツ）

> 🗣 プロンプト例：
> 「TypeScript + Express で `/api/chat` を作って。**OpenAIのAPIキーは `.env` から `process.env` で読み、フロントには絶対に出さない**で。`.gitignore` に `.env` を入れて。ブラウザからは `/api/chat` を叩くようにして」

出てきたコードの確認ポイント：

- [ ] 鍵が **`process.env` 経由**か（コード直書き・フロント埋め込みでないか）
- [ ] ブラウザは **`/api/...`（自分のサーバー）** を叩いているか（`api.openai.com` を直接でないか）
- [ ] `.gitignore` に **`.env`** が入っているか

---

## 📝 ことばメモ

- **APIキー**：LLMを使うための秘密の合鍵。料金は持ち主に請求される
- **環境変数（process.env）**：プログラムの外から渡す設定値。鍵などの秘密を置く場所
- **.env**：環境変数をまとめたファイル。gitに上げない
- **.gitignore**：gitで管理しないファイルの一覧
- **Secrets（本番の鍵置き場）**：Vercel / Supabase / Firebase などが用意する秘密の保管庫。本番では `.env` の代わりにここへ登録する
- **プロキシ**：あいだに立って代理で通信する役。ここでは自分のサーバー
- **gpt-4o-mini**：OpenAIのモデルの一つ（軽くて安い）。モデルと料金は付録D

---

## ➡️ 次の章へ

第3章では、いよいよ **はじめての会話（REST編）** を作ります。いまは1発言を送っただけ。次は **`messages` の形（system / user / assistant）** と、「送って待つ」往復をきちんと作ります。ここから **背骨①（記憶）** の話が始まります。

[← 目次・はじめにへもどる](README.md)
