# vibe_coding の CLAUDE.md の節を足す（Issue #139）

## ゴール

`development/vibe_coding.md` の「CLAUDE.md / md ファイルの書き方」に、ルールの置き場所の分け方、長くなったときの `docs/` への分割、skill とスクリプトへの切り出し、AGENTS.md（Codex など）の扱いを足す。

## 方針

- 「CLAUDE.md / md ファイルの書き方」を小見出しで分ける: 置き場所 / 長さ / docs に分ける / AGENTS.md
- 参照の書き方は二通りあることを書く。普通のパスは必要なときに開かれ、`@path` は起動時に毎回読み込まれる（Claude Code 公式文書）
- skill の話は既存の「定型タスクは skill 化する」に足す。重複させない。prompt → skill → スクリプトの順で固めていく、と書く
- AGENTS.md は公式文書で確認したことだけ書き、確認した時期を添える（数ヶ月で変わるため）

## AGENTS.md について確認したこと（2026年11月）

- Codex: `~/.codex/AGENTS.md`（共通）→ Git のルートから作業ディレクトリまでの各階層の `AGENTS.md`。近い階層が後に連結され優先される。`AGENTS.override.md` があればそちらを読む。合計の上限は `project_doc_max_bytes`（既定 32 KiB）
  - https://developers.openai.com/codex/guides/agents-md
- Codex の skill: `.agents/skills/`（リポジトリ）、`~/.agents/skills/`（ユーザー）。Agent Skills 標準の `SKILL.md`、`scripts/` を持てる
  - https://developers.openai.com/codex/skills
- Claude Code: CLAUDE.md がなければ AGENTS.md を読む（v2.1.277 以降）。CLAUDE.md があると既定では AGENTS.md は読まない。両方使うなら CLAUDE.md に `@AGENTS.md` と書くのが公式の勧め
  - https://code.claude.com/docs/en/memory
