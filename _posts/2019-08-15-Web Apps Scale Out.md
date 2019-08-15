---
layout: post
title:  "Web Apps Scale Out"
---

# Scale out するとどうなるのかの検証

1. ASP .Net を runtime として、適当に Web Apps を作る
1. Scale Out のメニューから Manual Scale させる（Basic で作ったので最大 3）
1. App Service Editor (preview) を開く
1. Default.aspx を新規作成する
1. 中身を書く
```
<%= Request.ServerVariables("LOCAL_ADDR") %>
```
6. Web Apps の URL にアクセスすると、Web Apps 自身の IP アドレスが出力される
1. 別のアクセスに見せかけるために InPrivate ブラウズして同 URL にアクセスすると、さっきの IP アドレスとは別の IP アドレスが表示される、ことがある

結論としては、Load Balance はされていそう、という話。

# Reference

https://www.w3schools.com/asp/coll_servervariables.asp にどんな環境変数が使えるか書いてある。
書き方はこのリンクに書いてある形式じゃなくて、上に書いてある感じにする。
``<%=`` で始めるか ``<%`` かどうかでなんか違うんだろう