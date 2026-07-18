---
title: "第2章　まず公式のサンプルを動かす — 天気サーバーをClaudeに繋ぐ"
parent: "MCPサーバーを作って学ぶ AIに道具を持たせる入門"
grand_parent: "開発の心得"
nav_order: 2
---

# 第2章　まず公式のサンプルを動かす — 天気サーバーをClaudeに繋ぐ

> 📖 MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 ← 目次に戻る
> 第1章では地図を配りました。この章から**手を動かします**。MCP公式チュートリアルの「天気サーバー」をそのまま写経して、あなたの Claude Desktop に繋ぎます。

---

> ⚠️ **先に、隠さず言っておきます。このサンプルは米国専用です。**
> これから作る天気サーバーは、アメリカの **国立気象局（NWS）** の公開API（`api.weather.gov`）を使います。だから——**東京の天気は取れません。** 大阪も、ロンドンも、取れません。
>
> 「じゃあ意味ないじゃん」と思いましたか？ いいえ、**この“取れない”を体験することが、この章のいちばん大事な狙い**です。なぜ取れないのか、そのときAIは何を考えているのか。ここが分かると、第5章から自分の道具を作る理由が腹に落ちます。だから最後まで付き合ってください。

---

## 📱 Claude ではこう見える 🟢

この章が終わると、あなたの Claude Desktop はこうなります。

入力欄の下あたりに、**接続されている道具のアイコン**が出ます。そこを開くと、こう並んでいるはずです。

```
weather
  ├ get_alerts     Get weather alerts for a state
  └ get_forecast   Get weather forecast for a location
```

そして、こう聞くと——

> 「テキサス州で今出ている気象警報を教えて」

Claude は「**weather の get_alerts を使っていいですか？**」と**許可を求めてきます**。許可すると、裏であなたのパソコンの中のプログラムが起動し、アメリカ国立気象局に問い合わせ、返ってきた文字列を Claude が日本語でまとめてくれます。

ここで起きていることを、第1章の地図に当てはめておきましょう。

| 役 | この章での正体 |
|---|---|
| **ホスト** | Claude Desktop |
| **クライアント** | Claude Desktop の中にいる通訳係（あなたは触らない） |
| **サーバー** | **これから書く `weather/build/index.js`** ← あなたの担当 |

> 💡 **許可を求められるのが正常です。** ツールは「AIが判断して実行するもの」なので、勝手に実行されないよう、ホストが人間に確認します。うるさく感じても、そこは安全のための設計です。

---

## 🤔 なぜ、いきなり自分のサーバーを作らないのか 🟢

「メモを検索するサーバーを作りたいのに、なんで天気？」——もっともな疑問です。理由は3つあります。

### ① 「動く状態」を1回作っておくと、切り分けができる

MCPは目に見えません。あなたが自分のサーバーを書いて動かなかったとき、原因の候補はこれだけあります。

- コードが間違っている
- ビルドを忘れている
- 設定ファイルのパスが違う
- Claude Desktop を再起動していない
- そもそも Claude Desktop 側の問題

**公式サンプルが1回でも動いた**なら、後ろの3つは「自分にもできる」と分かっています。**疑う範囲がぐっと狭まる**。これが効きます。

### ② 「他人の完成品」は、写経の教科書になる

このコードは公式が書いたものです。**書き方の型**——ツールの登録のしかた、返り値の形、エラーの扱い——が全部入っています。第5章で自分のサーバーを書くとき、あなたはこの型をそのまま流用します。

### ③ そして、**わざと限界にぶつかってもらうため**

これが本命です。この天気サーバーには**はっきりした限界**があります。それを自分の目で見てもらいます。

### サボるとどうなるか

「サンプルなんて飛ばして自分のを作る」と進むと、ほぼ確実にこうなります。

> 3時間ハマった末に、原因が「`npm run build` を忘れていた」だった。

これは煽りではありません。**全員が一度は通ります。** 先に安全な題材で通っておけば、傷は浅く済みます。

---

## 🛠 こう作る 🟢

ここからは手を動かします。**上から順に、そのまま**やってください。

### ステップ0：Node.js があるか確認する

ターミナル（Windows なら PowerShell）を開いて、こう打ちます。

```bash
node --version
npm --version
```

バージョン番号が2つとも出れば OK です。`command not found` と出たら、まず [Node.js](https://nodejs.org/) を入れてください。**Node.js 18以上**が必要です。

> 💡 公式クイックスタートには「16以上」と書かれていますが、**SDK（`@modelcontextprotocol/sdk` 1.29.0）の `package.json` は `engines.node` に `>=18` を要求しています**。ドキュメントの記述がSDKの実要件に追いついていないので、この教材では **18以上**でいきます。

### ステップ1：プロジェクトを作る

**macOS / Linux**:

```bash
# プロジェクト用のフォルダを作る
mkdir weather
cd weather

# npm プロジェクトとして初期化する
npm init -y

# 必要なライブラリを入れる
npm install @modelcontextprotocol/sdk zod@3
npm install -D @types/node typescript

# ソースファイルを置く場所を作る
mkdir src
touch src/index.ts
```

**Windows（PowerShell）**:

```powershell
md weather
cd weather
npm init -y
npm install @modelcontextprotocol/sdk zod@3
npm install -D @types/node typescript
md src
new-item src\index.ts
```

コマンドの意味を1行ずつ。

| コマンド | 何をしている？ |
|---|---|
| `npm init -y` | `package.json`（プロジェクトの設定書）を、質問なしで作る |
| `@modelcontextprotocol/sdk` | **MCPの公式ライブラリ**。これが本体 |
| `zod@3` | **入力の形をチェックするライブラリ**。「この引数は文字列です」と宣言するのに使う |
| `@types/node typescript` | TypeScript を書いてJavaScriptに変換するための道具。`-D` は「開発中だけ使う」印 |

> ⚠️ **`zod@3` の `@3` を落とさないでください。** zod にはバージョン4があり、**書き方が違います**。うっかり v4 が入ると、この章のコードはエラーになります。AIにコードを書かせたときも、ここは要注意ポイントです。

### ステップ2：`package.json` を書き換える

`npm init -y` が作った `package.json` を開いて、次の3つを足します。

```json
{
  "type": "module",
  "bin": {
    "weather": "./build/index.js"
  },
  "scripts": {
    "build": "tsc && chmod 755 build/index.js"
  },
  "files": ["build"]
}
```

1行ずつ。

- **`"type": "module"`** … 「このプロジェクトは新しい書き方（`import` を使うほう）で書きます」という宣言です。**これが無いと `import` の行でエラーになります。**
- **`"bin"`** … このプログラムの実行の入口はここですよ、という登録。今回は無くても動きますが、公式に合わせておきます。
- **`"scripts": { "build": ... }`** … `npm run build` と打ったとき何をするかの定義。`tsc` が TypeScript を JavaScript に変換し、`chmod 755` が「実行してよい」印を付けます（Windows では `chmod` の部分は無視されるか、エラーになった場合は `"build": "tsc"` だけにしてください）。
- **`"files"`** … 配布するとき何を含めるか。今回は関係ありません。

### ステップ3：`tsconfig.json` を作る

プロジェクトの一番上（`weather/` の直下）に **`tsconfig.json`** という新しいファイルを作り、まるごとコピーしてください。

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

全部は覚えなくていいです。**効いてくるのはこの3つ**だけ覚えてください。

| 設定 | 意味 | なぜ大事か |
|---|---|---|
| `"module": "Node16"` | Node.js 流の読み込み方式を使う | このせいで **import の末尾に `.js` が必須**になります（後述） |
| `"rootDir": "./src"` | 元のコードは `src/` にある | |
| `"outDir": "./build"` | 変換後のコードは `build/` に出す | **Claude Desktop の設定が指すのはこっち**。`src` ではありません |

> 💡 ここが初心者の最大の混乱ポイントです。**あなたが書くのは `src/index.ts`。Claude が動かすのは `build/index.js`。** 別のファイルです。だから「編集したのに反映されない」＝「変換（ビルド）していない」なのです。

### ステップ4：サーバーの土台を書く

ここから `src/index.ts` を書いていきます。**上から順に、追記していってください**（最後に全部つながって1つのファイルになります）。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const NWS_API_BASE = "https://api.weather.gov";
const USER_AGENT = "weather-app/1.0";

// サーバーの本体を作る
const server = new McpServer({
  name: "weather",
  version: "1.0.0",
});
```

1行ずつ見ます。

- **`McpServer`** … サーバー本体を作るための部品。「道具箱そのもの」だと思ってください。
- **`StdioServerTransport`** … **stdio（標準入出力）で話すための部品**。第1章で「Claude はあなたのプログラムを起動して、入力の口と出力の口で会話する」と説明した、あの通信路です。
- **`z`** … zod のこと。引数の形を宣言するのに使います。
- **`import` の末尾の `.js`** … `mcp.js`、`stdio.js` の `.js` を**絶対に消さないでください**。書いているのは TypeScript（`.ts`）なのに `.js` と書くのは奇妙に見えますが、`"module": "Node16"` ではこれが正解です。落とすと `Cannot find module` で落ちます。
- **`NWS_API_BASE`** … アメリカ国立気象局（National Weather Service）のAPIの住所です。**ここが「米国専用」の正体**。
- **`USER_AGENT`** … 「誰がアクセスしているか」の名乗り。NWS のAPIは名乗りを求めるので付けます。
- **`new McpServer({ name, version })`** … サーバーに名前を付けます。この `name` が Claude 側の一覧に出る名前になります。

### ステップ5：お手伝い関数を書く

天気APIを叩いて、結果を読みやすい文字列にする部分です。MCPそのものとは関係ない「ふつうのプログラム」なので、ここは流し読みで構いません。

```typescript
// NWS API にリクエストを投げる共通関数
async function makeNWSRequest<T>(url: string): Promise<T | null> {
  const headers = {
    "User-Agent": USER_AGENT,
    Accept: "application/geo+json",
  };

  try {
    const response = await fetch(url, { headers });
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return (await response.json()) as T;
  } catch (error) {
    console.error("Error making NWS request:", error);
    return null;
  }
}
```

- `fetch(url, { headers })` … インターネットにデータを取りに行きます。
- `response.ok` … 失敗（404など）なら例外を投げます。
- **`console.error`** … ⚠️ **ここが `console.log` ではないことに注目してください。** 第1章で予告した罠です。stdio では**標準出力（`console.log`）は Claude との会話チャンネルそのもの**なので、そこに文字を書くとサーバーが壊れます。**`console.error`（標準エラー出力）は安全**で、ホストがログとして拾ってくれます。第3章で実際に壊してもらいます。
- 失敗したら `null` を返します。**例外を投げっぱなしにしない**——これも後で効いてきます。

続いて、返ってきたデータの形を宣言し、整形する関数を書きます。

```typescript
interface AlertFeature {
  properties: {
    event?: string;
    areaDesc?: string;
    severity?: string;
    status?: string;
    headline?: string;
  };
}

// 警報1件を読みやすい文字列にする
function formatAlert(feature: AlertFeature): string {
  const props = feature.properties;
  return [
    `Event: ${props.event || "Unknown"}`,
    `Area: ${props.areaDesc || "Unknown"}`,
    `Severity: ${props.severity || "Unknown"}`,
    `Status: ${props.status || "Unknown"}`,
    `Headline: ${props.headline || "No headline"}`,
    "---",
  ].join("\n");
}

interface ForecastPeriod {
  name?: string;
  temperature?: number;
  temperatureUnit?: string;
  windSpeed?: string;
  windDirection?: string;
  shortForecast?: string;
}

interface AlertsResponse {
  features: AlertFeature[];
}

interface PointsResponse {
  properties: {
    forecast?: string;
  };
}

interface ForecastResponse {
  properties: {
    periods: ForecastPeriod[];
  };
}
```

- **`interface`** … 「このデータにはこういう項目が入っている」という説明書きです。TypeScript が間違いを見つけるために使います。
- **`?`** … 「無いかもしれない」印。天気APIは項目が欠けることがあるので、こう書いておきます。
- **`|| "Unknown"`** … 無かったら "Unknown" と書く、という保険。**AIに渡す文字列は、空にせず「分からない」と書く**ほうが親切です。

> 💡 注目してほしいのは、返しているのが**きれいなJSONではなく、人間が読める文章**だということです。**受け取るのはAI**なので、AIが読みやすい形——つまり素直な文章——がいちばん良いのです。これは第5章以降でずっと使う考え方です。

### ステップ6：道具その1 `get_alerts` を登録する 🟢★

**この章のいちばん大事なコード**です。よく見てください。

```typescript
server.registerTool(
  "get_alerts",
  {
    description: "Get weather alerts for a state",
    inputSchema: {
      state: z
        .string()
        .length(2)
        .describe("Two-letter state code (e.g. CA, NY)"),
    },
  },
  async ({ state }) => {
    const stateCode = state.toUpperCase();
    const alertsUrl = `${NWS_API_BASE}/alerts?area=${stateCode}`;
    const alertsData = await makeNWSRequest<AlertsResponse>(alertsUrl);

    if (!alertsData) {
      return {
        content: [
          {
            type: "text",
            text: "Failed to retrieve alerts data",
          },
        ],
      };
    }

    const features = alertsData.features || [];
    if (!features.length) {
      return {
        content: [
          {
            type: "text",
            text: `No active alerts for ${stateCode}`,
          },
        ],
      };
    }

    const formattedAlerts = features.map(formatAlert);
    const alertsText = `Active alerts for ${stateCode}:\n\n${formattedAlerts.join("\n")}`;

    return {
      content: [
        {
          type: "text",
          text: alertsText,
        },
      ],
    };
  },
);
```

`registerTool` は**引数を3つ**取ります。この3つの構造だけは、しっかり覚えてください。**この先ずっと同じ形です。**

| 引数 | 中身 | 誰のためのもの？ |
|---|---|---|
| **①名前** | `"get_alerts"` | **AIが読む**。呼ぶときの識別子 |
| **②説明の束** | `description` と `inputSchema` | **AIが読む** ←★ここが全て |
| **③中身の関数** | `async ({ state }) => {...}` | **AIは読まない**。実際に動く処理 |

とくに②の中を見てください。

- **`description: "Get weather alerts for a state"`** … 「州の気象警報を取ります」。**AIはこの一文を読んで、この道具を使うかどうか決めます。**
- **`inputSchema`** … 受け取る引数の宣言です。ここでは `state` が1つだけ。
- **`z.string()`** … 文字列であること。
- **`.length(2)`** … **ちょうど2文字**であること。
- **`.describe("Two-letter state code (e.g. CA, NY)")`** … 「2文字の州コード（例：CA、NY）」。**★この一文を覚えておいてください。この章の山場で戻ってきます。**

> ⚠️ **`inputSchema` は `z.object({...})` で包みません。** zod のフィールドをそのまま並べた**素のオブジェクト**です。AIに書かせると `z.object()` で包んでくることがありますが、それは古い（あるいは別の）書き方です。

③の処理側で、もうひとつ大事な作法があります。

> **失敗しても `throw` していません。** データが取れなければ `"Failed to retrieve alerts data"` という**文章を返して**います。警報が0件なら `"No active alerts for TX"` と返します。
> なぜか。**受け取るのはAIだから**です。例外で落ちるとAIには「なんか失敗した」としか伝わりませんが、文章で返せば、AIはそれを読んで「じゃあ別の州を試そう」「ユーザーに聞き返そう」と**次の手を考えられます**。

そして返り値の形。**これも全部同じです。**

```typescript
{ content: [{ type: "text", text: "…文字列…" }] }
```

### ステップ7：道具その2 `get_forecast` を登録する

もう1つの道具です。構造はまったく同じなので、違うところだけ見ます。

```typescript
server.registerTool(
  "get_forecast",
  {
    description: "Get weather forecast for a location",
    inputSchema: {
      latitude: z
        .number()
        .min(-90)
        .max(90)
        .describe("Latitude of the location"),
      longitude: z
        .number()
        .min(-180)
        .max(180)
        .describe("Longitude of the location"),
    },
  },
  async ({ latitude, longitude }) => {
    // まず座標から「予報の担当区画」を調べる
    const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
    const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl);

    if (!pointsData) {
      return {
        content: [
          {
            type: "text",
            text: `Failed to retrieve grid point data for coordinates: ${latitude}, ${longitude}. This location may not be supported by the NWS API (only US locations are supported).`,
          },
        ],
      };
    }

    const forecastUrl = pointsData.properties?.forecast;
    if (!forecastUrl) {
      return {
        content: [
          {
            type: "text",
            text: "Failed to get forecast URL from grid point data",
          },
        ],
      };
    }

    // 予報データ本体を取りに行く
    const forecastData = await makeNWSRequest<ForecastResponse>(forecastUrl);
    if (!forecastData) {
      return {
        content: [
          {
            type: "text",
            text: "Failed to retrieve forecast data",
          },
        ],
      };
    }

    const periods = forecastData.properties?.periods || [];
    if (periods.length === 0) {
      return {
        content: [
          {
            type: "text",
            text: "No forecast periods available",
          },
        ],
      };
    }

    // 期間ごとに読みやすく整形する
    const formattedForecast = periods.map((period: ForecastPeriod) =>
      [
        `${period.name || "Unknown"}:`,
        `Temperature: ${period.temperature || "Unknown"}°${period.temperatureUnit || "F"}`,
        `Wind: ${period.windSpeed || "Unknown"} ${period.windDirection || ""}`,
        `${period.shortForecast || "No forecast available"}`,
        "---",
      ].join("\n"),
    );

    const forecastText = `Forecast for ${latitude}, ${longitude}:\n\n${formattedForecast.join("\n")}`;

    return {
      content: [
        {
          type: "text",
          text: forecastText,
        },
      ],
    };
  },
);
```

見どころは3つ。

1. **引数が2つ**（`latitude` / `longitude`）。増やすときはこう並べるだけです。
2. **`z.number()` で数値**、`.min()/.max()` で範囲を宣言しています。緯度は -90〜90、経度は -180〜180。
3. **APIを2回叩いています**。NWS は「座標 → 担当区画 → 予報」の2段構えなので、1つの道具の中で2回通信しています。**1ツール＝1API呼び出し、である必要はありません。**

そして——**エラーメッセージをよく読んでください**。

> `This location may not be supported by the NWS API (only US locations are supported).`
> 「この場所は NWS API が対応していないかもしれません（**米国内のみ対応**）」

公式サンプルは、**ちゃんと正直に書いてあります**。この一文が後で効きます。

### ステップ8：起動部分を書く

ファイルの最後です。

```typescript
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Weather MCP Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

- **`new StdioServerTransport()`** … 「stdio で話します」という通信路を用意します。
- **`server.connect(transport)`** … サーバーを通信路に繋ぎます。**この行以降、標準出力は Claude との会話チャンネルになります。**
- **`console.error("Weather MCP Server running on stdio")`** … 起動メッセージ。**ここも `console.log` ではありません。** ここを `console.log` にした瞬間、この章の全てが動かなくなります（第3章で実演します）。
- **`main().catch(...)`** … 起動に失敗したら、理由をエラー出力に書いて終了します。

### ステップ9：★ビルドする（絶対に忘れない）

```bash
npm run build
```

これで `src/index.ts` が `build/index.js` に変換されます。

> ⚠️ **この章で一番多い事故がこれです。** 公式ドキュメントにも「これはサーバーを繋ぐうえで**とても重要なステップ**です」とわざわざ書いてあります。
> **コードを直したら、毎回 `npm run build`。** 直しただけでは、Claude が動かすファイル（`build/index.js`）は古いままです。

エラーが出たら、この段階で全部つぶしてください。よくあるのは：

- `Cannot find module '@modelcontextprotocol/sdk/server/mcp.js'` → `.js` を消していないか／`npm install` したか
- `import` の行でエラー → `package.json` に `"type": "module"` を入れ忘れ

### ステップ10：Claude Desktop に繋ぐ

設定ファイルを開きます。**無ければ新しく作ってください。**

> ⚠️ **Claude Desktop は macOS と Windows にしか提供されていません（Linux版はありません）。** Linux で進めている人は、このステップ10〜12を飛ばしてください。代わりに[第10章](10-client-basics.md)の自作クライアント、または[付録B](apx-b-claude-code.md)の Claude Code から、いま作ったサーバーを同じように動かせます。

**macOS**:
```bash
code ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

**Windows**:
```powershell
code $env:AppData\Claude\claude_desktop_config.json
```

（`code` は VS Code のコマンドです。使っていなければ、メモ帳など普通のエディタで開いても構いません。）

中身をこう書きます。

**macOS**:
```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/PARENT/FOLDER/weather/build/index.js"]
    }
  }
}
```

**Windows**:
```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["C:\\PATH\\TO\\PARENT\\FOLDER\\weather\\build\\index.js"]
    }
  }
}
```

この設定が言っているのは、たった2つです。

1. **`weather` という名前のMCPサーバーがあります**
2. **起動方法は `node /……/weather/build/index.js` です**

`/ABSOLUTE/PATH/TO/PARENT/FOLDER/` は**あなたの環境の実際のパス**に置き換えます。調べ方は、`weather` フォルダの中でこう打つのが確実です。

```bash
pwd
```

出てきたパスの末尾に `/build/index.js` を足したものが、書くべき値です。

> ⚠️ **必ず絶対パス（`/` から始まる長いパス）にしてください。** `./build/index.js` のような相対パスは動きません。Claude Desktop がサーバーを起動するときの「今いるフォルダ」は決まっておらず、macOS では `/`（一番上）のことすらあります。詳しくは次節で。

### ステップ11：Claude Desktop を**完全終了**して再起動する

- **macOS**: ウィンドウを閉じるだけではダメです。**Cmd + Q**、またはメニューバーの Claude →「Claude を終了」。それから起動し直します。
- **Windows**: タスクトレイ（画面右下）に残っていることがあります。そこから終了させてください。

起動し直すと、入力欄のあたりに**道具のアイコン**が現れ、`get_alerts` と `get_forecast` が並んでいるはずです。

### ステップ12：動かしてみる

こう聞いてみてください。

> 「テキサス州で今出ている気象警報を教えて」

許可を求められたら承認します。警報が出ていれば内容が、出ていなければ `No active alerts for TX` を Claude が噛み砕いて答えます。**どちらも成功です。**

予報のほうも試しましょう。

> 「緯度37.77、経度-122.42（サンフランシスコ）の天気予報を教えて」

気温・風・天気が期間ごとに並べば、あなたのMCPサーバーは**完全に動いています**。おめでとうございます。

> 📦 **完成コードはここにあります。**
> [modelcontextprotocol/quickstart-resources](https://github.com/modelcontextprotocol/quickstart-resources) の **`weather-server-typescript`** フォルダが、いま書いたものの完成形です。どうしても動かないときは、自分のコードと**1行ずつ見比べて**ください。写経のズレは、だいたい `import` の `.js` か `package.json` にあります。

---

## 🔥 この章の山場：「東京の天気は？」と聞いてみる 🟢

さあ、いちばん大事なところです。**必ず自分の手でやってください。**

Claude にこう聞きます。

> 「東京の天気を教えて」

さて、何が起きたでしょうか。おそらく、次のどれかです。

### パターン①：道具を使おうともしない

Claude が「私はリアルタイムの天気情報にアクセスできません」などと答え、**`get_forecast` を呼びすらしない。**

### パターン②：呼んだけど、失敗する

Claude が東京の座標（緯度35.68、経度139.69）を入れて `get_forecast` を呼び、こう返ってくる。

```
Failed to retrieve grid point data for coordinates: 35.68, 139.69.
This location may not be supported by the NWS API (only US locations are supported).
```

### パターン③：州コードを聞き返してくる

`get_alerts` を使おうとして、「州コードを教えてください」と困る。あるいは「東京は米国の州ではないので、この道具では調べられません」と説明してくる。

**どれが出ても正解です。** そして、どれが出たかは、**Claude が何を読んだか**で決まっています。

### なぜこうなるのか — AIが読んでいたもの

思い出してください。Claude があなたの道具について知っていることは、**これだけ**です。

```
名前：  get_alerts
説明：  Get weather alerts for a state
引数：  state（文字列・2文字）
        「Two-letter state code (e.g. CA, NY)」

名前：  get_forecast
説明：  Get weather forecast for a location
引数：  latitude（数値・-90〜90）「Latitude of the location」
        longitude（数値・-180〜180）「Longitude of the location」
```

**以上。** `NWS_API_BASE` が `api.weather.gov` であることも、`makeNWSRequest` の中身も、Claude は**1文字も見ていません**。

だから、Claude はこう推論しています。

- `get_alerts` の引数は「**2文字の州コード（例：CA、NY）**」——**これは明らかにアメリカの話だ**。東京には州コードが無い。→ 使えない、と正しく判断できる
- `get_forecast` の引数は「**その場所の緯度**」「**その場所の経度**」——**どこの国とも書いていない**。東京にも緯度経度はある。→ **使えそうに見える**。だから呼んでみて、失敗する

面白いのはここです。

> **`get_alerts` の説明文は正直だったので、Claude は正しく諦められた。**
> **`get_forecast` の説明文は米国限定と書いていなかったので、Claude は騙された。**

Claude が悪いわけでも、間抜けなわけでもありません。**書いてあることを読んで、素直に判断しただけ**です。

### ここから持ち帰ってほしい2つのこと

> ### ① AIは、あなたが書いた「説明文」しか読んでいない
> 第1章で撒いた種が、ここで芽を出しました。ツールの**名前・説明・引数の説明**——AIが見ているのはこの3つだけです。中のコードも、繋ぐ先のAPIも、あなたの意図も、AIは知りません。
>
> だから `get_forecast` の説明が `"Get weather forecast for a location (US locations only)"` と書いてあれば、Claude は東京で呼ばずに済みました。**説明文を1行直すだけで、AIの振る舞いが変わる。** これがMCPの、意外なくらい大きな部分です。

> ### ② 道具にできないことは、AIにもできない
> 「Claude は賢いから、なんとかしてくれるだろう」——なりません。`api.weather.gov` は東京のデータを持っていないので、**どんなに賢いAIでも、東京の天気は返せません**。
>
> AIの能力は、**あなたが持たせた道具の能力を超えません**。第1章で「MCPは配管であって能力ではない」と書いたのは、これのことです。

### だから、第5章から自分の道具を作ります

天気サーバーは、**他人が、他人のために作った道具**です。よくできていますが、あなたの役には立ちません。あなたが本当に欲しいのは、たぶんこういうものでしょう。

- 自分のメモフォルダを検索してくれる
- 自分の書きかけの文章を読んでくれる
- 自分のルールで新しいメモを書いてくれる

**それは、自分で作るしかありません。** そして、作り方は——**いま写経したものと、まったく同じ形**です。`registerTool` に、名前と、説明と、引数の説明と、処理を書く。それだけです。

第5章で、`weather` の代わりに `memo` を作ります。**この章のコードが、そのまま下敷きになります。**

---

## ⚠️ ハマりどころ 🟢

ここは厚めにいきます。**この5つで、この章のトラブルの9割です。**

### ① `npm run build` を忘れている ←最頻出

**症状**: コードを直したのに、Claude の振る舞いがまったく変わらない。あるいは、そもそも道具が出てこない。

**原因**: あなたが編集したのは `src/index.ts`。Claude が動かすのは `build/index.js`。**ビルドしないと繋がりません。**

**直し方**:
```bash
npm run build
```
そのうえで **Claude Desktop を完全終了→再起動**。この2つはセットです。

**覚え方**: 「**直したら、ビルドして、再起動**」。声に出して3回。

### ② 相対パスを書いている

**症状**: 道具が一切出てこない。ログに `Cannot find module` や `MODULE_NOT_FOUND`。

**原因**: 設定ファイルに `./build/index.js` や `~/weather/build/index.js` と書いている。

**なぜダメか**: Claude Desktop があなたのサーバーを起動するとき、**「今いるフォルダ」が何になるかは決まっていません**。macOS では `/`（ディスクの一番上）のことすらあります。そこから見た `./build/index.js` は、どこにも存在しません。`~` も展開されないことがあります。

**直し方**: `/` から始まる**絶対パス**を書く。`weather` フォルダで `pwd` を打って出たパスに `/build/index.js` を足す。Windows なら `C:\\` から始めて、**バックスラッシュを2つ重ねる**（`\\`）ことも忘れずに。

### ③ Claude Desktop を「完全終了」していない

**症状**: 設定を書き換えたのに、まったく反映されない。

**原因**: ウィンドウを閉じただけ。**アプリはまだ生きています。**

**直し方**:
- macOS: **Cmd + Q**。またはメニューから「Claude を終了」
- Windows: タスクトレイのアイコンを右クリックして終了。それでも怪しければタスクマネージャーで確認

設定ファイルは**起動時にしか読まれません**。ここは何度でも引っかかるので、疑ったらまず再起動してください。

### ④ JSON の書式ミス

**症状**: 道具のアイコンすら出ない。設定が丸ごと無視されている。

**原因**: `claude_desktop_config.json` の書式エラー。**JSONは細かいことに厳しい**です。

よくあるミス：

| ミス | 例 | 正しく |
|---|---|---|
| **末尾のカンマ** | `"args": ["/path/index.js"],` の後に `}` | 最後の要素にカンマを付けない |
| **カンマ忘れ** | `"command": "node" "args": [...]` | 項目の区切りに `,` が要る |
| **シングルクォート** | `'command': 'node'` | JSONは**ダブルクォート `"` のみ** |
| **コメント** | `// サーバーの設定` | **JSONにコメントは書けません** |
| **Windowsのパス** | `"C:\path\index.js"` | `"C:\\path\\index.js"`（`\` は2つ重ねる） |

**確かめ方**: VS Code で開くと、書式が壊れていれば波線が出ます。それが一番早いです。

### ⑤ `zod` のバージョンが違う

**症状**: `inputSchema` のあたりで型エラー。あるいは実行時に引数の受け渡しがおかしい。

**原因**: `zod@3` ではなく v4 が入っている。**書き方が違います。**

**確かめ方**:
```bash
npm ls zod
```
`zod@3.x.x` になっていなければ、入れ直します。

```bash
npm install zod@3
```

---

### それでも動かないときは

**天気サーバーで丸一日を溶かさないでください。** これは通り道であって、目的地ではありません。

1. まず **[付録C　設定ファイルの場所とトラブル](apx-c-config.md)** を見る
2. エラー文言が分かっているなら **[付録D　よくあるエラー集](apx-d-errors.md)**
3. それでもダメなら——**この章は飛ばして、第3章へ進んでかまいません。**

第4章でデバッグの道具（MCP Inspector とログの読み方）を手に入れます。**そこで戻ってきたほうが、確実に速い**です。

---

## 🤖 AIに頼むなら 🟢

この章のコードを、AIコーディングツールに書かせたい人へ。**やめたほうがいい、とは言いません。** ただし注意点があります。

### 頼み方

> 「MCPの公式ドキュメント（modelcontextprotocol.io）の**最新の書き方**で、TypeScript の公式SDK（`@modelcontextprotocol/sdk`）を使って、`get_alerts` と `get_forecast` を持つ天気サーバーを書いて。`zod@3` を使い、`server.registerTool` の形式で。」

### AIが間違えやすい3か所

MCPは動きが速い規格です。AIの学習データには**古い書き方が混ざっています**。とくにこの3つは要チェックです。

| よくある間違い | 正しくは |
|---|---|
| `server.tool(...)` を使う | **`server.registerTool(...)`**（この教材はこちらで統一） |
| `inputSchema: z.object({ ... })` と包む | **`z.object` で包まない**。素のオブジェクトにフィールドを並べる |
| `import ... from "…/mcp"` （`.js` 無し） | **`.js` 必須**（`"module": "Node16"` のため） |

加えて、**`console.log` を混ぜてこないか**を必ず見てください。AIは親切心で「起動しました」というログを `console.log` で入れることがあります。**それだけで通信が壊れます**——しかも、いつも派手に壊れるとは限りません。**なんとなく動いてしまう**こともあり、そのほうが厄介です（第3章）。

### いちばん大事なこと

**AIに書かせても、写経は1回やってください。** この章のコードは、この先の全章の下敷きです。`registerTool` の3つの引数——名前・説明の束・処理——が指に入っていると、第5章以降が別物のように楽になります。

---

## 📗 ことばメモ

| ことば | よみ | 意味 |
|---|---|---|
| **NWS** | エヌダブリューエス | National Weather Service。アメリカ国立気象局。`api.weather.gov` を公開している |
| **ビルド** | — | TypeScript（`src/*.ts`）を JavaScript（`build/*.js`）に変換すること。`npm run build` |
| **絶対パス** | ぜったいパス | `/` や `C:\` から始まる、どこから見ても同じ場所を指す道順 |
| **相対パス** | そうたいパス | `./` などから始まる、今いる場所を基準にした道順。**設定ファイルでは使えない** |
| **`registerTool`** | — | サーバーに道具を1つ登録する関数。引数は「名前・説明の束・処理」の3つ |
| **`inputSchema`** | インプットスキーマ | 道具が受け取る引数の宣言。zod のフィールドを並べた素のオブジェクト |
| **`.describe()`** | ディスクライブ | 引数の説明文を付ける。**AIが読む唯一の手がかり** |
| **zod** | ゾッド | 入力の形を宣言・検査するライブラリ。この教材では **v3** を使う |
| **`console.error`** | — | 標準エラー出力への書き込み。**stdio サーバーで使ってよいログの出し方** |
| **claude_desktop_config.json** | — | Claude Desktop がどのMCPサーバーを起動するかを書く設定ファイル |

---

## ➡️ 次へ

動きましたか。動いたなら、あなたはもう**MCPサーバーを1つ、自分のパソコンで動かした人**です。動かなかったなら、それは第4章で必ず回収します。

そして、この章で分かったことを1行にすると、こうです。

> **AIは、あなたが書いた説明文しか読んでいない。道具ができないことは、AIにもできない。**

次の [第3章　標準入出力の正体 — `console.log` 1行でサーバーが壊れる](03-stdio.md) では、**いま動かしたこのコードを、わざと壊します**。

やることは、たった1行足すだけ。`console.log("started")` と書きます。**それだけで、Claude との会話が壊れます。** しかも、いつも派手に壊れてくれるわけではありません。書き方によっては**なんとなく動いてしまう**ことがあり、そのほうがずっとタチが悪いのです。なぜそんなことが起きるのか、そのときログには何が出るのか、直し方は何通りあるのか——**壊れ方を先に知っておくと、あとがずっと楽になります**。

`weather` フォルダは消さないでおいてください。次の章でも使います。

## 関連ページ

- MCPサーバーを作って学ぶ AIに道具を持たせる入門 — はじめに・目次 — 目次
- [第1章　「コネクタ」の正体 — MCPは道具の共通規格](01-what-is-mcp.md) — ホスト／クライアント／サーバーの三役
- [第3章　標準入出力の正体 — `console.log` 1行でサーバーが壊れる](03-stdio.md) — 次の章。このコードを壊します
- [第5章　自分の道具を作りはじめる — メモを検索する](05-memo-server.md) — 「東京の天気」の答え。ここから自分の道具
- [付録C　設定ファイルの場所とトラブル](apx-c-config.md) — 繋がらないときの駆け込み寺
- [付録D　よくあるエラー集](apx-d-errors.md) — エラー文言から引く
