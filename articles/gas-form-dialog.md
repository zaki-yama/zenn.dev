---
title: "GASでフォームを含むダイアログを実装する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas"]
published: false
---

# 基本のダイアログを作る

まず、基本となるダイアログを表示するところまで実装します。
GAS においてダイアログを実装する方法はいくつかありますが、今回のようにダイアログ内にフォーム要素を設置したい場合、カスタムダイアログを使います。
参考: https://developers.google.com/apps-script/guides/dialogs#custom_dialogs

メインスクリプトの `Code.gs` に加え、独立した HTML ファイル（ここでは `Dialog.html`）を用意します。

```javascript:Code.gs
function onOpen() {
  SpreadsheetApp.getUi() // Or DocumentApp or SlidesApp or FormApp.
    .createMenu("Custom Menu")
    .addItem("Show dialog", "showDialog")
    .addToUi();
}

function showDialog() {
  var html = HtmlService.createHtmlOutputFromFile("Dialog")
    .setWidth(400)
    .setHeight(300);
  SpreadsheetApp.getUi() // Or DocumentApp or SlidesApp or FormApp.
    .showModalDialog(html, "My custom dialog");
}
```

```html:Dialog.html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top" />
  </head>
  <body>
    <h1>Hello, World!</h1>
  </body>
</html>
```

メニューを作成している箇所を除けば、ダイアログ表示のためにやっていることは以下の 2 つです。

1. [`HtmlService.createHtmlOutputFromFile(filename)`](https://developers.google.com/apps-script/reference/html/html-service#createhtmloutputfromfilefilename) というメソッドに HTML ファイル名を渡す
   - メソッドの戻り値は [`HtmlOutput`](https://developers.google.com/apps-script/reference/html/html-output.html) クラスのインスタンス
2. 1 の戻り値を [`Ui.showModalDialog()`](<https://developers.google.com/apps-script/reference/base/ui#showModalDialog(Object,String)>) というメソッドに渡す

このコードの結果、以下のようにメニューからダイアログが表示できるようになります。

![](https://storage.googleapis.com/zenn-user-upload/50e6564c0686-20220922.gif)

# フォームを作り、 送信された値を受け取る

# 処理結果をダイアログに表示する

# 保存済みの値を初期値としてセットする

# CSS フレームワークを利用する
