# デバッグ

## アプリケーションのデバッグ

Tauriのすべての可動部品を使用して、デバッグを必要とする問題に遭遇するかもしれません。
エラーの詳細が出力される場所が多くあり、そしてTauriにはより簡単にデバッグ処理するいくつかのツールがあります。

### Rustコンソール

エラーを確認する最初の場所はRustコンソール内です。
これは例えば`cargo tauri dev`を実行したターミナル内です。
Rustファイル内からそのコンソールに何かを出力するために、`次のコードを使用できます。

```rust
println!("Message from Rust: {}", msg);
```

よくRustコード内にエラーがあることがあり、そしてRustコンパイラから多くの情報を得られることがあります。
例として、もし`cargo tauri dev`がクラッシュした場合、LinuxまたはmacOS上で次のようにそれを再起動できます。

```sh
RUST_BACKTRACE=1 cargo tauri dev
```

また、Windowsでは次のようにできます。

```dos
set RUST_BACKTRACE=1
cargo tauri dev
```

このコマンドは詳細なスタックトレースを与えてくれます。
一般的には、Rustコンパイラは次のように問題について詳細な情報を与えることによって助けます:

```text
error[E0425]: cannot find value `sun` in this scope
  --> src/main.rs:11:5
   |
11 |     sun += i.to_string().parse::<u64>().unwrap();
   |     ^^^ help: a local variable with a similar name exists: `sum`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0425`.
```

### WebViewコンソール

WebView内を右クリックして、`Inspect Element`を選択します。
これはあなたに使用されたことのあるChromeまたはFirefox開発ツールと同様なWebインスペクターを開きます。
インスペクターを表示するために、LinuxとWindowsでは`Ctrl + Shift + i`、macOS上では`Command + Option + i`ショートカットも使用できます。

#### プログラムで開発ツールを開く

インスペクターウィンドウの可視性を[Window::open_devtools](https://docs.rs/tauri/1/tauri/window/struct.Window.html#method.open_devtools)と[Window::close_devtools](https://docs.rs/tauri/1/tauri/window/struct.Window.html#method.close_devtools)関数を使用することで制御できます:

```rust
use tauri::Manager;
tauri::Builder::default()
    .setup(|app| {
        #[cfg(debug_assertions)]    // デバッグビルドでのみこのコード(次のブロック)を含む
        {
            let window = app.get_window("main").unwrap();
            window.open_devtools();
            window.close_devtools();
        }
        Ok(())
    });
```

#### プロダクションでインスペクターを使用する

デフォルトで、インスベクターはCargoの機能で有効にしない限り、開発とデバッグビルドでのみ有効にされています。

#### デバッグビルドの作成

デバッグビルドを作成するために、`tauri build --debug`コマンドを実行します。

```sh
cargo tauri build --debug
```

通常のビルドと開発プロセスと同様に、最初にこのコマンドを実行したときは時間がかかりますが、後続の↑項ではとても高速です。
最終的にバンドルされたアプリは開発コンソールが有効化されて、`src-tauri/target/debug/bundle`に配置されます。

ターミナルからアプリをビルドすることもでき、Rustコンパイラは指摘（エラーの場合）または`println`メッセージを与えます。
`src-tauri/target/(release|debug)/[app name]`ファイルを探して、コンソールでそれを直接実行するか、ファイルシステム内でしの実行形式自信をダブルクリックすることで実行できます（注意: この方法においてエラーが発生するとコンソールが閉じます）。

##### 開発ツール機能の有効化

> 🔥 危険
>
> macOSにおいて、開発ツールAPIはプライベートです。
> macOSでプライベートAPIを使用することは、アプリケーションがApp Storeに受け入れられることを妨げます。

プロダクションビルドで開発ツールを有効にするために、`src-tauri/Cargo.toml`ファイルのCargo機能の`devtools`を有効にする必要があります。

```toml
[dependencies]
tauri = { version = "...", features = ["...", "devtools"] }
```

### コアプロセスのデバッグ

コアプロセスはRustによって強化されているため、それをデバッグするためにGDBまたはLLDBを使用できます。
TauriアプリケーションのコアプロセスをデバッグするためにLLDB VS Code Extensionを使用する方法を[VS Codeにおけるデバッグ](https://tauri.app/v1/guides/debugging/vs-code)ガイドに従うことができます。

## CLionにおけるデバッグ

このガイドにおいて、[Tauriアプリのコアプロセス](https://tauri.app/v1/references/architecture/process-model#the-core-process)をデバッグするために、IntelliJ CLionを準備します。

### 事前準備

Rustの機能を有効にするために、[IntelliJ Rust Plugin](https://plugins.jetbrains.com/plugin/8182-rust/docs)をインストールする必要があります。

### トップレベルのCargoワークスペース

デフォルトで、Tauriは`src-tauri`と呼ばれるサブディレクトリ内に配置されます。
CLionはトップレベルでないCargoプロジェクトを認識できないかもしれないため、その場合`Cargo -> Attach Cargo project`を使用することで、それに接続できるはずです。
もし、この機能が機能しない場合、メインのCargo.tomlを示すトップレベルワークスペースを作成する必要があります。

```toml
# Cargo.toml
[workspace]
members = ["src-tauri"]
```

進める前に、プロジェクトが完全にロードされていることを確認してください。
インデックス化が完了していて、Cargoツールウィンドウが全てのモジュールと目的のワークスペースを表示している場合、進めることができます。

### 実行／デバッグ構成

デバッグモードでTauriアプリを起動するために使用できる起動／デバッグ構成を準備します。
構成を作成するために、`Edit Configurations`に進み、`+`ボタンをクリックして、`Cargo`コマンドを選択します。

![Cargoコマンドの選択](https://tauri.app/assets/images/add-cargo-config-light-b1e6a7707f0058d915c80a5dadeb1743.png#gh-light-mode-only)

それを作成したら、CLionを構成して、デフォルトの機能を使用せずにアプリを構築するために、Cargoに指示する必要があります。
これは、ディスクからアセットを読み込む代わりに、開発サーバーを使用することをTauriに伝えます。
通常、このフラグはTauri　CLIによって渡されますが、ここでは完全に回避するため、手動でフラグを渡す必要があります。

![手動でフラグを渡す](https://tauri.app/assets/images/set-no-default-features-light-33d99ee27279541c27586bcc51cb0f8d.png#gh-light-mode-only)

これで、実行／デバッグ構成の名前を覚えやすい名前に変更でき、この例ではそれを「Tauriアプリの実行」と呼びますが、それに任意の名前をつけることができます。

![実行／デバッグ構成のリネーム](https://tauri.app/assets/images/rename-configuration-light-5740a3ead5c56967d74a1b7272b4687c.png#gh-light-mode-only)

> ⚠️ 注意
>
> Windowsにおいて、CLionが正しいデバッガーのツールチェインを使用していることも確認してください。
> これをするためには、設定(`File -> Settings...`)を開いて、`Build, Execution, Deployment -> Toolchains`を選択して、`Visual Studio`ツールチェインを一番上に移動します。

### 開発サーバーの起動

上記の構成は直接RustアプリケーションをビルドするためにCargoを使用して、それにデバッガーを接続します。
これは、完全にTauri CLIを回避するため、`beforeDevCommand`や`beforeBuildCommand`のような機能は実行されません。
新しいターミナルを開いて開発サーバーの実行を開始することでそれを処理する必要があります:

```sh
pnpm vite dev
```

> 現在、CLionはVS Codeにあるような実行／デバッグ構成でのバックグラウンドタスクをサポートしていないため、現在のところ開発サーバーを手動で実行する必要があります。

### デバッグセッションの起動

開発サーバーの起動と選択された起動／デバッグ構成の切り替えで、`Debug`をクリックすることで新しいデバッグセッションを起動できます。
CLionはプロジェクト内の任意のファイルに配置されたブレークポイントを自動的に認識します。

## VS Codeにおけるデバッグ

このガイドは、[Tauriアプリのコアプロセス](https://tauri.app/v1/references/architecture/process-model#the-core-process)をデバッグするために、VS Codeを準備する方法を一通り説明します。

### 事前準備

[vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)拡張機能をインストールしてください。

### launch.jsonの構成

`.vscode/launch.json`ファイルを作成してそれに次のJSONコンテンツを貼り付けてください。

```json
// .vscode/launch.json
{
  // (利用する)可能性(possible)のある属性について学ぶためにIntelliSenseを利用します。
  // 存在する属性の説明を表示するためにホバーしてください。
  // 詳細は、https://go.microsoft.com/fwlink/?linkid=830387 を参照してください。
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "Tauri Development Debug",
      "cargo": {
        "args": [
          "build",
          "--manifest-path=./src-tauri/Cargo.toml",
          "--no-default-features"
        ]
      },
      // `beforeDevCommand`を使用するタスク(もし使用するなら)は、`.vscode/tasks.json`に設定する必要があります。
      "preLaunchTask": "ui:dev"
    },
    {
      "type": "lldb",
      "request": "launch",
      "name": "Tauri Production Debug",
      "cargo": {
        "args": ["build", "--release", "--manifest-path=./src-tauri/Cargo.toml"]
      },
      // `beforeDevCommand`を使用するタスク(もし使用するなら)は、`.vscode/tasks.json`に設定する必要があります。
      "preLaunchTask": "ui:build"
    }
  ]
}
```

これは、開発そしてプロダクションモードの両方で、Rustアプリケーションをビルドして、それをロードするために`cargo`を直接使用します。

それはTauri CLIを使用しないため、排他的なCLI機能は実行されないことに注意してください。
`beforeDevCommand`や`beforeBuildCommand`スクリプトは、事前に実行するか、`preLaunchTask`フィールド内のタスクとして設定して、実行されなければなりません。
その2つのタスクを持つ`.vscode/launch.json`ファイルの例を次に示します。
1つは開発サーバーを起動する`beforeDevCommand`で、もう一方は`beforeBuildCommand`です:

```json
// .vscode/launch.json
{
  // task.jsonフォーマットについてのドキュメントはhttps://go.microsoft.com/fwlink/?LinkId=733558を参照してください。
  "version": "2.0.0",
  "tasks": [
    {
      "label": "ui:dev",
      "type": "shell",
      // `dev`はバックグラウンドで実行を続けます。
      // 理想的には、`problemMatcher`を設定するべきです。
      // https://code.visualstudio.com/docs/editor/tasks#_can-a-background-task-be-used-as-a-prelaunchtask-in-launchjsonを参照してください。
      "isBackground": true,
      // これを`beforeDevCommand`に変更してください:
      "command": "yarn",
      "args": ["dev"]
    },
    {
      "label": "ui:build",
      "type": "shell",
      // これを`beforeBuildCommand`に変更してください:
      "command": "yarn",
      "args": ["build"]
    }
  ]
}
```

これで、`src-tauri/src/main.rs`または任意の他のRustファイルにブレイクポイントを設定でき、`F5`を押すことでデバッグが開始します。
