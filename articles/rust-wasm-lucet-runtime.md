---
title: "RustをWebAssemblyにコンパイルしてlucet-runtimeで動かす"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "webassembly"]
published: false
---

# はじめに

[Lucet](https://github.com/bytecodealliance/lucet) は Fastly が開発している WebAssembly のコンパイラ & ランタイムです。
https://github.com/bytecodealliance/lucet

公式ドキュメント：
https://bytecodealliance.github.io/lucet

Lucet は WebAssembly ファイルを実行するための CLI に加え、Rust のプログラムから Lucet を利用できる [`lucet-runtime`](https://docs.rs/lucet-runtime) という ランタイム API も提供しています。

今回、この lucet-runtime を公式ドキュメントの通りに試したところ意外とハマるポイントが多かったことや、
公式ドキュメントには WebAssembly にコンパイルするサンプルコードがC 言語で書かれたものしかなく「Rust で書くには？」と試行錯誤したため、備忘録的に手順をまとめておきます。

公式ドキュメントの該当箇所は
- [2.2. Lucet "Hello World"](https://bytecodealliance.github.io/lucet/Your-first-Lucet-application.html)
- [2.3. Using the Lucet runtime API from Rust](https://bytecodealliance.github.io/lucet/lucet-runtime-example.html)

あたりです。

また、Lucet のソースコードをコンパイルしたもので動作確認しましたが、記事執筆時点の最新コミットは [`51fb1ed
`](https://github.com/bytecodealliance/lucet/tree/51fb1ed414fe44f842db437d94abb6eb439d7c92) です。

# 前提：必要なものをインストール

- Lucet 本体

「[2.1. Compiling Lucet](https://bytecodealliance.github.io/lucet/Compiling.html)」に従い Lucet をローカル環境でコンパイルします。
ドキュメントでは Linux と macOS でのインストール手順が記載されており、私は macOS でコンパイルしました。

:::message
おおむねドキュメントに沿って必要なツールをインストールすれば OK ですが、以下は変更したほうが良いポイントです。

- Rust も Homebrew でインストールするようになっているが rustup を使ったほうがいい
  - ref. https://www.rust-lang.org/ja/learn/get-started
- 記事執筆時点での llvm の最新バージョンは 12 のため clang のパスは `/usr/local/opt/llvm/lib/clang/10*` ではなく `/usr/local/opt/llvm/lib/clang/12*` となる
:::

- Rust のビルドターゲット `wasm32-wasi`

WASI に対応した WebAssembly ファイルにコンパイルするため、ビルドターゲットを追加しておきます。

```
$ rustup target add wasm32-wasi
```

# 手順

## 1. Rust でパッケージを作成し、WebAssembly にコンパイルする

はじめに、WebAssembly ファイル作成用のパッケージを作成します。パッケージの置き場所、パッケージ名は自由に決めて大丈夫です。

```zsh
$ cd ~/workspace
$ cargo new example
$ cd example
```

`src/main.rs` の中身は、次のように `println!` だけの main 関数とします。

```rust:src/main.rs
fn main() {
  println!("Hello, Lucet!");
}
```

保存したら、このファイルをビルドします。その際、ターゲットに `wasm32-wasi` を指定します。

```zsh
$ cargo build --target wasm32-wasi
```

`target/wasm32-wasi/debug` ディレクトリの下に `***.wasm` （`***` はパッケージ名。この場合 `example`）が生成されています。

## 2. WebAssembly ファイルをネイティブコードにコンパイルする

Lucet が実行するのは WebAssembly ファイルそのものではなく、事前に WebAssembly をネイティブコードにコンパイルしたファイルです。
（このあたり私は詳しくありませんが、AOT (Ahead of Time) コンパイル方式と呼ぶそうです。
参考：[WASMで様々なリソースにアクセス. How Lucet runs WebAssembly | by FUJITA Tomonori | nttlabs | Medium](https://medium.com/nttlabs/lucet-wasi-f93820c515f1)）

ネイティブコードへのコンパイルには `lucetc-wasi` というコマンドを使います。
Lucet のコンパイルの手順の中で `source /opt/lucet/bin/setenv.sh` を実行していればパスが通っているはずです。

```
$ lucetc-wasi -o example.so target/wasm32-wasi/debug/example.wasm
$ ls
Cargo.lock  Cargo.toml  example.so*    src/        target/
```

## 3. lucet-runtime を使ったパッケージを Lucet リポジトリ内に作成する

作成した .so ファイルを読み込み、lucet-runtime で実行するための別のパッケージを作成します。

:::message
lucet-runtime は Cargo クレートとしても提供されていますが、記事を書いた 2021-04-27 時点の最新バージョンである [0.6.1](https://docs.rs/lucet-runtime/0.6.1/lucet_runtime/) では動作しなかったため、Lucet リポジトリ内にパッケージを作成して最新のコードを使うようにしています。
:::

Lucet リポジトリ内にパッケージを作成する際、2通りの方法があります。

- リポジトリ内に [`docs/lucet-runtime-example`](https://github.com/bytecodealliance/lucet/tree/main/docs/lucet-runtime-example) というサンプルパッケージがあるため、これをそのまま編集する
- リポジトリ内の任意の位置に `cargo new` でパッケージを作成し、リポジトリのルートにある Cargo.toml の `members` に追加する

後者の場合で説明します。前者の場合、3-1 〜 3-3 は不要ですが 3-4 main.rs の修正は必要です。

### 3-1. リポジトリのルートの Cargo.toml にパッケージへのパスを追加

```bash
# 例: src/ ディレクトリの下に hello-lucet-runtime というパッケージを作る
$ cd path/to/lucet
$ mkdir src
$ cd src
$ cargo new hello-lucet-runtime
```

でパッケージを作成した後、リポジトリのルートの Cargo.toml にパッケージへのパスを追加します。

```diff:Cargo.toml
 [workspace]
 members = [
   "benchmarks/lucet-benchmarks",
   "docs/lucet-runtime-example",
+  "src/hello-lucet-runtime",
   "lucet-concurrency-tests",
   "lucet-module",
   ...
```

この後は基本的には「[2.3. Using the Lucet runtime API from Rust](https://bytecodealliance.github.io/lucet/lucet-runtime-example.html)」に従い、作成したパッケージを編集します。

### 3-2. .cargo/config ファイルを追加

```toml:src/hello-lucet-runtime/.cargo/config
[build]
rustflags = ["-C", "link-args=-rdynamic"]
```

### 3-3. Cargo.toml の `[dependencies]` に lucet-runtime および lucet-wasi を相対パスで追加

```toml:src/hello-lucet-runtime/Cargo.toml
# 相対パスはパッケージディレクトリの深さによって変わる
[dependencies]
lucet-runtime = { path = "../../lucet-runtime" }
lucet-wasi = { path = "../../lucet-wasi" }
```

### 3-4. src/main.rs を編集

src/main.rs を次のようにします。

```rust:src/hello-lucet-runtime/src/main.rs
use lucet_runtime::{DlModule, Limits, MmapRegion, Region};
use lucet_wasi::WasiCtxBuilder;

fn main() {
    // ensure the WASI symbols are exported from the final executable
    lucet_wasi::export_wasi_funcs();
    // load the compiled Lucet module
    let dl_module = DlModule::load("example.so").unwrap();
    // (*1) create a new memory region with default limits on heap and stack size
    let region = MmapRegion::create(
        1,
        &Limits::default().with_heap_memory_size(100 * 16 * 64 * 1024),
    )
    // instantiate the module in the memory region
    let mut instance = region.new_instance(dl_module).unwrap();
    // prepare the WASI context, inheriting stdio handles from the host executable
    let wasi_ctx = WasiCtxBuilder::new().inherit_stdio().build().unwrap();
    instance.insert_embed_ctx(wasi_ctx);
    // (*2) run the WASI main function
    instance.run("_start", &[]).unwrap();
}
```

ドキュメントとは違う点が2箇所あります。

- `(*1)` heap memory size の割り当て
  - `&Limits::default()` のままだとエラーになるので、 `.with_heap_memory_size()` というメソッドを使用して割り当てるメモリサイズを変更する
  - 数値に根拠はないが、コードから [デフォルト値は `16 * 64 * 1024` だった](https://github.com/bytecodealliance/lucet/blob/51fb1ed414fe44f842db437d94abb6eb439d7c92/lucet-runtime/lucet-runtime-internals/src/alloc/mod.rs#L494) ので 100倍を指定している
- `(*2)` 最後の行で `instance.run(...)` に渡す文字列は、 `"main"` ではなく `"_start"` が正しい

## 4. .so ファイルをコピー

ステップ2. で作成した .so ファイルを Lucet 内に作成したパッケージ下に配置します。

```
$ cd path/to/lucet/src/hello-lucet-runtime
$ cp ~/workspace/example/example.so .
$ ls
Cargo.toml       example.so*      src/
```

## 5. 実行

`cargo run` で実行します。手順1. で `println!` マクロに記載した文字列が表示されれば成功です。

```
$ cargo run
Hello, Lucet!
```

# 応用

ここまでで main 関数だけのスクリプトは WebAssembly にコンパイルして動かすことができました。
以下はもう少し複雑なことを WebAssembly 側でやろうとして動作を確認したメモです。

## 任意の関数名で実行したい

main.rs 側で `#[no_mangle]` つきで定義します。

```rust:main.rs (WebAssembly にコンパイルする側)
#[no_mangle]
fn hello() {
    println!("hello");
}
```

lucet-runtime を使う側では、`instance.run()` に指定する関数名を変更すれば OK です。

```rust:main.rs (lucet-runtime 使う側)
fn main() {
    // ...
    // run the WASI main function
    instance.run("hello", &[]).unwrap();
}
```
## 関数の引数に値を渡したい & 関数からの戻り値を得たい

まず、WebAssembly にコンパイルする側。こちらは通常の Rust の関数と同じように引数、戻り値を定義します。

```rust:main.rs (WebAssembly にコンパイルする側)
#[no_mangle]
fn add(a: i32, b: i32) -> i32 {
    println!("{}", a + b);
    return a + b;
}
```

lucet-runtime を使う側では、`instance.run()` の第二引数で引数を指定します。
また戻り値は `unwrap_returned()` を呼び出すことで取得できるようです。

```diff:
 fn main() {
     // ...

     // instantiate the module in the memory region
     let mut instance = region.new_instance(dl_module).unwrap();
     // prepare the WASI context, inheriting stdio handles from the host executable
     let wasi_ctx = WasiCtxBuilder::new().inherit_stdio().build().unwrap();
     instance.insert_embed_ctx(wasi_ctx);
     // run the WASI main function
-    instance.run("_start", &[]).unwrap();
+    let retval = instance
+        .run("add", &[5i32.into(), 3i32.into()])
+        .unwrap()
+        .unwrap_returned();
+    println!("{}", i32::from(retval));
+}
```

ここについては少し理解が不十分ですが、[https://docs.rs/lucet-runtime/0.6.1/lucet_runtime/](https://docs.rs/lucet-runtime/0.6.1/lucet_runtime/) を読んだところ
引数は `Val`、戻り値は `UntypedVal` という型であり、どちらも適切な型でキャストする必要があるためこのような書き方をする必要がある、という理解です。

また [`Val`](https://docs.rs/lucet-runtime/0.6.1/lucet_runtime/enum.Val.html) を見る限り文字列などの直接のやり取りはできなさそうです。（要調査）

# トラブルシューティング

調べる過程で遭遇したエラー達です。

## ドキュメントの通りに main.rs 書いたけどエラー
最初に遭遇したやつ。

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: SymbolNotFound("main")', docs/lucet-runtime-example/src/main.rs:17:31
```

**解決策💡**
`instance.run("main", ...)` ではなく `instance.run("_start", ...)` が正しかった。
Issue がある。

https://github.com/bytecodealliance/lucet/issues/598



## lucet-runtime, lucet-wasi のバージョンを指定して実行したけどエラー

Lucet リポジトリとは無関係のディレクトリでパッケージを作成し、

```toml:Cargo.toml
[dependencies]
lucet-runtime = "0.6.1"
lucet-wasi = "0.6.1"
```

として実行したところ以下のエラー。

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: DlError(Custom { kind: Other, error: "dlopen(/Users/yamazaki/workspace/lucet-runtime-example/example.so, 2): Symbol not found: _hostcall_wasi_snapshot_preview1_fd_close\n  Referenced from: /Users/yamazaki/workspace/lucet-runtime-example/example.so\n  Expected in: flat namespace\n in /Users/yamazaki/workspace/lucet-runtime-example/example.so" })', src/main.rs:8:50
```

**解決策💡**
上に書いたとおり、Lucet リポジトリ内にパッケージを作って最新のコードを使う。

## lucet-runtime, lucet-wasi をリポジトリの最新バージョンを指定したけどエラー

Cargo.toml の dependencies には git のリポジトリも指定できるので、これで Lucet 内に置いたのと同様に動くのでは？と思って試したやつ。

```toml:Cargo.toml
[dependencies]
lucet-runtime = { git = "https://github.com/bytecodealliance/lucet", branch = "main" }
lucet-wasi = { git = "https://github.com/bytecodealliance/lucet", branch = "main" }
```

結果は以下のエラー

```
  --> /Users/yamazaki/.cargo/git/checkouts/lucet-d29c7ec0f6d2e0d4/51fb1ed/wasmtime/crates/wasi-common/cap-std-sync/src/file.rs:84:14
   |
84 |             .set_times(convert_systimespec(atime), convert_systimespec(mtime))?;
   |              ^^^^^^^^^ method cannot be called on `cap_std::fs::File` due to unsatisfied trait bounds
   |
  ::: /Users/yamazaki/.cargo/registry/src/github.com-1ecc6299db9ec823/cap-std-0.13.9/src/fs/file.rs:32:1
   |
32 | pub struct File {
   | ---------------
   | |
   | doesn't satisfy `_: unsafe_io::unsafe_handle::AsUnsafeFile`
   | doesn't satisfy `cap_std::fs::File: SetTimes`
   |
   = note: the following trait bounds were not satisfied:
           `cap_std::fs::File: unsafe_io::unsafe_handle::AsUnsafeFile`
           which is required by `cap_std::fs::File: SetTimes`
   = help: items from traits can only be used if the trait is in scope
   = note: the following trait is implemented but not in scope; perhaps add a `use` for it:
           `use fs_set_times::set_times::SetTimes;`
```

**解決策💡**
同上

## ヒープメモリサイズが足りないというエラー

[`docs/lucet-runtime-example`](https://github.com/bytecodealliance/lucet/tree/main/docs/lucet-runtime-example) のコードのまま実行したところ以下のエラー。

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: LimitsExceeded("heap spec initial size: HeapSpec { reserved_size: 4194304, guard_size: 4194304, initial_size: 1114112, max_size: None }")', docs/lucet-runtime-example/src/main.rs:102:55
```

**解決策💡**
上に書いたとおり、 `MmapRegion::create()` の第二引数を変更する。

```rust
let region = MmapRegion::create(
    1,
    &Limits::default().with_heap_memory_size(100 * 16 * 64 * 1024),
)
```

## ヒープメモリサイズに適当な値を指定するとエラー

一つ前で、サイズが足りないと言うので適当に大きな値を指定したところ

```rust
let region = MmapRegion::create(
    1,
    &Limits::default().with_heap_memory_size(1000000),
)
```

それはそれで別のエラー。

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: InvalidArgument("memory size must be a multiple of host page size")', docs/lucet-runtime-example/src/main.rs:96:91
```

**解決策💡**
同上
