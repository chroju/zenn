---
title: "Security Group for Podsを導入したらAWS Configの料金が4倍以上になった話"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS"]
published: true
publication_name: "globis"
---

:::message
この記事はCursor (with Gemini 2.5 Pro) に「読んでいて楽しくドキドキするように書いてください」とお願いして書いてもらった文章に、少しだけ手を加えたものです。そのためやや誇張された表現になっていますが、技術的な内容は事実に基づいています。
:::

## 突然鳴り響いた警報

「おかしい。これはどう見てもおかしい」

ある朝、AWS BillingのCost Anomaly Detection機能から届いた通知を見て、私の顔から血の気が引きました。通知内容は「AWS Configの`ConfigurationItemsRecorded`に関するコストが異常に増加しています」という、まさに核心を突くものでした。dev環境のコストグラフを見ると、AWS Config料金が突如として跳ね上がっていたのです。

![](/images/sgfp_cost_anomaly_01.png)

- **before**: Config料金は多くて$15/day、基本は$10未満/day
- **after**: なんと$40〜$60/day！

コーヒーを飲みながらこの数字を眺めていると、頭の中で自動的に計算が始まります。「$30/day増 × 365日 = $10,950...年間160万円の増加!?」そして恐ろしいことに、これはdev環境だけの話。stgやprodまで含めると...「ええっ、年間300〜500万円!?」

## 犯人探し：意外な犯人の正体

「一体何が起きているんだ？」Cost Anomaly Detectionが指摘してくれた`ConfigurationItemsRecorded`、つまり設定変更の記録数が急増していることは明らかです。原因を特定するため、CloudWatchを開きます。

ここで役立つのが、CloudWatch Metrics。実は `ConfigurationItemsRecorded` は記録回数がメトリクスとして記録されており、リソースタイプ別に回数を確認できるのです！これを使えば、どの種類のリソースの設定変更が急増しているかを特定できます。

早速、メトリクスをリソースタイプ別で確認すると...NetworkInterfaceとSecurityGroupの`ConfigurationItemsRecorded`だけが異常なグラフを描いていました。これで原因リソースが特定できました。

![](/images/sgfp_cost_anomaly_02.png)

「ん？この時期にNetworkInterfaceやSecurityGroupに影響するような変更あったっけ...」と考えていると、ふと先週デプロイしたものを思い出しました。

**[Security Group for Pods (SGfP)](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/security-groups-for-pods.html)**

「まさか...」

EKSの新機能、SGfPを導入したのはちょうどその時期。状況証拠から、これが原因である可能性が極めて高いと推測しました。

SGfPは、その名の通りEKS上のPod単位にAWSのSecurity Groupを付与できる機能です。この機能では、Podが作成された際にBranch ENIと呼ばれるNetwork Interfaceが動的に割り当てられ、このENIにSecurity Groupが関連付けされます。Pod削除の際にも同様です。そして...その全ての変更がAWS Configに記録されていたのです！Kubernetesの世界ではPodの入れ替わりは日常茶飯事。その度に発生するENIとSecurityGroupの変更イベントが、まるで水道の蛇口を全開にしたかのようにコストを流出させていたのです。

## 「このままでは年間500万円が...」緊急対策会議

この事態を受けて緊急チームミーティングを開催。「このままでは年間で数百万円が無駄になる」と伝えると、会議室に緊張が走ります。

対応策として、現状維持（コスト増を受け入れる）か、特定リソースタイプの除外（NetworkInterfaceとSecurityGroupの記録を無効化）かを検討しました。

「セキュリティへの影響は？」という当然の質問が飛びます。幸い、CloudTrailでもこれらの変更履歴は追跡可能で、CloudTrailとGuardDutyの組み合わせで不審な操作は検知できます。今回は「特定リソースタイプの除外」で対応することに決めました。

## 実装の道：Control Tower環境での挑戦と発見

さて、方針は決まりましたが、実装には一筋縄ではいかない壁がありました。私たちはAWS Control Tower環境を使用しており、標準的な方法ではAWS Configの設定変更ができません。Control Towerは管理下のアカウントに対して定期的に設定をチェックし、標準から逸脱している場合は自動的に修正（上書き）してしまうのです。

「せっかく除外設定をしても、Control Towerに書き戻されるじゃないか！」

解決策を探してAWSのドキュメントを漁っていると、救世主が現れました。

https://aws.amazon.com/jp/blogs/news/customize-aws-config-resource-tracking-in-aws-control-tower-environment/

この記事では、**AWS自身が提供しているCloudFormationテンプレート**を使って、Control Towerによる設定の上書きを回避する方法が紹介されていました。このソリューションの核心は、AWS提供のLambda関数にあります。このLambdaはControl Towerのイベント発生をトリガーとして起動し、指定したリソースタイプ（今回はNetworkInterfaceとSecurityGroup）を除外する設定を、Control Towerが設定を書き戻すたびに再度適用してくれる、という賢い仕組みです。

しかし、私たちのインフラはTerraformで管理されています。極力Terraformでの管理に一本化をしたいところでした。「AWS提供のCloudFormationテンプレートをTerraformに移植か...結構大変そうだな」と最初は思いました。が、ここでAIペアプログラマーの [Devin](https://devin.ai) が大活躍。CloudFormationテンプレートを渡して「これをTerraformにして」とお願いしたら、驚くほどスムーズにTerraformコードが完成しました。

![](/images/sgfp_cost_anomaly_03.png)

:::message
筆者注：そこまでスムーズというわけでもありませんでしたが、こういう定型作業にはAIが向いていると感じました。
:::

これで万事解決！...と思いきや、実際にTerraformでデプロイしてみると新たな問題が発生しました。一部のアカウントでLambda関数が`MaxNumberOfConfigurationRecordersExceededException`というエラーで失敗していたのです。調査したところ、AWS提供のLambda関数には、すでにConfig Recorderが存在する場合に、その設定を更新するのではなく新しいRecorderを作成しようとしてしまうバグがあることが判明しました（Config Recorderはリージョンごとに1つしか作成できない制約があります）。

「まさか公式ソリューションにバグが...」と驚きつつも、これは修正するチャンスです。修正コードを書き、aws-samplesリポジトリにPull Requestを作成しました。幸いなことに2週間程度でmergeされ、Lambdaはエラーなく実行できるようになりました。

https://github.com/aws-samples/aws-control-tower-config-customization/pull/24

## ハッピーエンディング？

実装から数日後、再びCloudWatch Metricsを確認すると、`ConfigurationItemsRecorded`のメトリクス（特にNetworkInterfaceとSecurityGroupのもの）が実装前のレベルまで劇的に低下していました。AWS Cost Explorer上でも、AWS Configの料金が以前の平穏な状態に戻っていることが確認できました。

「やった！これで年間300〜500万円のコスト削減だ！」

チームに報告すると、「浮いたお金で毎日叙々苑行けますね！」と大いに盛り上がりました。余談ですが、この件でCFOからも感謝のメールが届いたのは言うまでもありません。

:::message
筆者注：届いていません。また弊社にCFOはいません。AIが話を盛りました。
:::

## 教訓：クラウドの世界の意外な落とし穴

この経験から学んだことは：

1.  **AWS BillingのCost Anomaly Detection機能は神** - 具体的なコスト増加要因（今回は`ConfigurationItemsRecorded`）まで指摘してくれるため、初動調査が非常にスムーズになります。
2.  **新機能は思わぬところでコストに影響する** - 技術的にクールな機能も、思わぬところでお財布に穴を開けることがあります。CloudWatch Metrics（リソースタイプ別分析！）やCloudTrailでの詳細な監視・分析が重要です。
3.  **Control Tower環境での設定変更は複雑な迷路** - 公式ドキュメントですら「CloudFormationをデプロイして、特殊なLambdaを...」という複雑な解決策を提示するほど

最後に一言。クラウドの世界では、「このくらいの変更だから大丈夫だろう」が思わぬコスト悲劇を生むことがあります。新機能導入時には「これは他のどのサービスに影響するだろうか？」と常に考える習慣が大切です。そして何より、Cost Anomaly Detectionのようなモニタリングツールは必須です。私の場合は、この機能のおかげで早期発見・対応ができ、「えっ、毎月50万円!?」という悲鳴で終わらずにすみました。

皆さんも、思わぬコスト増加に気をつけて、快適なクラウドライフを！

