---
layout: post
title: "Secure SQL Database connection from Web Apps by Private Endpoint"
image: "/assets/thumbnail-web-apps-sql-database-private-endpoint.png"
---

## Prerequisites

今回は、Web Apps から SQL Database への接続を Private Endpoint でセキュアにする感じのをやります。

- Web Apps

    代表的な PaaS の一つで、ソースコード投げ込めば Web Application が動く。
    [App Service の概要](https://docs.microsoft.com/ja-jp/azure/app-service/overview)

- SQL Database

    代表的な PaaS の一つで、SQL Server (完全互換ではない) のマネージドサービス。
    SQL を投げ込めばデータの出し入れができて、脆弱性の管理とかはお任せ。
    [Azure SQL Database サービスとは](https://docs.microsoft.com/ja-jp/azure/sql-database/sql-database-technical-overview)

- Private Endpoint

    2020 年 3 月頃に GA になったイケてるサービス。
    これがあることでネットワーク構成はだいぶ選択肢が広がった。
    [Azure プライベート エンドポイントとは](https://docs.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview)

- Key Vault

    ConnectionString とか SSL 証明書とかを安全に保存しておいてくれる。
    アクセスは Azure AD を利用した Managed ID を使えば、Key Vault へのアクセスのための id/pass はどこに保存すれば? といったループを断ち切れる。
    [Azure Key Vault とは?](https://docs.microsoft.com/ja-jp/azure/key-vault/key-vault-overview)

## Architecture

今回、図がきれいにかけたのでとてもうれしい (今回の post の趣旨はここ)

![Architecture](/assets/web-apps-sql-database-private-endpoint.svg)

## Disclaimer

今回、`TrustServerCertificate=True` で設定していて、`TrustServerCertificate=False` の状態でつながるまでには至っていません。
private なので許してもらえる世界もあれば、ダメな世界もあると思います。

## Configuration

設定のポイントをいくつか。

### Web Apps

VNet Integration を利用するので、[アプリを Azure 仮想ネットワークに統合する](https://docs.microsoft.com/ja-jp/azure/app-service/web-sites-integrate-with-vnet) のとおり Standard 以上の SKU の App Service Plan を利用して、Web Apps を作成します。

- VNet Integration

    最近 GA になった (Regional) VNet Integration を有効にしておきます。
    こちらは専用の Subnet が必要です。

    補足までに、`WEBSITE_VNET_ROUTE_ALL` は今回設定していません。

- Application Settings

    ConnectionString は Application Settings の画面のところで設定しています。
    今回は、[App Service と Azure Functions の Key Vault 参照を使用する](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-key-vault-references) を利用して、ConnectionString 自体は Key Vault に保存しています。

- Managed ID

    また、Key Vault との連携のため、Managed ID を利用しています。
    Azure Portal から設定できるので難しくはないですが、詳しくは [マネージド ID で Key Vault の認証を提供する](https://docs.microsoft.com/ja-jp/azure/key-vault/managed-identity) を参照。

### Key Vault

メインのコンポーネントではないので、Architecture の図には出てきていませんが、利用しているので記載しておきます。
要件次第ですが、SKU は Standard で問題ないと思います。

- Secret

    上記の、[App Service と Azure Functions の Key Vault 参照を使用する](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-key-vault-references) を利用しているので、Secret として ConnectionString を保存します。

- Access Policy

    そのうえで、Access Policy として、Web Apps の Managed ID に対して「Get Secret」の Permission だけを与えておきます。

### SQL Database

PaaS としての SQL Server も作る必要がありますが、適当に現行モデル Gen5 の vCore x2 で作成しておきます。

- Firewalls and virtual networks

    SQL Server の「Deny public network access」は「Yes」に変更します。
    これにより、Public IP からのアクセスをすべて遮断して、Private Endpoint のみから利用できるよう設定します。
    また、SQL Database の Service Endpoint も今回利用しません。

- ConnectionString

    ConnectionString の今回のポイントとしては、

    > Server=tcp:10.80.40.4,1433;Initial Catalog=xxxxxxxx;Persist Security Info=False;User ID=xxxxxxxx;Password=xxxxxxxxxxxx;MultipleActiveResultSets=False;Encrypt=True;
    `TrustServerCertificate=True`;Connection Timeout=30;

    として、SQL Server から提示された証明書に対して証明書チェーンの信頼を確認しない設定となっています。
    Disclaimer にも書きましたが、ここは解消したいポイントではあります。

### Private Endpoint

[クイック スタート:Azure portal を使用してプライベート エンドポイントを作成する](https://docs.microsoft.com/ja-jp/azure/private-link/create-private-endpoint-portal) を見ながら作っておきます。
今回は Endpoint 用に専用の Subnet を作った絵になっていますが、必ずしも専用にする必要はないです。

### ASP .NET Application

Web の通信であれば Web Apps の Console から curl たたけば確認できるんですが、sqlcmd がどうもうまく動かなかったので適当な ASP .NET Application を借りてきています。
[チュートリアル:SQL Database を使用して Azure に ASP.NET アプリを作成する](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-dotnet-sqldatabase) から、.zip を落としてきて、Web Apps にデプロイします。
今回試したケースだと、Web Apps をすでに作ってあったので、Publish Settings をダウンロードしてきてから publish してます。
また、MyDatabaseContext.cs に対して
```
public MyDatabaseContext() : base(ConfigurationManager.ConnectionStrings["MyDbConnection"].ConnectionString)
```
の箇所だけ少し修正を加えています。

## Test

ここまで設定ができれば問題なく ASP .NET のアプリが動作し
Todo の追加/削除ができるのではと思います。
別の VM でも建てて、SSMS を利用して接続すればデータが保存されていることが確認できます。

![ASP .Net Todo Application](/assets/asp-net-todo-app.png)