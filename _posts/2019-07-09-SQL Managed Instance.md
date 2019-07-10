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