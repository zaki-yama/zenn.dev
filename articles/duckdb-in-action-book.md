---
title: "「DuckDB in Action」でDuckDBに入門した"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["duckdb"]
publication_name: "loglass"
published: false
---

![](https://storage.googleapis.com/zenn-user-upload/bd39bda997ad-20241221.png)

:::message
この記事は **[株式会社ログラス Productチーム Advent Calendar 2024](https://qiita.com/advent-calendar/2024/loglass)** のシリーズ 1、**22日目** の記事です。
:::


こんにちは。
株式会社ログラスで CRE をやっている山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）と申します。

毎年アドベントカレンダーの季節になると、「これを機に今まで気になっていたことを調べてみよう」という気持ちになります。
今年はここ数ヶ月社内外で耳にすることの多かった「DuckDB」をテーマに選びました。

また、DuckDB の何を学ぶべきかと考えていたとき、ふと
「DuckDB in Action という無料で読める書籍があるらしい」
という話をどこかで聞いたのを思い出しました。
ちょうどいい機会だと思い、実際に読んでみることにしました。

本記事では、この書籍を通して学べることや読んでみた感想をご紹介します。

## 書籍「DuckDB in Action」について

「DuckDB in Action」（以下、「本書」）は Manning 社から 2024 年 7 月に出版された書籍です。

https://www.manning.com/books/duckdb-in-action

本書自体は有料ですが、PDF 版を [Motherduck](https://motherduck.com/) 社が無料で提供しているようです。
こちらからダウンロードできます。

https://motherduck.com/duckdb-book-brief/

なお、Motherduck 社は、DuckDB を基盤にしたサーバーレスのクラウド分析プラットフォームを提供している企業です。

## 本書の内容紹介

本書の内容について、先ほどの [Manning 社の紹介ページ](https://www.manning.com/books/duckdb-in-action) の「about the book」から引用します。

> DuckDB in Action guides you example-by-example from setup, through your first SQL query, to advanced topics like building data pipelines and embedding DuckDB as a local data store for a Streamlit web app.

また、目次は以下の通りです。

```
 1 An introduction to DuckDB
 2 Getting started with DuckDB
 3 Executing SQL queries
 4 Advanced aggregation and analysis of data
 5 Exploring data without persistence
 6 Integrating with the Python ecosystem
 7 DuckDB in the cloud with MotherDuck
 8 Building data pipelines with DuckDB
 9 Building and deploying data apps
10 Performance considerations for large datasets
11 Conclusion
```

印象に残ったところを中心に、各章の内容をかいつまんで紹介します。

### 1章、2章

1 章では DuckDB の概要や特長、どういったユースケースに DuckDB が適しているのか等が紹介されています。
「とりあえず DuckDB がどんなものなのか、手を動かして体験してみたい」という方は、後続の章を読んでからここに戻ってくるのも良いと思います。

2 章「Getting started with DuckDB」から、早速 DuckDB をインストールし、CLI を用いたデータ分析方法について学んでいきます。
ユニークだなと思ったのは、最初に登場するのがこのような SELECT 文である点です。

```sql
SELECT count(*)
FROM 'https://github.com/bnokoro/Data-Science/raw/master/'
      'countries%20of%20the%20world.csv';
```

CSV ファイル、かつローカルではなく Web 上に存在するデータソースに対してのクエリになっています。
DuckDB の特長である、多様なデータ形式（CSV、JSON、Parquet）をサポートしている点や、ローカルだけでなくオンラインのデータや S3 などのクラウドストレージにもアクセスできる点が現れているなと感じました。

### 3章、4章

3 章、4 章については、DuckDB というよりは SQL の説明が中心だったので、そのあたりの知識があれば読み飛ばしてもいいかもしれません。
ただ、3.5 節では DuckDB 独自の構文として `EXCLUDE` や `REPLACE` といった句が紹介されていました。

```sql
SELECT * REPLACE (round(kWh)::int AS kWh)
FROM v_power_per_day;
```

という書き方で、元の列の値を変換できます。

### 5章

5 章では JSON ファイルをデータソースとして扱う方法や、CSV ファイルを読み込んで Parquet ファイルに変換する方法、また保存した Parquet ファイルもデータソースとして読み込み、クエリできることなどが紹介されています。

CSV ファイルから Parquet ファイルに変換する方法（5.4 節）として、本書で紹介されているのは以下のようなクエリです。

```sql
COPY (
	SELECT * EXCLUDE (
    player,
    wikidata_id,
  )
  REPLACE (
    cast(strptime(ranking_date::VARCHAR, '%Y%m%d') AS DATE) AS ranking_date,
    cast(strptime(dob, '%Y%m%d') AS DATE) AS dob
  ),
  name_first || ' ' || name_last AS name
	FROM 'atp/atp_rankings_*.csv' rankings
	JOIN (FROM 'atp/atp_players.csv') players
	  ON players.player_id = rankings.player
)
TO 'atp_rankings.parquet'
(FORMAT PARQUET, CODEC 'SNAPPY', ROW_GROUP_SIZE 100000);
```

- `REPLACE` を使い、文字列型と認識されている日付関連の列を DATE 型に変換する
- `FROM 'atp/atp_rankings_*.csv'` という正規表現を使うことで、複数の CSV ファイルをマージして 1 つの Parquet ファイルを生成する

といった処理を行っています。

また後半はさらに多様なデータ形式の一例として、SQLite や Excel ファイルの扱いについても述べられています。（ここはあまり読んでないですが）

### 6 章以降

ここまでは CLI で DuckDB を実行していましたが、6 章では Python のパッケージを使って Python ファイルから DuckDB を利用する方法が紹介されています。
その中で、DuckDB のライブラリからであれば Pandas や Polars の DataFrames に対しても、データベースのテーブルやビューと同じように操作できることが述べられていました。

8 章や 9 章はより実用的な例として、8 章では DuckDB を dbt などと組み合わせてデータパイプラインを構築する方法、9 章では Streamlit と組み合わせて GUI アプリケーションを構築する方法が紹介されています。
8 章に関しては、私があまりこの分野の知識を持っていないため、やっていることはわかるものの表面的な理解に留まってしまいました。

## 読んだ感想

DuckDB に対する予備知識なしで読み始めましたが、手元で実行しながら学ぶことができたため、使い方や特徴を学ぶにあたって特にハードルはありませんでした。
特に、私の場合「DuckDB というものが何なのか知りたいが、特に作りたいものはない」という状態だったので、豊富な例を用いて解説されていた点は非常に良かったです。

また、各章の独立性が高いので、具体例の中でも気になったものだけかいつまんで読むのも良いと思います。

唯一の難点が英語であることでしたが、書籍が PDF 形式で入手できるおかげで NotebookLM や ChatGPT に助けてもらいながら読み進めることができました。
良い時代になりました。

個人的には、本書では触れられなかった [DuckDB-Wasm](https://duckdb.org/docs/api/wasm/overview.html) を使ってブラウザでも動かせるというのが気になっています。
次はこれで何か作ってみたいなあ。
