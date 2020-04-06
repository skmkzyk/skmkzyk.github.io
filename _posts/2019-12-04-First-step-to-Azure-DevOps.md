---
layout: post
title: "First step to Azure DevOps"
---

## First step to Azure DevOps

こちらは [Microsoft Azure Tech Advent Calendar 2019](https://qiita.com/advent-calendar/2019/microsoft-azure-tech) の 4 日目です。
Qiita ではなく GitHub Pages で書いているので、上のリンクを踏んでいただいて、ぜひほかのメンバのも見ていただければと！！

内容としては、先日 Azure DevOps の bootcamp に参加してきたので、それを踏まえて、すでに Azure Portal で deploy した環境を DevOps にどう移すか、みたいなところを考えていこうかなと思っています。

## Deploy options

Azure の resouce の deploy の仕方は様々ありますが、一番簡単なのは Azure Portal かと思っています。
そのための UI でもありますし、横に伸びる画面は慣れるまでは操作しづらいですが、触っているうちに慣れましたね。
その他には、Azure PowerShell や Azure CLI、ARM Template、REST API といった方法があります。

### Azure Portal

Azure Portal が先日ガラッと見た目が変わりましたね。アイコンもほとんど SVG に変わりまして、きれいにはなったんですが、手順書を作っている運用だとつらいですね。
左のハンバーガーメニューは常時表示に切り替えられますが、そもそもメンバによってメニューの順番が異なったりするのでそこまで揃えますか？という気はしますね。

### Azure PowerShell / Azure CLI

こういった Azure Portal の変更に対して、Azure PowerShell や Azure CLI を利用した手順書を作成することは有効な対応策だと考えられます。
ただ、これはこれで難しい点があって、手続き型と宣言型の違いというか、コマンドを実行するたびに状態が変わっていって、実際にどのようになったのかは Get-AzXXX などの cmdlet を実行する形になります。
手順書を作成することを考えると、まずは Get-AzXXX cmdlet を実行し、その結果が意図した **実行前** の状態であることを確認し、Set-AzXXX cmdlet を実行、再度 Get-AzXXX を実行して意図した **実行後** の状態であることを確認する、となるでしょう。

### ARM Template

これに対して、ARM Template はあるべき状態を記述し、それに添わせるように deploy をしていく形になります。
この ARM Template は (乱暴かもしれませんが) 一種のパラメータシートとしても扱うことができるため、ドキュメント変更漏れなどの心配もない、といえると思います。
今回は、この ARM Template を Azure DevOps 上で git で管理し、Build Pipeline と Release Pipeline を組んで実際に resource を変更していくところまでの流れを書いてみます。

### REST API

最終的には REST API にはなるんですが、JSON を作って REST API に投げれば、究極的には Azure に関して一番なんでもできる形で request を投げられます。
というのも、REST API での実装が一番早く、その後 PowerShell/CLI/Portal で操作できるようになる、という機能がよく見られます。
また、現在でも REST API に比べて Azure Portal は多くの操作ができない状態にはなっています。
最近は Azure CLI でも `az rest` というのが実装され、Postman を使わずとも REST API をそのままの形で叩くことができるようになっています。
また、[Resource Explorer](https://resources.azure.com/) はブラウザから利用する REST API フロントとして非常に便利です。

## Prerequisites

慣れないと読めないですね、この単語。
先日の bootcamp で何度も音として聞いたので spell もちゃんと書けるようになりました。
ぷりりくいじっつ、という感じです。

- git

    git の知識があるとよいです。
    ざっくり説明すると、なんかファイルを書く、保存する、commit する (保存したことをログに書く)、push する (ログ含めてファイルをリモートに保存する)、そんな流れがわかっているとよいです。
    あと、pull-request の知識もあると楽しいです。

- JSON

    ARM Template が JSON なので、JSON が読めたほうがよいです。
    細かい文法は覚えてなくてもよいですが、同じ要素を繰り返し書くなら `,` を間に挟むとかそういうやつです。
    `[` `]` は配列で、`{` `}` はオブジェクトみたいなもんです。
    文字列は `"` `"` でくくりましょう。
    [Visual Studio Code](https://code.visualstudio.com/) 使っておけば formatter も見つかるので便利です。

- Azure の基礎的な知識

    ここを見に来るのであればおそらくある程度お持ちかとは思っていますが最後に補足します。

## Migrate to Azure DevOps

とてもざっくり説明すると、1) Azure Portal から一旦リソースを作成し、2) ARM Template を吐かせて、3) Azure DevOps の Build/Release pipeline を組んで、4) ARM teplmate の内容を git push すると resource が変更される、というのをやってみます。

1. deploy by Azure Portal

    とりあえず Azure Portal でなんか deploy しましょう。
    どの順で作成してもよいですが、以下の resource を作ったとします。

    - Resource Group
    - Virtual Network
    - Network Security Group
    - Virtual Machine
    - Network Interface
    - Disk
    - Public IP Address

1. export ARM Template

    ARM Template を export します。
    ここでは、まずは VNet を選択し、"export template" をクリックします。
    しばらくすると ARM Template が表示されるので、上の "Download" をクリックします。
    ExportedTemplate-xxx.zip みたいなのが落ちてくるので、一旦開いておきます。

1. prepare Azure DevOps

    いきなり飛びますが、[Azure DevOps](https://dev.azure.com/) に行きます。
    Account がないようであれば作成し、最初の Project を作成します。

1. initialize your repos

    Repository を初期化するため、"Repos" をクリックします。
    簡単にするため、最後の "or initialize with a README or gitignore" で始めます。

1. clone your repos to your PC

    この辺から git の操作になります。
    git を入れていないようであれば [Git - Downloads](https://git-scm.com/downloads) から download、install しておいてください。
    私のおすすめは [Chocolatey Software | Git 2.24.0.2](https://chocolatey.org/packages/git) です。

    初期化した repos を clone しますが、URL は右上の "clone" ボタンを押すと出てきます。
    `https://xxxxxxxx@dev.azure.com/xxxxxxxx/yyyyyyyy/_git/yyyyyyyy` なんかこんな感じのです。
    `git clone https://--` とすると、初期化した repos が手元に clone されます、download するようなもんです。

1. add ARM template

    clone してきた repos に "export ARM Template" で download した ARM Template を追加します。
    フォルダ構造はなんでもいいんですが、なんとなく一旦 Template フォルダを作ってから、その中に template.json と parameters.json を置きます。
    また、なんとなくそれぞれのファイルを migrate-to-devops-vnet.template.json と migrate-to-devops-vnet.parameters.json にします。

1. modify template

    このまんまだとうまくいかなかったので、少しだけ Template を修正します。

    migrate-to-devops-vnet.parameters.json を開いて、

    ```json
    "virtualNetworks_xxxxxxxx_name": {
        "value": null
    }
    ```

    を

    ```json
    "virtualNetworks_xxxxxxxx_name": {
        "value": "migrate-to-devops-vnet"
    }
    ```

    にして、保存します。

1. push to remote

    `git add .` として追加したファイルを次の commit に含めるようにします。

    `git commit -m 'add template'` と叩いて commit します。(コメントは適宜変えていただければと)

    `git push origin` と叩いて commit したものを remote に push します。(クラウドに保存するようなイメージ)

1. create a build pipeline

    こっから、2 種類の pipeline を作ります。
    一つ目は Build pipieline です。

    左のメニューから "Pipelines" をクリックして、"Build" を開いた状態にします。
    (もしなかったら右上の人のアイコンから、"Preview Features" で、"Multi-stage pipeline" を off にする)

    "New Pipeline" をクリックします。

    次は "Use the classic editor" をクリックします。

    "select a source" はそのまま、"Continue" をクリックします。

    上にある、小さな "Empty job" をクリックします。

    "Agent job 1" の右にある "+" をクリックし、"Copy files" を探して "Add" します。
    また、同じように "Publish build artifact" も "Add" します。

    "Copy File" を選び、Contents を `**\*.json` に少し書き換えます。また、Target Foler は `$(Build.ArtifactStagingDirectory)` とします。

    "Publish Artifact" についてはそのままで問題ないはずです。

    保存するため上のフロッピーマーク "Save & Queue" をクリックします。

1. wait for build completion

    画面が自動的に切り替わり、Build の進捗画面になります。
    すべてが緑のチェックマークになれば問題ないです。
    どこかでコケてしまうようであればメッセージに沿って修正します。

1. create a release pipeline

    次は Release pipeline を作っていきます。
    Build の完了画面の右上に "Release" というのが目立っているのでこれをクリックします。

    画面がばっと切り替わりますが、ここでも右上の "empty job" を選んで進めていきます。

    "1 job, 0 task" というのをクリックし、"Azure resouce group deployment" という task を "Add" します。

    ここで Subscription を連携していきます。
    Azure Subscription の pulldown から該当の Subscription を選択し、"Authorize" します。

    Resource Group や Location は Azure Portal から deploy したときのものを入れておきます。

    Template や Template Parameter は右の `...` をクリックし、Build が正しく通っていれば json が選択できるようになっているはずです。
    ここで選べないようであれば、"Copy files" や "Publish build artifact" の設定内容が間違っている可能性が高いです。

    Release pipeline を右上の "save" で保存します。

1. wait for release completion

    問題がなく、Stage が緑色になれば Release Pipeline が完成しました。

1. change template.json

    ここでは、subnet を追加してみます。

    migrate-to-devops-vnet.template.json を開き、以下の内容を修正します。

    ```json
    "subnets": [
        {
            "name": "default",
            "properties": {
                "addressPrefix": "10.8.0.0/24",
                "delegations": [],
                "privateEndpointNetworkPolicies": "Enabled",
                "privateLinkServiceNetworkPolicies": "Enabled"
            }
        }
    ],
    ```

    を

    ```json
    "subnets": [
        {
            "name": "default",
            "properties": {
                "addressPrefix": "10.8.0.0/24",
                "delegations": [],
                "privateEndpointNetworkPolicies": "Enabled",
                "privateLinkServiceNetworkPolicies": "Enabled"
            }
        },
        {
            "name": "app",
            "properties": {
                "addressPrefix": "10.8.1.0/24",
                "delegations": [],
                "privateEndpointNetworkPolicies": "Enabled",
                "privateLinkServiceNetworkPolicies": "Enabled"
            }
        }
    ],
    ```

    といった感じにします。
    ポイントとしては、同じ `{` `}` のブロックをコピーして "name" や "addressPrefix" を修正するのと同時に、前側のブロックの最後に `,` を追加する部分です。

1. push to remote

    migrate-to-devops-vnet.template.json の修正を保存します。

    `git add .` します。

    `git commit -m 'add subnet'` します。

    `git push origin` します。

1. run a build pipeline

    今回、Continuous integration を有効にしていないため、手動で build pipeline を動かします。
    左のメニューから "Pipelines" の "Build" を選択し、右上の "Queue" をクリックすることで新しく Build が Queue されます。
    Build と Release は連携しているので、Build が正しく完了すれば Release も実行され、結果として VNet の subnet が追加されたことを確認できます。

## Conclusion

とりあえず VNet だけですが、Azure Portal で作ったリソースを Azure DevOps から操作できるよう、Repository の設定、Pipeline の設定までをひととおり書いてみました。
Azure DevOps を最初っから使い始めるわけではなく、今ある環境を DevOps に載せる、そういった観点の説明はなかなかないような気がするので、DevOps に取り組みたいけど今の環境どうするの、というときには参考になればと思います。

ほかにも、Multi-stage pipeline や、branch を使った pull-request、pre-deployment approval を使った内容も書きたかったのですが、時間切れでした。
また別の記事として書いてみたいと思います。

## Appendix

- Resource Group

    Virtual Network とか Virtual Machine、Disk などの各 component は resource と呼ばれています。
    その resource を束ねる概念が Resource Group で、これ自体には計算したりデータを保存する機能はありません。
    ただ、これ単位で Owner/Contributor/Reader といった権限を付与したり、これ単位で削除したりできるので、便利に使えます。

- Virtual Network

    仮想ネットワーク、VNet と省略して書くことがほとんどです。
    Private IP アドレス空間を定義して利用します。
    下位の概念として subnet があり、VNet のアドレス空間に subnet を複数定義する、という感じです。
    一般的にエンジニアが会話する意味の subnet (ほぼアドレス空間と同義) よりもかなり限定的な意味を持っています。

- Network Security Group

    いわゆる L4 ACL を Network Security Group、略して NSG と呼びます。
    送信元/先の IP/port および UDP/TCP/ICMP などの protocol を合わせた 5 tuple で allow/deny を定義できます。

- Virtual Machine

    普通に VM です。
    IaaS といって VM のことを指すことも多いです。
    AWS EC2 などと異なり、固有の名称を持ちません。。

- Network Interface

    もちろん仮想的なものですが、NIC が resource として存在します。
    NIC を複数作成すれば、VM に 2 NIC 以上生やすことも可能です。

- Disk

    もちろん仮想的なものですが、Disk も resource として存在します。
    複数作成すれば E:、F: というように複数の disk を持つ VM が作成できます。

- Public IP Address

    Azure から払い出される Public IP アドレスも resource として存在します。
    NIC と紐づけることで利用でき、Public IP アドレス経由で RDP/SSH させるような際に利用したりします。

## Changelog

- もとの Advent Calendar へのリンクがないとの天啓を受けたので修正 (2019-12-05)
