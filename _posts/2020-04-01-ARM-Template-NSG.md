---
layout: post
title: "ARM Template for Network Security Group"
---

## See Also
DevOps までやりきるのであればこちらに同じように ARM Template から移行する手順が書いてあります。
[First step to Azure DevOps]({% post_url 2019-12-04-First-step-to-Azure-DevOps %})

## Prerequisites

前提知識として、NSG と ARM Template を書いておきます。

### Network Security Group

送信元/宛先の IP/port + protocol の 5 tuple で通信を許可/拒否する L4 ACL の機能を提供する。
NIC にも Subnet (仮想ネットワークを区切ったもの) にも NSG を適用できるので、AWS とはちょっと雰囲気が違う。

### ARM Template
Azure の各リソースはそれぞれ JSON で書かれた template (= ARM Template) で表現できる。
この ARM Template と parameter file を組み合わせることで、IaC (Infrastructure as Code) を実現できる

## 今回やること
DevOps で管理するのが多いケースかとは思いますが、その前段として Azure Portal の Template (まだ preview ですが) 機能を使ってみる。
今回は、NSG の送信元/送信先の IP アドレスを変数で書いてみる。

## 概要
1. NSG を作る
1. なにかルールを追加する
1. NSG の ARM Template を表示させ、Template として保存する
1. 内容を少し編集する
1. deploy してみる (この時点では何も変わらない)
1. variables を使った template の編集、deploy する
1. その他のパターン

## 詳細な手順

1. NSG を作る

    省略します

1. なにかルールを追加する

    今回は検証なので、自分の Wi-Fi ルータの IP アドレスを確認くん (https://www.ugtop.com/spill.shtml) とかで調べて 3389/tcp とかを開けてみる。
    ここでは、198.51.100.103 から 3389/tcp を開けたことにします。

1. NSG の ARM Template を表示させ、Template として保存する。

    NSG の「テンプレートのエクスポート」をクリックすると、だいたいこんな感じかと思います。
    NSG の名前や rule の名前が違うかと思いますが、気にせず進めます。

    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "networkSecurityGroups_vm_poc01_nsg_name": {
                "defaultValue": "vm-poc01-nsg",
                "type": "String"
            }
        },
        "variables": {},
        "resources": [
            {
                "type": "Microsoft.Network/networkSecurityGroups",
                "apiVersion": "2019-11-01",
                "name": "[parameters('networkSecurityGroups_vm_poc01_nsg_name')]",
                "location": "japaneast",
                "properties": {
                    "securityRules": [
                        {
                            "name": "allow-rdp-test",
                            "properties": {
                                "protocol": "TCP",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "198.51.100.103",
                                "destinationAddressPrefix": "*",
                                "access": "Allow",
                                "priority": 100,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                            }
                        }
                    ]
                }
            },
            {
                "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                "apiVersion": "2019-11-01",
                "name": "[concat(parameters('networkSecurityGroups_vm_poc01_nsg_name'), '/allow-rdp-test')]",
                "dependsOn": [
                    "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('networkSecurityGroups_vm_poc01_nsg_name'))]"
                ],
                "properties": {
                    "protocol": "TCP",
                    "sourcePortRange": "*",
                    "destinationPortRange": "3389",
                    "sourceAddressPrefix": "198.51.100.103",
                    "destinationAddressPrefix": "*",
                    "access": "Allow",
                    "priority": 100,
                    "direction": "Inbound",
                    "sourcePortRanges": [],
                    "destinationPortRanges": [],
                    "sourceAddressPrefixes": [],
                    "destinationAddressPrefixes": []
                }
            }
        ]
    }
    ```

    このテンプレートを「ライブラリに追加 (プレビュー)」をクリックして、Template として保存します。
    名前と説明は適宜入力します。

1. 内容を少し編集する

    Azure Portal のテンプレートを開き、該当のテンプレートをクリックします。

    一部内容が冗長なので「編集」>「ARM テンプレート」をクリックして削除します。

    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "networkSecurityGroups_vm_poc01_nsg_name": {
                "defaultValue": "vm-poc01-nsg",
                "type": "String"
            }
        },
        "variables": {},
        "resources": [
            {
                "type": "Microsoft.Network/networkSecurityGroups",
                "apiVersion": "2019-11-01",
                "name": "[parameters('networkSecurityGroups_vm_poc01_nsg_name')]",
                "location": "japaneast",
                "properties": {
                    "securityRules": [
                    ]
                }
            },
            {
                "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                "apiVersion": "2019-11-01",
                "name": "[concat(parameters('networkSecurityGroups_vm_poc01_nsg_name'), '/allow-rdp-test')]",
                "dependsOn": [
                    "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('networkSecurityGroups_vm_poc01_nsg_name'))]"
                ],
                "properties": {
                    "protocol": "TCP",
                    "sourcePortRange": "*",
                    "destinationPortRange": "3389",
                    "sourceAddressPrefix": "198.51.100.103",
                    "destinationAddressPrefix": "*",
                    "access": "Allow",
                    "priority": 100,
                    "direction": "Inbound",
                    "sourcePortRanges": [],
                    "destinationPortRanges": [],
                    "sourceAddressPrefixes": [],
                    "destinationAddressPrefixes": []
                }
            }
        ]
    }
    ```

    最初の SecurityRules を空 (`[]`) にします。
    その上で、「OK」>「保存」で保存します。

1. deploy してみる (この時点では何も変わらない)

    テンプレートの画面から「展開」をクリックし、テンプレートを deploy します。
    パラメータ等は変更不要なので、もとの NSG があったものと同じ Resource Group を選びます。
    「上記の使用条件に同意する」をチェックし、「購入」をクリックします。

    特に問題がなく deploy が完了したことを確認します。
    Azure Portal 右上の通知で「展開が成功しました」と出れば問題ありません。
    また、該当の Resource Group を選択し、「デプロイ」を選択、一番上にあるであろう最新のデプロイが 成功 であれば問題ありません。

1. variables を使った template の編集、deploy する

    ここからテンプレートを少しずつ修正してみます。

    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "networkSecurityGroups_vm_poc01_nsg_name": {
                "defaultValue": "vm-poc01-nsg",
                "type": "String"
            }
        },
        "variables": {
            "my-address": "198.51.100.103"
        },
        "resources": [
            {
                "type": "Microsoft.Network/networkSecurityGroups",
                "apiVersion": "2019-11-01",
                "name": "[parameters('networkSecurityGroups_vm_poc01_nsg_name')]",
                "location": "japaneast",
                "properties": {
                    "securityRules": [
                    ]
                }
            },
            {
                "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                "apiVersion": "2019-11-01",
                "name": "[concat(parameters('networkSecurityGroups_vm_poc01_nsg_name'), '/allow-rdp-test')]",
                "dependsOn": [
                    "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('networkSecurityGroups_vm_poc01_nsg_name'))]"
                ],
                "properties": {
                    "protocol": "TCP",
                    "sourcePortRange": "*",
                    "destinationPortRange": "3389",
                    "sourceAddressPrefix": "[variables('my-address')]",
                    "destinationAddressPrefix": "*",
                    "access": "Allow",
                    "priority": 100,
                    "direction": "Inbound",
                    "sourcePortRanges": [],
                    "destinationPortRanges": [],
                    "sourceAddressPrefixes": [],
                    "destinationAddressPrefixes": []
                }
            }
        ]
    }
    ```

    変更点としては、1) variables として、送信元 IP アドレスを `my-address` という変数に入れ、2) `sourceAddressPrefix` を `my-address` を使う形に変更しています。

    この編集を「OK」>「保存」して再度「展開」し、正しくデプロイされ、また NSG の rule には何も変更がないことを確認します。

    さらに、この Template を変更することで NSG の rule を実際に変更していきます。

    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "networkSecurityGroups_vm_poc01_nsg_name": {
                "defaultValue": "vm-poc01-nsg",
                "type": "String"
            }
        },
        "variables": {
            "my-address": "198.51.100.0/24"
        },
        "resources": [
            {
                "type": "Microsoft.Network/networkSecurityGroups",
                "apiVersion": "2019-11-01",
                "name": "[parameters('networkSecurityGroups_vm_poc01_nsg_name')]",
                "location": "japaneast",
                "properties": {
                    "securityRules": [
                    ]
                }
            },
            {
                "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                "apiVersion": "2019-11-01",
                "name": "[concat(parameters('networkSecurityGroups_vm_poc01_nsg_name'), '/allow-rdp-test')]",
                "dependsOn": [
                    "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('networkSecurityGroups_vm_poc01_nsg_name'))]"
                ],
                "properties": {
                    "protocol": "TCP",
                    "sourcePortRange": "*",
                    "destinationPortRange": "3389",
                    "sourceAddressPrefix": "[variables('my-address')]",
                    "destinationAddressPrefix": "*",
                    "access": "Allow",
                    "priority": 100,
                    "direction": "Inbound",
                    "sourcePortRanges": [],
                    "destinationPortRanges": [],
                    "sourceAddressPrefixes": [],
                    "destinationAddressPrefixes": []
                }
            }
        ]
    }
    ```

    ここでは、送信元 IP アドレスを 198.51.100.103 から 198.51.100.0/24 に変更しました。
    この編集した template を deploy し、該当する rule の送信元 IP アドレスが変わったことを確認します。

1. その他のパターン

    その他、例えば 22/tcp (SSH) を追加することを考えてみます。
    パターンを調べるために、まずは NSG を Azure Portal 上で直接編集し、Template を表示させます。
    NSG では 拡張セキュリティ規則 が使えるので、1 行の rule で「22,3389」と書くことができます。
    この状態で Template を表示させた断片が以下のとおりです。

    ```json
    "properties": {
        "protocol": "TCP",
        "sourcePortRange": "*",
        "sourceAddressPrefix": "198.51.100.0/24",
        "destinationAddressPrefix": "*",
        "access": "Allow",
        "priority": 100,
        "direction": "Inbound",
        "sourcePortRanges": [],
        "destinationPortRanges": [
            "22",
            "3389"
        ],
        "sourceAddressPrefixes": [],
        "destinationAddressPrefixes": []
    }
    ```

    ではこれを variables を使った形に変更します。
    variables はこのように記載します。

    ```json
    "variables": {
        "my-address": "167.220.0.0/16",
        "mgmt-port": [
            "22",
            "3389"
        ]
    },
    ```

    これを利用するため、`destinationPortRange` ではなく、`destinationPortRanges` の記載に変えます。

    ```json
    {
        "properties": {
            "protocol": "TCP",
            "sourcePortRange": "*",
            "sourceAddressPrefix": "[variables('my-address')]",
            "destinationAddressPrefix": "*",
            "access": "Allow",
            "priority": 100,
            "direction": "Inbound",
            "sourcePortRanges": [],
            "destinationPortRanges": "[variables('mgmt-port')]",
            "sourceAddressPrefixes": [],
            "destinationAddressPrefixes": []
        }
    }
    ```

    念のため全体の template はこちらです。
    
    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "networkSecurityGroups_vm_poc01_nsg_name": {
                "defaultValue": "vm-poc01-nsg",
                "type": "String"
            }
        },
        "variables": {
            "my-address": "198.51.100.0/24",
            "mgmt-port": [
                "22",
                "3389"
            ]
        },
        "resources": [
            {
                "type": "Microsoft.Network/networkSecurityGroups",
                "apiVersion": "2019-11-01",
                "name": "[parameters('networkSecurityGroups_vm_poc01_nsg_name')]",
                "location": "japaneast",
                "properties": {
                    "securityRules": [
                    ]
                }
            },
            {
                "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                "apiVersion": "2019-11-01",
                "name": "[concat(parameters('networkSecurityGroups_vm_poc01_nsg_name'), '/allow-rdp-test')]",
                "dependsOn": [
                    "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('networkSecurityGroups_vm_poc01_nsg_name'))]"
                ],
                "properties": {
                    "protocol": "TCP",
                    "sourcePortRange": "*",
                    "sourceAddressPrefix": "[variables('my-address')]",
                    "destinationAddressPrefix": "*",
                    "access": "Allow",
                    "priority": 100,
                    "direction": "Inbound",
                    "sourcePortRanges": [],
                    "destinationPortRanges": "[variables('mgmt-port')]",
                    "sourceAddressPrefixes": [],
                    "destinationAddressPrefixes": []
                }
            }
        ]
    }
    ```

    こちらを同様に「OK」>「保存」>「展開」を実施することで、NSG の rule が変わったことを確認します。