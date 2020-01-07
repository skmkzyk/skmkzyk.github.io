---
layout: post
title:  "New-SelfSignedCertificate"
---

自己署名証明書の作り方毎回忘れるから書いておく。

```powershell
$cert = New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname hoge.jp
$pwd = ConvertTo-SecureString -String 'passw0rd!' -Force -AsPlainText
$path = 'cert:\localMachine\my\' + $cert.thumbprint
Export-PfxCertificate -cert $path -FilePath c:\temp\cert.pfx -Password $pwd
```

[こちら](https://aboutssl.org/how-to-create-a-self-signed-certificate-in-iis/) を見ると IIS Manager で Self-Signed Certificate は作れるので上記はいらないかもしれない。
