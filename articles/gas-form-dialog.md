---
title: "GASでシンプルなフォームつきダイアログを実装する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas"]
published: false
---

# はじめに

Google Apps Script（以下、GAS）を使ったアプリケーションを開発していると、なんらかの設定値を画面から入力させるしくみが必要になることがあります。
例えば Slack と連携する機能において、トークンや Webhook URL をコードには直接埋め込まずユーザーに入力させたい、などです。

やりたいことは非常にシンプルですが、調べてみるとなかなかまとまった情報が見つからなかったため、本記事ではこのような「ユーザーからの入力を受け取って保存する」機能の実装方法について簡単にまとめます。

# この記事で作るもの

本記事で実装するのは以下のような機能です。

![](https://storage.googleapis.com/zenn-user-upload/0075fe930ae4-20220925.gif)

- メニューからダイアログを表示する
- ダイアログ内のフォームに入力して保存を実行すると、入力値を保存する
- 保存成功時にメッセージを表示する
- 次回以降ダイアログを開いたときには、保存済みの値が初期値としてセットされる

前提として、スプレッドシートと連携した GAS を想定していますが、コードの一部（`SpreadsheetApp` など）を変えればドキュメント等にも転用できると思います。

# 実装方法

## 1. 基本のダイアログを作る

まず、基本となるダイアログを表示するところまで実装します。
GAS においてダイアログを実装する方法はいくつかありますが、今回のようにダイアログ内にフォーム要素を設置したい場合、カスタムダイアログを使います。
参考: https://developers.google.com/apps-script/guides/dialogs#custom_dialogs

メインスクリプト（ここでは `Code.gs`）に加え、独立した HTML ファイル（ここでは `Dialog.html`）を用意します。

```javascript:Code.gs
function onOpen() {
  SpreadsheetApp.getUi() // Or DocumentApp or SlidesApp or FormApp.
    .createMenu("Custom Menu")
    .addItem("Show dialog", "showDialog")
    .addToUi();
}

function showDialog() {
  const html = HtmlService.createHtmlOutputFromFile("Dialog")
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
   - メソッドの戻り値は [`HtmlOutput`](https://developers.google.com/apps-script/reference/html/html-output.html) クラスのオブジェクト
2. 1 の戻り値を [`Ui.showModalDialog()`](https://developers.google.com/apps-script/reference/base/ui#showmodaldialoguserinterface,-title) というメソッドに渡す

このコードの結果、メニューからダイアログが表示できるようになります。

![](https://storage.googleapis.com/zenn-user-upload/50e6564c0686-20220922.gif)

## 2. フォームを作り、 送信された値を受け取る

続いて、ダイアログにフォーム機能を実装します。まず、 `Code.gs` には以下の関数を追加します。

```javascript:Code.gs(抜粋)
function processForm(formObject) {
  console.log(`Hello, ${formObject.name}!`);
}
```

続いて `Dialog.html` は以下のようにします。

```html:Dialog.html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top" />
    <script>
      // Prevent forms from submitting.
      function preventFormSubmit() {
        const forms = document.querySelectorAll("form");
        forms.forEach((form) => {
          form.addEventListener("submit", function (event) {
            event.preventDefault();
          });
        });
      }
      window.addEventListener("load", preventFormSubmit);

      function handleFormSubmit(formObject) {
        google.script.run.processForm(formObject);
      }
    </script>
  </head>
  <body>
    <form id="myForm" onsubmit="handleFormSubmit(this)">
      <input name="name" type="text" />
      <input type="submit" value="Submit" />
    </form>
  </body>
</html>
```

form を設置し、その submit イベントハンドラ内で `google.script.run.processForm()` を呼び出しています。 [`google.script.run`](https://developers.google.com/apps-script/guides/html/reference/run) は HTML から Apps Script の関数を呼び出すための API です。
スクリプト側は渡されたフォームオブジェクトの中身を `console.log` に出力しているだけですが、ちゃんと送られてきているのを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/559d65dc83c4-20220924.png)

なお、 `preventFormSubmit()` ではフォームのデフォルトの動作をキャンセルしています。
これは公式ドキュメントの https://developers.google.com/apps-script/guides/html/communication#forms には

> Note that upon loading all forms in the page have the default submit action disabled by `preventFormSubmit`. This prevents the page from redirecting to an inaccurate URL in the event of an exception.

と記載されています。 `in the event of an exception` と書かれてはいますが、確認した限りではこの処理を省くと成功時でもブラウザのコンソールで以下のエラーが発生していたため、残しておいた方がよさそうです。

```
Unsafe attempt to initiate navigation for frame with origin 'https://docs.google.com' from frame with URL 'https://n-gxieqqv47obd6y...4i-0lu-script.googleusercontent.com/userCodeAppPanel'. The frame attempting navigation of the top-level window is sandboxed, but the flag of 'allow-top-navigation' or 'allow-top-navigation-by-user-activation' is not set.
```

## 3. フォームから送信されたデータを保存する

先ほどのステップでフォームの値をスクリプト側に送信できたため、保存処理を追加します。
スクリプトの `processForm()` を以下のように変更します。

```javascript:Code.gs(抜粋)
function processForm(formObject) {
  const userProperties = PropertiesService.getUserProperties();
  userProperties.setProperties({
    name: formObject.name,
  });
}
```

`PropertiesService` を使用すると、key-value 形式で任意のデータを保存できます。
ここではユーザー固有の設定値を保存するという想定で User Properties を使用していますが、全ユーザーで同じ値を共有するための Script Properties もあります。
詳しくは公式ドキュメントの https://developers.google.com/apps-script/guides/properties を参照ください。

## 4. 処理結果をダイアログに表示する

ここまででフォームの値を保存できるようになりましたが、保存が成功したことがダイアログ上からはわかりません。
そのため、次に「保存に成功したらダイアログ上にメッセージを表示する」という処理を追加します。

`Dialog.html` を以下のように変更します。

```diff:Dialog.html
 <!DOCTYPE html>
 <html>
   <head>
     <base target="_top" />
     <script>
       // Prevent forms from submitting.
       function preventFormSubmit() {
         const forms = document.querySelectorAll("form");
         forms.forEach((form) => {
           form.addEventListener("submit", function (event) {
             event.preventDefault();
           });
         });
       }
       window.addEventListener("load", preventFormSubmit);

+      function onSuccess() {
+        const div = document.getElementById("success-output");
+        div.innerHTML = "Successfully saved";
+      }
+
       function handleFormSubmit(formObject) {
-        google.script.run.processForm(formObject);
+        google.script.run.withSuccessHandler(onSuccess).processForm(formObject);
       }
     </script>
   </head>
   <body>
     <form id="myForm" onsubmit="handleFormSubmit(this)">
       <input name="name" type="text" />
       <input type="submit" value="Submit" />
     </form>
+    <div id="success-output"></div>
   </body>
 </html>
```

`google.script.run` に [`withSuccessHandler(Function)`](https://developers.google.com/apps-script/guides/html/reference/run#withsuccesshandlerfunction) というメソッドが追加されました。
これは文字通り、スクリプト側の関数（ここでは `processForm`）が成功したときのコールバック関数をセットするメソッドです。同様に、スクリプト側の関数が失敗したときのコールバック関数をセットする [`withFailureHandler(Function)`](https://developers.google.com/apps-script/guides/html/reference/run#withfailurehandlerfunction) もあり、これらはメソッドチェーンでつなげられます。
また、 **スクリプト側の関数に戻り値がある場合、コールバック関数の第一引数には戻り値が渡されます。**

成功したことをダイアログに表示するための方法は色々あると思いますが、ここではあらかじめ用意しておいた div 要素にテキストをセットするという素朴な方法をとっています。

これにより、処理が完了したことがユーザーから見てもわかりやすくなりました。

![](https://storage.googleapis.com/zenn-user-upload/e756393a52e1-20220925.gif)

## 5. 保存済みの値を初期値としてセットする

今のままでは過去に保存した値が存在するのか、またその値が何なのかを知る術がありません。
そこで、すでに保存済みの値があった場合、フォームの初期値としてセットする処理を追加します。

`Code.gs` でダイアログを表示していた部分を以下のように変更します。

```diff:Code.gs（抜粋）
 function showDialog() {
-  const html = HtmlService.createHtmlOutputFromFile("Dialog")
-    .setWidth(400)
-    .setHeight(300);
+  const template = HtmlService.createTemplateFromFile("Dialog");
+  template.name = getName();
+  const html = template.evaluate().setWidth(400).setHeight(300);
   SpreadsheetApp.getUi() // Or DocumentApp or SlidesApp or FormApp.
     .showModalDialog(html, "My custom dialog");
 }

+function getName() {
+  const userProperties = PropertiesService.getUserProperties();
+  return userProperties.getProperty("name");
+}
+
```

また、 `Dialog.html` は以下のようにフォーム要素に `value` を指定します。

```diff:Dialog.html（抜粋）
   <body>
     <form id="myForm" onsubmit="handleFormSubmit(this)">
-      <input name="name" type="text" />
+      <input name="name" type="text" value="<?= name ?>" />
       <input type="submit" value="Submit" />
     </form>
     <div id="success-output"></div>
```

`Code.gs` の変更点ですが、これまで `createHtmlOutputFromFile(filename)` を使用していたのが [`createTemplateFromFile(filename)`](https://developers.google.com/apps-script/reference/html/html-service#createtemplatefromfilefilename) に変わっています。
こうすると、引数で渡した HTML ファイルには `<?= foo ?>` で変数を埋め込んだり、 `<? ... ?>` という構文で if や for 文など任意の処理を実行することができます。（[Pug](https://pugjs.org/api/getting-started.html) や [EJS](https://ejs.co/) などと似たテンプレートエンジン、と捉えていただくとわかりやすいかもしれません）

詳しくは公式ドキュメントの https://developers.google.com/apps-script/guides/html/templates を参照ください。

`createTemplateFromFile(filename)` が返すのは `HtmlTemplate` クラスのオブジェクトとなり、これに任意の変数（ここでは `template.name = ...`）をセットすると HTML 側から参照できます（`<?= name ?>`）。
また、これを `evaluate()` した結果が `HtmlOutput` クラスのオブジェクトとなり、 `createHtmlOutputFromFile(filename)` の戻り値と同じものです。

`getName()` 内では User Properties から値を取り出しています。
上のコードのように [`getProperty(key)`](https://developers.google.com/apps-script/reference/properties/properties#getpropertykey) で特定の値を取り出すか、 [`getProperties()`](https://developers.google.com/apps-script/reference/properties/properties#getproperties) ですべての key-value のペアを取り出すこともできます。

## 6. （おまけ）CSS フレームワークを利用する

最後におまけで、外部の CSS フレームワークを使ってフォームの見た目を整えます。
GAS のダイアログは一般的な HTML と同じように `<link>` タグで外部の CSS を読み込むことができます。そのため、CDN で配信されている CSS フレームワークなどであれば比較的簡単に利用できます。
ここではその一例として GitHub が開発している [Primer CSS](https://primer.style/css/) を使ってみます。

コードは以下です（長いので折りたたんでいます）。

:::details Dialog.html

```html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top" />
    <link
      href="https://unpkg.com/@primer/css@^20.2.4/dist/primer.css"
      rel="stylesheet"
    />
    <script>
      // Prevent forms from submitting.
      function preventFormSubmit() {
        const forms = document.querySelectorAll("form");
        forms.forEach((form) => {
          form.addEventListener("submit", function (event) {
            event.preventDefault();
          });
        });
      }
      window.addEventListener("load", preventFormSubmit);

      function onSuccess() {
        const success = document.getElementById("success");
        success.style.display = "block";
      }

      function handleFormSubmit(formObject) {
        google.script.run.withSuccessHandler(onSuccess).processForm(formObject);
      }
    </script>
  </head>
  <body>
    <div id="success" class="flash flash-success" style="display: none">
      <!-- <%= octicon "check" %> -->
      <svg
        class="octicon"
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 16 16"
        width="16"
        height="16"
      >
        <path
          fill-rule="evenodd"
          d="M13.78 4.22a.75.75 0 010 1.06l-7.25 7.25a.75.75 0 01-1.06 0L2.22 9.28a.75.75 0 011.06-1.06L6 10.94l6.72-6.72a.75.75 0 011.06 0z"
        ></path>
      </svg>
      Saved successfully
    </div>
    <div id="form">
      <form id="myForm" onsubmit="handleFormSubmit(this)">
        <div class="form-group">
          <div class="form-group-header">
            <label for="name">Name</label>
          </div>
          <div class="form-group-body">
            <input
              class="form-control input-block"
              id="name"
              name="name"
              type="text"
              value="<?= name ?>"
            />
          </div>
        </div>
        <div class="form-actions">
          <button class="btn btn-primary" type="submit">Submit</button>
        </div>
      </form>
    </div>
  </body>
</html>
```

:::

これで、冒頭の GIF で紹介したような見た目のダイアログが完成しました。

![](https://storage.googleapis.com/zenn-user-upload/cc60ad18bfec-20220925.png)

# コード

記事中のサンプルコードはこちらのリポジトリにも置いてあります。

https://github.com/zaki-yama-labs/gas-form-dialog-example

# 参考リンク

- [HTML Service: Create and Serve HTML  |  Apps Script  |  Google Developers](https://developers.google.com/apps-script/guides/html)
- [HTML Service: Communicate with Server Functions  |  Apps Script  |  Google Developers](https://developers.google.com/apps-script/guides/html/communication)
- [Custom dialog in Google Sheets](https://spreadsheet.dev/custom-dialog-in-google-sheets#build-data-entry-forms-within-custom-dialogs-in-google-sheets-using-apps-script)
- [[GAS] Google Apps Script の HtmlService まとめ - dackdive's blog](https://dackdive.hateblo.jp/entry/2015/02/01/010540)
