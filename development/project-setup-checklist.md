# 新規プロジェクト立ち上げチェックリスト

新規プロジェクトの初日に固めるべき事項を、**MulmoClaude プロジェクトの実履歴 + 一般的な経験則**から洗い出したもの。

## 出典の区別

| セクション | 由来 |
|---|---|
| **§1〜§15** | MulmoClaude（Web アプリ）の履歴で「後付けで大規模リファクタを強いられた」領域を実際に git log から洗い出したもの。リファレンスとなった主要 PR / コミットを括弧内に併記。 |
| **§16〜§24** | MulmoClaude の履歴には現れないが、Web アプリ以外（ライブラリ・CLI）や、より広い領域（DB / 公開 API / クラウド / セキュリティ / ドキュメント）で同じく「後で困りやすい」経験則として追加した一般項目。MulmoClaude の PR 番号は付かない。 |

セクション分け：
- §1〜§15：**Web アプリ**（このプロジェクトが該当・実履歴ベース）
- §16：**ライブラリ（npm package）固有**（一般項目）
- §17：**CLI ツール固有**（一般項目）
- §18：**全プロジェクト共通の追加項目**（一般項目）
- §19：**DB を使う / ネット公開する場合のセキュリティ準備**（一般項目）
- §20：**クラウドアカウント・課金・API キー管理**（一般項目）
- §21：**ドキュメント管理**（一般項目）
- §22：**API キーの命名・設計（Stripe 流）**（一般項目）
- §23：**外部公開する API**（一般項目）
- §24：**WAF・クラウドセキュリティ**（一般項目）

---

## 1. リント・フォーマット（プロジェクト初日に固める）

- [ ] **ESLint flat config** + `test/` `e2e/` も対象に含める
- [ ] **`eslint-plugin-sonarjs`** 入れて `cognitive-complexity: error` / `no-nested-conditional: warn`
- [ ] **`@intlify/eslint-plugin-vue-i18n`** 入れて `no-raw-text: error` ※ i18n を後回しにすると今回みたいに 17 バッチかかる
  - `ignoreNodes: ["code","kbd","pre"]`、`ignorePattern` で Material Icon ligature を最初から除外
- [ ] **`id-length: error`** (min: 3, exceptions `_`,`i`,`j`,`ok`) ← PR #90bab91 で全域リネームに
- [ ] **`@typescript-eslint/no-explicit-any: error`**(テストは warn)
- [ ] **`import/no-cycle: error`**
- [ ] **`prefer-const`, `no-shadow`, `no-param-reassign`**
- [ ] **Prettier** `printWidth: 160` くらい広め、`singleQuote` などを最初に確定
- [ ] **ESLint cache** (`--cache --cache-location node_modules/.cache/eslint/`) を `lint` script に最初から
- [ ] **pre-commit hook** で format + lint + typecheck — 後から入れると差分が膨大になる

## 2. 国際化（i18n）

- [ ] 文字列 1 個目から **`useI18n()` + `t()` を必須**にする
- [ ] **TS 辞書** (`src/lang/en.ts` / `ja.ts`) で型安全 — JSON ダンプ (`batch/i18n-dump.ts`) は lint 用にだけ用意
- [ ] **`createI18n<[MessageSchema], Locale>(...)`** + module augmentation を最初に
- [ ] `VITE_LOCALE` 環境変数で切り替え（UI セレクタは要件次第）
- [ ] 命名規約：機能ごとに namespace（`pluginXxx`, `settingsXxx`, `common`）

## 3. 定数（マジックリテラル禁止を最初から）

- [ ] **時間定数** `ONE_SECOND_MS / ONE_MINUTE_MS / ONE_HOUR_MS` を `utils/time.ts` に最初から（PR #421 で全域置換になった）
- [ ] **API ルート定数** `API_ROUTES` (`src/config/apiRoutes.ts`) — フロント/サーバ/テストで共有
- [ ] **イベント種別** `EVENT_TYPES` などすべて `as const` オブジェクト
- [ ] **状態リテラル**（view mode, status, origin など）も最初から `const` 化（#509, #cddd673, #0acbcb9, #c11e340 で個別 PR）

## 4. データ保存ディレクトリ（一番取り返しが効かない）

- [ ] **トップレベル構造を最初に決定**（このプロジェクトは `data/` `artifacts/` `conversations/` `config/` を後から整理して #449999c などでパス移行に苦労）
- [ ] **絶対パス禁止、`WORKSPACE_PATHS` 経由**で全アクセス
- [ ] **デフォルト workspace 場所**（`~/<app>/` 等）を環境変数で上書き可能に
- [ ] **書き込み単位**（フラット / by-name / by-date 等）を最初に決める
- [ ] **読み取り専用参照ディレクトリ**の概念を入れるか入れないか早期決定（後から追加すると権限まわり全部書き直し）

## 5. ファイル I/O

- [ ] **`writeFileAtomic`** を初日に実装：tmp → rename、宛先と同じ FS に tmp（`os.tmpdir()` 不可）
- [ ] **Windows EPERM/EBUSY/EACCES リトライ**を最初から（PR #524 の atomic.ts 参照、`[30,100,300]ms` バックオフ）
- [ ] **ドメイン別 I/O モジュール** `server/utils/files/<domain>-io.ts` を最初から（#366 Phase 6 で全 route handler から fs 呼び出しを剥がした）
- [ ] **route handler では `fs.readFile`/`fs.writeFile` 禁止** をリント or レビュー規約で

## 6. ネットワーク I/O

- [ ] **クライアント共通ヘルパ** `apiGet/apiPost/apiPut/apiDelete` を最初から（auth トークン自動付与、Result 型返却）
- [ ] **`AbortController` でタイムアウト必須**（fetch 直書き禁止）
- [ ] **エラー処理**：`try/catch` + `!response.ok` の両方
- [ ] **リトライ戦略**を方針として決めておく（指数バックオフ vs 1回失敗即エラー）

## 7. 認証・セキュリティ

- [ ] **bearer token 認証**を初日から（PR #447 で「side-effects あり API がノーガード」だったのを修正）
- [ ] **`timingSafeEqual`** でトークン比較
- [ ] **CORS / CSRF / SSRF** ポリシーを最初に
- [ ] **入力検証は Zod**（schema → `z.infer` で型を派生、type 重複定義禁止）
- [ ] **境界（user input, 外部 API）でだけ検証**、内部コードは型を信頼

## 8. ロギング

- [ ] **構造化ロガー** `log.{error,warn,info,debug}(prefix, msg, data?)` を初日から（PR #91 / #181 で console.* を全置換）
- [ ] **`console.*` 禁止**を ESLint or レビュー規約で
- [ ] **シンク**：開発は console、本番はファイル+ローテート

## 9. クロスプラットフォーム CI

- [ ] **CI matrix `[ubuntu, windows, macos] × [node 22.x, 24.x]`** を最初から
- [ ] **`node:path` 必須** — `/` `\\` 直書き禁止
- [ ] **`fileURLToPath` / `pathToFileURL`** 使用、`__dirname` 直書き禁止
- [ ] **npm scripts**：`rm -rf` → `rimraf`、`cp` → `shx` か Node スクリプト
- [ ] **case-insensitive FS** 前提（macOS/Windows）でファイル名衝突しない命名

## 10. パッケージング / モジュール

- [ ] **`exports` フィールド**に `import` / `require` / `default` 全部入れる（Docker CJS 対応 — PR #429）
- [ ] **monorepo** にするなら最初から workspaces + `packages/<scope>/<name>` 構造を決める（bridges のサブディレクトリ移動 PR #ba0ebf1 で全パス書き換え）
- [ ] **再エクスポート barrel ファイル禁止**（理由のないものは作らない）
- [ ] **dynamic import は条件付き依存だけ**、常用は top-level import

## 11. テスト

- [ ] **`node:test` + `node:assert`**
- [ ] **E2E は Playwright**、`mockAllApis(page)` パターンを最初から
- [ ] **golden test** 方針を最初に（CLI/プログラム出力の比較）
- [ ] **`data-testid` 属性**を全 UI 要素に — クラス名/位置で取らない
- [ ] **テストを CI matrix の全 OS で走らせる**

## 12. 型安全

- [ ] **`as` キャスト禁止** → 型ガード関数 `isXxx(x): x is Type`
- [ ] **Zod スキーマ → `z.infer` で型派生**、ローカル type 二重定義禁止
- [ ] **既存ライブラリの型ガード使う** (`isObject` from graphai 等) — 自前で書かない

## 13. プラグイン / 拡張アーキテクチャ

- [ ] **拡張ポイント**（プラグイン、ブリッジ、bridge 種別など）を最初に設計
- [ ] **追加に必要な変更箇所をドキュメント化** — このプロジェクトは「local plugin 追加で 8 ファイル変更」と CLAUDE.md に明記してある
- [ ] **共有プロトコル**は npm package として最初から切る（後から切ると依存関係解きほぐしが苦痛）

## 14. リリース / バージョニング

- [ ] **`docs/CHANGELOG.md`** を初日に作る（PR #ec3cee4 で後付け）
- [ ] **タグ規約**を決める：`vX.Y.Z`（アプリ）/ `@scope/name@X.Y.Z`（パッケージ）
- [ ] **release script** (`/release-app`, `/publish`) を最初に整える

## 15. リポジトリ運用

- [ ] **main 直 push 禁止**（PR 必須、admin もバイパス不可）
- [ ] **review/CI 必須化**
- [ ] **bot レビュー**（CodeRabbit, Sourcery）を初日から有効化
- [ ] **PR テンプレ** に「User Prompt」「Summary」「Items to Confirm」セクションを入れる

---

## 16. ライブラリ（npm package）固有

公開する npm パッケージで後から困りやすい点。

### パッケージング

- [ ] **`exports` フィールド完備**：`import` / `require` / `types` / `default` を最初から
  ```json
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
  ```
- [ ] **`type: "module"`** を明示（ESM か CJS か曖昧にしない）
- [ ] **`main` / `module` / `types`** も後方互換のため併記（古いツール対応）
- [ ] **`files` フィールド**で公開対象を明示（`dist/`, `README.md`, `LICENSE` のみ）
- [ ] **`sideEffects: false`**（or 配列で副作用ファイルを列挙）でツリーシェイク有効化
- [ ] **`engines.node`** 明記（最低サポート Node バージョン）
- [ ] **TypeScript 型は `.d.ts` ＋ `.d.cts`** 両方出す（dual package hazard 対策）

### 依存関係

- [ ] **`peerDependencies`** を慎重に：framework 系（vue, react, etc.）は peer に
- [ ] **`peerDependenciesMeta.optional: true`** で optional peer を明示
- [ ] **bundle 内に他パッケージを抱え込まない**（同じ依存の二重ロード回避）
- [ ] **dependencies はミニマムに**（脆弱性面積を減らす）

### API 設計

- [ ] **named export のみ**、default export は避ける（ツリーシェイク・型推論で有利）
- [ ] **後方互換の方針を最初に決める**：semver 厳守 / canary タグ / experimental API 接頭辞
- [ ] **`@experimental` / `@deprecated` JSDoc タグ**を最初から運用
- [ ] **エラー型を export する**（catch 側で `instanceof` できるように）
- [ ] **型推論が効く API を優先**（generics 過剰は避ける）

### バージョニング

- [ ] **semver 厳守**：major bump = breaking change を必ず
- [ ] **breaking change は CHANGELOG の最上段＋ migration guide**
- [ ] **`npm publish --dry-run`** を CI に組み込んで漏れチェック
- [ ] **provenance** (`npm publish --provenance`) で信頼性向上

### ドキュメント

- [ ] **README は「30 秒で動く」サンプル冒頭**
- [ ] **TypeScript 型を README で見せる**（`.d.ts` 抜粋）
- [ ] **API リファレンスは TypeDoc** で自動生成
- [ ] **changesets** 等で changelog 自動化を最初から

---

## 17. CLI ツール固有

ツール（`npx foo` で動く類）で後から困りやすい点。

### 引数・入出力

- [ ] **argv パーサ**を最初に決める（`commander` / `yargs` / `clipanion`）— `process.argv` 直接触らない
- [ ] **`--help` / `--version`** を初日から
- [ ] **stdout = データ、stderr = ログ** の規律（pipe 可能に）
  - 進捗バー、警告、エラーは全部 stderr へ
- [ ] **stdin からのパイプ入力対応**（`process.stdin.isTTY` で判定）
- [ ] **JSON 出力モード**（`--json`）を最初から用意（他ツールと連携可能に）
- [ ] **exit code 規約**：0=成功、1=ユーザエラー、2=システムエラー、130=SIGINT

### TTY / 端末

- [ ] **`NO_COLOR` 環境変数尊重**（[no-color.org](https://no-color.org/)）
- [ ] **`FORCE_COLOR` も尊重**
- [ ] **TTY 判定で出力を変える**（`process.stdout.isTTY`）— パイプ時は色なし、進捗バーなし
- [ ] **シェル補完**（bash/zsh/fish）を後から追加できる構造で
- [ ] **CI 検出**（`process.env.CI`）でインタラクティブ要素を抑制

### シグナル・終了処理

- [ ] **SIGINT / SIGTERM ハンドラ**で graceful shutdown
- [ ] **作業中の一時ファイル cleanup** を必ず（`process.on("exit")` ＋ シグナル両方）
- [ ] **長時間処理の中断ポイント**を最初から設計（`AbortController` ベース）

### 設定・状態

- [ ] **設定ファイル探索順を決める**：`./<tool>.config.{js,json,ts}` → `~/.<tool>rc` → env vars
- [ ] **XDG Base Directory** 準拠（`$XDG_CONFIG_HOME`, `$XDG_CACHE_HOME`, `$XDG_DATA_HOME`）
- [ ] **更新通知**（`update-notifier` 等）を入れるか初日に決める
- [ ] **オフラインモード**を最初から想定（ネット必須にしない）

### 配布

- [ ] **`bin` フィールド**を `package.json` に
- [ ] **shebang `#!/usr/bin/env node`** を出力ファイル先頭に
- [ ] **Windows でも `npx <tool>` が動く**ことを CI で確認
- [ ] **single executable**（`pkg`, `bun build --compile`, Deno compile）にするか早期判断

---

## 18. 全プロジェクト共通の追加項目

### リポジトリ初期化（initial commit に入れる）

- [ ] **`LICENSE`** ファイル（MIT/Apache-2.0/その他）
- [ ] **`README.md`** with 動くサンプル
- [ ] **`.gitignore`** （`node_modules`, `dist`, `.env`, `.DS_Store`, OS固有）
- [ ] **`.gitattributes`** で改行コード固定（`* text=auto eol=lf`）
- [ ] **`.editorconfig`**
- [ ] **`.nvmrc` / `.tool-versions`** で Node バージョン固定
- [ ] **`CODEOWNERS`** を最初から
- [ ] **`SECURITY.md`** で脆弱性報告先を明記
- [ ] **`CONTRIBUTING.md`**（OSS なら）
- [ ] **`.github/dependabot.yml`** で依存自動更新
- [ ] **`.github/ISSUE_TEMPLATE/`** と **`PULL_REQUEST_TEMPLATE.md`**
- [ ] **`.github/FUNDING.yml`**（OSS で寄付受け付けるなら）

### AI エージェント / コーディング支援ツール設定

「AI が間違ったコードを生成しないように、最初に **何を守らせるか** を明文化しておく」領域。後から書くと既に作られた違反コードの掃除込みになる。

- [ ] **`CLAUDE.md`（プロジェクトルート）** を initial commit に：
  - パッケージマネージャ（yarn/npm/pnpm）の指定
  - lint/format/build/test の標準コマンド
  - コミット前必須チェック（`yarn format && yarn lint && yarn typecheck && yarn build`）
  - コーディング規約（関数 20 行以内、`as` キャスト禁止、any 禁止 等）
  - 禁止事項（`git add .` 禁止、main 直 push 禁止、git 操作の事前承認必須 等）
  - PR 作成フロー、コミットメッセージ prefix（`feat:` / `fix:` / `refactor:` / `docs:` / `chore:`）
  - **MUST / NEVER / SHOULD / MAY** キーワードを定義して使う（曖昧さ排除）
  - **「やるな」を明示的に書く**（やらせたい挙動より、間違いやすい挙動の禁止が効く）
- [ ] **エリア別 CLAUDE.md** をサブディレクトリに（`server/CLAUDE.md`, `src/CLAUDE.md`, `test/CLAUDE.md` 等）— 大きいリポでは細分化
- [ ] **`AGENTS.md`** を別途用意するか CLAUDE.md にシンボリックリンクするか決定（OpenAI Codex / 他エージェント互換）
- [ ] **`.claude/` ディレクトリ構成を最初に**：
  ```
  .claude/
    settings.json          ← プロジェクト共通設定（commit する）
    settings.local.json    ← 個人設定（gitignore）
    skills/                ← プロジェクト固有 skill
    commands/              ← プロジェクト固有 slash command
    hooks/                 ← イベントフック
  ```
- [ ] **`.claude/settings.local.json` を gitignore**（個人の API キー / preferences）
- [ ] **プロジェクト固有 Skill を `.claude/skills/` に**：
  - `/release` — リリース手順（version bump, changelog, tag, publish）
  - `/publish` — npm パッケージ公開
  - `/setup` — 初回セットアップ（依存インストール、DB 初期化、シード投入）
  - `/migrate` — DB マイグレーション実行
  - 「複数手順を毎回繰り返している」操作は skill 化候補
- [ ] **Slash command を `.claude/commands/` に**：プロンプトテンプレ（review、commit message 生成、PR 説明など）
- [ ] **MCP サーバ設定 `.mcp.json`** を最初に：
  - 共通 MCP（postgres, github, slack 等）はリポジトリで共有
  - 個人 MCP は `.mcp.local.json` で（gitignore）
- [ ] **Hooks**：
  - `pre-commit` 相当を `.claude/hooks/` に（lint、format、テスト）
  - 危険コマンドの確認 hook（`rm -rf`, `git push --force`, DB drop 等）
- [ ] **`.cursorrules` / `.cursor/rules/`**（Cursor 利用なら）— CLAUDE.md と内容同期するか同じファイルを参照
- [ ] **`.github/copilot-instructions.md`**（GitHub Copilot 利用なら）
- [ ] **`.windsurfrules`**（Windsurf 利用なら）
- [ ] **エージェント向け指示の単一ソース化**：複数ツール対応するなら `AGENTS.md` を真とし、各ツール用は symlink or include
- [ ] **「学んだことを CLAUDE.md に追記する」運用ルール**：会話中の指摘・修正をルール化（continuous learning）
- [ ] **CLAUDE.md / AGENTS.md 自体を PR レビュー対象**：勝手に強い禁則を増やさない

### 秘密情報・環境変数

- [ ] **`.env.example`** をコミット、`.env` は gitignore
- [ ] **環境変数の型定義**（Zod でバリデート）を起動時に
- [ ] **GitHub Secrets** の使い分けを決める（repo / environment / org）
- [ ] **コミット前 secret スキャン**（`gitleaks` or `trufflehog` を pre-commit に）

### 観測性（observability）

- [ ] **エラートラッキング**（Sentry, Rollbar 等）を入れるか初日決定
- [ ] **メトリクス収集**の方針（Prometheus / OpenTelemetry / なし）
- [ ] **トレース ID** を最初からリクエストに付与
- [ ] **ログレベルを env で制御**（`LOG_LEVEL`）

### パフォーマンス

- [ ] **bundle size 監視**（`size-limit`, `bundlewatch`）を CI に
- [ ] **依存サイズ監視**（`bundlephobia` 確認）
- [ ] **Lighthouse CI**（Web の場合）を PR ごとに

### アクセシビリティ（Web）

- [ ] **`eslint-plugin-jsx-a11y`** or **`eslint-plugin-vuejs-accessibility`** を初日から
- [ ] **axe-core** で自動 a11y テスト
- [ ] **キーボード操作のみで全機能**動くことを E2E で検証
- [ ] **prefers-reduced-motion / prefers-color-scheme** を最初から想定

### ローカル開発体験（DX）

- [ ] **`make dev` / `npm run dev`** 一発で全部立ち上がる
- [ ] **Docker Compose** でローカル DB / 依存サービスを再現可能に
- [ ] **VS Code `.vscode/settings.json` ＋ recommended extensions** をコミット
- [ ] **シードデータ・fixtures** を最初から整備
- [ ] **`/scripts/`** に頻用スクリプトをまとめる（README に index）

### スキーマ・データモデル

- [ ] **DB マイグレーションツール**（Prisma / Drizzle / Knex 等）を最初に
- [ ] **マイグレーション = forward-only** ポリシー（rollback はやらない、forward fix）
- [ ] **DB スキーマと TS 型を一元化**（Prisma generate / drizzle-zod / etc.）
- [ ] **API レスポンスは Zod スキーマ**で型を派生（OpenAPI なら zod-to-openapi）

### キャッシュ

- [ ] **HTTP キャッシュヘッダ**戦略を最初に（`Cache-Control`, `ETag`）
- [ ] **静的アセットは content-hash 付き**で immutable cache
- [ ] **アプリ内キャッシュ**（SWR / TanStack Query）を最初に決定

### 国際化（i18n 拡張）

- [ ] **日付・数値・通貨**は `Intl.*` を最初から（自前フォーマット禁止）
- [ ] **タイムゾーン**：UTC で保存、表示時にユーザ TZ で変換
- [ ] **複数形ルール**は vue-i18n の pluralization に任せる（自前 `if count === 1` 禁止）
- [ ] **RTL 言語**を将来サポートするなら CSS は `inline-start/end` で書く

### 法務・コンプライアンス

- [ ] **依存ライセンス監査**（`license-checker` 等）を CI に
- [ ] **プライバシーポリシー / 利用規約** を最初に決める
- [ ] **GDPR / CCPA** 想定なら cookie consent + データ削除 API を最初から
- [ ] **エクスポート規制**（暗号輸出など）の確認

---

## 19. DB を使う / ネット公開する場合のセキュリティ準備

「動いてから後付け」が一番しんどい領域。**初日に骨格だけでも入れておく**ことを強く推奨。

### 19.1 ORM / クエリビルダ

- [ ] **生 SQL 文字列連結を全面禁止** — Prisma / Drizzle / Kysely / TypeORM のいずれか必須
- [ ] **どうしても生 SQL が必要な箇所はパラメータ化必須**（`?` プレースホルダ、文字列補間禁止）
- [ ] **マイグレーションツール**を最初に決定（Prisma Migrate / Drizzle Kit / node-pg-migrate）
- [ ] **マイグレーションは forward-only**、down 書かない（rollback したいときは forward fix）
- [ ] **マイグレーション = レビュー必須**（DB スキーマは PR テンプレで強制セルフレビュー）
- [ ] **N+1 検出**（Prisma なら `?include`、Drizzle なら `?with`、または APM で監視）
- [ ] **トランザクション境界を最初に設計**（リクエスト単位 / unit-of-work）
- [ ] **コネクションプール上限**を環境ごとに（serverless では特に注意）
- [ ] **slow query log** を本番で常時有効に

### 19.2 入力検証（境界で全部）

- [ ] **Zod スキーマで全エンドポイントの入力を検証**（body / params / query / headers すべて）
- [ ] **Zod → TS 型派生**で「検証された型」しか handler に届かない構造に
- [ ] **長さ上限**を全文字列フィールドに（`z.string().max(N)` 必須）
- [ ] **配列長・ネスト深さの上限**（DoS 対策）
- [ ] **数値範囲**（`.min()` / `.max()`）— `parseInt` の罠回避
- [ ] **allowlist > denylist**（許可するものを列挙、禁止リストは穴だらけになる）
- [ ] **ファイルアップロード**：サイズ上限、MIME allowlist、マジックナンバー検証（拡張子だけは信用しない）
- [ ] **ファイル名サニタイズ**：path traversal（`../`）、null byte、Windows 予約名（`CON`, `PRN` 等）
- [ ] **JSON parse の深さ・サイズ制限**（Express なら `express.json({ limit: '100kb' })`）

### 19.3 出力エンコーディング / サニタイズ

- [ ] **テンプレートエンジンの自動エスケープ ON**（Vue / React は標準で OK、`v-html` / `dangerouslySetInnerHTML` は要審査）
- [ ] **Markdown / HTML 受け入れる箇所は DOMPurify**（サーバ側で sanitize、クライアントで安心しない）
- [ ] **JSON レスポンスは Content-Type: application/json 明示**（XSS 経由 JSON hijacking 対策）
- [ ] **ユーザ入力をログに直書きしない**（ログインジェクション、改行混入対策）
- [ ] **エラーメッセージにスタックトレース・SQL を出さない**（本番）
- [ ] **`X-Powered-By` ヘッダ削除**（フィンガープリンティング対策）

### 19.4 HTTP セキュリティヘッダ

- [ ] **`helmet`（Express）/ 同等品**を初日に入れる
- [ ] **Content-Security-Policy** を `default-src 'self'` ベースで最初から（後から入れると inline script の総ざらいになる）
- [ ] **Strict-Transport-Security** `max-age=31536000; includeSubDomains; preload`
- [ ] **X-Content-Type-Options: nosniff**
- [ ] **X-Frame-Options: DENY** or CSP `frame-ancestors`
- [ ] **Referrer-Policy: strict-origin-when-cross-origin**
- [ ] **Permissions-Policy** で不要な API（geolocation, camera, etc.）を全 deny
- [ ] **CORS は allowlist**（`Access-Control-Allow-Origin: *` 禁止、credentials 付きなら特に）
- [ ] **Cookie**：`Secure` + `HttpOnly` + `SameSite=Lax`（or `Strict`）デフォルト
- [ ] **CSRF トークン**：state-changing 操作（POST/PUT/DELETE）に必須、`SameSite=Strict` で十分か検討

### 19.5 ユーザアカウント / 認証

- [ ] **パスワードは bcrypt（cost 12+）or argon2id**、生 SHA-256 / MD5 絶対禁止
- [ ] **パスワードポリシー**：長さ重視（最低 12 文字）、複雑性は緩く、漏洩パスワード DB と照合（haveibeenpwned API）
- [ ] **アカウントロックアウト**：N 回失敗で一時的にレート制限（IP + アカウント単位）
- [ ] **セッション管理**を最初に決める：
  - server-side session（Redis）：取り消し可能、推奨
  - JWT：取り消し困難、短命にして refresh token と組み合わせる
- [ ] **JWT 使うなら**：`alg: none` 拒否、署名アルゴリズム固定、`exp` 必須、署名鍵ローテーション計画
- [ ] **メール検証**：登録時 + メアド変更時に必須
- [ ] **パスワードリセット**：トークンは 1 回限り、有効期限 30 分以内、URL に user id 入れない
- [ ] **2FA / MFA**：TOTP（Google Authenticator 互換）を最初から構造化、実装は後でも構造は最初から
- [ ] **OAuth / OIDC** で外部 IdP に寄せられるなら寄せる（自前パスワード管理は最後の手段）
- [ ] **権限モデル**を最初に決定：RBAC / ABAC / ACL のどれか
- [ ] **権限チェックは middleware で集約**、各 handler に `if user.isAdmin` を散らさない
- [ ] **「自分のリソースか」チェックを忘れない**（IDOR / BOLA：他人の ID をパラメータに入れて読めてしまうやつ — 最頻発脆弱性）
- [ ] **メアド・電話番号変更時に通知メール**（アカウント乗っ取り検知）
- [ ] **退会時のデータ削除フロー**を最初から（GDPR right to erasure）

### 19.6 レート制限・DoS 対策

- [ ] **エンドポイント全体にデフォルトのレート制限**（`express-rate-limit` 等）
- [ ] **重い操作・認証系には厳しめのレート制限**（ログイン、パスワードリセット、検索）
- [ ] **IP 単位 + アカウント単位の両方**（IP 単位だけだと共有 NAT で誤爆、アカウント単位だけだと未ログイン攻撃に無力）
- [ ] **Redis or memcached** を rate limit ストアに（メモリ単独だとマルチインスタンスで穴）
- [ ] **CDN / WAF（Cloudflare 等）の前段配置**を本番デプロイ前に決定
- [ ] **CAPTCHA**：パブリックフォーム（登録、問い合わせ）に hCaptcha / Turnstile

### 19.7 注入攻撃の網羅

- [ ] **SQL injection**：ORM 使用 + 生 SQL 禁止（§19.1）
- [ ] **NoSQL injection**：MongoDB 使うなら `$where` / `$regex` のユーザ入力直結禁止、`mongo-sanitize`
- [ ] **コマンドインジェクション**：`exec` / `spawn` の shell モード禁止、引数は配列で渡す
- [ ] **SSRF**：外部 URL fetch 時は private IP（10.x, 172.16-31.x, 192.168.x, 169.254.169.254 メタデータ）拒否
- [ ] **XXE**：XML パース時は外部エンティティ無効化
- [ ] **テンプレート injection**：ユーザ入力をテンプレート文字列にしない
- [ ] **prototype pollution**：`Object.create(null)` か Map を使う、`__proto__` キーは reject

### 19.8 シークレット管理

- [ ] **シークレットは環境変数 + シークレットマネージャ**（AWS Secrets Manager / GCP Secret Manager / Vault）
- [ ] **`.env` を絶対に commit しない**（gitleaks / trufflehog を pre-commit + CI に）
- [ ] **GitHub Actions の secrets** はジョブ単位で最小権限に
- [ ] **DB 認証情報をローテート可能な設計**（hardcode しない、再起動なしで切り替え可能に）
- [ ] **API キー漏洩時の revoke 手順** をドキュメント化
- [ ] **クライアント（フロント）に秘密を埋め込まない**（公開鍵・publishable key だけ）

### 19.9 暗号化

- [ ] **TLS 1.2 以上のみ**、HTTPS 強制（HTTP は HSTS で 301 リダイレクト）
- [ ] **証明書**：Let's Encrypt 自動更新を最初に
- [ ] **DB 接続も TLS**（社内ネットでも油断しない）
- [ ] **DB at-rest 暗号化**（クラウド DB なら標準で ON、確認）
- [ ] **PII / クレカ番号 / 健康情報など機微データは追加暗号化**（カラムレベル暗号化）
- [ ] **クレカは保存しない**（Stripe 等にトークン化を委譲、PCI DSS 範囲を最小化）

### 19.10 監査・ログ

- [ ] **監査ログ**：認証、権限変更、データ削除、admin 操作は別 log stream に
- [ ] **ログから PII / シークレットを除外**（パスワード、トークン、クレカ、メアド の masking）
- [ ] **ログ改竄防止**：append-only ストア（CloudWatch / Stackdriver）or 署名付きログ
- [ ] **不審アクセス通知**（同一アカウントで地理的に離れた IP からのログイン等）
- [ ] **保管期間ポリシー**を最初に決める（GDPR / 業界規制に応じて）

### 19.11 バックアップ・DR

- [ ] **DB の自動バックアップ**を初日から（クラウド DB なら設定確認）
- [ ] **リストア手順を実際に試す**（バックアップだけ取って戻したことないが最悪パターン）
- [ ] **point-in-time recovery (PITR)** を有効に
- [ ] **バックアップは別リージョン or 別アカウントへ**
- [ ] **RTO / RPO を明文化**

### 19.12 テスト環境

- [ ] **本番と完全分離**：別 DB、別シークレット、別ドメイン（`test.example.com`）
- [ ] **本番データを test/dev に持ち込まない**（必要なら必ず PII 匿名化スクリプト経由）
- [ ] **環境変数 `NODE_ENV` / `APP_ENV`** で挙動を制御、`if (env === 'production')` を散らさない
- [ ] **staging 環境**を本番にできるだけ近く（同じインフラ、同じスキーマ、ダミーデータ）
- [ ] **feature flag** を最初から（LaunchDarkly / GrowthBook / 自前）— 本番に出さずにマージできる
- [ ] **テスト環境は IP 制限 or basic auth** でクロール / 検索エンジン回避（`X-Robots-Tag: noindex` も）
- [ ] **テスト DB は毎回リセット可能**（migration を applied 状態 → empty にする npm script）

### 19.13 テストデータ

- [ ] **シードスクリプト** (`scripts/seed.ts`) を最初に整備、リポジトリにコミット
- [ ] **`@faker-js/faker` で生成**、ただし**シード固定**で再現性確保
- [ ] **エッジケース fixtures**：unicode（絵文字、結合文字、RTL）、最大長、空文字、NULL、SQL メタ文字
- [ ] **本番データ → テスト** 持ち込みパイプラインを作るなら **匿名化必須**（k-anonymity 考慮）
- [ ] **PII を含むテストデータをコミットしない**（実在 email / 電話 / 住所 NG、`@example.com` 等を使う）
- [ ] **大量データテスト**：本番想定の 10x データ量での性能テストを CI に組み込む

### 19.14 依存・サプライチェーン

- [ ] **`npm audit` / `yarn audit` を CI に**、high 以上で fail
- [ ] **Dependabot or Renovate** で自動 PR
- [ ] **Socket Security** など typosquat / マルウェア検出ツール
- [ ] **lockfile を必ずコミット**（`yarn.lock` / `package-lock.json`）
- [ ] **`npm ci` を本番ビルドで使う**（`npm install` は lockfile を書き換える）
- [ ] **依存追加 PR は人がレビュー**（自動マージしない、minor でも）
- [ ] **SBOM（CycloneDX / SPDX）生成**を CI に

### 19.15 静的解析・脆弱性スキャン

- [ ] **Semgrep / CodeQL** で SAST を CI に
- [ ] **OWASP ZAP / Burp** で DAST（staging 環境定期実行）
- [ ] **コンテナイメージスキャン**（Trivy / Snyk Container）
- [ ] **シークレットスキャン**を pre-commit + CI 両方で

### 19.16 Webhook / 外部連携

- [ ] **受信 webhook は署名検証必須**（Stripe / GitHub / Slack 等、各社の HMAC 仕様に従う）
- [ ] **timing-safe comparison**（`crypto.timingSafeEqual`）で署名比較
- [ ] **送信 webhook は HTTPS のみ**、URL は allowlist or ユーザ確認
- [ ] **idempotency key** を受け入れて重複処理回避
- [ ] **再送ポリシー**：失敗時の retry / backoff / dead letter queue

### 19.17 GDPR / プライバシー

- [ ] **データ収集の目的を最初に明文化**
- [ ] **データ削除 API**（right to erasure）— 退会時に依存テーブル全削除できる構造
- [ ] **データエクスポート API**（right to portability）— ユーザ自身の全データを JSON で出せる
- [ ] **同意管理（cookie consent）** — 必須 cookie と analytics cookie を分離
- [ ] **third-party 送信を最小化**（Google Analytics / Sentry 等のデータ送信先をプライバシーポリシーに列挙）
- [ ] **データ保管リージョン**を最初に決定（EU データは EU 内、等）

---

## 20. クラウドアカウント・課金・API キー管理

「いつの間にか月数十万円」「root アカウント乗っ取り」が一番怖い領域。**請求書が飛んでくる前**に固める。

### 20.1 クラウドアカウント運用

- [ ] **root（ルート）アカウントは日常使用しない** — 作成直後にロック
  - root メアドは個人ではなく `aws-root@<company>` 等の MTA で受ける
  - root のパスワードは password manager に、physical security key（YubiKey）で MFA
  - root credential はインシデント時のみ使用、使ったら必ずローテート
- [ ] **MFA 全アカウント必須**（人間も機械も） — 設定強制ポリシーを最初に
- [ ] **環境ごとに別アカウント / 別プロジェクト**：
  - AWS：`prod` / `staging` / `dev` を別 Account（AWS Organizations + SCP）
  - GCP：別 Project + 別 Folder
  - Azure：別 Subscription
  - 「同一アカウントで dev と prod を分ける」は事故のもと
- [ ] **SSO / SAML で組織 IdP に統合**（Google Workspace, Okta 等）— 個別アカウントを作らない
- [ ] **退職者 offboarding チェックリスト**を最初に作る（IdP 削除でクラウド・GitHub・Slack 全部切れる構成に）
- [ ] **CloudTrail / Cloud Audit Logs を全リージョンで有効化**、別アカウントに集約（消せない場所へ）
- [ ] **使わないリージョンを SCP で全 deny**（攻撃面減＋誤コスト防止：`ap-northeast-3` 等で勝手に EC2 立てられた事例あり）

### 20.2 IAM / 権限

- [ ] **最小権限原則**：admin role は限定、サービス role は scope を絞る
- [ ] **個人にポリシー直付けしない**、必ず group / role 経由
- [ ] **長期 access key を避ける**：人間は SSO、CI は OIDC（GitHub Actions → AWS / GCP は OIDC でキーレス）
- [ ] **Service Account の鍵が必要なら短期化**（rotation スケジュール明記）
- [ ] **権限変更は IaC で管理**（Terraform / CDK / Pulumi）、コンソール直編集禁止
- [ ] **`AdministratorAccess` を CI に与えない**（必要な権限を都度切り出す）
- [ ] **break-glass account**：障害時用の高権限アカウントを別途用意、使用は必ず alert + 監査

### 20.3 課金アラート

- [ ] **Budget アラート**を初日に設定：
  - 日次（想定の 1.5x で警告 — 異常急増の早期発見）
  - 月次（予算の 50% / 80% / 100% / 120%）
  - サービス別（EC2, S3, データ転送 等カテゴリ別）
- [ ] **アラート送信先を複数チャンネル**：email + Slack + on-call ページ
- [ ] **自動シャットダウン閾値**（破滅回避）：「月 $10,000 超えたら non-prod を自動停止」等を IaC で
- [ ] **Cost Explorer / 課金ダッシュボード**を週次で見る習慣（Slack に summary bot）
- [ ] **タグ付けポリシー**を最初に決める：`env`, `service`, `team`, `cost-center` 必須 — タグなしリソース禁止 SCP
- [ ] **無料枠の罠を理解**：
  - 12 ヶ月無料 → 13 ヶ月目に課金開始
  - リージョンごとに無料枠（`us-east-1` だけ等）
  - データ転送 OUT が高額（特に inter-region, internet egress）
- [ ] **未使用リソース定期 sweep**：止め忘れ EC2、未アタッチ EBS、空 S3 バケット
- [ ] **Reserved Instance / Savings Plan** の検討は本番 stable 後（事業方向性決まってから）
- [ ] **個人実験用には別アカウント + 強い予算上限**（$50 等）— 本番アカウントで遊ばない

### 20.4 API キー / シークレット管理

- [ ] **API キーをリポジトリに commit しない**（pre-commit + CI で gitleaks / trufflehog / Socket）
- [ ] **シークレットマネージャ必須**：
  - クラウド：AWS Secrets Manager / GCP Secret Manager / Azure Key Vault
  - 自前：HashiCorp Vault / Doppler / 1Password Secrets Automation
- [ ] **環境変数は起動時にシークレットマネージャから取得**（`.env` ファイルに本物の鍵を書かない、`.env.example` だけコミット）
- [ ] **キーごとに用途・所有者・有効期限をメタデータで管理**
- [ ] **ローテーション計画**を最初に：
  - DB パスワード：90 日
  - API キー：180 日
  - JWT 署名鍵：1 年（rotation 中は新旧両方を accept する設計）
- [ ] **キー漏洩時の手順をドキュメント化**：revoke → ローテート → 影響範囲調査 → 通知
- [ ] **第三者 API（OpenAI, Anthropic, Stripe, SendGrid 等）の鍵**：
  - サービスごとに別キー、用途ごとに別キー（test/prod、機能別）
  - 各社のダッシュボードで使用量上限・spending limit 設定
  - 不審な急増アラートを各社で ON
  - **コードに直書きしない、フロントエンドに出さない**（公開鍵・publishable key だけ OK）
- [ ] **GitHub Personal Access Token / OAuth App**：
  - fine-grained PAT を使う（classic は scope が広すぎる）
  - 期限を必ず付ける
  - 用途別に分ける、共有 PAT 禁止
- [ ] **CI/CD のシークレット**：
  - GitHub Actions Secrets / Environment Secrets（環境別に保護ルール）
  - 本番 deploy には manual approval 必須
  - secret を echo / log 出力しない（GitHub は自動マスクしてくれるが油断しない）
- [ ] **ローカル開発の API キー**：1Password CLI / `direnv` で env に注入、`.env` を git 管理しない
- [ ] **開発者個別の API キーを発行**（共有キー禁止 — 漏洩時の影響範囲限定 + 監査可能）

### 20.5 SaaS / 外部サービス管理

- [ ] **サブスクリプション台帳**：何を契約しているか、誰が owner か、いつ更新か
- [ ] **支払い方法を集約**（個人カード払い禁止、コーポレートカード / 請求書払い）
- [ ] **使ってないサービスを定期棚卸**（四半期ごと）
- [ ] **データ持ち出しポリシー**：解約時にデータをエクスポートできるか事前確認

---

## 21. ドキュメント管理

「コードはあるがドキュメントがない」「ドキュメントはあるが古い」「どこに何が書いてあるか分からない」を最初に防ぐ。

### 21.1 ドキュメントの種類と置き場

- [ ] **置き場を最初に決める**：
  - **コード近接（in-repo）**：`README.md`, `docs/`, JSDoc/TSDoc — コードと一緒に PR レビュー、stale 検知しやすい
  - **Wiki（Confluence, Notion, GitHub Wiki）**：人事・契約・組織情報など非エンジニア共有
  - **「両方に同じこと書く」を絶対避ける**（必ずどちらかが古くなる）
- [ ] **`docs/` ディレクトリ構造を最初に決定**：
  ```
  docs/
    README.md              ← 目次（どこに何があるか）
    getting-started.md     ← 30 分で動かす
    architecture.md        ← 全体像、図
    developer.md           ← 開発者向け（このプロジェクトでも採用）
    api/                   ← API リファレンス（自動生成 or 手書き）
    runbooks/              ← oncall 手順
    decisions/             ← ADR（後述）
    CHANGELOG.md           ← リリースノート
  ```

### 21.2 README（最重要）

- [ ] **README は「30 秒で何のプロジェクトか分かる」冒頭**：
  1. 1 行説明
  2. スクリーンショット or デモ GIF
  3. インストール / クイックスタート（コピペで動く）
  4. ドキュメント・サポートへのリンク
- [ ] **長くなったら `docs/` に分割**、README は index に
- [ ] **バッジ**（CI status, npm version, license, downloads）で health 一目把握
- [ ] **multilingual README**（README.md + README.ja.md）が必要なら最初に決める

### 21.3 ADR（Architecture Decision Records）

- [ ] **重要な技術判断は ADR に記録**：`docs/decisions/0001-use-postgres.md` 等
- [ ] **テンプレ固定**：Context / Decision / Consequences / Alternatives considered
- [ ] **immutable**：間違いだったら新しい ADR で supersede（古いものは残す）
- [ ] **判断者と日付を明記**
- [ ] **「なぜそうしたか」を残す**（「何をしたか」はコードを読めば分かる）

### 21.4 API ドキュメント

- [ ] **コードから自動生成**：
  - REST：OpenAPI（`zod-to-openapi` / `tRPC` / `@asteasolutions/zod-to-openapi`）
  - TypeScript ライブラリ：TypeDoc
  - GraphQL：schema 自体がドキュメント
- [ ] **手書きドキュメントとコードを乖離させない仕組み**を初日に
- [ ] **example request/response** を必ず添える
- [ ] **API バージョニング戦略**を最初に決定（`/v1/`、ヘッダ、ドメイン分離）

### 21.5 Runbook / Oncall

- [ ] **障害対応手順書**を本番デプロイ前に：
  - 各アラートに対応する runbook を 1:1 で
  - 手順、確認 SQL、復旧コマンドを具体的に
  - 「困ったらこの人」連絡先
- [ ] **postmortem テンプレ**を最初に用意（blameless ポストモーテム文化）
- [ ] **過去のインシデント一覧**を `docs/incidents/` に蓄積（同じことを繰り返さない）

### 21.6 オンボーディング

- [ ] **新メンバーが Day 1 で動かせるドキュメント**：
  - リポジトリ clone → 起動までのコマンド列
  - 必要なアカウント発行手続き（IdP, GitHub, Slack, Vault）
  - 開発環境セットアップ（推奨 IDE + 拡張機能）
  - 用語集（プロジェクト固有の名前、略語）
- [ ] **オンボーディング自体も Issue 化**（毎回チェックリストとして使う）

### 21.7 図・ダイアグラム

- [ ] **Mermaid をリポジトリ内 markdown で**（GitHub が自動描画、PR で diff 見える）
- [ ] **複雑なものは Excalidraw / draw.io**、ソースファイル（`.excalidraw`, `.drawio`）も commit
- [ ] **PNG / JPG だけ commit しない**（編集できなくなる）
- [ ] **C4 model** で抽象度を統一（System / Container / Component / Code）

### 21.8 コード内ドキュメント

- [ ] **JSDoc / TSDoc** を public API に必須
- [ ] **「何をするか」ではなく「なぜそうするか」をコメントに**（コードは「何」を語る）
- [ ] **TODO / FIXME / HACK タグ**：必ず Issue 番号を併記、orphan TODO 禁止
- [ ] **重要な不変条件・前提条件**を関数 docstring の冒頭に
- [ ] **複雑なアルゴリズムは参考リンク**（論文 URL、Wikipedia、Stack Overflow 回答）

### 21.9 CHANGELOG

- [ ] **`docs/CHANGELOG.md` を初日に作る**（`Keep a Changelog` 形式推奨）
- [ ] **リリースごとに必ず更新**（CI で「未更新ならブロック」も検討）
- [ ] **breaking change / new / fix / deprecated を分けて書く**
- [ ] **changesets / semantic-release** で自動化も検討

### 21.10 ドキュメントの鮮度維持

- [ ] **ドキュメント変更を PR レビュー必須に**（コード変更で挙動が変わるならドキュメントも同じ PR で）
- [ ] **`grep` で古い情報を検出**できるよう、参照箇所を集約（同じファイルパスを何箇所にも書かない）
- [ ] **link checker を CI に**（リンク切れ検出）
- [ ] **四半期ごとにドキュメント棚卸**（chore タスクとして作る）
- [ ] **stale doc を削除する勇気**（古い情報を残すより削除）

### 21.11 ナレッジ共有

- [ ] **個人の Slack DM に解決策を書かない**（必ずチャンネル / ドキュメント化）
- [ ] **「30 分悩んだら聞く / 書く」ルール**：解決したらドキュメント化
- [ ] **週次振り返り or 月次の learning share**（過去の躓きを共有資産に）
- [ ] **ドキュメント書きを評価対象に含める**（書く人が損しない仕組み）

---

## 22. API キーの命名・設計（Stripe 流）

API キーは「**見ただけで何のキーか・どの環境か・公開可否か分かる**」設計が事故防止に効く。Stripe の命名規則がデファクト。

### 22.1 命名フォーマット

`<role>_<env>_<random>` の 3 パートで構成：

| 役割 (role) | 用途 | 公開可否 |
|---|---|---|
| `pk_` | Publishable / public — クライアント側に埋め込み可 | 公開 OK |
| `sk_` | Secret — サーバ側のみ、絶対公開不可 | 秘密 |
| `rk_` | Restricted — scope を絞った secret（特定 API のみ） | 秘密 |
| `whsec_` | Webhook signing secret — webhook 署名検証専用 | 秘密 |

| 環境 (env) | 用途 |
|---|---|
| `live` | 本番環境 |
| `test` | テスト / sandbox 環境 |
| `dev`  | 開発環境（必要なら） |

例：

```
pk_live_51H...     ← 本番フロント埋め込み用
sk_live_51H...     ← 本番サーバ
sk_test_51H...     ← テスト環境サーバ
rk_live_51H...     ← 本番だが Read-only など限定権限
whsec_1234...      ← Webhook 署名検証
```

### 22.2 設計チェックリスト

- [ ] **prefix で role + env を必ず示す**（`sk_live_` を見たら緊張する文化を作る）
- [ ] **prefix だけでも grep / lint で検出可能に**（誤コミット時の自動検知）
- [ ] **本番キーと test キーは見た目で区別**（`live` / `test` の文字列）
- [ ] **キー本体は十分長く（128 bit 以上）、URL-safe 文字のみ**（`a-zA-Z0-9_-`）
- [ ] **後ろ N 文字だけログに残す**（`sk_live_***...XYZW`、4〜6 文字程度）
  - 全文ログ NG、完全マスク（`***`）だと識別不能
- [ ] **キー作成時にメタデータ必須**：所有者、用途、有効期限、許可 IP
- [ ] **管理画面でキーを表示するのは作成直後のみ**（再表示不可、忘れたら再発行）
- [ ] **キー単位で revoke 即時反映**（revoke リスト or キー自体を DB から削除）
- [ ] **キー単位の使用量メトリクス**（誰がどれだけ叩いたか）
- [ ] **scope（権限）をキーに紐付け**：read-only キー、write キー、admin キー を別 prefix で
- [ ] **rate limit もキー単位**（共有キーで上限食い潰し防止）

### 22.3 ライフサイクル

- [ ] **発行フロー**：管理画面 → 用途記入 → メタデータ付き → 表示は1回限り
- [ ] **ローテーション**：新キー発行 → 両方 valid 期間（grace period）→ 旧キー revoke
- [ ] **失効通知**：有効期限の N 日前に owner に email
- [ ] **無効化トリガー**：怪しい使用パターン（地理的飛び、急激な頻度上昇）で自動 revoke + alert

---

## 23. 外部公開する API

社内・自社製品内部の API と違い、**契約**として扱う必要がある。

### 23.1 設計

- [ ] **バージョニング戦略**を最初に決定：
  - URL パス：`/v1/users` — 一番分かりやすい、推奨
  - ヘッダ：`Accept: application/vnd.myapi.v1+json` — Stripe / GitHub 流
  - クエリ：`?version=1` — 非推奨（キャッシュしにくい）
  - 日付ベース：`Stripe-Version: 2024-04-22` — 細粒度バージョン
- [ ] **breaking change ポリシー**を明文化：N ヶ月予告 → deprecation header → 廃止
- [ ] **`Sunset` HTTP ヘッダ**で deprecation を機械可読に
- [ ] **REST vs GraphQL vs gRPC** を最初に決定（後変更は事実上不可）
- [ ] **OpenAPI / GraphQL schema をソース・オブ・トゥルース**に
- [ ] **エラーレスポンス形式を統一**：[RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) 推奨
  ```json
  { "type": "https://api.example.com/errors/invalid-input",
    "title": "Invalid input", "status": 400,
    "detail": "field 'email' must be a valid address",
    "instance": "/v1/users" }
  ```
- [ ] **リクエスト / レスポンスとも UTC ISO 8601** で日時統一
- [ ] **金額は整数（minor unit / 銭）**、float 禁止
- [ ] **enum 値は文字列**（数値だと意味不明）

### 23.2 認証・認可

- [ ] **認証方式を最初に決定**：API key / OAuth 2.0 / OIDC / mTLS
- [ ] **API key は §22 の Stripe 流命名**
- [ ] **OAuth 2.0 ならスコープ設計を最初に**（`read:users`, `write:users`, `admin:*` 等）
- [ ] **Bearer token は `Authorization: Bearer <token>` ヘッダ**（クエリ NG — log に残る）
- [ ] **権限は最小スコープから出発**、後から広げる（最初に admin くれてやると剥がせない）

### 23.3 レート制限・公平利用

- [ ] **レート制限を最初から実装**（後付けすると「いきなり 429 出すな」と苦情）
- [ ] **plan 別レート上限**（free: 100/min, pro: 1000/min 等）
- [ ] **レート制限ヘッダを必ず返す**：
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 873
  X-RateLimit-Reset: 1714000000
  Retry-After: 30
  ```
- [ ] **429 と 503 を使い分ける**（429 = client が多すぎ、503 = サーバ側問題）
- [ ] **idempotency key 受け入れ**（重複 POST 対策、Stripe 流 `Idempotency-Key` ヘッダ）
- [ ] **長時間処理は async pattern**（202 Accepted + polling URL or webhook）

### 23.4 ペジネーション

- [ ] **ペジネーション方式を最初に決定**：
  - cursor-based（Stripe 流：`starting_after` / `ending_before`）— 推奨
  - offset-based（`?page=2&limit=20`）— 簡単だが大量データで遅い、追加削除に弱い
- [ ] **デフォルトページサイズと最大値**を決める（default 20、max 100 等）
- [ ] **次/前ページの URL or cursor をレスポンスに**

### 23.5 ドキュメント・SDK

- [ ] **OpenAPI から自動生成された API リファレンス**を初日から公開
- [ ] **公式 SDK の言語を最初に決定**（後から増やすのは大変）
- [ ] **SDK も自動生成（OpenAPI Generator / Speakeasy）**を検討
- [ ] **Postman / Insomnia コレクション**を配布
- [ ] **ライブ playground**（Swagger UI / GraphQL Playground）
- [ ] **changelog をユーザ向けに別管理**（内部 CHANGELOG とは別）
- [ ] **status page**（statuspage.io / instatus / 自前）

### 23.6 公開時の運用

- [ ] **CORS 設定**：自社ドメインの allowlist、`*` 禁止
- [ ] **HTTPS only** + HSTS
- [ ] **API トラフィック分析**：どの endpoint が遅い、どのキーが叩きすぎ
- [ ] **deprecation の予告 → 移行期間 → 廃止 → 410 Gone** の流れを SOP 化
- [ ] **公開 API への変更は外部レビュー必須**（社内だけで決めない）
- [ ] **SLA を明文化**（uptime, latency, support response time）

### 23.7 セキュリティ追加

- [ ] **入力サイズ上限を厳しく**（公開 API は攻撃対象）
- [ ] **レスポンスサイズ上限**（DoS 対策）
- [ ] **認証失敗のレスポンス時間を一定化**（timing attack 対策）
- [ ] **エラーレスポンスから内部情報を漏らさない**（DB の table 名、stack trace 厳禁）
- [ ] **`OPTIONS` preflight の許可 method / header を最小に**
- [ ] **bot detection**（hCaptcha, Cloudflare Bot Management）

---

## 24. WAF・クラウドセキュリティ

「**アプリの前段で防げるものはアプリより前で防ぐ**」が鉄則。

### 24.1 WAF（Web Application Firewall）

- [ ] **公開エンドポイントの前に WAF を必ず置く**：
  - **CDN 統合型**：Cloudflare WAF / Fastly Next-Gen WAF / AWS CloudFront + AWS WAF — 推奨（CDN とまとめて運用）
  - **クラウド単独**：AWS WAF / GCP Cloud Armor / Azure WAF
  - **オープンソース**：ModSecurity（Nginx / Apache）
- [ ] **マネージドルールセット有効化**：
  - OWASP Top 10（Core Rule Set）
  - Known bad IPs（脅威 intel feed）
  - Known bots（良性 / 悪性の振り分け）
- [ ] **カスタムルール**：自アプリ固有の path / pattern を block / rate limit
- [ ] **DDoS 対策レイヤ**：
  - L3/L4：CDN / クラウド標準（AWS Shield Standard, Cloudflare 等）
  - L7：WAF + rate limit
  - **Shield Advanced / Cloudflare Enterprise** は事業規模次第
- [ ] **ログを別ストレージに集約**（S3 + Athena, BigQuery）— 攻撃分析に必須
- [ ] **誤検知（false positive）対応プロセス**：count モードで観察 → block に切り替え
- [ ] **ジオブロック**：サービスを提供しない国を block（攻撃面減）
- [ ] **Tor 出口ノードの block**（必要なサービスでなければ）

### 24.2 ネットワーク

- [ ] **VPC 設計を最初に**：public / private / database サブネットの 3 層
- [ ] **DB は private サブネット必須**（インターネット直接到達不可）
- [ ] **アプリも極力 private**、ALB / API Gateway のみ public
- [ ] **Security Group は最小許可**（`0.0.0.0/0` を慎重に、SSH は VPN / SSM 経由）
- [ ] **NACL でサブネット境界の deny**
- [ ] **NAT Gateway / VPC Endpoint** で egress 制御
- [ ] **PrivateLink / VPC Peering** で社内サービス間通信
- [ ] **bastion / SSM Session Manager** で SSH 廃止（鍵管理から解放）
- [ ] **VPN（Tailscale / Cloudflare Zero Trust）** を最初に決定

### 24.3 IaC（Infrastructure as Code）

- [ ] **全インフラを Terraform / CDK / Pulumi で**：手動コンソール変更禁止
- [ ] **state file を S3 + DynamoDB lock**（安全な保管 + 同時更新防止）
- [ ] **plan を PR で必ず確認、apply は CI 経由**
- [ ] **drift 検出を定期実行**（手動変更があれば検知）
- [ ] **モジュール化**：`prod` / `staging` で環境差は変数のみ
- [ ] **secret は IaC に書かない**（外部参照のみ）

### 24.4 クラウドセキュリティ全般

- [ ] **CIS Benchmark / クラウド各社のベストプラクティス**を最初に通読
- [ ] **CSPM ツール導入**：
  - AWS：Security Hub + Config + GuardDuty
  - GCP：Security Command Center
  - Azure：Defender for Cloud
  - サードパーティ：Wiz / Lacework / Prisma Cloud
- [ ] **GuardDuty / equivalent を全アカウントで有効**（異常検知）
- [ ] **S3 / GCS バケットのデフォルト public block** を強制
- [ ] **EC2 / GCE インスタンスにメタデータ v2 強制**（IMDSv2 — SSRF + EC2 認証情報窃取対策）
- [ ] **EBS / Persistent Disk の encryption 強制**（IaC ポリシーで）
- [ ] **DB スナップショットを public 禁止**
- [ ] **ECR / Artifact Registry の image scan 有効化**
- [ ] **container は non-root user で実行**（Dockerfile に `USER` 必須）
- [ ] **read-only root filesystem**（書き込みが必要な場所だけ volume mount）
- [ ] **Pod Security Standards / Kubernetes RBAC** を最初に厳しく

### 24.5 シークレット & 認証情報

- [ ] **Long-term access key を使わない**：
  - GitHub Actions → AWS / GCP は **OIDC でキーレス**
  - 人間は **SSO / Identity Center**
  - サービス間は **IAM Role / Service Account** + 短期トークン
- [ ] **キー必須の場合**は §22 の命名規則 + ローテーション
- [ ] **ハードコード検知**：Trufflehog / Gitleaks / Socket を pre-commit + CI

### 24.6 監視・インシデント対応

- [ ] **CloudTrail / Audit Logs の API 操作監視**：
  - root アカウントの API 操作（即 alert）
  - IAM 変更
  - Security Group 変更
  - リソース大量削除
- [ ] **異常な課金パターン**（§20.3 と連動）
- [ ] **不正アクセスシナリオの runbook**：
  - キー漏洩時：revoke → 影響範囲調査 → 強制ローテート → notify
  - インスタンス侵害：isolate → forensic snapshot → 再構築
  - DB 侵害：read-only snapshot → 影響データ特定 → 通知義務確認
- [ ] **incident response 訓練**を年 1 回（chaos engineering 含む）

### 24.7 コンプライアンス

- [ ] **業界規制を最初に把握**：
  - 一般：GDPR（EU）、CCPA（カリフォルニア）、APPI（日本個人情報保護法）
  - 金融：PCI DSS（クレカ）、SOX、ISO 27001
  - 医療：HIPAA（米）、各国医療規制
- [ ] **データレジデンシー**（EU データは EU 内、等）を最初に設計
- [ ] **audit 用のエビデンス自動収集**を最初から（後から証跡集めるのは地獄）
- [ ] **third-party の SOC 2 Type II / ISO 27001 認証**を契約前に確認

---

## 使い方の目安

新規プロジェクトを始めるとき、最低でも以下は **initial commit に含める**：

1. §1（リント）+ §15（リポジトリ運用）
2. §3（定数）+ §4（データ保存ディレクトリ）
3. §5（ファイル I/O）+ §6（ネットワーク I/O）
4. §9（クロスプラットフォーム CI）+ §11（テスト）
5. §18 のリポジトリ初期化セット（LICENSE, README, .gitignore, etc.）

§2（i18n）と §7（認証）は要件次第だが、**「あとから入れる」のは確実に高くつく**ので「不要」と判断するならその判断を README に明記しておく（後任者が迷わないように）。

