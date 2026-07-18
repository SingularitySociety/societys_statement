---
title: "第10章　使う側のしくみ — 繋ぐ・一覧をもらう・AIに渡す"
parent: "MCPサーバーを作って学ぶ AIに道具を持たせる入門"
grand_parent: "開発の心得"
nav_order: 10
---

# 第10章　使う側のしくみ — 繋ぐ・一覧をもらう・AIに渡す

> 📖 MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 ← 目次に戻る
> **ここから第2部です。** 第9章までで作ったのは「道具を差し出す側（サーバー）」でした。この章からは、これまで Claude Desktop が代わりにやってくれていた**使う側（クライアント）を、自分で書きます**。

---

## 📱 Claude ではこう見える 🟢

第2章から第9章まで、あなたは `memo` サーバーを作り、Claude Desktop の設定ファイルにパスを書いて、再起動して、動かしてきました。

そのあいだ、**Claude Desktop の中では、あなたが一度も書いていない仕事**が動いていました。

- あなたのサーバーのプログラムを**起動する**
- 標準入出力で**繋ぐ**
- 「どんな道具があるの？」と**一覧を聞く**
- AIが「`search_notes` を使いたい」と言ったら**代わりに呼んで、結果を返す**

この4つです。無料でやってもらっていた、と言ってもいい。

**この章では、この4つを自分の手で書きます。** 書き上がると、Claude Desktop なしで、ターミナルから自分のメモサーバーを叩けるようになります。

---

## 🤔 なぜ自分でクライアントを書くのか 🟢

### 第1章の3人の図を思い出してください

```
┌─────────────────────────────────────────────┐
│  ホスト（＝あなたが作るアプリ）  ←★第2部の主役 │
│                                             │
│   ┌──────────────────┐                      │
│   │ クライアント       │ ←─ サーバー1つにつき1人 │
│   └────────┬─────────┘                      │
└────────────┼────────────────────────────────┘
             │  ← ここでMCPの会話が行われる
             ↓
   ┌────────────────────┐
   │ サーバー            │ ＝第9章までに作ったもの
   │ （memo）            │
   └────────────────────┘
```

第1章で「クライアントは**ホストの中にいる通訳係**」と説明しました。第1部では、そのホストが Claude Desktop でした。

第2部では、**ホストがあなたのアプリ**になります。そして通訳係（クライアント）も、あなたが用意します。

### やらないとどうなる？

やらなくても、Claude Desktop では動きます。困りません。

でも、**背骨①「一度作れば、どのAIからでも使える」を確かめられないまま終わります**。

規格のありがたみは、「同じサーバーが、別のホストからも動いた」という体験でしか分かりません。この章と第11章は、そのための章です。

> 💡 ついでに言うと、**世の中の "MCP対応アプリ" が中で何をしているか**が分かります。Cursor も Claude Code も、ここでやることを（もっと丁寧に）やっているだけです。

---

## 🛠 こう作る 🟢

### ①　まず、サーバーをビルドしておく

クライアントが起動するのは **`build/index.js`**（コンパイル後のファイル）です。`src/index.ts` ではありません。

```bash
cd /絶対パス/memo
npm run build
```

これを忘れると、この章はまるごと動きません。**先にやってください。**

### ②　クライアント用のプロジェクトを作る

サーバーとは**別のフォルダ**を作ります。混ぜないでください。

```bash
mkdir memo-client
cd memo-client
npm init -y
npm install @modelcontextprotocol/sdk
npm install -D @types/node typescript
mkdir src
```

`package.json` に、サーバーのときと同じ要領で2行を足します。

```json
{
  "type": "module",
  "scripts": { "build": "tsc" }
}
```

`tsconfig.json` は**サーバーと同じもの**でOKです（第5章のものをコピーしてください）。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### ③　読み込むもの

`src/index.ts` を作ります。まず、上から2行。

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
```

サーバーのときは `server/mcp.js` と `server/stdio.js` でした。**今回は `client/` の下です。** 同じSDKの、反対側の部品です。

- **`Client`** … MCPの会話をする本体。サーバーの `McpServer` に対応する相手役
- **`StdioClientTransport`** … 標準入出力で話すための「線」。**サーバーのプログラムを起動するのも、こいつの仕事**

> ⚠️ `.js` 拡張子を落とさないでください。第5章と同じ理由（Node16 のモジュール解決）です。

### ④　クライアントを1つ作る

```typescript
const mcp = new Client({ name: "my-client", version: "1.0.0" });
```

`name` と `version` は**自己紹介**です。サーバー側が「どのアプリから繋がれているか」を知るために使います。好きな名前でかまいません。

### ⑤　サーバーを起動して、繋ぐ

ここが「①サーバーを起動する」と「②繋ぐ」にあたります。

```typescript
const transport = new StdioClientTransport({
  command: "node",
  args: ["/絶対パス/memo/build/index.js"],
});
await mcp.connect(transport);
```

たった4行ですが、中では**プログラムが1つ起動しています**。

- `command: "node"` … 「node コマンドで動かして」
- `args: [...]` … 「このファイルを渡して」

つまりこれは、あなたがターミナルで `node /絶対パス/memo/build/index.js` と打つのと同じことを、プログラムからやっています。**Claude Desktop の設定ファイルに書いていた `command` と `args` が、そのままコードになった**と思ってください。

```json
"memo": {
  "command": "node",
  "args": ["/ABSOLUTE/PATH/TO/memo/build/index.js"]
}
```

↑これ（第2章の設定ファイル）と、↑↑これ（さっきのコード）は、**同じもの**です。設定ファイルとは、要するに「クライアントに渡すパラメータを書いた紙」だったわけです。

`await mcp.connect(transport)` で、起動 → 挨拶（初期化のやりとり）まで済みます。

### ⑥　道具の一覧をもらう

「③ツール一覧をもらう」です。

```typescript
const toolsResult = await mcp.listTools();
```

`toolsResult.tools` が配列で返ってきます。1つ1つに、こういう中身が入っています。

| 中身 | 何が入っているか |
|---|---|
| `tool.name` | `"search_notes"` |
| `tool.description` | `"ユーザーのローカルのメモ（.mdファイル）を全文検索する。ユーザーが自分の過去のメモ・議事録・日記・書きかけの文章について尋ねたときに使う。ヒットしたファイル名と、その前後の抜粋を返す。"` |
| `tool.inputSchema` | 引数の形（`query` は文字列、など） |

**見覚えがありますよね。** 第5章であなたが `registerTool` に書いた、あの3つです。あれが線の向こうから、そのまま返ってきています。

画面に出してみましょう。

```typescript
console.log("=== 使える道具 ===");
for (const tool of toolsResult.tools) {
  console.log("-", tool.name, ":", tool.description);
}
```

> 💡 **ここで `console.log` を使っていることに注目してください。** サーバー側では絶対禁止だった `console.log` が、**クライアント側では普通に使えます**。理由は後の「ハマりどころ」で説明します。

### ⑦　道具を呼んでみる

「④callTool して結果を返す」の、呼ぶところです。**この章ではAIを繋がないので、自分で決め打ちで呼びます。**

```typescript
const result = await mcp.callTool({
  name: "search_notes",
  arguments: { query: "会議" },
});

console.log("=== search_notes の結果 ===");
console.log(result.content);
```

- `name` … 呼びたい道具の名前（`listTools()` で返ってきたもの）
- `arguments` … 引数。**`inputSchema` が要求している形**で渡します

返ってくる `result.content` は、第5章でサーバーが返していた `{ content: [{ type: "text", text: "…" }] }` の `content` です。**サーバーで書いた戻り値が、そのまま届いています。**

### ⑧　全体（コピペで動く最小クライアント）

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const mcp = new Client({ name: "my-client", version: "1.0.0" });

async function main() {
  const transport = new StdioClientTransport({
    command: "node",
    args: ["/絶対パス/memo/build/index.js"],
  });
  await mcp.connect(transport);

  const toolsResult = await mcp.listTools();
  console.log("=== 使える道具 ===");
  for (const tool of toolsResult.tools) {
    console.log("-", tool.name, ":", tool.description);
  }

  const result = await mcp.callTool({
    name: "search_notes",
    arguments: { query: "会議" },
  });
  console.log("=== search_notes の結果 ===");
  console.log(result.content);

  await mcp.close();
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

最後の `await mcp.close()` は**後片付け**です。これを呼ぶと、裏で起動していたサーバーのプログラムも終わります。忘れるとターミナルが返ってこないことがあります。

動かします。

```bash
npm run build
node build/index.js
```

こう出れば成功です。

```
=== 使える道具 ===
- search_notes : ユーザーのローカルのメモ（.mdファイル）を全文検索する。ユーザーが自分の過去のメモ・議事録・日記・書きかけの文章について尋ねたときに使う。ヒットしたファイル名と、その前後の抜粋を返す。
- read_note : メモを1つ選んで、その全文を返す。search_notes で見つけたファイル名を渡す
- write_note : 新しいメモを保存する。すでに同じ名前のメモがある場合は上書きされる
=== search_notes の結果 ===
[ { type: 'text', text: '…' } ]
```

**Claude Desktop を1回も開かずに、自分のMCPサーバーを叩けました。** これが第2部の第一歩です。

---

## 🔑 この章の山場：LLMに渡す形に「変換」する 🟢

さて、ここからが本題です。

`listTools()` でもらったツール定義を、**そのままLLMのAPIに渡せると思いますか？**

**渡せません。**

### MCPの形と、LLMのAPIの形は別物

比べてみてください。

**MCPが返してくる形**（`listTools()` の中身）

```
name        : "search_notes"
description : "ユーザーのローカルのメモ（.mdファイル）を全文検索する…"
inputSchema : { … }        ← キー名は inputSchema
```

**LLMのAPI（Anthropic）が要求する形**

```
name         : "search_notes"
description  : "ユーザーのローカルのメモ（.mdファイル）を全文検索する…"
input_schema : { … }        ← キー名は input_schema
```

……**`inputSchema` が `input_schema` になっているだけ**です。拍子抜けするくらい、ほとんど同じ。

でも「ほとんど同じ」は「同じ」ではありません。だから、こう変換します。

```typescript
const tools = toolsResult.tools.map((tool) => {
  return {
    name: tool.name,
    description: tool.description,
    input_schema: tool.inputSchema,
  };
});
```

この `tools` を、LLMのAPIに `tools:` として渡します（実際に渡すのは第11章です）。

### なぜこれが山場なのか 🟢

この3行のためだけに節を立てているのは、**ここに MCP の正体が出ているから**です。

> **MCPは「AIと道具の共通語」であって、「LLM各社のAPIの形」ではありません。**

MCPは、**道具を差し出す側**（サーバー）と**道具を集める側**（クライアント）のあいだの規格です。集めた道具をどのLLMに、どういうJSONで渡すかは、**MCPの管轄外**です。

- Anthropic のAPIは `input_schema` というキーを使います
- OpenAI のAPIは別のキー名・別の入れ子で要求します（第11章で実際に書きます）
- 将来また別のAPIが出てきても、変わるのは**この変換の3行だけ**です

だからクライアントの仕事は、こう整理できます。

```
MCPサーバー（何個でも）→ [ 共通語で受け取る ] → [ 変換 ] → 使いたいLLMのAPI
                                                 ↑
                                       ここだけがLLM各社に依存する
```

### 第3弾との対比：手書きしなくてよくなった 🟢

第3弾（自作CLIエージェント）の第3章を思い出してください。あそこでは、AIに渡すツール定義を**自分で1個ずつ手書き**していました。名前を書き、説明を書き、引数のスキーマを書いて、配列に並べて。

道具を1つ足すたびに、**エージェント側のコードを書き換える**必要がありました。

いまはどうでしょう。

```typescript
const toolsResult = await mcp.listTools();
```

**この1行で、道具の一覧が勝手に入ってきます。**

つまり——**サーバーに `write_note` を足してビルドし直すだけで、クライアントは1文字も直さずに新しい道具を使えるようになります。** クライアントは、どんな道具が来るかを事前に知らなくていい。

これが「規格で作る」ことの、いちばん具体的な見返りです。第1章で書いた N×M 問題の話が、ここでコードになりました。

---

## 📋 クライアントがやっていること4段階 🟢

この章のまとめです。**クライアントの仕事は、たった4つ**です。

| # | 仕事 | コード |
|---|---|---|
| ① | **サーバーを起動する** | `new StdioClientTransport({ command, args })` |
| ② | **繋ぐ** | `await mcp.connect(transport)` |
| ③ | **ツール一覧をもらう** | `await mcp.listTools()` → LLMの形に変換 |
| ④ | **AIが呼びたいと言ったら呼んで、結果を返す** | `await mcp.callTool({ name, arguments })` |

この章では、①②③と、④の「呼ぶ」ところだけを、AI抜きでやりました。

**残っているのは④の「AIが呼びたいと言ったら」の部分だけ**です。そこを埋めるのが第11章です。

---

## ⚠️ ハマりどころ 🟢

### ①　パスは絶対パスで

```typescript
args: ["../memo/build/index.js"]   // ❌ たいてい動きません
args: ["/Users/you/memo/build/index.js"]  // ✅
```

相対パスは「どこから見た相対か」が状況によって変わります。第2章の設定ファイルで絶対パスを要求されたのと**まったく同じ理由**です。

パスが分からなくなったら、サーバーのフォルダで確認できます。

```bash
cd /絶対パス/memo
pwd
```

出てきた文字列に `/build/index.js` を足したものが答えです。

### ②　サーバーをビルドし忘れる（いちばん多い）

エラーはこう出ます。

```
Error: Cannot find module '/絶対パス/memo/build/index.js'
```

`build/index.js` が**存在しない**のです。原因は9割これ。

```bash
cd /絶対パス/memo
npm run build
```

**サーバーのコードを直したのに、クライアントの挙動が変わらない**ときも、これを疑ってください。クライアントが動かしているのは古い `build/index.js` のままです。「直したのに変わらない＝ビルドし忘れ」は、この教材でずっと付いて回ります。

### ③　クライアントでは `console.log` を使ってよい 🟢

第3章であれだけ「`console.log` は絶対に書くな」と言ったのに、この章では普通に使っています。矛盾ではありません。

理由はシンプルです。

> **`console.log` が禁止だったのは「stdout（標準出力）が、MCPの会話チャンネルになっている」プログラムだけ**です。

- **サーバー**は、自分の stdout に**MCPの返事を書いている** → そこに文字を混ぜたら会話が壊れる → **禁止**
- **クライアント**の stdout は、**ただのターミナル**です。誰も会話に使っていない → **自由**

クライアントがサーバーと話しているのは、**起動した子プロセスの stdin / stdout** であって、クライアント自身の画面ではありません。**口が別**なのです。

覚え方はこれで十分です。

> **stdio サーバー ＝ 画面が会話チャンネル（黙る）／クライアント ＝ 画面はただの画面（喋ってよい）**

第8章の HTTP サーバーで `console.log` が問題なかったのも、同じ理屈でした（会話チャンネルが HTTP のほうに移っているから）。

### ④　「繋がったのにツールが0個」

`listTools()` が空の配列を返すときは、**サーバー側**を疑います。

1. ターミナルで直接 `node /絶対パス/memo/build/index.js` を実行してみる（第4章の切り分け手順①）
2. `npx @modelcontextprotocol/inspector node /絶対パス/memo/build/index.js` で見てみる（同②）

Inspector の Tools タブに出ないなら、原因はクライアントではなくサーバーです。**第4章に戻ってください。**

### ⑤　メモが見つからない（`NOTES_DIR`）

`search_notes` が「1件も見つかりません」と返すときは、サーバーが見ているフォルダを確認します。

第5章でサーバーには**既定のメモ置き場**を持たせてあるので、クライアントから何も渡さなくても動くはずです。それでも空なら、そのフォルダに `.md` ファイルがあるか確かめてください（対象は `.md` だけです）。

> 🔧 別のフォルダを見せたい場合、クライアントからサーバーに環境変数を渡す方法があります。ただし**扱いがSDKの版で変わりやすい**ところなので、必要になったら公式ドキュメント（[modelcontextprotocol.io](https://modelcontextprotocol.io/docs/develop/build-client)）で現行の書き方を確認してください。手っ取り早いのは、サーバー側の既定値を書き換えてビルドし直すことです。

### ⑥　プログラムが終わらない

`await mcp.close()` を忘れると、起動したサーバーが生き残ってターミナルが返ってこないことがあります。最後に必ず呼んでください。止まらなくなったら `Ctrl + C` で大丈夫です。

---

## 🤖 AIに頼むなら 🟢

AIに「MCPクライアントを書いて」と頼むときのコツです。

**言うべきこと**

> 「`@modelcontextprotocol/sdk` の **`Client` と `StdioClientTransport`** を使って、TypeScript で最小のMCPクライアントを書いて。**LLMはまだ繋がなくていい**。`listTools()` の結果を表示して、`search_notes` を1回呼んで終わるだけ」

**「LLMはまだ繋がなくていい」を必ず付けてください。** これを言わないと、AIは親切心で OpenAI や Anthropic のAPI呼び出し、対話ループ、エラーハンドリングまで全部盛りで書いてきます。**動かないときに、どこが悪いか分からなくなります。**

**気をつけること**

- MCPは動きが速い分野です。AIは**古い書き方**（廃止された繋ぎ方や、昔のクラス名）を自信満々に出してきます。「公式ドキュメント（modelcontextprotocol.io）の最新の書き方で」と添えてください
- **絶対パスは自分で入れる。** AIはパスを想像で埋めます。`pwd` で調べた本物に置き換えてください
- **サーバー側とクライアント側の import を混ぜていないか**を目視で確認。`server/stdio.js` と `client/stdio.js` は名前が似ていて、AIもたまに取り違えます

---

## 📗 ことばメモ

| ことば | よみ | 意味 |
|---|---|---|
| **クライアント** | — | ホストの中にいる、サーバーとの通訳係。この章から**あなたが作る**もの |
| **`Client`** | クライアント | SDKのクラス。MCPの会話をする本体。サーバー側の `McpServer` の相手役 |
| **`StdioClientTransport`** | — | 標準入出力でサーバーと話すための「線」。**サーバーのプログラムを起動するのもこれ** |
| **`listTools()`** | リストツールズ | サーバーに「どんな道具がある？」と聞く。中身は `tools/list` |
| **`callTool()`** | コールツール | 道具を実際に呼ぶ。中身は `tools/call` |
| **`inputSchema`** | インプットスキーマ | 引数の形。**MCP側のキー名**（キャメルケース） |
| **`input_schema`** | インプットスキーマ | 同じものの、**Anthropic API側のキー名**（スネークケース）。ここを変換する |
| **変換（マッピング）** | — | MCPの形を、使いたいLLMのAPIの形に詰め替えること。**LLM各社に依存するのはここだけ** |

---

## ➡️ 次へ

クライアントの骨格はできました。サーバーを起動し、繋ぎ、道具の一覧をもらい、呼べる。そして一覧を**LLMが読める形に変換する**ところまで来ました。

でも、まだ**AIがいません**。`search_notes` を「会議」で呼ぶと決めているのは、あなたが書いた決め打ちのコードです。

[第11章　自作ChatGPTクローンに繋ぐ](11-connect-gptclone.md) では、ここで作った変換済みの `tools` を実際にLLMへ渡し、**AIが「この道具を使いたい」と言ってきたら `callTool()` して結果を返す**——④の残り半分を埋めます。繋ぎ先は第2弾で作った ChatGPT クローンです。

そのとき、**あなたが第5章で書いたメモサーバーが、Claude Desktop でも、自分のアプリでも、まったく同じコードのまま動く**ところを見ることになります。背骨①の答え合わせです。

## 関連ページ

- MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 — 目次
- ChatGPTクローンで学ぶ LLMアプリ開発入門 — はじめに・目次 — 第2弾。第11章で繋ぐ相手がこれです
- 自作CLIエージェントで学ぶ AIエージェント開発入門 — はじめに・目次 — 第3弾。ツール定義を**手書きしていた**のがこちら
