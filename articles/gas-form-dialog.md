---
title: "GASでフォームを含むダイアログを実装する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas"]
published: false
---

# 基本のダイアログを作る

まず、土台となるダイアログを作り、表示できるところまで実装します。
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
t
```

# フォームを作り、 送信された値を受け取る

# 処理結果をダイアログに表示する

# 保存済みの値を初期値としてセットする

# CSS フレームワークを利用する
