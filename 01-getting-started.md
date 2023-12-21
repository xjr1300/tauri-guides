# はじめる

## 事前準備

省略

## クイックスタート

### HTML、CSS及びJavaScript

このガイドは単にHTML、CSS及びJavaScriptを使用して最初のTauriアプリを作成することを一通り説明します。
Web開発を新しく始めた人にとって、おそらくこれは始めるために最も良い場所です。

> ℹ️ INFO
> 続ける前に、開発作業環境を得るために[前提条件](https://tauri.app/v1/guides/getting-started/prerequisites)が完了していることを確認してください。

Tauriは任意のフロントエンドフレームワークとRustのコアを使用して、デスクトップアプリケーションを構築するフレームワークです。
それぞれのアプリは2つの部分で構成されます。

1. ウィンドウを作成してそれらのウィンドウにRustの(native)関数を公開するRustバイナリ
2. ウィンドウ内にユーザーインターフェースを生成する選択したフロントエンド

次において、フロントエンドの最初の足場をつくり、Rustプロジェクトを準備して、最後に2つの間で会話する方法を説明します。

> 💡 `create-tauri-app`
> 新しいプロジェクトの足場を作成する最も簡単な方法は、`create-tauri-app`ユーティリティです。
> それはバニラのHTML/CSS/JavaScriptとReact、SvelteそしてYewのような多くのフロントエンドフレームワークのために、こだわりを持ったテンプレートを提供します。
>
> ```sh
> # Cargo
> cargo install create-tauri-app --locked
> cargo create-tauri-app
> ```
>
> もし`create-tauri-app`を使用した場合、このガイドの残りに従う必要がないことに注意してください。しかし、セットアップを理解するためにそれを読むことを推奨します。

これは、構築したもののプレビューです。

![Welcome from Tauri](https://tauri.app/assets/images/html-css-js-light-66f279206592cdc7b5aaaee9bc495166.png#gh-light-mode-only)

#### フロントエンドの作成

HTMLファイルを使用して非常に最小限のUIを作成します。
ただし、綺麗にものを維持するために、そのために分離したフォルダを作成しましょう。

```sh
mkdir ui
```

次に、そのフォルダ内に、次の内容を持つ`index.html`ファイルを作成します。

```html
<!-- ui/index.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <h1>Welcome from Tauri!</h1>
  </body>
</html>
```

このガイドではUIを最小に保ちますが、多くのコンテンツで遊んだり、CSSを介してスタイリングを追加することは自由です。

#### Rustプロジェクトの作成

すべてのTauriアプリの心臓部はウィンドウ、`webview`を管理するRustバイナリで、`tauri`と呼ばれるRustクレートを経由してオペレーティングシステムから呼び出されます。
このプロジェクトは、公式パッケージマネージャとRustの汎用ビルドツールである[Cargo](https://doc.rust-lang.org/cargo/)によって管理されます。

Tauri CLIは内部でCargoを使用しているため、それと直接対話する必要があることは稀です。
CargoはTauriの(our)CLIを介して公開していないテスト、リンティングそしてフォーマットのような、より多くの便利な機能を持っているため、詳細はそれらの[公式ドキュメント](https://doc.rust-lang.org/cargo/commands/index.html)を参照してください。

> ℹ️ Tauri CLIのインストール
>
> もし、まだTauri CLIをインストールしていない場合、次のコマンドの1つでそれができます。
> どれを使用するか確かめることができませんか？
> [FAQエントリ](https://tauri.app/v1/guides/faq#node-or-cargo)を確認してください。
>
> ```sh
> # Cargo
> cargo install tauri-cli
> ```

Tauriを使用するために事前準備された最小のRustプロジェクトの足場を作成するために、ターミナルを開いて次のコマンドを実行してください。

```sh
# Cargo
cargo tauri init
```

それは、一連の質問を経由して一通りのことを行います。

1. *What is your app name?*
   これは最終的なバンドルそしてOSがアプリを呼び出す名前です。
   ここでは希望する任意の名前を使用できます。
2. *What should the window title be?*
   これはデフォルトのメインフィン堂のタイトルになります。
   ここでは希望する任意のタイトルを使用できます。
3. *Where are your web assets (HTML/CSS/JS) located relative to the `<current dir>/src-tauri/tauri.conf.json` file that will be created?*
   これはTauriが**プロダクション**用にバンドルするフロントエンドのアセットをロードするパスです。
   この値に`../ui`を使用してください。
4. *What is the URL of your dev server?*
   これはTauriが**開発**中に（アセットを）ロードするURLまたはファイルパスのどちらかを使用できます。
   この値に`../ui`を使用してください。
5. *What is your frontend dev command?*
   これはフロントエンドの開発サーバーを開始するために使用されるコマンドです。
   コンパイルする必要のない間、これをブランクのままにできます。
6. *What is your frontend build command?*
   これはフロントエンドファイルをビルドするコマンドです。
   コンパイルする必要のない間、これをブランクのままにできます。

> ℹ️ INFO
>
> もし、Rustに慣れている場合、`tauri init`が`cargo init`のように見えたり機能したりすることに気づくでしょう。
> もし、完全に手動で準備したい場合、単に`cargo init`を使用した後、必要なTauriの依存関係を追加できます。

`tauri init`コマンドは、`src-tauri`と呼ばれるフォルダを生成します。
それは、そのフォルダの中にコアまたは関連するすべてのファイルを配置する、Tauriアプリにとって便利です。
このフォルダ内の内容を簡単に確認しましょう。

- `Cargo.toml`
  Cargoのマニフェストファイルです。
  アプリが依存するRustのクレート、アプリのメタデータなどを宣言できます。
  完全なリファレンスは[Cargoのマニフェストフォーマット](https://doc.rust-lang.org/cargo/reference/manifest.html)を参照してください。
- `tauri.conf.json`
  このファイルはアプリの名前から許可されるAPIのリストまで、Tauriアプリケーションの様々な要素を構成またはカスタマイズできます。
  サポートされているオプションの完全なリストとそれぞれの詳細な説明は[TauriのAPI構成](https://tauri.app/v1/api/config)を参照してください。
- `src/main.rs`
  これはRustプログラムのエントリポイントで、Tauriを起動する場所です。
  その中に2つのセクションを見つけるでしょう。

  ```rust
  /// src/main.rs
  #![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

  fn main() {
  tauri::Builder::default()
     .run(tauri::generate_context!())
     .expect("error while running tauri application");
  }
  ```

  [cfg!マクロ](https://doc.rust-lang.org/rust-by-example/attribute/cfg.html)を使用した開始行は単に1つの目的を提供します: バンドルされたアプリを起動した場合、それはWindowsにおいて通常ポップアップされるコマンドプロンプトウィンドウを無効にします。

  `main`関数はエントリポイントで、プログラムが起動した時に呼び出される最初の関数です。

- `icons`
  アプリにおしゃれなアイコンが必要な可能性があります。
  すぐに作業を開始できるように、デフォルトのアイコンのセットが含まれています。
  アプリケーションを公開する前に、これらを切り替える必要があります。
  さまざまなアイコン形式の詳細については、Tauriの[アイコン機能ガイド](https://tauri.app/v1/guides/features/icons)を参照してください。

以上です！
これでアプリの開発ビルドを開始するために、ターミナルで次のコマンドを実行できます。

```sh
# Cargo
cargo tauri dev
```

アプリケーションのプレビューです。

![アプリケーションプレビュー](https://tauri.app/assets/images/html-css-js-dev-light-4c95c978b120ed30080617ee92e08357.png#gh-light-mode-only)

#### コマンドの呼び出し

Tauriはネイティブな能力でフロントエンドを強化します。
これらを[コマンド](https://tauri.app/v1/guides/features/command)と読んでいて、フロントエンドのJavaScriptから呼び出すことができる本質的にRustの関数です。
これは、はるかにパフォーマンスの高いRustコードで、重たいプロセスまたはOSの呼び出しを処理することを可能にします。

単純な例を作成しましょう。

```rust
/// src-tauri/src/main.rs
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

コマンドは、JavaScriptコンテキストで関数と会話させる`#[tauri::command]`属性マクロが追加された単にRustの普通の関数のようです。

最後に、Tauriがそれに応じて呼び出しをルーティングできるように、新しく作成したコマンドについてTauriに伝える必要もあります。
これは、次に見ることができる`invoke_handler()`関数と`generate_handler!()`マクロのコンビネーションで行われます:

```rust
/// src-tauri/src/main.rs
fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

これでフロントエンドからコマンドを呼び出す準備ができました。

通常、ここでは[@tauri-apps/api](https://tauri.app/v1/api/js/)パッケージを推奨しますが、このガイドでバンドラーを使用しないため、`tauri.conf.json`ファイルで[withGlobalTauri](https://tauri.app/v1/api/config#buildconfig.withglobaltauri)を有効にしてください。

```json
// src-tauri/tauri.config.json
{
    "build": {
        "beforeBuildCommand": "",
        "beforeDevCommand": "",
        "devPath": "../ui",
        "distDir": "../ui",
        "withGlobalTauri": true
  },
}
```

これはフロントエンドに事前にバンドルされたバージョンのAPI関数を挿入します。

これでコマンドを呼び出すために`index.html`を変更できます。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <h1 id="header">Welcome from Tauri!</h1>
    <script>
      // 事前バンドルされたグローバルAPI関数にアクセス
      const { invoke } = window.__TAURI__.tauri

      // これでコマンドをよびだせます!
      // 「Welcome from Tauri」が「Hello, World!」に置き換わっていることを確認
      invoke('greet', { name: 'World' })
        // `invoke` returns a Promise
        .then((response) => {
          window.header.innerHTML = response
        })
    </script>
  </body>
</html>
```

> 💡 TIP
>
> もしRustとJavaScriptの会話について詳細を知りたいのであれば、Tauriの[内部プロセスコミュニケーション](https://tauri.app/v1/references/architecture/inter-process-communication/)を読んでください。
