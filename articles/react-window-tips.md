---
title: "react-windowでこんなことできる？のまとめ"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript"]
published: true
---

この記事は [React Advent Calendar 2021](https://qiita.com/advent-calendar/2021/react) 21 日目の記事です。

## はじめに

以前、とあるプロジェクトに [react-window](https://github.com/bvaughn/react-window) というライブラリの導入を検討したことがあり、
そのとき調査したことを Tips としてまとめた記事です。

記事中に登場するサンプルコードの完全版はこちらのリポジトリに置いてあるほか、

https://github.com/zaki-yama-labs/react-window-playground

Storybook 上で動作するサンプルもこちらにあります。
[https://61c1f9a280e634003a329ca6-nxwkgquwen.chromatic.com](https://61c1f9a280e634003a329ca6-nxwkgquwen.chromatic.com)

なお、react-window は一次元の List と二次元の Grid という 2 種類のコンポーネントを提供していますが、当時の調査対象が Grid だったため、ここでも Grid を扱います。

## 前提知識: Grid の基本的な使い方

react-window の Grid は [FixedSizeGrid](https://react-window.vercel.app/#/api/FixedSizeGrid) および [VariableSizeGrid](https://react-window.vercel.app/#/api/VariableSizeGrid) という 2 つのコンポーネントを提供しています。(これは List も同様)

両者は行や列ごとの幅(高さ)が固定 (Fixed) か可変 (Variable) か、という点が異なります。
例として、VariableSizeGrid を使ったサンプルコードを以下に示します。

```tsx
import { VariableSizeGrid, GridChildComponentProps } from "react-window";

const columnWidths = new Array(1000)
  .fill(true)
  .map(() => 75 + Math.round(Math.random() * 50));
const rowHeights = new Array(1000)
  .fill(true)
  .map(() => 25 + Math.round(Math.random() * 50));

const Cell = ({ columnIndex, rowIndex, style }: GridChildComponentProps) => (
  <div style={style}>
    Item {rowIndex},{columnIndex}
  </div>
);

const Grid = () => {
  return (
    <VariableSizeGrid
      width={300}
      height={150}
      columnCount={1000}
      rowCount={1000}
      columnWidth={(index) => columnWidths[index]}
      rowHeight={(index) => rowHeights[index]}
    >
      {Cell}
    </VariableSizeGrid>
  );
};
```

- Grid 全体の幅・高さ (`width` & `height`)
- 行・列方向の要素数 (`columnCount` & `rowCount`)
- 行または列ごとの幅 (`columnWidth` & `rowHeight`)

を必須 props として受け取ります。
VariableSizeGrid は `(index: number) => number` という関数で 1 行(列)ごとに幅を変えられるようになっていますが、FixedSizeGrid はすべての行(列)で同じ幅になるため、単純に `number` を受け取ります。

また、Grid 内部に描画する Cell コンポーネントは children として渡します。

それ以外の必須ではない props も含めた props 一覧については以下に記載します。

:::details FixedSizeGrid/VariableSizeGrid の props

| name                 | required |                       type                        | description                                                                                           |
| :------------------- | :------: | :-----------------------------------------------: | :---------------------------------------------------------------------------------------------------- |
| children             |   Yes    | `({ columnIndex, rowIndex, style }) => Component` | Cell コンポーネント                                                                                   |
| width                |   Yes    |                     `number`                      |
| height               |   Yes    |                     `number`                      |
| columnCount          |   Yes    |                     `number`                      |
| columnWidth          |   Yes    |            `(index: number) => number`            | FixedSizeGrid は `number`                                                                             |
| rowCount             |   Yes    |                     `number`                      |
| rowHeight            |   Yes    |            `(index: number) => number`            | FixedSizeGrid は `number`                                                                             |
|                      |          |
| className            |          |                   `string = ""`                   |
| style                |          |                  `Object = null`                  |
| direction            |          |                  `ltr` \| `rtl`                   |
| initialScrollLeft    |          |                   `number = 0`                    | Horizontal scroll offset                                                                              |
| initialScrollTop     |          |                   `number = 0`                    | Vertical scroll offset                                                                                |
| innerRef             |          |         `function` \| `createRef` object          |
| innerElementType     |          |            `React$ElementType = "div"`            |
| itemData             |          |                       `any`                       | 指定すると、Cell コンポーネントに props として渡される                                                |
| itemKey              |          |   `({ columnIndex, data, rowIndex }) => string`   |
|                      |          |
| onItemsRendered      |          |                    `function`                     |
| onScroll             |          |                    `function`                     |
| outerRef             |          |         `function` \| `createRef` object          |
| outerElementType     |          |            `React$ElementType = "div"`            |
| overscanColumnCount  |          |                   `number = 1`                    |
| overscanRowCount     |          |                   `number = 1`                    |
| useIsScrolling       |          |                 `boolean = false`                 | `true` にすると children に `isScrolling` という prop が渡される。スクロール中は `isScrolling = true` |
| estimatedColumnWidth |          |                   `number = 50`                   | VariableSizeGrid のみ                                                                                 |
| estimatedRowHeight   |          |                   `number = 50`                   | VariableSizeGrid のみ                                                                                 |

:::

## Cell に任意のデータを渡したい

Grid で描画したいデータはコンポーネントの外部から渡すケースがほとんどかと思います。

![](https://storage.googleapis.com/zenn-user-upload/c2123e72d3d7-20211221.png)

この場合、**`itemData` prop を使います。**
Grid コンポーネントは `itemData` という prop を受け取れるようになっており、ここに渡したデータは Cell コンポーネント側では `data` という prop で扱うことができます。
Cell に渡したいデータは基本的にすべてこの `itemData` に詰めて渡すことになります。

:::details サンプルコード

```tsx
import { VariableSizeGrid, GridChildComponentProps } from "react-window";

import records from "./data.json";

type Record = {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  city: string;
};

const columnWidths = {
  id: 40,
  firstName: 100,
  lastName: 100,
  email: 300,
  city: 100,
};

type Data = Record[];

const Cell = ({
  columnIndex,
  rowIndex,
  style,
  data,
}: GridChildComponentProps) => {
  const record = (data as Data)[rowIndex];
  const key = (Object.keys(record) as Array<keyof Record>)[columnIndex];

  return (
    <div className="cell" style={style}>
      {record[key]}
    </div>
  );
};

const GridWithItemData = () => {
  return (
    <VariableSizeGrid
      className="grid"
      columnCount={5}
      columnWidth={(index) => {
        return columnWidths[
          (Object.keys(records[0]) as Array<keyof Record>)[index]
        ];
      }}
      height={300}
      rowCount={records.length}
      rowHeight={(index) => 50}
      width={600}
      itemData={records}
    >
      {Cell}
    </VariableSizeGrid>
  );
};
```

:::

TypeScript 的には Cell の `data` は `any` 型になっているので、取り出すときにキャストするのがいいでしょう。

## マウスオーバーで行に色をつけたい

これもテーブルでよくあるケースです。

![](https://storage.googleapis.com/zenn-user-upload/f5ae5916effe-20211221.gif)

残念ながら react-window には行または列という単位のコンポーネントはないため、**Cell にイベントハンドラーを設定して自前で実装する必要があります。**

:::details サンプルコード

```tsx
import { Dispatch, SetStateAction, useState } from "react";
import { FixedSizeGrid, GridChildComponentProps } from "react-window";

type Data = {
  hoveredRowIndex: number | null;
  setHoveredRowIndex: Dispatch<SetStateAction<number | null>>;
};

const Cell = ({
  columnIndex,
  rowIndex,
  style,
  data,
}: GridChildComponentProps) => {
  const { hoveredRowIndex, setHoveredRowIndex } = data as Data;
  const className = `cell ${hoveredRowIndex === rowIndex ? "row-hovered" : ""}`;
  return (
    <div
      className={className}
      style={style}
      onMouseEnter={() => {
        console.log("mouseenter", rowIndex, columnIndex);
        setHoveredRowIndex(rowIndex);
      }}
    >
      Item {rowIndex},{columnIndex}
    </div>
  );
};

const GridWithRowHover = () => {
  const [hoveredRowIndex, setHoveredRowIndex] = useState<number | null>(null);

  return (
    <FixedSizeGrid
      className="grid"
      columnCount={1000}
      columnWidth={100}
      height={300}
      rowCount={1000}
      rowHeight={50}
      width={600}
      itemData={{
        hoveredRowIndex,
        setHoveredRowIndex,
      }}
    >
      {Cell}
    </FixedSizeGrid>
  );
};
```

:::

参考: [Is there any way to have an :hover css modifier at the row level ? · Issue #252 · bvaughn/react-window](https://github.com/bvaughn/react-window/issues/252)

## Grid のサイズを自動調整したい

親となる要素のリサイズに合わせて Grid 自体のサイズも調節したいケースです。

![](https://storage.googleapis.com/zenn-user-upload/c8dcd42fffc6-20211221.gif)

これは **[`react-virtualized-auto-sizer`](https://github.com/bvaughn/react-virtualized-auto-sizer/) というライブラリを併用すると実現できます。**
README にも記載されています。
[Frequently asked questions > Can a list or a grid fill 100% the width or height of a page?](https://github.com/bvaughn/react-window#can-a-list-or-a-grid-fill-100-the-width-or-height-of-a-page)

:::details サンプルコード

```tsx
import { VariableSizeGrid, GridChildComponentProps } from "react-window";
import AutoSizer from "react-virtualized-auto-sizer";

const Cell = ({ columnIndex, rowIndex, style }: GridChildComponentProps) => (
  <div className="cell" style={style}>
    Item {rowIndex},{columnIndex}
  </div>
);

const GridWithAutoResize = () => {
  return (
    <div className="container">
      <AutoSizer>
        {({ width, height }) => (
          <VariableSizeGrid
            className="grid"
            columnCount={1000}
            columnWidth={(index) => columnWidths[index]}
            height={height}
            rowCount={1000}
            rowHeight={(index) => rowHeights[index]}
            width={width}
          >
            {Cell}
          </VariableSizeGrid>
        )}
      </AutoSizer>
    </div>
  );
};
```

:::

## 複数 Grid 間でスクロールを同期したい

たとえば一部の行または列を固定するために、複数の Grid を結合したコンポーネントを作るといったケースです。
この場合、全体として 1 つの Grid に見えるようスクロールなどは Grid 間で同期したくなります。

![](https://storage.googleapis.com/zenn-user-upload/9715efea9932-20211221.gif)

このようなケースは **`onScroll` props および `scrollTo({scrollLeft: number, scrollTop: number})` メソッドを使うと実現できます。**

:::details サンプルコード

```tsx
const GridWithScrollSync = () => {
  const leftGridRef = useRef<VariableSizeGrid>(null);
  const rightGridRef = useRef<VariableSizeGrid>(null);

  const onScroll = ({ scrollTop }: GridOnScrollProps) => {
    if (leftGridRef.current) {
      leftGridRef.current.scrollTo({ scrollTop });
    }
    if (rightGridRef.current) {
      rightGridRef.current.scrollTo({ scrollTop });
    }
  };

  return (
    <div className="container">
      <div style={{ width: columnWidths[0], overflow: "hidden" }}>
        <VariableSizeGrid
          ref={leftGridRef}
          onScroll={onScroll}
          className="leftGrid"
          style={{ overflowX: "hidden", overflowY: "auto" }}
          width={columnWidths[0] + scrollbarSize()}
          height={300}
          columnWidth={(index) => columnWidths[index]}
          rowHeight={(index) => rowHeights[index]}
          columnCount={1}
          rowCount={20}
        >
          {LeftGridCell}
        </VariableSizeGrid>
      </div>
      <VariableSizeGrid
        ref={rightGridRef}
        onScroll={onScroll}
        className="rightGrid"
        width={500}
        height={300}
        columnWidth={(index) => columnWidths[index]}
        rowHeight={(index) => rowHeights[index]}
        columnCount={40}
        rowCount={20}
      >
        {RightGridCell}
      </VariableSizeGrid>
    </div>
  );
};
```

:::

Grid は `onScroll` という prop を指定できるようになっており、スクロールしたときにこちらのイベントハンドラーが呼ばれます。
イベントハンドラーには `scrollLeft` および `scrollTop` というスクロール位置に関する情報が引数として渡されます。
また、Grid のインスタンスには `scrollTo({scrollLeft: number, scrollTop: number})` というメソッドが備わっているため、`onScroll` で得られたスクロール位置をそのまま `scrollTo()` に渡してあげることでスクロールの同期を実現しています。

なお、ちょっとしたポイントとしては、2 つの Grid をそのまま並べるだけだとスクロールバーも両方の Grid に表示されてしまいます。

![](https://storage.googleapis.com/zenn-user-upload/47bfe70ac884-20211221.png)

それを防止するため、ここでは

- スクロールバーを表示したくない側の Grid には元の Grid をラップする div 要素を足す。
  width は元の Grid と同じで、`overflow: hidden` をつける
- 元の Grid はスクロールバーの分だけ width をプラスして、はみ出させる。かつ `overflow-y: auto` とする

とすることでスクロールバーを隠しています。
これは、[react-virtualized](https://github.com/bvaughn/react-virtualized) の [MultiGrid](https://bvaughn.github.io/react-virtualized/#/components/MultiGrid) というコンポーネントの実装を参考にしました。
(ソースコードはこのあたり: https://github.com/bvaughn/react-virtualized/blob/v9.22.3/source/MultiGrid/MultiGrid.js#L632-L665)

さらに余談ですが、スクロールバーのサイズの算出には [dom-helpers](https://www.npmjs.com/package/dom-helpers) というライブラリを利用しています。

参考: [Compatibility with ScrollSync · Issue #86 · bvaughn/react-window](https://github.com/bvaughn/react-window/issues/86)
