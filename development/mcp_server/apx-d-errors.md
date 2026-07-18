---
title: "付録D　よくあるエラー集"
parent: "MCPサーバーを作って学ぶ AIに道具を持たせる入門"
grand_parent: "開発の心得"
nav_order: 17
nav_exclude: true
---

# 付録D　よくあるエラー集

> 📖 MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 ← 目次に戻る
> このページは**読み物ではありません**。詰まったときに開く**逆引き辞典**です。ブラウザの検索（Ctrl+F / Cmd+F）でエラーの文字列を探してください。

---

## 🚑 まずこの順で確かめる 🟢

深く考える前に、この3つです。**動かない原因の大半はここにあります。**

1. **ビルドしたか？** — `npm run build` を実行しましたか。設定ファイルが指しているのは `src/index.ts` ではなく **`build/index.js`** です
2. **絶対パスか？** — 設定ファイルの `args` は `/Users/…` `C:\\…` から始まる**フルパス**ですか。`./build/index.js` は動きません
3. **再起動したか？** — Claude Desktop を**完全終了**して開き直しましたか。ウィンドウの×を押しただけでは足りません

この3つを確かめてから、下の症状一覧に進んでください。

---

## 🔎 症状から探す

| # | こんな症状 | 見出し |
|---|---|---|
| 1 | ツール一覧にサーバーが出てこない | [#1](#1-claude-desktop-にサーバーが出てこない) |
| 2 | `Cannot find module` と出る | [#2](#2-cannot-find-module-と出る) |
| 3 | コードを直したのに反映されない | [#3](#3-コードを直したのに動きが変わらない) |
| 4 | パスは合っているのに起動しない | [#4](#4-設定に相対パスを書いた) |
| 5 | ログを足したら壊れた | [#5](#5-consolelog-を書いたらサーバーが壊れた) |
| 6 | zod まわりで動かない | [#6](#6-zod-のバージョンが違う) |
| 7 | `Cannot use import statement outside a module` | [#7](#7-type-module-を書き忘れた) |
| 8 | 設定を書いたのに何も起きない | [#8](#8-設定ファイルの-json-が壊れている) |
| 9 | 再起動したつもりなのに古いまま | [#9](#9-claude-desktop-を再起動していない) |
| 10 | ツールはあるのにAIが呼ばない | [#10](#10-ツールはあるのにaiが呼んでくれない) |
| 11 | メモが1件も見つからない | [#11](#11-環境変数-notes_dir-が渡っていない) |
| 12 | 「東京の天気」が取れない | [#12](#12-天気サーバーで東京の天気が取れない) |

---

## 1. Claude Desktop にサーバーが出てこない

**症状**
設定ファイルを書いて再起動したのに、ツールの一覧にあなたのサーバーが**まったく現れない**。エラーメッセージも出ない。

**原因**
これは1つの原因ではなく、**「サーバーが起動できなかった」ときの共通の見え方**です。Claude Desktop はサーバーの起動に失敗しても、画面には基本的に何も出しません。原因は下のどれかであることがほとんどです。

- 設定ファイルの JSON が壊れている（→ [#8](#8-設定ファイルの-json-が壊れている)）
- パスが違う／相対パス（→ [#4](#4-設定に相対パスを書いた)）
- ビルドしていないので `build/index.js` が存在しない（→ [#3](#3-コードを直したのに動きが変わらない)）
- 再起動していない（→ [#9](#9-claude-desktop-を再起動していない)）
- サーバー自体が起動時にクラッシュしている

**対処**
順番に切り分けます。**この順序が大事**です。

1. **ターミナルで直接動かす**。設定と同じコマンドを手で打ちます。

   ```bash
   node /ABSOLUTE/PATH/TO/memo/build/index.js
   ```

   `Memo MCP Server running on stdio` と出て、そのまま待ち状態になれば**サーバー自体は無事**です（Ctrl+C で止めます）。ここでエラーが出たら、それが本当の原因です。
2. **ログを見る**。
   - macOS: `tail -n 20 -F ~/Library/Logs/Claude/mcp*.log`
   - Windows: `type "$env:AppData\Claude\logs\mcp*.log"`
3. **MCP Inspector で繋いでみる**。

   ```bash
   npx @modelcontextprotocol/inspector node /ABSOLUTE/PATH/TO/memo/build/index.js
   ```

   Inspector で繋がるのに Claude Desktop で出ないなら、**問題はサーバーではなく設定ファイル**です。

くわしい手順は [第4章　デバッグの作法 — 見えない通信を覗く](04-debug.md)、設定ファイルまわりは [付録C　設定ファイルの場所とトラブル](apx-c-config.md) を見てください。

---

## 2. `Cannot find module` と出る

`Cannot find module` は**2つの別の場面**で出ます。まずどちらか見分けてください。

### 2-a. 実行時（`node build/index.js` で出た）

**症状**

```
Error: Cannot find module '/Users/you/memo/build/index.js'
```

**原因**
そのファイルが**本当に存在しません**。`npm run build` をしていないか、パスの綴りが違うかです。

**対処**

```bash
npm run build
ls build/index.js
```

`ls`（Windows は `dir`）で見えなければビルドが失敗しています。ビルドのエラーメッセージを先に読んでください。

### 2-b. ビルド時（`npm run build` で出た）— import の `.js` 忘れ

**症状**

```
error TS2307: Cannot find module '@modelcontextprotocol/sdk/server/mcp' or its corresponding type declarations.
```

**原因**
import のパスから **`.js` を落としています**。

```typescript
// ✗ 動かない
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp";

// ✓ 正しい
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
```

`tsconfig.json` の `"moduleResolution": "Node16"` では、**import 先の拡張子まで書くのが必須**です。ここは「TypeScript を書いているのに `.js` と書く」ので、初心者が全員「間違いでは？」と思うところですが、**`.js` が正しい**です。出力される JavaScript から見た名前を書くからです。

自分のファイルを import した場合は、TypeScript がもっと親切に教えてくれます。

```
error TS2835: Relative import paths need explicit file extensions in ECMAScript imports
when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './notes.js'?
```

**対処**
`src/index.ts` の import 行をすべて見直し、末尾に `.js` を付けます。付け終わったら `npm run build` をやり直してください。

> 💡 AIコーディングツールは、この `.js` をよく落とします。指摘すれば直ります。

---

## 3. コードを直したのに動きが変わらない

**症状**
`src/index.ts` を編集して Claude Desktop を再起動したのに、**前と同じ動き**をする。直したはずのバグが直っていない。新しく足したツールが出てこない。

**原因**
**ビルドしていません。** Claude Desktop が実行しているのは `src/index.ts` ではなく、コンパイル済みの **`build/index.js`** です。ソースを編集しただけでは `build/index.js` は古いままなので、**昔のサーバーが動き続けます**。

```
memo/
  src/index.ts      ← あなたが編集するファイル
  build/index.js    ← Claude Desktop が実行するファイル（npm run build で作られる）
```

**対処**

```bash
npm run build
```

そのうえで Claude Desktop を完全再起動します。**「直した → ビルド → 再起動」が1セット**だと覚えてください。

編集中は、別ターミナルで監視ビルドを回しておくと楽です。

```bash
npx tsc --watch
```

（それでも Claude Desktop の再起動は必要です。サーバーは起動時に読み込まれるからです。）

---

## 4. 設定に相対パスを書いた

**症状**
パスは合っているつもりなのに、サーバーが出てこない。ログには `Cannot find module './build/index.js'` のようにファイルが見つからない旨が出る。

**原因**
設定ファイルに**相対パス**を書いています。

```json
{ "args": ["./build/index.js"] }
```

Claude Desktop がサーバーを起動するときの**カレントディレクトリ（作業フォルダ）は決まっていません**。macOS では `/`（ルート）になることがあります。つまり `./build/index.js` は `/build/index.js` を探しにいって、当然見つかりません。

**対処**
**絶対パス**にします。

```json
{
  "mcpServers": {
    "memo": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/memo/build/index.js"],
      "env": { "NOTES_DIR": "/ABSOLUTE/PATH/TO/notes" }
    }
  }
}
```

パスの調べ方:

- macOS / Linux: プロジェクトのフォルダで `pwd` を実行し、末尾に `/build/index.js` を足す
- Windows: エクスプローラーでファイルを Shift+右クリック →「パスのコピー」

同じ理由で、`env` に書く `NOTES_DIR` も絶対パスです。`~/notes` の `~` は展開されないことがあるので、`/Users/you/notes` のように書き切ってください。

→ [付録C　設定ファイルの場所とトラブル](apx-c-config.md)

---

## 5. `console.log` を書いたらサーバーが壊れた

**症状**
デバッグのつもりで `console.log("started")` を1行足したら、**サーバーが繋がらなくなった**。それまで動いていたのに。

**原因**
この教材で一番有名な罠です。**stdio では、標準出力（stdout）は Claude との会話チャンネルそのもの**です。そこに `console.log` で文字を流すと、**JSON-RPC のメッセージに文字列が混ざり**、相手が解釈できなくなって会話が壊れます。

MCP の仕様でも、stdio では「**有効な MCP メッセージ以外を stdout に書いてはならない（MUST NOT）**」と定められています。

**対処**
出力先を変えます。3つの選択肢があります。

| 方法 | 書き方 | 使いどころ |
|---|---|---|
| **標準エラー出力** | `console.error("…")` | いちばん手軽。ホストがログとして拾う |
| **ファイルに追記** | `fs.appendFileSync(…)` | じっくり追いたいとき |
| **MCP のログ機能** | `await server.server.sendLoggingMessage({ level: "info", data: "…" })` | クライアントに届けたいとき |

`console.log` を全部 `console.error` に置き換えて、ビルドし直せば直ります。

> ⚠️ MCP のログ機能を使うなら、サーバーを作るときに `capabilities: { logging: {} }` を宣言してください。**宣言していないと、エラーも出ないまま黙って何も送りません**（[第3章](03-stdio.md)参照）。

> 💡 **HTTP なら stdout に書いても壊れません**（通信路が別だから）。逆に Streamable HTTP では **stderr はクライアントに拾われません**。トランスポートによってログの正解が変わります。

くわしくは [第3章　標準入出力の正体 — `console.log` 1行でサーバーが壊れる](03-stdio.md) と [第4章　デバッグの作法](04-debug.md) を。

---

## 6. zod のバージョンが違う

**症状**
`registerTool` の `inputSchema` のところで型エラーが出る。あるいはビルドは通るのに、ツールを呼んだ瞬間に実行時エラーで落ちる。ネットのサンプルどおりに書いたのに動かない。

**原因**
**zod の v4 が入っています。** この教材と MCP 公式クイックスタートは **zod のバージョン3** を前提にしています。v4 は書き方が変わっているため、そのままでは噛み合いません。

`npm install zod` とだけ打つと最新版（v4系）が入ってしまいます。

**対処**
まず、今どれが入っているか確認します。

```bash
npm ls zod
```

`3.x.x` でなければ、入れ直します。

```bash
npm install zod@3
npm run build
```

> ⚠️ **AIに書かせたときも要注意です。** 「zod を入れて」と頼むと最新版を入れてきます。**「zod@3 で」**と明示してください。

---

## 7. `"type": "module"` を書き忘れた

**症状**

```
SyntaxError: Cannot use import statement outside a module
```

あるいは、

```
Warning: To load an ES module, set "type": "module" in the package.json
or use the .mjs extension.
```

**原因**
`package.json` に `"type": "module"` がありません。Node.js は既定では古い形式（CommonJS）としてファイルを読むため、`import` 文を理解できずに止まります。

**対処**
`package.json` に1行足します。

```json
{
  "type": "module",
  "scripts": { "build": "tsc && chmod 755 build/index.js" },
  "files": ["build"]
}
```

足したら `npm run build` をやり直してください。

> 💡 `npm init -y` で作った `package.json` には**この行は入りません**。自分で足すものだと覚えておいてください。

---

## 8. 設定ファイルの JSON が壊れている

**症状**
設定ファイルを書き足したのに、**何も起きない**。サーバーが出てこないどころか、それまで動いていた他のサーバーまで消えることがあります。

**原因**
`claude_desktop_config.json` の JSON として文法が壊れています。**JSON は思っているより厳しい**フォーマットです。よくある3つ:

**① 末尾のカンマ**

```json
{
  "mcpServers": {
    "memo": { "command": "node", "args": ["/path/build/index.js"] },   ← ✗ 最後の要素の後ろにカンマ
  }
}
```

**② コメント**

```json
{
  // メモサーバー   ← ✗ JSON にコメントは書けません
  "mcpServers": { }
}
```

**③ Windows のパスのバックスラッシュ**

```json
"args": ["C:\Users\you\memo\build\index.js"]     ← ✗ 壊れます
"args": ["C:\\Users\\you\\memo\\build\\index.js"] ← ✓ 2本重ねる
"args": ["C:/Users/you/memo/build/index.js"]      ← ✓ スラッシュでもOK
```

JSON では `\` が特別な意味を持つため、**そのまま1本では書けません**。

**対処**
壊れていないか、機械に確かめさせるのが確実です。

```bash
node -e "JSON.parse(require('fs').readFileSync(process.argv[1],'utf8')); console.log('OK')" ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

`OK` と出れば文法は正しいです。エラーが出れば、何行目かを教えてくれます。VS Code で開いて、赤い波線が出ていないか見るだけでもかなり分かります。

→ [付録C　設定ファイルの場所とトラブル](apx-c-config.md)

---

## 9. Claude Desktop を再起動していない

**症状**
設定を直した／ビルドし直したのに、**まったく変化がない**。「さっきと同じエラー」がずっと出続ける。

**原因**
**ウィンドウを閉じただけ**になっています。とくに macOS では、ウィンドウの左上の赤い×を押してもアプリは終了せず、裏で動き続けます。MCPサーバーの設定は**アプリの起動時に読み込まれる**ので、アプリが終了していなければ何も変わりません。

**対処**
**完全終了**してから開き直します。

- **macOS**: Claude Desktop を前面にして **Cmd + Q**。またはメニューバーの「Claude」→「Claude を終了」。Dock のアイコンに小さな点が残っていないか確認します
- **Windows**: ×で閉じたあと、**タスクトレイ（右下）に Claude のアイコンが残っていないか**確認し、右クリックして終了します。それでも不安ならタスクマネージャーで Claude のプロセスを終了します

再起動が必要になるのは、次のときです。

- 設定ファイル（`claude_desktop_config.json`）を編集したとき
- `npm run build` でサーバーのコードを作り直したとき

→ [第2章　まず公式のサンプルを動かす](02-quickstart-weather.md)

---

## 10. ツールはあるのにAIが呼んでくれない

**症状**
ツール一覧にはちゃんと出ている。MCP Inspector から手で呼べば正しく動く。**なのに、Claude と会話しても呼んでくれない。** 「メモを検索して」と頼んでも、Claude が「ファイルにアクセスできません」と答える。

**原因**
これは**バグではなく、説明文の問題**であることがほとんどです。

思い出してください。**AIがあなたの道具について知っているのは、あなたが書いた説明文だけ**です。中身のコードは1文字も読みません。だから——

- 説明が雑（「検索する」だけ）だと、AIは**何を検索する道具なのか分からず**、使いません
- 引数の `.describe()` が無いと、**何を渡せばいいか分からず**、避けます
- 名前と説明が食い違っていると、**信用されません**

**対処**
説明文を書き直します。ポイントは「**何ができて、いつ使うべきか**」を書くことです。

```typescript
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
  async ({ query }) => { /* … */ },
);
```

- `description` は**動詞で、具体的に**。返ってくるものまで書く
- **すべての引数に `.describe()` を付ける**。ここを省略しないでください

書き換えたら `npm run build` → 完全再起動です（[#3](#3-コードを直したのに動きが変わらない) [#9](#9-claude-desktop-を再起動していない)）。

説明文の設計は [第5章　自分の道具を作りはじめる — メモを検索する](05-memo-server.md) の中心テーマです。呼ばれないときは、まずここに戻ってください。

---

## 11. 環境変数（`NOTES_DIR`）が渡っていない

**症状**
サーバーは起動している。ツールも呼ばれている。**なのに検索結果がいつも0件**。あるいは「フォルダが見つかりません」というテキストが返ってくる。ターミナルから直接動かしたときは正常に動く。

**原因**
**環境変数が Claude Desktop から自動では引き継がれません。** あなたのターミナルで `export NOTES_DIR=…` していても、Claude Desktop が起動するサーバーには**その値は届きません**。結果、`process.env.NOTES_DIR` が `undefined` になります。

**対処**
設定ファイルの **`env` キー**で明示的に渡します。

```json
{
  "mcpServers": {
    "memo": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/memo/build/index.js"],
      "env": { "NOTES_DIR": "/ABSOLUTE/PATH/TO/notes" }
    }
  }
}
```

あわせて確認したいこと:

- **絶対パスか**（`~/notes` ではなく `/Users/you/notes`）→ [#4](#4-設定に相対パスを書いた)
- そのフォルダが**実際に存在するか**
- 中に **`.md` ファイル**が入っているか（このメモサーバーが対象にするのは `.md` だけです）
- 書き換えたあと**完全再起動したか** → [#9](#9-claude-desktop-を再起動していない)

サーバー側では、値が来ていないときに黙って0件を返すのではなく、**理由をテキストで返す**ようにしておくと自分が助かります（エラーは throw せず、説明文を返すのがこの教材の作法です）。

→ [第5章　自分の道具を作りはじめる](05-memo-server.md) / [第9章　安全 — 書ける道具は、危ない道具](09-safety.md)

---

## 12. 天気サーバーで「東京の天気」が取れない

**症状**
第2章の天気サーバーを繋いだ。ツールも出ている。呼ばれてもいる。**でも「東京の天気を教えて」には答えられない。** 「カリフォルニアの警報」なら返ってくる。

**原因**
**仕様です。バグではありません。**

公式クイックスタートの天気サーバーは、アメリカ国立気象局（NWS）のAPIを使っています。**このAPIはアメリカ国内のデータしか持っていません**。だから東京のデータは、そもそも取りようがないのです。

ツールの説明文を見てください。`get_alerts` の引数はこう書いてあります。

> **Two-letter state code**（2文字の州コード。例: `CA`, `NY`）

**州コード**を受け取る道具に、「東京」を渡す方法はありません。

**対処**
**直しません。これは体験してもらうための仕掛けです。**

ここから受け取ってほしいことが2つあります。

1. **AIは説明文しか読んでいない。** Claude は「東京」を渡せないことを、コードを読んで知ったのではありません。**説明文に "Two-letter state code" と書いてあったから**そう判断しました
2. **道具にできないことは、AIにもできない。** AIをどれだけ賢くしても、道具が持っていない情報は出てきません。**能力の上限を決めるのは、あなたが作る道具のほう**です

日本の天気が欲しければ、**日本の天気APIを呼ぶ道具を自分で作る**しかありません。そして、それができるようになるのが第5章からです。

→ [第2章　まず公式のサンプルを動かす](02-quickstart-weather.md) / [第5章　自分の道具を作りはじめる](05-memo-server.md)

---

## それでも直らないときは

ここまで試して直らないなら、**「動かない」の中身をもっと細かく特定**しましょう。[第4章　デバッグの作法](04-debug.md) では、次の4類型に分けて切り分ける手順を扱っています。

| 類型 | まず見るところ |
|---|---|
| **繋がらない** | ターミナルで直接起動 → ログ → 設定ファイル |
| **ツールが出ない** | MCP Inspector の Tools タブ |
| **呼ばれない** | 説明文（[#10](#10-ツールはあるのにaiが呼んでくれない)） |
| **落ちる** | `console.error` のログ・stdout 混入（[#5](#5-consolelog-を書いたらサーバーが壊れた)） |

> 💡 **AIに助けを求めるときのコツ。** 「動きません」だけでは、AIはあなたのパソコンの中を見られません。**①どのコマンドで ②何と出たか（エラー全文） ③何を試したか**——この3つを貼ってください。解決の速さがまるで変わります。
> そしてMCPは動きの速い分野です。AIは**古い書き方**を自信満々に出すことがあります。「**modelcontextprotocol.io の最新の書き方で**」と添えてください。

---

## 📗 ことばメモ

| ことば | よみ | 意味 |
|---|---|---|
| **ビルド** | — | TypeScript を JavaScript に変換すること。`npm run build`。結果は `build/` に出る |
| **絶対パス** | ぜったいパス | `/` や `C:\` から始まる、フルの住所。途中から書いたものが相対パス |
| **stdout / stderr** | 標準出力／標準エラー出力 | プログラムの出口2種。stdio では stdout が通信路、stderr がログ |
| **JSON-RPC** | ジェイソンアールピーシー | MCP が中で使っている、命令と返事のやりとり形式 |
| **環境変数** | かんきょうへんすう | プログラムに外から渡す設定値。`NOTES_DIR` など |
| **zod** | ゾッド | 入力の形（型）をチェックするライブラリ。この教材では**v3** |

---

## 関連ページ

- MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 — 目次
- [第3章　標準入出力の正体 — `console.log` 1行でサーバーが壊れる](03-stdio.md)
- [第4章　デバッグの作法 — 見えない通信を覗く](04-debug.md)
- [第5章　自分の道具を作りはじめる — メモを検索する](05-memo-server.md)
- [付録C　設定ファイルの場所とトラブル](apx-c-config.md)
