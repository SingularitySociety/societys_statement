# Gitを使った開発について

以下のような使い方を推奨しています。
- 2人以上での開発の場合は、Pull Request + Reviewを必ず行う
- rebaseではなくmerge。
- commit/Pull Request
  - commitの粒度は極力小さめに。
  - 最大でも数時間、できれば数分でreviewできる程度にPull Requestをつくる
    - review時間がかかると
      - マージされないので、開発がとまる。
      - conflictする可能性も増える
  - 粒度が大きくすると、commitの頻度が減り、進捗の状況がわからない。
    - commit = 進捗
  - commitの粒度が小さい場合は、開発の方向性や実装に間違いが合った場合にすぐに軌道修正できる
    - 粒度が大きい場合は間違った場合に軌道修正のコストも大きくなる（捨てるコードや変更が大きくなる）

- 整形
  - editor configや、prettierなどのformatterを使って、各環境、エディターによる差分はなくす
