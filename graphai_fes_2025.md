# GraphAI Contribution Fes 2025 開催のお知らせ

[メルマガ「週刊 Life is Beautiful」](https://www.mag2.com/m/0001323030)やGitHubですでに告知しておりますが、**GraphAI Contribution Fes 2025** を開催します。

本イベントは、[メルマガ「週刊 Life is Beautiful」](https://www.mag2.com/m/0001323030) / [Receptron](https://github.com/receptron/) が主催し、NPO法人 [Singularity Society](https://singularitysociety.org/) / [Raycast Community Japan](https://raycast.connpass.com/) / [DevX](https://devx.jp/) がサポートする形で実施されます。

イベント概要や詳細については、次のリンクをご参照ください: [イベント概要 - GitHub](https://github.com/snakajima/life-is-beautiful)

---

## **GraphAI Contribution Fes 2025 の目的**

本イベントでは、**GraphAI** フレームワークを用いたアプリケーション・サービス・ツールの開発、または GraphAI 自体への貢献を目的としています。  
GraphAI は、フロントエンドとバックエンドでシームレスに動作する Multi-Agent AI Workflow フレームワークで、以下の特徴を持っています：

- **TypeScript** で記述されている  
- 複雑な非同期処理を簡潔に記述可能  
- サーバー・ブラウザ双方でコード変更なしで動作可能  
  - CLI で開発・デバッグしたものをそのままブラウザで動作させる  
  - ブラウザで設計したワークフローをサーバーバッチで動作させる
  - ブラウザで Agent を実行し、ブラウザの API を呼び出す  
- グラフ構造のワークフローと Agent 実装を分離し、HTTP 経由で別環境に Agent を配置可能
  - Workflow 実行と Agent 実行の分離を実現
  - 各 Agent の実行環境を個別に制御可能
  - TypeScript 以外で記述された Agent の呼び出しも可能  
- 複雑な分散環境でも Stream 処理を容易に実現  
- Agent 自体がスキーマやサンプルコードといったメタ情報を保持し、AI による動的グラフ構成が可能  
- 標準的な Agent ホスティング機能を提供  
- 複数サーバーに分散された環境を考慮  


---

## GraphAI Vision
GraphAIは、マルチエージェントシステムを構築する際に、各処理をマイクロサービスのように細かく分割することが可能です。この処理はサーバーやブラウザなど、さまざまな環境で動作させることができます。

### 将来のAI処理の展望
今後、AIに関連する処理は、粒度の大小を問わず、世界中に分散されたサーバーやデバイスから提供されるようになると考えられます。GraphAIを使用することで、こうした分散されたエージェントを柔軟に組み合わせ、効率的に活用することが可能です。

### 分散エージェントと動的ワークフローの生成
GraphAIでは、ワークフローを定義するグラフデータがAIにとって学習しやすい構造を持っています。そのため、分散されたエージェントのリストやユーザーからのリクエストを基に、動的にワークフローを生成することができます。

GraphAIはこの未来に向けた機能やツールをすでに開発済みです。GraphAIが描くビジョンは、単なる未来像ではなく、今この瞬間から実現できる現実です。
サーバー、ブラウザ、そしてあらゆるデバイスで動作するエージェントを作り、世界中に分散された知能をつなぎ合わせる仕組みを、一緒に作り上げていきませんか？
AIの可能性を解放し、次世代のインフラを共に構築する仲間として、あなたの参加を心からお待ちしています。
さあ、GraphAIとともに、未来を切り拓きましょう！

---

## **参加方法**

以下の内容を Singularity Society の問い合わせフォームから送信してください：  
[問い合わせフォーム](https://docs.google.com/forms/u/7/d/e/1FAIpQLSfk1cnTUwBvalTqm9un_p9Oyl9LckVgmKC40ifyhtAU3BcTuw/viewform)

* 参加申し込み、Slackへの参加をしていないと、いくらコントリビュートしても審査の対象にならないので忘れず参加をお願いします

### 必要情報
1. **名前**（個人またはチーム）  
   - チームの場合、全メンバーの氏名を記載（本名、旧姓の本名、通名のいずれか）  
2. **GitHub アカウント名（必須）**  
   - チームの場合は Organization 名、個人の場合はアカウント名  
3. **SNS 情報**（X, Zenn, Qiita など）  
   - GraphAI に関する投稿も審査対象とします  
4. **過去に作成したプロダクトやオープンソース URL**  
5. **参加計画**（フェスで行う予定の内容）  
   - 変更可、応募時点での案を記載してください  
6. **メールアドレス（Slack 招待用）**  
   - チームの場合、全員分
---

## **Slack 参加について**
登録後、Slack の専用チャンネルへ招待いたします。イベント情報や連絡はすべて Slack を通じて行うため、**1 日 1 回以上の確認**をお願いします。

### 注意事項
- **行動規範の遵守**
  - [行動規範 - GitHub](https://github.com/SingularitySociety/societys_statement/blob/main/code-of-conduct.md)  
- **Slack のルール**
  - [Slack 参加ルール - GitHub](https://github.com/SingularitySociety/societys_statement/blob/main/SlackRule.md)  
- 個人情報や機密情報の取り扱いには十分ご注意ください。

---

## **ハッカソンの詳細**

- **日時**: 2025年3月22日(土)～3月23日(日)  
- **場所**: 渋谷  

Contribution Fes参加者を対象にハッカソンを行います。ハッカソンでは、2 日間でプロジェクトを仕上げる機会を提供します。3 月 23 日(日)の午後には成果物のプレゼンを行う場を用意しています。  

- **プレゼンの参加は任意**ですが、オンライン(調整中)/ オフラインでの発表が可能です。  

---

### Contribution Fes 成果物提出締切
- **3 月 22 日(土)** までに成果物を一覧を[GitHub上](https://github.com/snakajima/life-is-beautiful) のmdファイルに、PRの形で提出してください。mergeでコンフリクトには気をつけてください。
- 3 月 23 日(日)のプレゼン終了後、審査結果を発表予定です。
- スケジュール（予定）

   |     | 3/22(土) | 3/23(日) |
   |:--- | :---     | :---  |
   |午前  | ハッカソン | ハッカソン  |
   |午後  | ハッカソン | Contribution Fes <br> - プレゼン <br> - 審査 <br> - 結果発表

## 審査
- 2025年3月23日までに行った全コントリビューションが評価対象となります。  
- 成果物を一覧提出してください
- 3月23日(日)プレゼンの発表終了後、審査、結果発表の予定です
- ハッカソンへの参加、3/23のプレゼンは任意です。参加、発表をしなくても審査の対象になります。
- ハッカソン参加および 3/23 プレゼンの登録は別途行います。

---

多くの方の参加を心よりお待ちしております！
