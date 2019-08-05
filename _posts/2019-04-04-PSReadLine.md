---
layout: post
title:  "PSReadLine"
---

# PSReadLine

Powershell で ^P^P とかでて腹立たしい時には PSReadLine を入れるといいらしい。
自分の環境だとすでに Install-Module はできてたので、Import-Module をするための profile.ps1 を書くのと、Set-PSReadLineOption -EditMode Emacs を追加するのが必要だった。

![PSReadLine](/assets/psreadline.png)
![Post Installation](/assets/psreadline-post-installation.png)
![Usage](/assets/psreadline-usage.png)

もともとの PowerShell の Profile は "%USERPROFILE%\Documents\WindowsPowerShell\profile.ps1" で、PowerShell 6 は "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" なのね。