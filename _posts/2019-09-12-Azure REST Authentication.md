---
layout: post
title:  "Azure REST Authentication"
---

# Azure REST API

Azure の各種サービスを操作するにあたって Azure Portal がもちろん一番便利ですね。
また、自動化とかいう観点だと、PowerShell や CLI を使って Azure を操作することも可能です。
ただ、これらの手段を使っても Azure のすべての操作を実施することはできず、結局は REST API がすべての操作をサポートしています。
というか、まずは REST API の口ができてから、他の操作方法がついてくる、という感じですね。

## REST API for Storage Account

で、今回はその中でも Storage Account に関する REST API を試していて、ちょっと難しい点があったんのでメモしておきます。
API を叩くのには信頼と実績の Postman (ref. https://www.getpostman.com/) を使っています。

## ハマりポイント

認証をどうやってやるかなんですよね。
普通にブラウザ等々でアクセスする際には SAS (Shared Access Signature) を後ろにくっつけるだけ問題ありません。

    https://myaccount.blob.core.windows.net/container/blob?sas-key

ただ、REST API アクセスの場合はそうではないようです。

## `Authorization` header の組み立て方

例えば Pub Blob の REST API を見てみます。(ref. https://docs.microsoft.com/en-us/rest/api/storageservices/put-blob)
URL をどう構成するか、HTTP header をどう構成するかを参考にするのですが、その中でもはまったのが `Authorization` header でした。

> Required. Specifies the authentication scheme, account name, and signature. For more information, see [Authentication for the Azure Storage Services](https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key).

see と書かれたページに行ってみるのですが、まー API 系なので読み込まなきゃいけないのがひたすらつらい。
まずは、いくつかの header を組み合わせた文字列を作ります。

```
StringToSign = VERB + "\n" +  
               Content-Encoding + "\n" +  
               Content-Language + "\n" +  
               Content-Length + "\n" +  
               Content-MD5 + "\n" +  
               Content-Type + "\n" +  
               Date + "\n" +  
               If-Modified-Since + "\n" +  
               If-Match + "\n" +  
               If-None-Match + "\n" +  
               If-Unmodified-Since + "\n" +  
               Range + "\n" +  
               CanonicalizedHeaders +   
               CanonicalizedResource;
```

`CanonicalizedResource` も何ぞやという感じですが、例を見ると、

```
Get Container Metadata  
   GET http://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata   
CanonicalizedResource:  
    /myaccount/mycontainer\ncomp:metadata\nrestype:container  
  
List Blobs operation:  
    GET http://myaccount.blob.core.windows.net/container?restype=container&comp=list&include=snapshots&include=metadata&include=uncommittedblobs  
CanonicalizedResource:  
    /myaccount/mycontainer\ncomp:list\ninclude:metadata,snapshots,uncommittedblobs\nrestype:container  
  
Get Blob operation against a resource in the secondary location:  
   GET https://myaccount-secondary.blob.core.windows.net/mycontainer/myblob  
CanonicalizedResource:  
    /myaccount/mycontainer/myblob
```

という感じなので比較的簡単ですね。
blob.core.windows.net という文字列を除く感じですね、ざっくり。

で、んじゃこれらを組み合わせて `Authorization` header を作るんですが、

> Next, encode this string by using the HMAC-SHA256 algorithm over the UTF-8-encoded signature string, construct the `Authorization` header, and add the header to the request. The following example shows the `Authorization` header for the same operation:

とざっくり書いてある。
最後に書いてわかりづらいんだけど pseudocode はここに書いてある。(ref. https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key#encoding-the-signature)

言いたいことは分かるものの、これを Postman でどうやるのかがわからなくて悩んでいたんですが、結果から言うと Pre-request Script というのを使います。
結果を先に示しておきますが、こんな感じの script を Postman の Pre-request Script に書きます。
CryptoJS というライブラリが使えるんですね、知らなかったです。

```
postman.setEnvironmentVariable("x-ms-version", "2018-11-09");

var current_timestamp = new Date();
postman.setEnvironmentVariable("x-ms-date", current_timestamp.toUTCString());

var message = "PUT\n" /*HTTP Verb*/
+ "\n"    /*Content-Encoding*/
+ "\n"    /*Content-Language*/
+ "\n"    /*Content-Length (empty string when zero)*/
+ "\n"    /*Content-MD5*/
+ "\n"    /*Content-Type*/
+ "\n"    /*Date*/
+ "\n"    /*If-Modified-Since */
+ "\n"    /*If-Match*/
+ "\n"    /*If-None-Match*/
+ "\n"    /*If-Unmodified-Since*/
+ "\n"    /*Range*/
+ "x-ms-blob-content-length:32214352384\n"
+ "x-ms-blob-type:PageBlob\n"
+ "x-ms-date:" + current_timestamp.toUTCString() + "\n"
+ "x-ms-version:2018-11-09\n"    /*CanonicalizedHeaders*/
+ "/xxxxxxxxxxxxxxxxxxxxxxxx/vhds/xxxxxxxxxxxxxxxxxxxxx.vhd";    /*CanonicalizedResource*/

var access_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";

var hash = CryptoJS.HmacSHA256(message, CryptoJS.enc.Base64.parse(access_key));
var hashInBase64 = CryptoJS.enc.Base64.stringify(hash);
postman.setEnvironmentVariable("Authorization", "SharedKey xxxxxxxxxxxxxxxxxxxxxxxx:" + hashInBase64);
```

いつかこの Blog を見るときには、`x-ms-version` は変わっているかもしれません。
その場合には、これらの記載もガラッと変わっている可能性があります。

`x-ms-blob-content-length` については、よしなにサイズを変えてください。
`CanonicalizedResource` も環境に合わせて変えてください。
`var access_key` のところについては、access key を書いておきます。(key1、key2 たぶんどちらでも大丈夫です)
`postman.setEnvironmentVariable` の `SharedKey xxxxxxxxxxxxxxxxxxxxxxxx` については、Storage Account の名前を書くだけです。

で、これらを有効に使うために、Postman の Headers タブの各値については、`{{x-ms-version}}` とか、`{{Authorization}}` とかにしておけば Pre-request Script によってそれらの値が書き換わります。

![postman-headers](/assets/postman-headers.png)

## ちなみに

結果を保存していないのですが、ある程度 `Authorization` header が惜しいところまで来ていると、REST API の response として、こういう内容で認証しようとしているよ、というヘッダの内容を教えてくれます。(上記 `var message` として組み立てていた内容)
実際は、ハッシュかけているので `CanonicalizedHeaders` の中身も順序があるわけで、そこらへんが Azure 側と一致しない場合にはこういう順序で書いてね、という意味で送ってきてるのかもしれません。
上記 Script 例はそれに合わせた形、実績のある形なのでほぼこれをパクればいい感じに認証できるはずです。

REST API をちゃんと読み込むのも初めてでしたが、慣れがいるなと感じました。