---
title: "第5章　自分の道具を作りはじめる — メモを検索する"
parent: "MCPサーバーを作って学ぶ AIに道具を持たせる入門"
grand_parent: "開発の心得"
nav_order: 5
---

# 第5章　自分の道具を作りはじめる — メモを検索する

> 📖 MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 ← 目次に戻る
> ここまでは**他人が作ったサーバー**を動かし、壊し、直してきました。この章から**自分のためのサーバー**を作ります。作るのはツール1つだけ。そのかわり、この章の後半で「**説明文を書き換えると、AIの動きが変わる**」という、この教材でいちばん意外な実験をします。

---

## 📱 Claude ではこう見える 🟢

### 第2章の天気サーバーは、あなたの役に立たなかった

第2章で動かした天気サーバーは、たしかに動きました。でも「**東京の天気は？**」と聞いても答えは返ってこなかったはずです。あれは米国気象局のAPIを使う、**米国専用**の道具だったからです。

そして大事なのは、Claude が悪かったわけでも、あなたの書き方が悪かったわけでもない、ということです。

> **道具ができないことは、AIにもできません。**

つまり——**自分の役に立つ道具は、自分で作るしかない**。この章はそこから始まります。

### この章のゴール

こうなります。

```
あなた： 先週の打ち合わせのメモある？

Claude： メモを検索しますね。
        [🔨 search_notes を使用]

        3件見つかりました。

        ・2026-07-10-打ち合わせ.md
          「…先週の打ち合わせでは、次のリリース日を…」
        ・weekly.md
          「…打ち合わせの前に資料を…」
        …
```

あなたのパソコンの中にある `.md` ファイル（メモ）を、Claude が**自分で探しに行ってくれる**状態です。アップロードもコピペも要りません。

> 💡 **メモの置き場所は、あなたが決めます。** Obsidian のフォルダでも、`~/notes` でも、`~/Documents/memo` でも構いません。`.md` ファイルが入っているフォルダなら何でもOKです。

---

## 🤔 なぜ「1つだけ」なのか 🟢

これから作るツールは、たった1つです。

| ツール名 | 引数 | やること |
|---|---|---|
| `search_notes` | `query`（文字列） | メモを全文検索して、ファイル名と抜粋を返す |

「読む」「書く」も欲しいでしょう。ですが**それは第6章**です。今は1つだけにします。理由は3つあります。

1. **動かない原因が1つに絞れる。** ツールを3つ同時に書くと、動かないときに「どれが悪いのか」が分かりません。第4章でやった切り分けの話と同じです
2. **検索だけでも、じゅうぶん実用的。** 「あのこと、前にメモした気がするんだけど…」を解決できるだけで、道具としてはかなり強いです
3. **この章の本題は、コードではないから。** 本題は後半の「**説明文の実験**」です。コードが複雑だと、そこに集中できません

> ⚠️ **ツール名と引数名は、絶対に変えないでください。** `search_notes` と `query` です。この2つは**第6章から第9章まで、ずっと同じものを育てていきます**。ここで `searchNotes` や `keyword` にすると、後の章のコードと食い違って苦労します。

### やらないとどうなる：メモ置き場を「決め打ち」すると詰む

初心者がやりがちなのが、コードの中に自分のフォルダ名を直接書いてしまうことです。

```typescript
// ❌ これをやると後で困る
const NOTES_DIR = "/Users/satoshi/notes";
```

一見動きます。でも——

- フォルダを移したら、**コードを書き換えてビルドし直し**
- 人に見せられない（あなたのユーザー名が入っている）
- 第9章で「どこまで読ませるか」を絞るとき、**手の入れどころが無い**

なので、**メモ置き場は外から渡します**。これを**環境変数**と言います。この章では `NOTES_DIR` という名前で受け取ります。

---

## 🛠 こう作る 🟢

### ステップ1：`memo` プロジェクトを新しく作る

第2章の `weather` フォルダは**そのまま残しておいてください**。混ぜません。まったく別のプロジェクトとして作ります。

```bash
mkdir memo
cd memo
npm init -y
mkdir src
```

`npm init -y` は「とりあえず標準的な `package.json` を作って」という意味です。`-y` は全部の質問に「はい」と答える指定です。

### ステップ2：必要な部品を入れる

```bash
npm install @modelcontextprotocol/sdk zod@3
npm install -D @types/node typescript
```

- `@modelcontextprotocol/sdk` … MCPサーバーを作るための公式の部品
- `zod@3` … 「この引数は文字列です」と型を宣言するための道具
- `-D` が付いた2つ … **開発中だけ**使うもの（TypeScript本体と、Nodeの型情報）

> ⚠️ **`zod@3` の `@3` を落とさないでください。** バージョン4は書き方が変わっており、この教材のコードがそのままでは動きません。`npm install zod` とだけ打つと最新版（v4系）が入ってしまいます。

### ステップ3：`package.json` を整える

`npm init -y` が作った `package.json` を開いて、次の3つを足します（他の行は残したままでOK）。

```json
{
  "type": "module",
  "scripts": { "build": "tsc && chmod 755 build/index.js" },
  "files": ["build"]
}
```

| 行 | 意味 |
|---|---|
| `"type": "module"` | 今どきの `import` の書き方を使う、という宣言 |
| `"scripts"` の `build` | `npm run build` で TypeScript を JavaScript に変換する |
| `"files"` | 配るときに含めるフォルダ（今は気にしなくてOK） |

`chmod 755` は「このファイルを実行してもいいですよ」という許可を付けるコマンドです。Windows では効きませんが、エラーにはならないので気にしないでください。

### ステップ4：`tsconfig.json` を作る

`memo` フォルダの直下に `tsconfig.json` というファイルを新規作成して、これを丸ごと貼ります。

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

とくに大事なのは2つだけです。

- `"outDir": "./build"` … **変換後のファイルが `build/` に出る**。Claude Desktop の設定ファイルが指すのは、この `build/index.js` です
- `"module": "Node16"` … この設定のせいで、`import` の行末に **`.js` を付ける必要**があります（後述）

### ステップ5：メモ置き場を用意する

まだメモが1枚も無い人は、テスト用に作ってしまいましょう。

```bash
mkdir -p ~/notes
echo "# 打ち合わせ 2026-07-10

次のリリース日を7月末に決めた。担当は自分。
資料は来週までに用意する。" > ~/notes/2026-07-10-打ち合わせ.md
```

> ⚠️ **メモが0件だと、この章は何も返ってきません。** 「動いていないのか、ヒットしていないだけなのか」が区別できず、いちばん時間を溶かすパターンです。**先に2〜3枚、日本語のメモを置いてください。**

### ステップ6：`src/index.ts` を書く

これが本体です。まず全体を貼ってから、上から順に読んでいきます。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import fs from "node:fs/promises";
import path from "node:path";

const NOTES_DIR = process.env.NOTES_DIR;

const server = new McpServer({ name: "memo", version: "1.0.0" });

// メモ置き場にある .md ファイルの名前を集める
async function listNoteFiles(dir: string): Promise<string[]> {
  const entries = await fs.readdir(dir, { withFileTypes: true });
  return entries
    .filter((entry) => entry.isFile() && entry.name.endsWith(".md"))
    .map((entry) => entry.name);
}

server.registerTool(
  "search_notes",
  {
    description:
      "ユーザーのローカルのメモ（.mdファイル）を全文検索する。ユーザーが自分の過去のメモ・議事録・日記・書きかけの文章について尋ねたときに使う。ヒットしたファイル名と、その前後の抜粋を返す。",
    inputSchema: {
      query: z
        .string()
        .describe(
          "メモの中から探す検索語。日本語でよい。文章ではなく、1〜2語のキーワードにすること（例: 「打ち合わせ」「リリース日」）",
        ),
    },
  },
  async ({ query }) => {
    if (!NOTES_DIR) {
      return {
        content: [
          {
            type: "text",
            text: "メモ置き場が設定されていません。設定ファイルの env で NOTES_DIR に絶対パスを指定してください。",
          },
        ],
      };
    }

    try {
      const files = await listNoteFiles(NOTES_DIR);
      const needle = query.toLowerCase();
      const hits: string[] = [];

      for (const filename of files) {
        const body = await fs.readFile(path.join(NOTES_DIR, filename), "utf-8");
        const found = body.toLowerCase().indexOf(needle);
        if (found === -1) continue;

        const start = Math.max(0, found - 60);
        const end = Math.min(body.length, found + query.length + 60);
        const excerpt = body.slice(start, end).replace(/\n/g, " ");
        hits.push(`【${filename}】\n…${excerpt}…`);
      }

      if (hits.length === 0) {
        return {
          content: [
            {
              type: "text",
              text: `「${query}」を含むメモは見つかりませんでした。（${files.length}件の.mdを調べました）`,
            },
          ],
        };
      }

      return {
        content: [
          { type: "text", text: `${hits.length}件見つかりました。\n\n${hits.join("\n\n")}` },
        ],
      };
    } catch (error) {
      return {
        content: [
          {
            type: "text",
            text: `メモ置き場を読めませんでした（${NOTES_DIR}）。フォルダが存在するか確認してください。詳細: ${error}`,
          },
        ],
      };
    }
  },
);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Memo MCP Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

長く見えますが、**新しい話は真ん中のフォルダを読むところだけ**です。順に見ていきます。

#### 上の5行（読み込み）

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import fs from "node:fs/promises";
import path from "node:path";
```

上の3行は第2章と同じです。下の2行が今回の新顔です。

- `node:fs/promises` … **fs = file system（ファイルシステム）**。ファイルを読み書きするための、Node標準の部品です。`/promises` が付いているほうを使うと `await` で書けます
- `node:path` … フォルダ名とファイル名を**正しく繋ぐ**ための部品

> ⚠️ `.js` を落とさないでください。TypeScript のファイルを書いているのに `.js` と書くのは変に見えますが、**変換後のファイルを指しているので正しい**です。ここを消すと `Cannot find module` で止まります。

#### メモ置き場を受け取る1行

```typescript
const NOTES_DIR = process.env.NOTES_DIR;
```

`process.env` は、**このプログラムに外から渡された設定の入れ物**です。ここから `NOTES_DIR` を取り出しています。

渡す側は、あとで書く `claude_desktop_config.json` の `env` です。つまり——**コードにフォルダ名は一切書かれていません**。

注意点が1つ。`process.env.NOTES_DIR` は「**渡されていないかもしれない**」ので、型は `string | undefined` です。だから後ろのほうで `if (!NOTES_DIR)` と確認しています。ここを飛ばすと `tsc` に怒られます（`strict: true` のおかげです）。

#### フォルダを読む関数（今回の新しいところ）

```typescript
async function listNoteFiles(dir: string): Promise<string[]> {
  const entries = await fs.readdir(dir, { withFileTypes: true });
  return entries
    .filter((entry) => entry.isFile() && entry.name.endsWith(".md"))
    .map((entry) => entry.name);
}
```

1行ずつ読みます。

| 行 | 何をしている |
|---|---|
| `fs.readdir(dir, ...)` | **readdir = read directory（フォルダを読む）**。そのフォルダの中身の一覧をもらう |
| `{ withFileTypes: true }` | 「名前だけじゃなく、**ファイルなのかフォルダなのかも教えて**」という指定 |
| `await` | 一覧が返ってくるまで**待つ**。ディスクを触る処理は一瞬では終わらないため |
| `.filter(...)` | 条件に合うものだけ残す |
| `entry.isFile()` | **フォルダを弾く**（`.md` で終わるフォルダを作る人はいませんが、念のため） |
| `entry.name.endsWith(".md")` | **`.md` で終わる名前だけ**残す。`.txt` や `.DS_Store` は無視 |
| `.map((entry) => entry.name)` | 残ったものから**名前の文字列だけ**取り出す |

戻り値は「`["a.md", "b.md"]` のような文字列の配列」です。それが `Promise<string[]>` の意味です。

> 💡 **サブフォルダの中は見ません。** 一階層だけです。「フォルダの中のフォルダも探したい」は、まずこれが動いてからにしてください。動くものを1つ持っているほうが、あとで必ず得をします。

#### 1ファイルずつ中を見るところ

```typescript
for (const filename of files) {
  const body = await fs.readFile(path.join(NOTES_DIR, filename), "utf-8");
  const found = body.toLowerCase().indexOf(needle);
  if (found === -1) continue;
```

| 行 | 何をしている |
|---|---|
| `for (const filename of files)` | 集めたファイル名を**1つずつ**取り出して繰り返す |
| `path.join(NOTES_DIR, filename)` | `/Users/you/notes` と `a.md` を繋いで `/Users/you/notes/a.md` にする。**自分で `+ "/" +` と書かない**（Windows では区切り文字が違うため） |
| `fs.readFile(..., "utf-8")` | ファイルの中身を**文字列として**読む。`"utf-8"` を書き忘れると、日本語が読めない生データが返ってきます |
| `.indexOf(needle)` | 検索語が**何文字目にあるか**を返す。無ければ `-1` |
| `if (found === -1) continue;` | 見つからなかったら、このファイルは**飛ばして次へ** |

`needle`（針）は、この関数の少し上でこう作っています。

```typescript
const needle = query.toLowerCase();
```

`toLowerCase()` は**すべて小文字にする**という意味です。`body` のほうも `.toLowerCase()` してから探しているので、`MCP` と `mcp` のどちらで検索してもヒットします。

#### 抜粋を切り出すところ

```typescript
const start = Math.max(0, found - 60);
const end = Math.min(body.length, found + query.length + 60);
const excerpt = body.slice(start, end).replace(/\n/g, " ");
hits.push(`【${filename}】\n…${excerpt}…`);
```

ヒットした場所の**前後60文字**を切り出しています。

- `Math.max(0, ...)` … マイナスにならないように。ファイルの1行目でヒットした場合の保険です
- `Math.min(body.length, ...)` … ファイルの終わりを超えないように
- `.replace(/\n/g, " ")` … 改行をスペースに置き換えて、**1行の抜粋**にする（読みやすさのため）

**ファイル全部をAIに渡さない**のがポイントです。メモが100枚あったら、全文を返すと処理しきれません。**必要なところだけ**返します。

#### エラーを「返す」ところ

```typescript
} catch (error) {
  return {
    content: [{ type: "text", text: `メモ置き場を読めませんでした（${NOTES_DIR}）。…` }],
  };
}
```

ここは MCP らしい書き方です。**エラーが起きても、投げっぱなし（throw）にしません。**

なぜなら、throw するとサーバーが落ちて、Claude 側には「なんか失敗した」としか伝わらないからです。**文章で返せば、AIはそれを読んで次の手を考えられます**——「フォルダのパスが違うようです。設定を確認してみてください」と、あなたに教えてくれるのです。

> 💡 **道具のエラーメッセージも、AIへの説明文の一種**です。この考え方は第6章でもう一度出てきます。

### ステップ7：ビルドする

```bash
npm run build
```

エラーが出なければ、`build/index.js` ができています。確認しましょう。

```bash
ls build
```

> ⚠️ **これを忘れる人が、毎回います。** `src/index.ts` をどれだけ直しても、`npm run build` しなければ `build/index.js` は古いままです。Claude が見ているのは `build/index.js` のほうです。**直したら、ビルド。** 覚えてください。

### ステップ8：Claude Desktop の設定ファイルに `memo` を足す

設定ファイルの場所です。

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%AppData%\Claude\claude_desktop_config.json`

第2章の `weather` を残したまま、`memo` を足します。

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/weather/build/index.js"]
    },
    "memo": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/memo/build/index.js"],
      "env": { "NOTES_DIR": "/ABSOLUTE/PATH/TO/notes" }
    }
  }
}
```

新しいのは **`env`** の行です。ここに書いたものが、コードの `process.env.NOTES_DIR` に届きます。

パスを調べるコマンドはこれです。

```bash
# memo フォルダの中で
pwd
# → /Users/あなた/dev/memo   ← これに /build/index.js を足す

# メモ置き場の絶対パス
cd ~/notes && pwd
```

> ⚠️ **`~/notes` とは書かないでください。** `~` は展開されません。`/Users/あなた/notes` のように**書き下した絶対パス**にしてください。ここは全員が一度は踏みます。

### ステップ9：完全に再起動する

**ウィンドウを閉じるだけでは足りません。**

- **macOS**: `Cmd + Q`（またはメニューから「Claude を終了」）
- **Windows**: タスクトレイのアイコンからも終了する

再起動したら、入力欄のあたりにある**ツールのアイコン**を開いて、`search_notes` が並んでいるか確認してください。ここに出てこないなら、Claude はまだあなたのサーバーを知りません。**第4章に戻って切り分け**です。

### ステップ10：聞いてみる

```
打ち合わせについてのメモある？
```

初回は「memo の search_notes を使ってもいいですか？」と許可を聞かれます。許可すると、検索結果が返ってきます。

**おめでとうございます。あなたの Claude が、あなたのメモを読みました。**

---

## 🧪 この章の山場：説明文を変えると、呼ばれ方が変わる 🟢

さて、ここからが本題です。**さっき書いたコードを、わざと悪くします。**

第1章と第2章で「AIが読むのは説明文だけ」という話をしました。ここでそれを、**自分の手で確かめます**。

### 実験A：説明文を「検索」だけにしてみる

`src/index.ts` の `description` を、これに書き換えてください。

```typescript
    description: "検索",
```

そして——**忘れずに**。

```bash
npm run build
```

Claude Desktop を **完全に再起動**して、新しい会話を開き、こう聞きます。

```
打ち合わせについてのメモある？
```

### 何が起きるか

高い確率で、こうなります。

- Claude が **道具を使わずに**「メモの内容にはアクセスできません」と答える
- あるいは、**関係ない話**（Web検索が要るのでは、など）を始める
- あるいは、使うには使うが、**変な `query`** を渡してくる（「打ち合わせについてのメモ」のような長い文章など）

道具は**ちゃんとそこにあります**。コードも1文字も壊れていません。動くのに、**呼ばれない**。

> **AIから見ると、「検索」という道具は「何を検索するのか分からない道具」です。**
> あなたには自明でも、AIには何の手がかりもありません。Web検索かもしれない、社内DBかもしれない、辞書引きかもしれない。**分からない道具は、使われません。**

### 実験B：`.describe()` も雑にしてみる

もう一段いきます。引数の説明も潰します。

```typescript
    inputSchema: {
      query: z.string().describe("クエリ"),
    },
```

ビルド → 完全再起動 → 同じ質問。今度は**呼ばれても中身が変**になりがちです。

```
[🔨 search_notes]  query: "先週の打ち合わせで決まったことについて教えて"
```

これでは1文字もヒットしません。**AIは「query に何を入れればいいか」を、`.describe()` からしか知りません。** 「クエリ」とだけ書いてあれば、質問文をそのまま入れてくるのは当然です。

### 実験C：良い説明文に戻す

最初のコードに戻します。

```typescript
    description:
      "ユーザーのローカルのメモ（.mdファイル）を全文検索する。ユーザーが自分の過去のメモ・議事録・日記・書きかけの文章について尋ねたときに使う。ヒットしたファイル名と、その前後の抜粋を返す。",
    inputSchema: {
      query: z
        .string()
        .describe(
          "メモの中から探す検索語。日本語でよい。文章ではなく、1〜2語のキーワードにすること（例: 「打ち合わせ」「リリース日」）",
        ),
    },
```

ビルド → 完全再起動 → 同じ質問。今度は、

```
[🔨 search_notes]  query: "打ち合わせ"
```

**きれいに呼ばれます。** キーワードも短く切ってきます。

### 何が効いたのか

3つの成分に分解できます。**あなたが説明文を書くときのチェックリスト**にしてください。

| 成分 | 悪い例 | 良い例 |
|---|---|---|
| **何をするか** | 「検索」 | 「ローカルのメモ（.mdファイル）を全文検索する」 |
| **いつ使うか** | （書いていない） | 「ユーザーが自分の過去のメモ・議事録・日記について尋ねたとき」 |
| **何が返るか** | （書いていない） | 「ヒットしたファイル名と、その前後の抜粋」 |

とくに効くのが真ん中の「**いつ使うか**」です。AIは毎回「今この道具を出すべきか？」を判断しています。その判断材料を**あなたが書いていない**なら、判断できないのは当たり前です。

引数側（`.describe()`）も同じ構造です。

| 成分 | 良い例 |
|---|---|
| **何を入れるか** | 「メモの中から探す検索語」 |
| **どういう形式か** | 「文章ではなく、1〜2語のキーワード」 |
| **具体例** | 「例: 「打ち合わせ」「リリース日」」 |

**具体例を1つ入れるだけで、精度がはっきり変わります。** これは覚えて損がありません。

### プログラミングの上手さより、説明の上手さ

ここで立ち止まってほしいのです。

いま実験Aから実験Cまでで、**あなたはロジックを1行も変えていません**。検索の処理は最初から最後まで完璧に動いていました。それでも、**道具の役に立ち方はまるで違いました**。

> **MCPサーバー作りでいちばん効くスキルは、コードを速く書く力ではありません。「この道具は何者で、いつ使うのか」を、他人（AI）に一発で伝わるように書く力です。**

これは意外と、プログラミングというより**説明書を書く仕事**に近いです。そして——**あなたが日本語で丁寧に説明できるなら、それはもう強みです**。

> 💡 **やりすぎには注意。** 説明文が長すぎると、今度は他の道具の説明を圧迫します（AIが一度に読める量には限りがあります）。**3〜4文が目安**です。第12章で「道具が多すぎるとAIが選べなくなる」問題として、もう一度出てきます。

### 🔧 もう一歩：どう呼ばれたか、後から見る 🔧

実験のたびに再起動するのが面倒な人へ。第4章でやった **MCP Inspector** を使えば、Claude Desktop を挟まずに説明文を確認できます。

```bash
npx @modelcontextprotocol/inspector node /絶対パス/memo/build/index.js
```

Tools タブを開くと、**AIが見ているのと同じ情報**（名前・description・引数と `.describe()`）がそのまま表示されます。ここに出ている文字が、AIが持っている情報の**すべて**です。この画面を見ながら説明文を練るのが、いちばん速い書き方です。

---

## ⚠️ ハマりどころ 🟢

### ① `NOTES_DIR` を渡し忘れる

**症状**：Claude が「メモ置き場が設定されていません」と言う。

`claude_desktop_config.json` の `memo` に `env` の行があるか確認してください。第2章の `weather` には `env` が無かったので、**まるごとコピーして貼ると、`env` ごと消えます**。

```json
"memo": {
  "command": "node",
  "args": ["/絶対パス/memo/build/index.js"],
  "env": { "NOTES_DIR": "/絶対パス/notes" }
}
```

> 💡 ちなみに、あなたのターミナルで `export NOTES_DIR=...` しても**効きません**。Claude Desktop は、あなたのターミナルの環境変数を引き継がないからです。渡す場所は `env` だけです。

### ② `~` や相対パスを書いてしまう

**症状**：「メモ置き場を読めませんでした」と返ってくる。

```json
"env": { "NOTES_DIR": "~/notes" }        ← ❌ ~ は展開されない
"env": { "NOTES_DIR": "./notes" }        ← ❌ 起動時のカレントフォルダは不定
"env": { "NOTES_DIR": "/Users/you/notes" } ← ✅
```

`args` のパスも同じです。**MCPサーバーは、どのフォルダで起動されるか分かりません**（macOS では `/` のこともあります）。絶対パスだけが安全です。

### ③ ビルドを忘れる

**症状**：コードを直したのに、動きが変わらない。

`npm run build` です。**説明文の実験のときが、いちばん忘れます。**「description を書き換えた → 再起動 → 変わらない → 悩む」の正体は、たいていこれです。手順は毎回この3点セットだと覚えてください。

> **直す → `npm run build` → Claude を完全終了して再起動**

### ④ メモが1件も無い／`.md` じゃない

**症状**：エラーは出ないが、いつも「見つかりませんでした」。

- そのフォルダに `.md` ファイルはありますか？（`.txt` は対象外です）
- そもそもフォルダは空ではないですか？

コードの最後のメッセージで `（3件の.mdを調べました）` と件数を出しているのは、このためです。**`0件` と出たら、フォルダかパスの問題**。**`5件` と出たのに見つからないなら、検索語の問題**。この2つを切り分けられるようにわざと書いてあります。

### ⑤ 日本語の検索と、大文字小文字

`toLowerCase()` は英字の大文字小文字を吸収しますが、日本語ではこんな取りこぼしが起きます。

- **全角と半角**：`ＭＣＰ` と `MCP` は別物として扱われます
- **表記ゆれ**：「打ち合わせ」「打合せ」「ミーティング」は別物です
- **活用**：「決めた」で探しても「決定」はヒットしません

これは**バグではなく、単純な文字列検索の限界**です。今は割り切ってください。むしろ大事なのは、`.describe()` に「**1〜2語のキーワードにすること**」と書いておくことです。AIが短く切ってくれれば、ヒット率は上がります。

> 💡 **道具の弱点は、説明文でカバーできる。** これも「説明の上手さ」のうちです。

### ⑥ `Cannot find module` が出る

`import` の行末の **`.js`** が落ちていないか確認してください。`tsconfig.json` の `"module": "Node16"` を使う以上、これは必須です。

### ⑦ ツール一覧に `search_notes` が出てこない

ここまで来たら、第4章の手順です。順番に。

1. ターミナルで直接 `node /絶対パス/memo/build/index.js` を実行 → `Memo MCP Server running on stdio` が出るか（出たら `Ctrl+C` で止める）
2. Inspector で繋がるか
3. Claude Desktop のログ（macOS: `tail -n 20 -F ~/Library/Logs/Claude/mcp*.log`）
4. `claude_desktop_config.json` の**カンマ・波かっこ**（`weather` の後ろにカンマを足し忘れていませんか？）

---

## 🤖 AIに頼むなら 🟢

この章のコードは、AIに書かせても構いません。ただし**渡し方**にコツがあります。

### 仕様を先に固定して渡す

```
MCPサーバーを TypeScript で作ってください。
- 公式SDK @modelcontextprotocol/sdk、zod は v3
- server.registerTool を使う。inputSchema は zod のフィールドを並べた素のオブジェクト
  （z.object で包まない）
- ツールは search_notes ひとつだけ。引数は query: string
- メモ置き場は process.env.NOTES_DIR から取る
- .md ファイルだけを対象に全文検索し、ファイル名と前後の抜粋を返す
- 戻り値は必ず { content: [{ type: "text", text: ... }] }
- エラーは throw せず、理由を text で返す
- stdout には絶対に書かない（ログは console.error）
```

**名前と引数名を先に書いて渡す**のが肝心です。放っておくと `searchNotes` や `keyword` にされます。第6章以降と食い違います。

### AIが古い書き方を出してきたら

MCPは動きの速い分野です。AIが `server.tool(...)` や `setRequestHandler(...)` といった**別の書き方**を出してくることがあります。間違いとは限りませんが、**この教材のコードとは混ざりません**。そのときはこう言ってください。

> **「modelcontextprotocol.io の最新のクイックスタートの書き方（`registerTool`）に揃えてください」**

### 説明文こそ、AIに手伝ってもらう

そして、この章の本題です。**説明文の推敲は、AIがかなり得意です。**

```
このMCPツールの description を書き直してください。
条件：
・何をする道具か
・どういうときに使うべきか
・何が返るか
の3点を、3〜4文で。読む相手は人間ではなくAIです。
```

「**読む相手はAIです**」と伝えるのがポイントです。マーケティング的な言い回しではなく、**判断材料**を書いてくれます。

---

## 📗 ことばメモ

| ことば | よみ | 意味 |
|---|---|---|
| **環境変数** | かんきょうへんすう | プログラムに外から渡す設定値。コードに書かずに済む |
| **`process.env`** | プロセス・エンブ | Node で環境変数を取り出す入れ物 |
| **`NOTES_DIR`** | ノーツディア | この教材で決めた、メモ置き場を渡すための環境変数名 |
| **fs** | エフエス | file system。ファイルを読み書きする Node 標準の部品 |
| **`readdir`** | リードディア | フォルダの中身の一覧をもらう |
| **`readFile`** | リードファイル | ファイルの中身を読む。日本語は `"utf-8"` を指定 |
| **`path.join`** | パス・ジョイン | フォルダ名とファイル名を、OSに合った形で繋ぐ |
| **絶対パス** | ぜったいパス | `/` から始まる、完全な住所。設定ファイルでは必須 |
| **description** | ディスクリプション | ツールの説明文。**AIが読む唯一の手がかり** |
| **`.describe()`** | ディスクライブ | 引数1つひとつの説明。ここも AI が読む |

---

## ➡️ 次へ

この章で、あなたは**自分のための道具**を1つ持ちました。そして——道具の良し悪しを決めるのは、コードの巧みさではなく**説明文**だ、ということも体験しました。この感覚は、次の章からさらに効いてきます。

[第6章　道具を増やす — 読む・書く](06-more-tools.md) では、`read_note`（1ファイルの全文を読む）と `write_note`（メモを保存する）を足します。検索でファイル名が分かったら、次は中身を読みたくなりますよね。その自然な流れです。

ただし、`write_note` からは話の温度が変わります。**読む道具は間違っても困るだけですが、書く道具は間違うと壊します**。その怖さは第6章で予告し、第9章でまとめて手当てします。

## 関連ページ

- MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 — 目次
- [第4章　デバッグの作法 — 見えない通信を覗く](04-debug.md) — 動かないときはここへ
- [第6章　道具を増やす — 読む・書く](06-more-tools.md) — 次の章
- [第9章　安全 — 書ける道具は、危ない道具](09-safety.md) — `NOTES_DIR` の外に出さない話
- 自作CLIエージェントで学ぶ AIエージェント開発入門 — はじめに・目次 — 第3弾。道具の説明文の話はこちらにも
