---
layout: post
title:  "SQL Managed Instance"
---

# SQL Managed Instance

## Cost

割りと最低限のリソースに絞ったつもり（4 vCore、ストレージ 32GB）だとしても 10 万円コースからになるので、手軽に作れるものではないですね。。

![cost](/assets/cost.png)

## Deploy Time

ASE も同じだが、VNet Injection の PaaS はかなり時間がかかるということで覚悟していたんだが、意外と 3.5h くらいで作成できた。

![deploy-time](/assets/deploy-time.png)

## Deliverables

Deploy の結果作成されるもの。今回は全く新しい環境に作成したので VNet から NSG、Route Table も作成されてる。中心的なものは Virtual Cluster と SQL Managed Instance かなと。Virtual Cluster はなかなか見慣れないものが作成されている。その他に標準で見えていないもの（非表示の型の表示で出る）は networkIntentPolicies というのがあって、これによって Route Table と NSG がある subnet に確実に適用されるように制限されているみたい。

![deliverables](/assets/deliverables.png)