---
title: "一定時間以上稼働している ECS タスクを検知する"
emoji: "🕵️" 
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "ecs", "datadog"]
published: false
publication_name: "atamaplus"
---

# はじめに
atama plusではAWS ECSを様々なケースで利用しています。サービスの基盤としてはもちろん、タスクをCIから実行することでデータベースのマイグレーションを行ったり、運用作業の一環として手動でタスクを実行したりすることもあります。

この記事ではECSタスクが意図せず起動し続けてしまうという問題に直面したので、その検知方法と調査過程での学びを紹介します。

# ECSのタスクが意図せず長時間起動している
ある時CIのマイグレーションのジョブが失敗したため再実行をしました。その後ジョブは正常に終了したのですが、失敗したジョブで実行されたタスクが終了しておらず起動したままになっているのを2日後に発見しました。

![](/images/monitor_long_running_ecs/1.png)

具体的に問題が発生していたわけではありませんが、このままでは次のような問題を引き起こす可能性があります。
- インフラコストの増加
- データベースの競合状態
- データ不整合

また、他の環境も調べてみると運用上の利用で手動実行されたタスクがいくつか起動し続けていることも分かりました。

弊社ではサービスで利用しているタスクは日次で再起動をしているので、24時間以上起動しているタスクを異常とみなし検知する仕組みを実装することにしました。

# 検知方法の調査

弊社ではDatadogを利用しているのでDatadogに送信されているメトリクスからモニターを作成し監視することをまずは考えました。ただ、[ECSのメトリクス](https://docs.datadoghq.com/ja/containers/amazon_ecs/data_collected/)にはタスクの起動時間を計るものがなかったので以下の代替案を検討しました。

- LambdaとEventBridgeを用いて、ECSの[describe tasksのAPI](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-tasks.html)を定期的に呼び出し、タスク毎に`startedAt`プロパティが24時間以上経過しているかどうかを確認

しかし、やりたいことに対し実装がオーバースペックに感じたためこちらの案は一旦見送りとしました。

さらに調査していった結果、以下の案も出たのですがまた後日に詳細に実装方法の検討をすることになりました。
- [Docker integration](https://docs.datadoghq.com/ja/containers/docker/integrations/?tab=labels)を有効化して[docker.uptime](https://docs.datadoghq.com/ja/containers/docker/data_collected/#%E3%83%A1%E3%83%88%E3%83%AA%E3%82%AF%E3%82%B9)メトリクスを監視する
- CloudWatch Agentを使って`/proc/uptime`をカスタムメトリクスとしてDatadogに送信


# 解決へのアプローチと検知方法の決定

後日調査を進める中で、ECSタスクの起動時間に関するメトリクスが本当に取れていないのか、Datadogの[Quick Graphs](https://docs.datadoghq.com/dashboards/guide/quick-graphs/)を使ってメトリクスを確認してみました。すると、ECSタスク内の各コンテナについて[container.uptime](https://docs.datadoghq.com/containers/docker/data_collected/#container-integration)が取得できていることが分かりました。

`container.uptime`はDatadogの[Container Integration](https://docs.datadoghq.com/integrations/container/)により取得できるメトリクスでECS上では自動的に有効になっており取得できていました。

> Container is a core Datadog Agent check and is automatically activated if any supported container runtime is detected.

Quick Graphsでグラフ化して見ても意図通りのメトリクスが取れていることが分かります。

```
max.container.uptime{task_name: 監視したいタスク定義の名前} by {task_arn}
```

![](/images/monitor_long_running_ecs/2.png)

よって、この`container.uptime`を監視するモニターを作成し24時間以上起動しているタスクを検知することに決めました。

# 調査に時間を要してしまった原因
最初の調査時に[Dockerのメトリクスのドキュメント](https://docs.datadoghq.com/containers/docker/data_collected/)を確認していたのですが`container.uptime`のメトリクスを見つけることができませんでした。`container.uptime`は[日本語のページ](https://docs.datadoghq.com/ja/containers/docker/data_collected/)には記載がなく[英語のページ](https://docs.datadoghq.com/containers/docker/data_collected/)にのみ記載があったため見落としてしまいました。他のクラウドのドキュメントでもよくありますが、英語のドキュメントを第一に確認すれば情報の抜け漏れは防げると学びました。

また最初からQuick Graphsで実際に取得できているメトリクスを確認していたら`container.uptime`に気づけたかもしれません。ドキュメントの確認だけで判断せずに、実際にメトリクスデータを確認して使えそうなメトリクスがあるかを判断すべきでした。

# まとめ
ECSタスクの起動時間を監視するために、`container.uptime`メトリクスを利用し、24時間以上稼働しているタスクを検知できる仕組みを構築しました。これにより、インフラのコスト増加やデータベースの競合などのリスクを未然に防ぎやすくなりました。

今回の調査で、ドキュメントだけに頼らず、実際のメトリクスを確認することの重要性を学びました。また、英語版ドキュメントを優先的に確認することで情報漏れを防ぐことができるということも再認識できました。


# We're hiring!
atama plusは教育を一緒にテクノロジーで解決していけるエンジニアを探しています！
興味があればぜひご応募ください！
https://herp.careers/v1/atamaplus
