# 付録C　ツール定義（JSON Schema）の書き方ミニ辞典

> 📖 この付録のゴール：道具(tool)を渡すときの **`input_schema`（＝引数の設計図）** を、こわがらずに書けるようになる。「どんな形の引数を、どう説明すればLLMが正しく道具を使ってくれるか」を、本編の道具（`read_file` / `write_file` / `run_command`）を題材に、1行ずつ読み解きます。
> [← 目次・はじめにへもどる](README.md)

---

本編の第3章で、道具はこんな形で渡しました。

```ts
const tools: Anthropic.Tool[] = [{
  name: "get_current_time",
  description: "今の日時を返す。引数は不要。",
  input_schema: {
    type: "object",
    properties: {},
    required: [],
  },
}];
```

この `input_schema` の部分が、今回の主役です。**「引数(ひきすう)＝道具に渡す材料」の形を、機械が読める言葉で書いたもの**で、その言葉が **JSON Schema（ジェイソン・スキーマ）** です。

> 🍳 たとえ：道具は「電話の向こうの天才シェフ（LLM）」に渡すレシピカードです。`name`＝道具の名前、`description`＝使い方の説明、`input_schema`＝**「材料はこの形でちょうだいね」という注文票**。注文票がちゃんとしているほど、シェフは間違った材料を送ってこなくなります。

---

## ① いちばん基本：3点セット 🟢

JSON Schema は見た目はゴツいですが、**最初に覚えるのはたった3つ**だけです。

```ts
input_schema: {
  type: "object",
  properties: {
    path: { type: "string", description: "読みたいファイルのパス" },
  },
  required: ["path"],
}
```

**1行ずつ読むと：**
- `type: "object"` … 引数全体は **「名前つきの箱の集まり」** ですよ、という宣言。道具の引数は**ほぼ必ずこれ**で始まります（材料を `path` のような名札つきで渡すから）。
- `properties: { ... }` … その箱の中に**どんな名札の材料が入るか**を並べる場所。ここに `path` を1つ定義しました。
- `path: { type: "string", ... }` … `path` という材料は **文字列(string)** ですよ、という指定。
- `description: "..."` … その材料が**何なのか**の説明（後で詳しく。これがいちばん大事）。
- `required: ["path"]` … **絶対に欠かせない材料の名札リスト**。`path` が無いとこの道具は動かない、という意味。

> 💡 おぼえ方：**`type` で「箱」と言って、`properties` で「中身の名札と型」を並べて、`required` で「絶対いる名札」を挙げる**。この3点セットがJSON Schemaの土台です。

引数がまったく要らない道具（さっきの `get_current_time` など）は、`properties` を空の `{}`、`required` を空の `[]` にします。**「箱はあるけど、中に入れる材料はゼロ」**という意味です。

---

## ② 材料の「型」いろいろ 🟢

材料（プロパティ）には **型(かた)** ＝「どんな種類のデータか」を指定します。よく使うのは次の6つです。

| 型の書き方 | 意味 | 例 |
|---|---|---|
| `"string"` | 文字列 | `"README.md"`、`"こんにちは"` |
| `"number"` | 数（小数もOK） | `3.14`、`42` |
| `"integer"` | 整数（小数はダメ） | `1`、`100` |
| `"boolean"` | はい/いいえ（真偽） | `true` / `false` |
| `"array"` | 並び（リスト） | `["a", "b", "c"]` |
| `"object"` | 名前つきの箱（入れ子） | `{ city: "東京" }` |

書き方はどれも `{ type: "型の名前", description: "説明" }` の形でそろっています。

```ts
properties: {
  path:      { type: "string",  description: "ファイルのパス" },
  max_lines: { type: "integer", description: "読み込む最大行数" },
  overwrite: { type: "boolean", description: "既存ファイルを上書きしてよいか" },
}
```

**1行ずつ読むと：**
- `path` … 文字列。ファイルの場所を文字でもらう。
- `max_lines` … **整数(integer)**。行数は「2.5行」みたいな小数だと困るので `number` ではなく `integer` を選びます。**「ここは整数しか来ないでほしい」をスキーマで伝えられる**のがポイント。
- `overwrite` … **真偽(boolean)**。`true`（上書きする）か `false`（しない）の二択。

> 🔧 `number` と `integer` の違い：`number` は小数も整数も許す広い型、`integer` は整数だけ。**「個数」「行数」「ページ番号」のように小数があり得ない材料は `integer`** にしておくと、シェフが `1.5` のような変な値を送りにくくなります。

---

## ③ いちばん大事：`description` はLLMへの指示そのもの 🟢

ここが付録Cで**いちばん覚えてほしい**ところです。

> 🗣 **`name` と `description` は、ただのメモではありません。LLMがそれを読んで「いつ・どの道具を・どんな引数で使うか」を決める“指示文”そのもの**です。

LLMはあなたのコードの中身を見ていません。**見えているのは `name` と `description` の文章だけ**。だから——

- **道具名（`name`）が曖昧** → LLMが「どの道具を使えばいいか」を間違える
- **引数の `description` が雑** → LLMが**変な値**を入れてくる、または**必要な道具を呼ばない**

逆に言えば、**説明をていねいに書くほど、道具は正しく使われます**。プロンプトを書くのと同じ気持ちで、説明文を書いてください。

### よい例 / わるい例

同じ「ファイルを読む道具」でも、説明しだいで賢さが変わります。

```ts
// 😖 わるい例：曖昧で、何を渡せばいいか分からない
{
  name: "read",
  description: "読む",
  input_schema: {
    type: "object",
    properties: { p: { type: "string", description: "値" } },
    required: ["p"],
  },
}
```

```ts
// 😀 よい例：いつ使うか・何を渡すかが具体的
{
  name: "read_file",
  description:
    "テキストファイルの中身を読んで返す。中身を確認・編集したいときに使う。" +
    "引数 path には、作業フォルダからの相対パス（例: src/index.ts）を渡す。",
  input_schema: {
    type: "object",
    properties: {
      path: {
        type: "string",
        description: "読みたいファイルのパス。例: README.md, src/index.ts",
      },
    },
    required: ["path"],
  },
}
```

**1行ずつ読むと（よい例）：**
- `name: "read_file"` … **動詞＋目的語**で具体的。`read` だけだと「何を読む？」が伝わらないので `read_file` にします。
- `description` の前半 … **「いつ使うか」**（中身を確認・編集したいとき）を書く。LLMは“使いどころ”が分かると道具を取り違えません。
- `description` の後半 … **「何を渡すか」**（作業フォルダからの相対パス、例つき）を書く。例があると形を間違えにくい。
- `path` の `description` … プロパティ側にも**具体例**を添える。`description: "値"` のような中身ゼロの説明は、無いのと同じです。

> 🍳 たとえ：シェフへの注文票に「材料：適量」とだけ書いたら、何がどれだけ来るか分かりません。「**塩：小さじ1（粒の細かいもの）**」のように具体的に書くほど、思いどおりの料理（道具の使われ方）になります。

> 💡 こつ：説明はLLM（読み手）目線で書く。「この道具は**いつ使うべきで／いつ使うべきでないか**」「引数は**どんな形式・単位か**」「**禁止事項**（例：作業フォルダの外は読まない）」まで書けると、誤用がぐっと減ります。

---

## ④ 選択肢を絞る `enum`・並びの `array`・入れ子 🔧

基本の3点セットに慣れたら、引数を**もっと正確に**縛れる道具を覚えましょう。

### `enum`：選べる値を決まったものだけにする

「上書きモードは `overwrite` か `append` の**どちらか**しか許さない」——そんなときは `enum`（イーナム＝**選択肢**）です。

```ts
properties: {
  mode: {
    type: "string",
    enum: ["overwrite", "append"],
    description: "書き込み方法。overwrite=丸ごと置換 / append=末尾に追記",
  },
}
```

**1行ずつ読むと：**
- `type: "string"` … この材料は文字列。
- `enum: ["overwrite", "append"]` … **この2つ以外の文字列はダメ**、という制限。LLMが `"replace"` のような勝手な値を作っても弾けます。
- `description` … それぞれの選択肢が**何を意味するか**を説明。`enum` だけだと言葉の意味までは伝わらないので、ここで補足します。

> 💡 `enum` は**タイプミスや表記ゆれを防ぐ最強の道具**。「決まった値しか来ないはず」の引数には積極的に使いましょう。

### `array`：並び（リスト）には `items` で中身の型を書く

複数のファイルをまとめて渡したいときは `array`（配列＝並び）。**並びの中の1個1個がどんな型か**を `items` で指定します。

```ts
properties: {
  paths: {
    type: "array",
    items: { type: "string" },
    description: "読みたいファイルのパスの一覧。例: ["a.ts", "b.ts"]",
  },
}
```

**1行ずつ読むと：**
- `type: "array"` … この材料は**並び（リスト）**ですよ、という宣言。
- `items: { type: "string" }` … **その並びの中身は、ぜんぶ文字列**ですよ、という指定。`items` を書き忘れると「中身が何の並びか」が伝わりません。
- `description` … 例つきで「ファイルのパスの一覧」と説明。

### 入れ子（ネスト）：箱の中に箱を入れる

`object` の中にさらに `object` を入れて、**まとまった材料**を渡すこともできます。

```ts
properties: {
  target: {
    type: "object",
    properties: {
      path:    { type: "string",  description: "書き込み先のパス" },
      content: { type: "string",  description: "書き込む中身" },
    },
    required: ["path", "content"],
    description: "書き込み対象（場所と中身のセット）",
  },
}
```

**1行ずつ読むと：**
- `target: { type: "object", ... }` … `target` という材料が**さらに箱**になっている（入れ子）。
- 中の `properties` … その箱の中身（`path` と `content`）を、いつもの3点セットで定義。
- 中の `required: ["path", "content"]` … **箱の中でも「絶対いる材料」を指定できる**。外側と同じルールが、入れ子の中でも効きます。

> ⚠️ 入れ子は深くしすぎないこと。**1〜2段まで**が読みやすさの目安です。深くするほどLLMも人間も形を間違えやすくなります。本編の道具は基本フラット（入れ子なし）で十分です。

---

## ⑤ 🔧 strict tool use：形を「厳密」にする

ここまでのスキーマは「**こういう形で来てね**」というお願いでした。さらに一歩進めて、「**この形以外は受け付けない**」と厳しくしたいときは、**strict tool use（ストリクト＝厳密モード）** を使います。

ねらいは2つ：**①余計な材料を混ぜさせない／②必須の材料を必ず付けさせる**。組み合わせは次の3点セットです。

```ts
{
  name: "write_file",
  description: "テキストファイルに中身を書き込む（上書き）。",
  input_schema: {
    type: "object",
    properties: {
      path:    { type: "string", description: "書き込み先のパス" },
      content: { type: "string", description: "書き込む中身（全文）" },
    },
    required: ["path", "content"],
    additionalProperties: false,
  },
  strict: true,
}
```

**1行ずつ読むと：**
- `required: ["path", "content"]` … **すべての材料を必須**にする。strict では「全部 `required`」が基本ルールです（省略可の材料を作らない）。
- `additionalProperties: false` … **`properties` に書いていない材料は禁止**。LLMが勝手に `mode` などを足してきても弾かれます。
- `strict: true` … 上の2つとセットで、「**スキーマぴったりの形しか通さない**」を有効にするスイッチ。

> 🛡 背骨②（安全）とのつながり：strict は「LLMが余計な引数を紛れ込ませる」事故を**入口で**止めてくれます。とくに `write_file` や `run_command` のような**危険な道具**ほど、形を厳密に固めておくと安心です。ただし `strict: true` を使うと**必須にできない（省略OKの）引数が作りにくくなる**ので、「全部必須でいい道具」から試すのがおすすめ。

> 💡 strict は「あれば便利な安全弁」。本編は**まず strict 無し**で動かして仕組みを理解し、**慣れたら危険な道具から strict を足す**、という順番でOKです。

---

## ⑥ まとめ：本編の道具を“スキーマ目線”で読む 🟢

最後に、本編で作る3つの道具のスキーマを並べて、**型・必須・説明の決め方**を振り返ります。

```ts
const tools: Anthropic.Tool[] = [
  {
    name: "read_file",
    description: "テキストファイルの中身を読んで返す。中身を確認したいときに使う。",
    input_schema: {
      type: "object",
      properties: {
        path: { type: "string", description: "読みたいファイルのパス（作業フォルダからの相対パス）" },
      },
      required: ["path"],
    },
  },
  {
    name: "write_file",
    description: "テキストファイルに中身を書き込む（上書き）。新規作成・修正に使う。",
    input_schema: {
      type: "object",
      properties: {
        path:    { type: "string", description: "書き込み先のパス" },
        content: { type: "string", description: "書き込む中身（ファイル全文）" },
      },
      required: ["path", "content"],
    },
  },
  {
    name: "run_command",
    description: "シェルコマンドを実行して、その出力を返す。テスト実行やビルドに使う。",
    input_schema: {
      type: "object",
      properties: {
        command: { type: "string", description: "実行するコマンド。例: npm test, ls -la" },
      },
      required: ["command"],
    },
  },
];
```

**1行ずつ読むと：**
- `read_file` … 材料は `path`（文字列）1つ、必須。読むだけなので**安全**な道具。
- `write_file` … 材料は `path` と `content`（どちらも文字列）、**両方とも必須**。中身を全文もらうので `content` の説明に「ファイル全文」と明記します（**危険**：上書き）。
- `run_command` … 材料は `command`（文字列）1つ、必須。説明に**使いどころ（テスト・ビルド）と例**を入れて、何でもかんでも実行させない方向に誘導します（**いちばん危険**）。

> 🍳 たとえで総まとめ：JSON Schema は「シェフへの注文票」。**`type` で材料の種類を、`required` で絶対いる材料を決め、`description` でその材料が何かを“言葉で”伝える**。そして `enum`／`array`／strict は、注文票をより正確にする“追記”です。注文票がていねいなほど、道具は思いどおりに、安全に使われます。

---

> 📌 もっと知りたい人へ：JSON Schema には他にも `minimum` / `maximum`（数の範囲）、`minLength` / `maxLength`（文字数）、`pattern`（正規表現で形を縛る）などがあります。ただし**本編のCLIエージェントはここまでの基本でじゅうぶん**。まずは「3点セット＋ていねいな `description`」を体にしみこませましょう。

[← 目次・はじめにへもどる](README.md)
