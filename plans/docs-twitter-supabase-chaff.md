# Twitterクローン教材を chaffjs で直す（Issue #101）

同じことを別のシリーズでやるときの手順として残す。

## chaff は何をする道具か

prose の linter。**文章は書き換えない**（`chaff --help` の冒頭に "It never rewrites the text." と書いてある）。
指摘を出すところまでが chaff の仕事で、直すのは人間か AI の側。

使ったコマンド:

| コマンド | 何をするか |
| --- | --- |
| `npx chaffjs <dir> --compact` | 検査。1 指摘 2 行で出る |
| `npx chaffjs genres` | ジャンル（文書の種類）の一覧 |
| `npx chaffjs explain <rule>` | ルールの理由と段階（strict/normal/relaxed/off） |
| `npx chaffjs compare <before> <after>` | 書き換えで事実が落ちていないか |
| `npx chaffjs relax\|off <rule> --why "理由"` | ルールの段階を変え、理由と日付を chaff.yaml に書く |
| `npx chaffjs suppressions <dir>` | stet で黙らせた指摘の一覧 |
| `npx chaffjs skill` | Claude Code 用の skill を書き出す（手順が入っている） |

## 順番

1. **ジャンルを先に決める。** 決めないと `blog/tech` で検査され、絵文字の見出しなどが全部指摘になる。
   この教材は手順書なので `docs/manual`。`chaff.yaml` に書いて固定した。
2. **検査して、指摘の多いルールから見る。** 表示に出る不具合（リンク切れ・表示されない強調）を先に直す。
3. **指摘ごとに三択**（chaff の skill が示す形）
   - 直す / その場所だけ `stet` ＋ 理由 / ルールを `relax`・`off` ＋ 理由
   - **どれを選んでも理由を書く。** 黙らせた理由は `suppressions` で一覧でき、ルールの理由は chaff.yaml に残る
4. **書き換えたら必ず `compare`。** 下の失敗は compare が見つけた。

## 失敗から学んだこと

- **機械的な置換は引用の中まで届く。** 英数字前後の空白をそろえるとき、AI へのプロンプト例（鉤括弧で引いた指示文）の中まで書き換えてしまった。
  compare が「引用が落ちて別の引用が足された」と出して気づいた。chaff 自身のルールにも「鉤括弧で引いた中は数えない」と書いてある。
- **鉤括弧の除外は 1 行の中だけでは足りない。** 複数行にまたがる引用（プロンプト例）は、行ごとに見ると開きと閉じが別の行にあるので素通りする。
- **伏せ字で保護するなら、入れ子を戻しきるまで繰り返す。** 1 回だけ戻すと伏せ字が本文に残り、`invisible-character` が大量に出た。

## 残したもの

- `colloquial-opener`（文頭の「でも」「なので」）は relaxed のまま。完全未経験者向けの語りかけとして意図したもの
- `stet` で黙らせた 4 箇所は、すべてその場に理由を書いた

## chaff 側の誤検知（報告する価値があるもの）

PR の本文にまとめた。再現手順つき。
