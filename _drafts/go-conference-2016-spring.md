---
title: Go Conference 2016 Spring でElastic Beatsについて話してきました。
---

こんにちは。xxxの平田 ([@daichild](https://twitter.com/daichild)) です。

少し時間が立ってしまいましたが、4/23に開催された[Go Conference 2016 Spring](http://gocon.connpass.com/event/27521/)で[Elastic Beats](https://www.elastic.co/products/beats)について登壇させていただきました。CAリワードでは積極的にGoの導入を進めており、その一環として導入したElastic Beatsと、そこで得られた知見を発表する事が出来とても良い機会でした。今回はその時の発表資料を公開します。

## ElasticBeatsを導入してみた話

<script async class="speakerdeck-embed" data-id="50ffafdc20ba4b63a3f36289207fba6d" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

資料でもある通り、弊社ではリアルタイム処理基盤としてApache Kafkaを導入しています。その処理の性質上遅延は出来る限り避けたい所ですので、リアルタイムなシステムのモニタリングが課題としてありました。モニタリングツールの導入を検討していたタイミングでは、既存のツールのKafka 0.9以降に導入されたKafka-based offset storageへの対応が不完全なものが多かったため、Elastic Beatsを導入することにしました。既にElasticsearch + Kibanaで構築されたアクセスログの分析・可視化基盤が存在しており、環境としての相性と導入の容易さも選定の理由としてあります。

Kafkaのメトリクス収集部分に関しては独自で開発する必要がありましたが、Elastic Beatsが独自のプラグイン開発を前提として作られていて、実際に開発してみると本質的な部分以外を書く必要がほとんど無い所が好印象でした。この辺の話は是非資料を見ていただければと思います。

## GoやGCPへの取り組みについて

CAリワードでは、APIサーバーや運用ツールにGoの導入を進めています。既存のPHPで動いているシステムに関してもどんどんGoへのリプレースが進んでいます。また、今回のようなログの分析基盤には積極的にGCPを活用しており、今回のシステム以外にもBig QueryやData Procを使った分析システムも稼働し始めています。

これからも積極的にGoを取り入れながら新しいことに挑戦し続けて行きますので、GoやGCP、ハイパフォーマンスなシステムの構築に興味のある方は是非連絡ください。

