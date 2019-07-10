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

Deploy の結果作成されるもの。今回は全く新しい環境に作成したので VNet から NSG、Route Table も作成されてる。中心的なものは Virtual Cluster と SQL Managed Instance かなと。Virtual Cluster はなかなか見慣れないものが作成されている。その他に標準で見えていないリソース（非表示の型の表示で出る）は networkIntentPolicies というのがあって、これによって Route Table と NSG がある subnet に確実に適用されるように制限されているみたい。

![deliverables](/assets/deliverables.png)

## Route Table

作成された Route Table の中身はこんな感じ。なんとなく隠したほうがいいと思って blur をかけたんだが、普通に [docs](https://docs.microsoft.com/azure/sql-database/sql-database-managed-instance-connectivity-architecture#user-defined-routes) に載ってた。正確に言うと、これ以外に SqlManagement_automation_backup、SqlManagement_controlPlane_az01、SqlManagement_security_primary といった Route も追加されている。これは作成した SQL Managed Instance によって異なるから docs に書かれていないのかもしれない。

![route-table](/assets/route-table.png)

Route Table に対して、強制トンネリングのための Default Route（0.0.0.0/0）を追加することはできるが、既存の mi-103-25-156-22-nexthop-internet といった Route を消そうとすると networkIntentPolicies のエラーが出ることになる。もともと Route Table がある状態で SQL Managed Instance を作成した場合にどう整合性をとるのかは検証していない。同様に、そもそもの Route Table と subnet の紐付きを外そうとすると大量のエラーが出る。

![route-table-delete-error](/assets/route-table-delete-error.png)