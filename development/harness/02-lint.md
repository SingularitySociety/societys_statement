---
title: "第2章 AI向けガチガチ lint(Vue/React + Express)"
parent: "AIハーネス入門 — AIに安全に良い仕事をさせる環境づくり"
grand_parent: "開発の心得"
nav_order: 2
---

# 第2章 AI向けガチガチ lint — Vue/React + Express の ESLint 設定

この章では、AI に大量のコードを書かせる前提で、**遠慮なく厳しくした ESLint 設定**を作ります。題材は、実際に AI 主体で数百 PR を回しているプロジェクト [MulmoClaude の eslint.config.mjs](https://github.com/receptron/mulmoclaude/blob/main/eslint.config.mjs)(Vue + Express + TypeScript)です。

## なぜ「ガチガチ」にするのか

人間のチームでは、厳しい lint は嫌われます。「この程度いいじゃん」という摩擦のコストが、ルールの利益を上回るからです。AI ではこの前提が逆転します。

1. **AIは文句を言わない** — lint エラーを見せれば黙って直します。摩擦コストがほぼゼロ
2. **AIは自分で直せる** — 「エラーを検出 → AI が修正 → 再チェック」のループが自動で回る。**厳しいルール = 自動品質改善の燃料**
3. **AIは書く量が多い** — 人間レビューの前に、機械が9割の問題を止めてくれないとレビューが破綻する
4. **AIは一貫性を機械に頼る** — セッションをまたいだ「書き方の記憶」がないので、ルールがコードの一貫性を担保する唯一の手段

CLAUDE.md(第1章)に「関数は20行以下」と**書く**のが約束なら、lint は**破れなくする**装置です。約束はすり抜けることがありますが、CI で回る lint はすり抜けられません。

### 道具の役割分担

| 道具 | 役割 | 例 |
| --- | --- | --- |
| **Prettier** | 見た目の整形(議論の余地をなくす) | インデント、改行、クォート |
| **ESLint** | 書き方の質とバグの芽(この章の主役) | `any` 禁止、関数の長さ、`==` 禁止 |
| **tsc(typecheck)** | データの形の整合性 | 型の不一致、null の可能性 |
| **CI** | 上の全部を push のたびに強制 | GitHub Actions |

---

## ベースは「推奨セットの積み上げ」

ゼロからルールを書くのではなく、実績ある推奨セットを重ね、その上に自分のルールを足します。MulmoClaude では次の6段重ねです。

| セット | 提供元 | 何をくれるか |
| --- | --- | --- |
| `eslint.configs.recommended` | ESLint 本体 | 未定義変数などの基本 |
| `tseslint.configs.strict` + `stylistic` | typescript-eslint | 型まわりの厳格ルール一式(recommended より強い **strict** を選ぶのがポイント) |
| `sonarjs.configs.recommended` | eslint-plugin-sonarjs | 認知的複雑度、重複、バグパターン検出 |
| `securityPlugin.configs.recommended` | eslint-plugin-security | インジェクション等の危険パターン検出 |
| `vuePlugin.configs["flat/recommended"]` | eslint-plugin-vue | Vue テンプレートの品質(React なら React 系に差し替え。後述) |
| `eslintConfigPrettier` | eslint-config-prettier | Prettier と衝突する整形系ルールを無効化(**必ず最後に**置く) |

## サンプル設定(Vue + Express + TypeScript)

MulmoClaude の実設定から、どのプロジェクトでも使える部分を抜き出して整理したものです。コメントの説明ごとコピーして使ってください。

```js
// eslint.config.mjs
import eslint from "@eslint/js";
import globals from "globals";
import tseslint from "typescript-eslint";
import eslintConfigPrettier from "eslint-config-prettier";
import prettierPlugin from "eslint-plugin-prettier";
import sonarjs from "eslint-plugin-sonarjs";
import securityPlugin from "eslint-plugin-security";
import importPlugin from "eslint-plugin-import";
import vuePlugin from "eslint-plugin-vue";
import vueParser from "vue-eslint-parser";

export default [
  { files: ["{src,server,test}/**/*.{js,ts,vue}"] },
  { ignores: ["dist", "coverage", "node_modules"] },

  // ── 土台: 推奨セットの積み上げ ─────────────────────
  eslint.configs.recommended,
  sonarjs.configs.recommended,
  securityPlugin.configs.recommended,
  ...tseslint.configs.strict,
  ...tseslint.configs.stylistic,
  ...vuePlugin.configs["flat/recommended"],

  // ── 本体: AI向けの上乗せルール ─────────────────────
  {
    languageOptions: {
      globals: { ...globals.es2021, ...globals.node },
      ecmaVersion: "latest",
      sourceType: "module",
    },
    plugins: { prettier: prettierPlugin, import: importPlugin },
    rules: {
      // --- 量の上限: AIの「書きすぎ」を止める ---
      "max-lines-per-function": ["error", { max: 50, skipBlankLines: true, skipComments: true }],
      complexity: ["error", { max: 15 }],
      "sonarjs/cognitive-complexity": "error",
      "max-depth": ["error", { max: 4 }],
      "max-params": ["error", { max: 6 }],

      // --- 型の穴をふさぐ ---
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-non-null-assertion": "error",
      "@typescript-eslint/consistent-type-assertions": "error",
      "@typescript-eslint/no-use-before-define": ["error", { functions: false, ignoreTypeReferences: true }],

      // --- うっかりバグの検出 ---
      eqeqeq: ["error", "smart"],
      "array-callback-return": "error",
      "consistent-return": "error",
      "no-param-reassign": "error",
      "no-shadow": "error",
      "no-self-compare": "error",
      "no-unmodified-loop-condition": "error",
      "no-implicit-coercion": ["error", { boolean: true, number: true, string: true }],
      "no-throw-literal": "error",
      "prefer-promise-reject-errors": "error",
      "default-case-last": "error",
      "default-param-last": "error",
      "sonarjs/no-ignored-exceptions": "error",

      // --- 読みやすさの底上げ ---
      "id-length": ["error", { min: 3, properties: "never", exceptions: ["_", "i", "j", "ok"] }],
      "prefer-const": "error",
      "prefer-template": "error",
      "no-else-return": ["error", { allowElseIf: false }],
      "no-lonely-if": "error",
      "no-unneeded-ternary": ["error", { defaultAssignment: false }],
      "object-shorthand": "error",
      "prefer-arrow-callback": "error",
      "arrow-body-style": ["error", "as-needed"],

      // --- 構造を守る ---
      "import/no-cycle": "error",
      "import/no-duplicates": "error",
      "import/no-self-import": "error",
      "import/no-mutable-exports": "error",
      "import/first": "error",
      "import/newline-after-import": "error",
      "@typescript-eslint/no-require-imports": "error",

      // --- 整形は Prettier に一任し、違反は lint エラーにする ---
      "prettier/prettier": "error",
      indent: ["error", 2],
      semi: ["error", "always"],
      "linebreak-style": ["error", "unix"],
    },
  },

  // ── テストコードだけの緩和(理由は本文参照) ─────────
  {
    files: ["test/**/*.{ts,js}", "e2e/**/*.{ts,js}"],
    languageOptions: { globals: { ...globals.browser } },
    rules: {
      "max-lines-per-function": "off",
      "@typescript-eslint/no-explicit-any": "warn",
      "sonarjs/assertions-in-tests": "error",
    },
  },

  // ── Vue ファイル固有(main ルールの後に置く: 後勝ち) ──
  {
    files: ["**/*.vue"],
    languageOptions: {
      parser: vueParser,
      parserOptions: { parser: tseslint.parser, sourceType: "module", extraFileExtensions: [".vue"] },
      globals: { ...globals.browser },
    },
    rules: {
      "vue/no-v-html": "error",
      "vue/no-useless-mustaches": "error",
      "vue/no-useless-v-bind": "error",
      "vue/prefer-true-attribute-shorthand": "error",
      "vue/no-empty-component-block": "error",
    },
  },

  eslintConfigPrettier, // 必ず最後
];
```

---

## ルール解説 — なぜそのルールなのか

「AI がやりがちな事故」と対にして読むと、なぜこの選定なのかが分かります。

### 量の上限 — AIの「書きすぎ」を物理的に止める

AI の代表的な悪癖は、**動くけど長くて把握できないコード**を量産することです。サイズ系ルールはそれを機械的に止めます。

| ルール | 意味 | 防ぐ事故 |
| --- | --- | --- |
| `max-lines-per-function: 50` | 関数は実質50行まで | 200行の「なんでも関数」。CLAUDE.md の目標が20行でも、lint の強制ラインは50行のように**目標より緩い天井**にすると運用しやすい |
| `complexity: 15` | 分岐の数の上限 | if の海。テスト不能な関数 |
| `sonarjs/cognitive-complexity` | 「人間が読んで疲れる度」の上限 | ネスト+分岐+再帰の合わせ技 |
| `max-depth: 4` | ネストは4段まで | 矢印型(`if` の中の `if` の中の…)コード |
| `max-params: 6` | 引数は6個まで | 引数10個の関数(オブジェクト引数への合図) |

### 型の穴をふさぐ — 「とりあえず動かす」抜け道の封鎖

AI は型エラーに詰まると `any` や `as` や `!` で**黙らせる**誘惑に駆られます。全部エラーにしておくと、正面から型を直すしかなくなります。

| ルール | 防ぐ事故 |
| --- | --- |
| `no-explicit-any` | `any` で型チェックを丸ごと無効化。**AI向け lint で一番大事な1本** |
| `no-non-null-assertion` | `foo!.bar` — 「null じゃないはず」という祈り。実行時クラッシュの温床 |
| `consistent-type-assertions` | 乱暴な `as` キャストの制限。型ガード関数へ誘導 |
| `no-use-before-define` | 定義前の変数参照(TDZ エラー)。AI がコードを継ぎ足すときに起こしがち |

### うっかりバグの検出 — JavaScript の罠を全部エラーに

| ルール | 防ぐ事故 |
| --- | --- |
| `eqeqeq` (smart) | `==` の暗黙変換(`"" == 0` が true!)。`x == null` イディオムだけ許す設定 |
| `array-callback-return` | `map` のコールバックで return し忘れ → 全要素 undefined |
| `consistent-return` | 「値を返す経路と返さない経路が混在」する関数 |
| `no-param-reassign` | 引数の書き換え → 呼び出し元のオブジェクトが黙って変わる |
| `no-shadow` | 外側と同名の変数 → 「どっちの `user`?」事故 |
| `sonarjs/no-ignored-exceptions` | `catch (e) {}` — エラーの握りつぶし。AI が「とりあえず動かす」ためにやりがち |

### セキュリティ — 危険パターンの検出網

`eslint-plugin-security` と sonarjs は、eval・危険な正規表現(ReDoS)・子プロセス起動などの危険パターンに反応します。

ここで重要な考え方が**誤検知(false positive)との付き合い方**です。MulmoClaude では、初回監査で「527件警告が出て、本物が0件」だったルール(`detect-object-injection` など)は、**理由をコメントに書いた上で off** にしています。

```js
// detect-object-injection: 動的キーの obj[key] に全部反応する。
// 初回監査で 329 件の警告 / 本物 0 件 → シグナルを殺すので off。
// 本物のケースは sonarjs 側のルールが拾う。
"security/detect-object-injection": "off",
```

逆に、本当に危ないもの(`detect-unsafe-regex` = ReDoS)は **error に格上げ**し、正当な例外には1行ずつ理由つきの disable コメントを義務づけています。**「厳しくする」は「全部 on」ではなく「シグナルが濃くなるように調整する」**ことです。ノイズだらけの lint はAIも人間も無視するようになり、いちばん危険です。

### 構造を守る — アーキテクチャを lint で強制する

AI が最も壊しやすいのは、コードの「行」ではなく**構造**(どこから何を import してよいか)です。`no-restricted-imports` を使うと、アーキテクチャの境界を lint にできます。MulmoClaude の実例(プラグインは決められた API 経由でしかホストに触れない):

```js
{
  files: ["src/plugins/*/**/*.ts", "src/plugins/*/**/*.vue"],
  rules: {
    "no-restricted-imports": ["error", {
      patterns: [
        {
          group: ["**/server/**"],
          message: "プラグインからサーバー側モジュールを import しない。" +
                   "プラグインはクライアント側で動く — 通信は API 経由で。",
        },
        {
          group: ["**/config/**"],
          message: "ホストの設定を直接 import しない。../api の関数を使う。",
        },
      ],
    }],
  },
},
```

**`message` に「代わりにどうするか」を書く**のがコツです。AI はエラーメッセージを読んで自分で直すので、メッセージがそのまま矯正装置になります。

---

## テストコードは一部だけ緩める

テストは性質が違うので、機械的に同じ厳しさを当てるとノイズになります。ただし**緩めるのは狙った数本だけ**。バグを拾う系(no-shadow、cognitive-complexity など)はテストでも error のままにします。

| 扱い | ルール | 理由 |
| --- | --- | --- |
| off | `max-lines-per-function` | `describe()` に it が10個並ぶのは正常。20行/50行の目標は本番コードの読みやすさのためで、テストには当てはまらない |
| warn に格下げ | `no-explicit-any` | モック作りで DOM 型に any 的キャストが要ることがある。本番コードでは error のまま |
| **error に格上げ** | `sonarjs/assertions-in-tests` | **assert のないテスト**(実行するだけで何も検証しない)は、AI がテストを「書いたことにする」典型パターン。人間の目では見逃しやすいので CI で強制 |

最後の1本は特に重要です。AI は「テストを書け」と言われると、**通ることだけが目的の空っぽのテスト**を書くことがあります。lint がそれを検出できます。

---

## React の場合の差分

コアの上乗せルール(量の上限・型・バグ検出・構造)は**そのまま全部使えます**。差し替えるのはフレームワーク層だけです。

| Vue | React での対応物 |
| --- | --- |
| `eslint-plugin-vue` + `vue-eslint-parser` | `eslint-plugin-react` + `eslint-plugin-react-hooks` |
| — | `eslint-plugin-jsx-a11y`(アクセシビリティ) |
| `vue/no-v-html: error` | `react/no-danger: error`(`dangerouslySetInnerHTML` 禁止 = 同じ XSS 対策) |

```js
// eslint.config.mjs(React 版の差分イメージ)
import react from "eslint-plugin-react";
import reactHooks from "eslint-plugin-react-hooks";
import jsxA11y from "eslint-plugin-jsx-a11y";

export default [
  // ...共通の土台と上乗せルールは Vue 版と同じ...
  react.configs.flat.recommended,
  jsxA11y.flatConfigs.recommended,
  {
    files: ["**/*.{jsx,tsx}"],
    plugins: { "react-hooks": reactHooks },
    rules: {
      "react-hooks/rules-of-hooks": "error",   // フックの呼び出し規則違反(条件分岐の中で useState 等)
      "react-hooks/exhaustive-deps": "error",  // useEffect の依存配列の書き漏れ — AIが最も多発させるReactバグ
      "react/no-danger": "error",
      "react/jsx-key": "error",                // リストの key 忘れ
    },
    settings: { react: { version: "detect" } },
  },
];
```

> ⚠️ flat config の書き方(`configs.flat.recommended` など)はプラグインのバージョンで微妙に変わります。導入時は各プラグインの README の指示に従ってください。ルールの**選定**はこの表の通りで大丈夫です。

特に `react-hooks/exhaustive-deps` は error 推奨です。依存配列の書き漏れは「たまにしか起きない不可解なバグ」になり、AI・人間ともに原因究明に最も時間を溶かすパターンだからです。

---

## 例外(eslint-disable)のエチケット

どんなに調整しても、正当な例外は発生します。ルール化しておくべき作法:

1. **行単位で、理由コメントつきでのみ許可**

   ```ts
   // ReDoS 安全性はテスト test_htmlSrcAttrs.ts で上界を検証済み
   // eslint-disable-next-line security/detect-unsafe-regex
   const SRC_ATTR = /.../;
   ```

2. **ファイル全体・ルール全体の disable は禁止**(1つの例外のために検出網ごと外さない)
3. **CLAUDE.md に「NEVER: disable で黙らせず根本原因を直す」と書く**(第1章参照)。lint とルールの二段構えで、AI が安易な disable に流れるのを防ぐ
4. **既存コードが多くて一気に直せないとき**は「格付けラチェット」方式: 新規コードには error、直しきれていない既存ファイルだけ一時的に warn の例外リストに載せ、返済したらリストから消す。**例外リストに新規追加はしない**(MulmoClaude が `max-lines-per-function` で実際にやっている運用)

---

## 運用に組み込む

lint は**実行されなければ存在しないのと同じ**です。3か所に組み込みます。

```jsonc
// package.json
{
  "scripts": {
    "lint": "eslint src server test --cache",
    "format": "prettier --write '{src,server,test}/**/*.{ts,vue,json}'",
    "typecheck": "vue-tsc --noEmit && tsc -p server/tsconfig.json --noEmit"
  }
}
```

1. **CLAUDE.md** — 「変更後は必ず `yarn format` → `yarn lint` → `yarn typecheck` → `yarn build`」(第1章)
2. **Claude Code のフック** — `git commit` 前に自動実行、失敗したらコミットをブロック([シリーズ表紙](./README.md)の実例)
3. **CI(GitHub Actions)** — push / PR のたびに全チェック。ローカルをすり抜けても最後はここで止まる([CI/CD入門](../github_actions/README.md))

この三段構えで、「約束(CLAUDE.md)→ 手元の強制(フック)→ 最終防衛線(CI)」が揃います。

---

次章は、コードを書いた AI とは**別の目**にチェックさせるしくみ — subagent と cross review です。

→ [第3章 subagent と cross review — AIの仕事を別のAIがチェックする](./03-cross-review.md)
