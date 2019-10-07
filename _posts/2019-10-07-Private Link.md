---
layout: post
title:  "Private Link"
---

# Private Link についていろいろ調べてみた

基本的な内容は [docs](https://docs.microsoft.com/ja-jp/azure/private-link/private-link-overview) を見てくださいませ。
こちらは、とりあえずつくってみた、VM もおいてみていろんなアクセス方法を試してみた、という感じの記事です。

Disclaimer として、今のところ Preview の機能を試しているだけで、AS IS の状態を示しており、GA した際には変わる点も多いかとは思います。
また、[制限事項](https://docs.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview#limitations) もあるので、こちらも参照ください。

## Cost

Cost かかるんかなぁ、と思ったらかかるみたいです。
若干苦手な、あるだけで Cost がかかる系のサービスです。
ネットワーク系で同じようなのだと VPN Gateway、Azure Firewall などがあります。
公式の Document を参照する限りだと 0.56 x 24 x 30 = 403.2 円 (Preview 期間中は半額らしい) です。
と、Traffic に応じた課金の合算になるようです。

![Cost](/assets/private-link-cost.png)

## Route

Private Link って結局 subnet に所属してるので、特に追加の Route は差し込まれないかと思ったら /32 が追加されてた。
なんのために必要なんだろうね。
Next Hop Type は InterfaceEndpoint らしい。

![Effective Route](/assets/private-link-effective-route.png)

Private DNS Zone が紐づいている VNet #1 と紐づいてない VNet #2 を (Global) VNet Peering してみた結果、この /32 は聞こえてきます。
同じく、/16 が聞こえてきてるのでこれ要るんかなぁという気はする。

![Route Propagation - VNet Peering](/assets/private-link-route-vnet-peering.png)

## DNS

[docs](https://docs.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview#dns-configuration) に以下の記載があるとおり、DNS の名前解決は何もしなくても Private IP Address が返ってくるように変更されます。

> アプリケーションで接続 URL を変更する必要はありません。 パブリック DNS を使用して解決を試みると、DNS サーバーはプライベート エンドポイントに解決するようになります。 このプロセスはアプリケーションに影響しません。

具体的には、Private Link を作る際に、Private DNS Zone を作るかどうかを聞かれます。
いわれるがまま Private DNS Zone を作成すると、privatelink.database.windows.net のゾーンが作成されます。

![Private DNS Zone](/assets/private-link-private-dns-zone.png)

また、これが Private Link Endpoint の NIC が所属する Subnet (の所属する VNet) に紐づいて、Azure 既定の DNS サーバである 168.63.129.16 に問い合わせした際にこの Private Zone に CNAME が向いて、うまく Private IP Address が返ってくるように変わります。

![Private DNS Zone - VNet links](/assets/private-link-vnet-links.png)

で、Private DNS Zone の VNet links を追加すれば、その VNet でも同じ名前解決の結果となるように変わります。
もちろん、VNet 同士は何らかの方法で接続する必要はありますが。

その状態で、遊んでみようということで、VNet の外側から hoge.privatelink.database.windows.net. を叩くと、適宜 CNAME で曲がって、ちゃんと Public IP アドレスに解決されます。
まぁ、実際にこれを使う場面はないですが、ちゃんとしてますね、という確認までに。

![nslookup](/assets/private-link-nslookup.png)

## ARM Template

Resource Explorer (https://resources.azure.com/) で Resource にアクセスしてみて、どんな具合かを見てみる。
普通の NIC と比較してみると、まず `location` 属性がない。
この点が原因なのか、遊びで Private IP アドレスをつけてみようとしてもくっつかない。

![Metadata location](/assets/private-link-metadata-location.png)

それからこれも面白いのが、MAC Address が空になっている。

![Metadata MAC Address](/assets/private-link-metadata-mac-address.png)

ちなみに Subnet には `purpose` 属性がつく。

![Metadata Subnet purpose](/assets/private-link-metadata-subnet-purpose.png)

あとから気づいたけど、Resource Explorer が Private Link にまだ対応してない。
本来なら Microsoft.Network/privateEndpoints があるはず。

![Resouce Explorer missing](/assets/private-link-resource-explorer-missing.png)

というのも、deploy の template を見ると、ほかの parameter も設定されてる。

![missing property](/assets/private-link-missing-property.png)

## Monitoring

運用も大事ですからね、この辺確認しておきます。

Diganostics の設定は現状で見当たらず、Metric のみ。
といっても Byte In/Out しかとれないので現時点では最低限、という感じかと。

![metric](/assets/private-link-metric.png)

とはいえ、ほかに何が欲しいかといわれると悩む。
NSG 引っかからない気がするから Flow Log 的なものが Diagnostics としてあったらいいのかも。
あとは承認関連は Activity Log に出てくるのかな、まだ試してない。

## SQL Database deploy

SQL Database (と SQL Server) を deploy して、SSMS (SQL Server Management Studio) でアクセスしてみる。
DNS 名ではもちろん普通にアクセスはできます。

で、一応 Private IP アドレスでアクセスしてみるとどうか、という話なんですが、まぁつながりませんね。

![SQL Database - private IP](/assets/private-link-sql-database-private-ip.png)

VNet Peering した状態で、hosts を書けば SQL Database に Private IP Address で接続できます。
AD DS 立てて、S2S VPN で接続した形、より On-Prem <-> Azure っぽい検証はまた後日やります。