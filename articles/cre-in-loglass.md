---
title: "ログラスでCREを立ち上げた背景とこれから"
emoji: "🤝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["cre"]
publication_name: "loglass"
published: true
---

[株式会社ログラス](https://www.loglass.jp/)　エンジニアの山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）です。
ログラスではフロントエンドエンジニアとしてプロダクト開発を担当する傍ら、CRE（Customer Reliability Engineer）というロールも兼務しており、業務時間のうち一定割合をこの CRE としての活動に充てています。
本日はこの CRE というロールを立ち上げた経緯と、立ち上げにあたり具体的にどんなことをやってきたのか、そして今後取り組みたいことを紹介します。

# CRE 立ち上げの経緯

ログラスにおいてカスタマーサクセスは非常に重要な要素です。

ログラスが扱う「経営管理」という領域は非常に複雑なドメイン知識が求められます。弊社のカスタマーサクセス（CS）チームには、元々お客様と同じ経営企画業務を経験していた者も多く、単なるプロダクトの使い方にとどまらずお客様の業務に踏み込んだ支援を可能にしています。経営企画という、企業活動の根幹に関わる部分で CS チームがお客様と伴走できるというのは、ログラスの強みの 1 つとなっています。
一方、お客様に対するフロントとしては CS チームがいますが、その CS チームとプロダクトチームをつなぐ部分に関しては、まだまだ体系化できておりませんでした。具体的には、CS チームが日々お客様からいただいた声をプロダクトチームにどのように届けるか、CS チームが真にお客様と向き合えるためにプロダクトチームとしてはどういったサポートが必要なのか、といったことです。またそのつなぎ込みとして、プロダクトチームと CS チームをつなぐ専門のロールが必要なのでは、と模索しておりました。
そんな折、国内外で CRE（Customer Reliability Engineer）という職種について言及されている事例記事を見かけるようになりました。

- [CRE (Customer Reliability Engineer) 職 採用 - 採用情報 - 株式会社はてな](https://hatena.co.jp/recruit/career/cre)
- [Kyash の現状の課題と CRE 募集ページの解説 - Kyash Product Blog](https://blog.kyash.co/entry/2019/08/29/173359)
- [アンドパッドの CRE チームを紹介します！ - ANDPAD Tech Blog](https://tech.andpad.co.jp/entry/2020/11/30/120000)
- [とある教育サービスにおける CRE というロールのお話 - Qiita](https://qiita.com/kozy4324/items/294bd3b1d7f0f6ecab29)
- [「CSM」と「CRE」の 2 職種を新たに設けた背景や舞台裏 | Autify Blog](https://blog.autify.com/ja/from-csce-to-csm-and-cre)

ログラスにおいても、このような顧客に軸足を置いたエンジニアというロールが必要なのではと考え、ロールの立ち上げに至りました。

（以下、「お客様」を「顧客」という表現で統一します）

# 立ち上げにあたって：責務の言語化

CRE というロールを新たに立ち上げるにあたり、まず行ったのが責務の言語化です。
そのためにはまず CRE というロールに対する理解を深める必要があり、国内外の事例を参考にしました。
まずはじめに読んだのは、CRE を最初に提唱した Google による記事です。

https://cloudplatform-jp.googleblog.com/2016/10/google-cre.html

この記事から、従来の SRE という職で培われたプラクティスを顧客に適用したものが CRE であること、CRE とは「お客様の不安をゼロにする」ことが究極的なミッションであること、などを学びました。

しかしながら、職務が具体的にイメージできない箇所もありました。たとえば、以下のような記述です。

> CRE チームは、お客様の基幹アプリケーションにおける主要な要素を、コードからデザイン、実装、運用手順に至るまで綿密に調査します。そこで見つけたものを取り出し、アプリケーション（および関連チーム）に対して厳しく PRR を実施します。

（PRR とは "Production Readiness Review" の略で、Google 社内における検査プロセスを指すそうです）

これをログラスにそのまま適用できるイメージがわかない理由として、Google にとっての顧客 = Google のプラットフォームを利用してアプリケーション開発を行う開発者　であり、Google と顧客の間で技術的な話ができる、Google は顧客のコードを見れるという前提になっており、上記のような表現になるのだと解釈しました。
ログラスが提供するのは経営企画の方向けの SaaS であり、Google のようなプラットフォームとは特性が異なるため、ここについては自社にあった形で責務を定義する必要があると感じました。
その後、他社の事例記事なども参考にしつつ、以下のようなキーワードが重要であると考えました。

- 顧客の不安をゼロにするためにエンジニアリングで貢献する、というのが大まかなミッション
  - プロセス全体に責任は持つが、具体的な仕事として何をどこまで担当するかは各社それぞれ
- リアクティブ（受動的）とプロアクティブ（能動的）
  - 問い合わせ対応、要望対応などは前者
  - 顧客が直接声を上げなくても、顧客の不安を減らすためのアクションをこちらから能動的に起こせる、のが後者
- データドリブン：プロアクティブなアクションのためのデータ収集と分析

これらを踏まえ、ログラスにおける CRE の責務を以下のように定めました。
（社内の notion ページをそのまま添付しています）

![ログラス社内の「CREの責務」ドキュメントの一部](https://storage.googleapis.com/zenn-user-upload/bbcb6ba80204-20220820.png)
*ログラス社内の「CREの責務」ドキュメントの一部*

同時に、現時点で CRE に求められていること「ではない」ことも言語化しておくことにしました。

![](https://storage.googleapis.com/zenn-user-upload/d3c58bca65a6-20220820.png)

:::message
ここでの「UX 改善」とは「顧客からいただいた要望を起点とした機能改善」を意味しています
:::

特に言語化するまでは CRE の ”E” の部分、すなわち「どのような点でエンジニアとしてのスキルが求められるのか？」という点が自分の中で腹落ちしていませんでしたが、「**顧客が抱えている課題を計測・分析し、信頼性向上のためのプロアクティブな行動につなげる。そのためのデータモニタリングや分析基盤構築といった活動には大いにエンジニアリングのスキルが必要となる**」のだと解釈しました。
そのため、プロダクトの**直接的な改善**については CRE のみの責務ではなく、エンジニアチーム全体で受け持つべき、としました。ただこれはあくまで現時点のログラスに合うように定義したものであり、プロダクトや組織の特性によって変わるものだと思っています。また、ログラスも組織のフェーズが変われば見直されるべき点だと考えています。

# 具体的にやったこと

ここでは、社内で行った CRE としての活動の具体例を 1 つご紹介します。

CRE としての活動の第一歩として、まずは責務のうちの「**エンジニアチームと CS・PdM の橋渡しとなり、カスタマーサクセスを組織一丸となって実現する**」にフォーカスしました。
まずここにフォーカスした理由として、ログラスには CRE を立ち上げる前から、開発リソースの一定割合を UX 改善（顧客要望を起点としたプロダクトの機能改善、を意味しています）に充てるという文化がありました。しかしながら、そのプロセスは属人的な部分も多く、以下のような課題を抱えていました。

- 社内の各種定例 MTG がどのようにつながっており、それぞれでどういう議論がされているのかわからない
- 改善チケットの優先度などを、いつ・誰が意思決定しているのかが不明瞭

そのため、まずはここにオーナーシップを持って取り組むことにしました。

具体的にやったこととして、まずはエンジニア・CS にヒアリングしたうえで、プロセスを図におこしました。

![](https://storage.googleapis.com/zenn-user-upload/03ff0eba6467-20220820.png)

![](https://storage.googleapis.com/zenn-user-upload/e98c44cc3b16-20220820.png)

そしてこの図を元に CS・エンジニア間で認識を揃え、さらにお互いに感じている課題を洗い出しました。

これにより、各イベントが何を決める場なのか、優先度について誰がどこで最終決定しているのかが明確になり、またお互いどこに課題感を持っているのかがわかりました。その後、こうした課題を 1 つ 1 つ解決していくことで、以前よりも透明性が高いプロセスになり、納得感を持って UX 改善を進めることができました。

なお、上図に出てくる各種イベントについての詳細は以前弊社の CS チームが記事にしてくれています。よければご覧ください。

https://note.com/yukiasami/n/n089e32914f25

また、この改善プロセスがうまく運用されるようになると、別の課題が生まれてきました。

- 顧客要望をそのまま実装するだけでは受け身の改善になってしまう。ある程度要望を集約したうえで、どう解決するかはより大きな機能開発の枠組みでやるべきではないか
- 1 スプリントあたり X ストーリーポイント分、という枠を設ける運用だと、それを超える改善要望が挙がってきた際に消化できず、重要な改善がいつまでも優先度上がらなくなってしまう

そのため、徐々にこのプロセスで対応すべきは「緊急性の高い要望」のみに留め、それ以外の要望については一旦プロダクトマネージャーが取りまとめた後、新機能と同じようにプロダクト全体のエピックの中で優先度をつけて取り組むように変化していきました。

# これから

CRE として最初に取り組んだ活動は、泥臭いアプローチではありますが徐々に良い方向に進みだしました。引き続き「カスタマーサクセスを組織一丸となって実現する」ために必要なことは行っていきつつ、徐々に責務の 3 つめに記載した「**データに基づきプロアクティブな行動を起こす**」という領域にチャレンジし始めています。

ログラスは[今年の 4 月にシリーズ A の資金調達を終えた](https://prtimes.jp/main/html/rd/p/000000056.000052025.html)ばかりのスタートアップです。ビジネスの成長に伴い、顧客が増えるにつれて、CS チームがこれまでのように顧客に寄り添った支援を行うのは難しくなっていきます。そのため、「今注力すべき顧客」を素早く判断し、適切なアプローチができるような、データ収集および活用の仕組みを整えていく、というのが次の目標です。

具体的には、現在は**顧客の不安（信頼性）を定量的に評価できる指標作り**に取り組んでいます。たとえば機能の利用状況（よく使われている／いない機能）、エラー率などが得られると、顧客が十分に活用できていない機能を分析してプロダクトの改善につなげたり、CS チーム目線では「今サポートを必要としている顧客」に対して最適なタイミングでアプローチができるようになると期待しています。
また、これらの指標を元に CRE と CS が **「いつ」「誰に」「どのようなアクションを取るべきか」** といったオペレーションフローの構築につなげたいと考えています。そのために、何を指標とすべきか？を CS チームと連携しながら決めるだけでなく、必要なデータを収集・分析するためのツール選定や基盤作りといったスキルが求められ、技術的にも非常にチャレンジングだと感じています。

# おわりに

ログラスにおいて CRE というロールを立ち上げるにあたり考えたことや、立ち上げ後の具体的な活動をご紹介しました。特に責務を言語化するにあたって、先駆者である各社の事例記事が大変参考になりました。本記事も、CRE や同じような職種に携わる方々にとって少しでも参考になる部分があれば幸いです。
