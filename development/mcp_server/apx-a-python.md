---
title: "付録A　Python版で同じものを作る"
parent: "MCPサーバーを作って学ぶ AIに道具を持たせる入門"
grand_parent: "開発の心得"
nav_order: 14
nav_exclude: true
---

# 付録A　Python版で同じものを作る

> 📖 MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 ← 目次に戻る
> 本編（第5章〜第6章）で作った**メモサーバー**を、Python で書くとどうなるかを見ます。目的は「Pythonを覚える」ことではなく、**MCPが規格である**ことを実物で確かめることです。

---

## 📱 この付録の位置づけ 🟢

第1章で、MCPは**コンセントの形を決めた規格**だと言いました。プラグの形さえ合っていれば、中身が何でできていようと繋がる、と。

その言葉が本当かどうかは、**同じ道具を別の言語で書いて、同じ Claude Desktop に繋いでみれば**分かります。この付録でやるのはそれだけです。

> ✅ 結論を先に言います。**繋ぎ方も、Claude から見える姿も、まったく同じです。** 変わるのは、あなたが書くコードの見た目と、設定ファイルの `command` の行だけです。

だから——**言語は好みで選んでOK**です。Python が手に馴染んでいる人は、本編を読みながらこの付録の書き方に置き換えて進めても、最後まで同じゴールに着けます。

> 💡 本編で作るものと**仕様は完全に同じ**にします。ツールは `search_notes` / `read_note` / `write_note` の3つ、メモ置き場は環境変数 **`NOTES_DIR`**、対象は **`.md` ファイルのみ**。名前も引数名も変えません。

---

## 🤔 何が同じで、何が違う？ 🟢

先に地図を配ります。

### 同じもの（＝MCPの本体）

- **ツールの名前と引数**（`search_notes(query)` など）
- **stdio（標準入出力）でやりとりする**という繋ぎ方
- **AIが読むのは説明文だけ**という大原則
- **Claude Desktop 側の設定ファイルの場所と構造**
- **stdout に何か書くと壊れる**という罠（← ここが今回いちばん大事）

### 違うもの（＝言語の都合）

- パッケージの入れ方（`npm install` → `uv add`）
- 説明文の書き方（`.describe()` → **docstring（説明用の文字列）**）
- **ビルドが要らない**（TypeScript の `npm run build` に当たる工程がない）
- 設定ファイルの `command` が `node` ではなく **`uv`**

つまり違いは**全部「言語の作法」の側**にあります。MCPの側は一行も変わりません。

---

## 🛠 こう作る 🟢

以下は、**公式クイックスタート（[modelcontextprotocol.io/quickstart/server](https://modelcontextprotocol.io/quickstart/server) の Python タブ）で確認できた書き方**に沿っています。推測は入れていません。

### ① 必要なもの

公式が挙げている条件はこの2つです。

- **Python 3.10 以上**
- **Python MCP SDK 1.2.0 以上**

### ② uv を入れる

公式の Python 版は **`uv`**（Python のプロジェクト管理ツール）を使います。まずこれを入れます。

macOS / Linux:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Windows（PowerShell）:
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

> ⚠️ **入れたらターミナルを一度閉じて開き直してください。** そうしないと `uv` コマンドが見つかりません。公式にも明記されている注意です。

### ③ プロジェクトを作る

本編では `memo` というフォルダを作りました。Python 版でも同じ名前でいきます。

macOS / Linux:
```bash
uv init memo
cd memo

uv venv
source .venv/bin/activate

uv add "mcp[cli]"

touch memo.py
```

Windows（PowerShell）:
```powershell
uv init memo
cd memo

uv venv
.venv\Scripts\activate

uv add mcp[cli]

new-item memo.py
```

1行ずつ見ます。

| コマンド | 何をしているか |
|---|---|
| `uv init memo` | `memo` フォルダを作ってプロジェクトの土台を用意する |
| `uv venv` | このプロジェクト専用の**仮想環境**（他と混ざらない置き場）を作る |
| `source .venv/bin/activate` | 作った仮想環境に入る |
| `uv add "mcp[cli]"` | **MCPのPython用SDK**を入れる（パッケージ名は `mcp`） |
| `touch memo.py` | 中身を書くファイルを作る |

> 💡 公式の天気サーバーでは `httpx`（HTTP通信のライブラリ）も一緒に入れますが、メモサーバーは**インターネットに出ない**ので要りません。ファイルの読み書きは Python に最初から付いている機能で足ります。

### ④ サーバーの土台を書く

`memo.py` の先頭に書きます。

```python
import os
import sys
from pathlib import Path

from mcp.server.fastmcp import FastMCP

# MCPサーバー本体。"memo" が、このサーバーの名前
mcp = FastMCP("memo")

# メモ置き場。環境変数 NOTES_DIR で渡してもらう
NOTES_DIR = Path(os.environ.get("NOTES_DIR", "")).expanduser()
```

- `from mcp.server.fastmcp import FastMCP` … **FastMCP** が、Python 版の書きやすい書き方です。TypeScript の `McpServer` にあたります
- `FastMCP("memo")` … サーバーに名前を付けます。TypeScript の `new McpServer({ name: "memo", version: "1.0.0" })` と同じ役目
- `NOTES_DIR` … 第5章と同じく、**メモフォルダの場所は設定ファイルから渡してもらう**方針です。コードに直書きしません

> 📌 **FastMCP の一番おいしいところ**：公式はこう説明しています——**「FastMCP は Python の型ヒントと docstring から、ツールの定義を自動で組み立てる」**。
> つまり、TypeScript で `z.string().describe("検索したいキーワード")` と手で書いていた部分が、Python では**関数の書き方そのもの**に置き換わります。

### ⑤ `search_notes` を書く（第5章に対応）

```python
@mcp.tool()
async def search_notes(query: str) -> str:
    """メモを全文検索して、見つかったファイル名と抜粋を返す

    Args:
        query: 検索したいキーワード
    """
    if not NOTES_DIR.is_dir():
        return f"メモ置き場が見つかりません: {NOTES_DIR}"

    hits = []
    for path in sorted(NOTES_DIR.glob("*.md")):
        text = path.read_text(encoding="utf-8", errors="ignore")
        index = text.find(query)
        if index == -1:
            continue
        start = max(0, index - 40)
        excerpt = text[start:index + 120].replace("\n", " ")
        hits.append(f"{path.name}: …{excerpt}…")

    if not hits:
        return f"「{query}」に一致するメモはありませんでした。"

    return "\n---\n".join(hits)
```

ここが Python 版の核心なので、丁寧に見ます。

- **`@mcp.tool()`** … この1行が「この関数をツールとしてAIに見せる」という宣言です。TypeScript の `server.registerTool(...)` にあたります
- **関数名 `search_notes` が、そのままツール名になります**。だから名前を勝手に変えないでください
- **引数 `query: str` が、そのまま入力スキーマになります**。`str` という型ヒントが「文字列を受け取る」という情報になります
- **docstring の1行目が、ツールの説明文**になります。`Args:` 以下が引数の説明です
- **戻り値は `str`**（文字列）を返すだけでOK。TypeScript で `{ content: [{ type: "text", text: … }] }` と包んでいた部分は、FastMCP が代わりに包んでくれます

> 💡 **第1章と第5章の話をもう一度**。AIが読むのは、この **docstring だけ**です。中の `for` 文も `if` 文も、AIは1文字も見ていません。**Python に書き換えても、この事実は1ミリも変わりません。** むしろ、説明文が関数の真上に書かれるぶん、Python のほうが「説明文が主役」であることが見た目に出ています。

- `NOTES_DIR.glob("*.md")` … **`.md` ファイルだけ**を対象にします（本編と同じ仕様）
- 見つからないときも **エラーを投げず、理由を文章で返しています**。第6章で強調した作法と同じです

### ⑥ `read_note` と `write_note` を書く（第6章に対応）

```python
@mcp.tool()
async def read_note(filename: str) -> str:
    """指定したメモ1つの全文を返す

    Args:
        filename: 読みたいメモのファイル名（例: 2026-07-19.md）
    """
    if not filename.endswith(".md"):
        return "拡張子が .md のファイルだけを読めます。"

    path = NOTES_DIR / filename
    if not path.is_file():
        return f"{filename} が見つかりません。"

    return path.read_text(encoding="utf-8")


@mcp.tool()
async def write_note(filename: str, content: str) -> str:
    """メモを新しく保存する（または上書きする）

    Args:
        filename: 保存するファイル名（例: idea.md）
        content: 保存する本文（Markdown）
    """
    if not filename.endswith(".md"):
        return "拡張子が .md のファイルだけを保存できます。"

    path = NOTES_DIR / filename
    path.write_text(content, encoding="utf-8")
    return f"{filename} を保存しました。"
```

引数が2つになっても、書き方は同じです。**引数を並べれば、それがそのまま入力スキーマになります。**

> ⚠️ **このままでは第9章のぶんが足りません。** 本編の第9章では、`filename` に `../../` のような**外に出ていく指定**が来たときに拒否する仕組み（封じ込め）を足しました。Python でも同じことをします。`Path.resolve()` で実際の場所を確定させ、それが `NOTES_DIR` の**配下かどうか**を確かめてから読み書きする——考え方はまったく同じです。**この付録のコードは、第9章を読んでから必ず補強してください。**

### ⑦ 起動する部分を書く

ファイルの最後に置きます。

```python
def main():
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

`mcp.run(transport="stdio")` の一行が、TypeScript の

```typescript
const transport = new StdioServerTransport();
await server.connect(transport);
```

にあたります。**「標準入出力で話します」** という宣言です。

### ⑧ 動かしてみる

```bash
uv run memo.py
```

**何も表示されないまま止まっているように見えたら、正解です。** MCPサーバーは画面を持たないプログラムで、話しかけられるのを待っています（第1章のとおり）。`Ctrl-C` で止めてください。

> 💡 **TypeScript版と違って、`npm run build` に当たる工程がありません。** Python はそのまま実行できるからです。本編の付録Dで「ビルド忘れ」が定番エラーとして出てきますが、Python版ではその罠は最初から存在しません。

---

## ⚠️ ハマりどころ 🟢

### ① `print()` は、サーバーを壊します ★最重要

第3章でやった、あの罠です。**言語が変わっても、罠はそのまま付いてきます。**

第3章の主役は `console.log` でした。Python 版での主役は **`print()`** です。

> **`print()` は、書いたものを stdout（標準出力）に出します。**
> そして stdio サーバーにとって stdout は、**Claude との会話チャンネルそのもの**です。
> だから `print("処理中")` と1行書くだけで、その文字列が JSON-RPC のメッセージに混ざり込みます。
>
> しかも `print()` は末尾に改行を付けるので、第3章で見た**2通りの壊れ方のうち「気づきにくい方」**になりがちです。ホストによっては、その行だけ「読めなかった」と捨てて会話を続けてしまう。**エラーがログの奥に黙って積もるだけで、表面上は動いて見える**——だから気づけません。改行なしで stdout に書いた場合は、次のメッセージごと潰れて固まります。どちらにせよ、書いてはいけないことに変わりはありません。

公式ドキュメントもはっきり書いています——「**stdioベースのサーバーでは、絶対に stdout に書いてはいけない。stdout に書くと JSON-RPC メッセージが壊れ、サーバーが動かなくなる**」。

そして、**逃げ道も公式に書いてあります**。

```python
import sys
import logging

# ❌ ダメ（stdio）
print("Processing request")

# ✅ OK（stdio）
print("Processing request", file=sys.stderr)

# ✅ OK（stdio）
logging.info("Processing request")
```

`file=sys.stderr` を足すだけで、行き先が **stderr（標準エラー出力）** に変わります。stderr は会話チャンネルではないので、混ざりません。ホスト側がログとして拾ってくれます。

公式が挙げているベストプラクティスも、ひとことです——**「stderr かファイルに書くログライブラリを使うこと」**。

**TypeScript版との対応**はこうなります。頭の中でこの表を1枚持っておいてください。

| | 壊れる書き方 ❌ | 安全な書き方 ✅ |
|---|---|---|
| **TypeScript** | `console.log("started")` | `console.error("started")` |
| **Python** | `print("started")` | `print("started", file=sys.stderr)` |
| | | または `logging.info("started")` |

> 💡 **HTTP の場合は話が別**です。公式も「**HTTPベースのサーバーなら、標準出力へのログは問題ない**」と書いています。HTTP では stdout が会話に使われていないからです。第3章・第8章で扱ったのと同じ理屈です。

### ② 設定ファイルで `command` を間違える

第2章では `"command": "node"` と書きました。**Python版では `node` は動きません。** ここは次の節でまとめます。

### ③ 仮想環境の存在を忘れる

`uv add` で入れたパッケージは、**そのプロジェクトの仮想環境の中**にいます。だから、うっかり別の Python で `memo.py` を実行すると「`mcp` なんてモジュールは無い」と怒られます。

**`uv run memo.py` で起動する**——この形を守れば、uv が正しい環境を選んでくれます。設定ファイルでも同じ形を使います（次節）。

### ④ ターミナルを再起動していない

`uv` を入れた直後は、ターミナルがまだ `uv` の場所を知りません。**閉じて開き直す**。公式にもわざわざ書かれている、地味だが全員がやる失敗です。

---

## 🛠 Claude Desktop の設定ファイル 🟢

ここが、**言語を変えたときに実際に書き換わる唯一の場所**です。

ファイルの場所は本編とまったく同じです（付録Cも参照）。

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%AppData%\Claude\claude_desktop_config.json`

### TypeScript版（第2章・本編）

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

### Python版（この付録）

macOS / Linux:
```json
{
  "mcpServers": {
    "memo": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/PARENT/FOLDER/memo",
        "run",
        "memo.py"
      ],
      "env": { "NOTES_DIR": "/ABSOLUTE/PATH/TO/notes" }
    }
  }
}
```

Windows:
```json
{
  "mcpServers": {
    "memo": {
      "command": "uv",
      "args": [
        "--directory",
        "C:\\ABSOLUTE\\PATH\\TO\\PARENT\\FOLDER\\memo",
        "run",
        "memo.py"
      ],
      "env": { "NOTES_DIR": "C:\\ABSOLUTE\\PATH\\TO\\notes" }
    }
  }
}
```

見比べてください。**変わったのは `command` と `args` の中身だけ**です。

| | TypeScript版 | Python版 |
|---|---|---|
| `command` | `node` | `uv` |
| `args` | ビルド済み `build/index.js` の**絶対パス** | `--directory` + プロジェクトフォルダの**絶対パス** + `run` + `memo.py` |
| `env` | 同じ | 同じ |

`--directory` は「**このフォルダのプロジェクトとして実行してね**」という指定です。これがあるおかげで、uv が仮想環境を正しく見つけてくれます。

> ⚠️ **本編と同じ注意が、そのまま全部当てはまります。**
> - **絶対パス必須**（Claude Desktop が起動するときのカレントディレクトリは決まっていません）
> - **環境変数は自動で引き継がれません** → `env` キーで `NOTES_DIR` を渡す
> - **書き換えたら Claude Desktop を完全終了して再起動**（ウィンドウを閉じるだけでは足りません）
>
> くわしくは [付録C　設定ファイルの場所とトラブル](apx-c-config.md) を見てください。**トラブルの中身は言語によらず共通です。**

> 💡 **Windows で `uv` が見つからないと言われたら**、`command` に `uv` のフルパスを書く手があります。ターミナルで `where uv`（macOS/Linux なら `which uv`）と打つと場所が分かります。

---

## 📊 TypeScript版 ↔ Python版 対応表 🟢

**この表がこの付録の本体**です。同じ概念が、言語でどう書き分けられるかを並べました。

| やりたいこと | TypeScript（本編） | Python（この付録） |
|---|---|---|
| SDKを入れる | `npm install @modelcontextprotocol/sdk zod@3` | `uv add "mcp[cli]"` |
| SDKを読み込む | `import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"` | `from mcp.server.fastmcp import FastMCP` |
| サーバーを作る | `new McpServer({ name: "memo", version: "1.0.0" })` | `FastMCP("memo")` |
| ツールを登録する | `server.registerTool("search_notes", {…}, handler)` | `@mcp.tool()` を関数の上に付ける |
| ツール名を決める | 第1引数の文字列 `"search_notes"` | **関数名がそのままツール名** |
| ツールの説明を書く | `description: "メモを全文検索して…"` | **docstring の1行目** |
| 引数の型を決める | `inputSchema: { query: z.string() }` | **型ヒント** `query: str` |
| 引数の説明を書く | `.describe("検索したいキーワード")` | **docstring の `Args:` の行** |
| 結果を返す | `{ content: [{ type: "text", text: "…" }] }` | **`str` を `return` するだけ** |
| stdio で起動する | `new StdioServerTransport()` → `server.connect(transport)` | `mcp.run(transport="stdio")` |
| ログを出す（安全） | `console.error("…")` | `print("…", file=sys.stderr)` / `logging.info("…")` |
| ログを出す（**壊れる**） | `console.log("…")` ❌ | `print("…")` ❌ |
| ビルド | `npm run build` が**必要** | **不要**（そのまま実行できる） |
| 手動で起動して試す | `node build/index.js` | `uv run memo.py` |
| 設定ファイルの `command` | `"node"` | `"uv"` |

眺めていて気づくことがあると思います。

> **左の列と右の列は、書き方が違うだけで、言っていることが1対1で対応しています。**

これが「規格である」ということです。MCPが決めているのは**この表の一番左の列（やりたいこと）**だけで、それをどう書くかは各言語のSDKに任されている。だから**どちらで書いても、Claude から見える姿は同じ**になります。

---

## 🔧 MCP Inspector で確かめる 🔧

第4章で使ったデバッグ道具は、Python 版でもそのまま使えます。**Inspector は言語を気にしません**——「stdio で起動できるプログラム」でありさえすればいいからです。

```bash
npx @modelcontextprotocol/inspector uv --directory /ABSOLUTE/PATH/TO/memo run memo.py
```

TypeScript版で

```bash
npx @modelcontextprotocol/inspector node /絶対パス/memo/build/index.js
```

と書いていたところの、`node …` の部分を **設定ファイルに書いたのと同じ起動コマンド**に差し替えるだけです。

Tools タブを開くと、`search_notes` / `read_note` / `write_note` の3つが並び、**docstring に書いた説明文がそのまま表示されている**はずです。ここで「AIにはこう見えている」を自分の目で確認できます。

---

## 🤖 AIに頼むなら 🟢

「本編のTypeScriptコードをPythonに直して」とAIに頼むのは、**この付録でいちばん相性のいい使い方**です。やることが1対1で対応しているので、AIも間違えにくい。

ただし、第1章でも書いたとおり **MCPは動きの速い分野で、AIは古い書き方を自信満々に出してくること**があります。頼むときは条件を添えてください。

> **「MCPの公式ドキュメント（modelcontextprotocol.io）の最新の書き方で」**
> **「Python の公式SDK（`mcp` パッケージ）の `FastMCP` を使って」**
> **「stdio で動かすので、`print()` は使わず `print(..., file=sys.stderr)` にして」**

3つめは特に大事です。**AIは何も言わないと、デバッグ用に `print()` を平気で入れてきます。** そしてそれは、動かないサーバーです。

出てきたコードを受け取ったら、**まず `print(` を検索してください。** `file=sys.stderr` が付いていない `print` が1つでもあったら、それが犯人です。

---

## 📗 ことばメモ

| ことば | よみ | 意味 |
|---|---|---|
| **uv** | ユーブイ | Python のプロジェクト・パッケージ管理ツール。公式のPython版で使われている |
| **FastMCP** | ファストエムシーピー | Python版SDKの、書きやすいほうの書き方。型ヒントと docstring からツール定義を自動で作る |
| **docstring** | ドックストリング | 関数の中身の先頭に書く説明用の文字列。**MCPではこれがAIの読む説明文になる** |
| **型ヒント** | かたヒント | `query: str` のように、引数の種類を書き添える書き方。入力スキーマの元になる |
| **仮想環境** | かそうかんきょう | プロジェクト専用のパッケージ置き場。他のプロジェクトと混ざらないようにするもの |
| **stdout** | エスティーディーアウト／標準出力 | プログラムの出力の口。**stdioサーバーではここに書くと壊れる** |
| **stderr** | エスティーディーエラー／標準エラー出力 | エラーやログを流す別の口。**stdioサーバーではこちらが安全** |

---

## ➡️ 次へ

Python 版でも、同じ道具が、同じように Claude に繋がりました。第1章の「**コンセントの形さえ合えばいい**」というたとえは、比喩ではなく**そのままの事実**だった、ということです。

そして、繋ぐ先も Claude Desktop だけではありません。[付録B　Claude Code に繋ぐ](apx-b-claude-code.md) では、いま作ったサーバー（TypeScript版でも Python版でも構いません）を、ターミナルで動く Claude Code から使えるようにします。

**道具は1回作れば済む**——第13章まで読んだあとに、この付録に戻ってくると、その意味がもう一段はっきりすると思います。

## 関連ページ

- MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 — 目次
- [第3章　標準入出力の正体 — `console.log` 1行でサーバーが壊れる](03-stdio.md) — この付録の「`print()` の罠」の元ネタ
- [第5章　自分の道具を作りはじめる — メモを検索する](05-memo-server.md) — TypeScript版の `search_notes`
- [第6章　道具を増やす — 読む・書く](06-more-tools.md) — TypeScript版の `read_note` / `write_note`
- [第9章　安全 — 書ける道具は、危ない道具](09-safety.md) — Python版でも必ず足すこと
- [付録C　設定ファイルの場所とトラブル](apx-c-config.md) — 設定まわりのトラブルは言語共通
