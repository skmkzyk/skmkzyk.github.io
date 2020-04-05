---
layout: post
title: "SQL Database error"
---

## A connection was successfully established with the server, but then an error occurred during the login process. (provider: SSL Provider, error: 0 - The target principal name is incorrect.)

というエラーが出たら、
> `Server=tcp:x.x.x.x,1433;Initial Catalog=xxxxxxxx;Persist Security Info=False;User ID=xxxxxxxx;Password=xxxxxxxxxxxx;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=True;Connection Timeout=30;`
とやればとりあえずうまくいく。

が、[暗号化を使用した接続](https://docs.microsoft.com/ja-jp/sql/connect/jdbc/connecting-with-ssl-encryption) によると

> encrypt プロパティが true に設定され、trustServerCertificate プロパティが true に設定されている場合、SQL Server 用 Microsoft JDBC ドライバー では SQL Server の TLS 証明書が検証されません。 これは、通常、テスト環境 (SQL Server インスタンスが自己署名入りの証明書しか備えていない環境など) で接続を許可する場合に必要になります。

という感じらしいので、注意する必要がある。。

そもそもどういう証明書を提示してきてるのか気になるので調べる。