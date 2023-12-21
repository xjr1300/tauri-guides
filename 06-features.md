# 機能

## 自身のCLIを作成

Tauriは、強固なコマンドライン引数パーサーである[clap](https://github.com/clap-rs/clap)を経由してアプリがCLIを持つようにできます。
`tauri.conf.json`ファイル内の単純なCLI定義を使用して、インターフェイスを定義して、JavaScriptとRustでマッチする引数を読み込むことができます。

### 基本的な構成

`tauri.conf.json`の配下で、インターフェイスを構成するために次の構成を持てます。

```json
{
    "tauri": {
        "cli": {
            "description": "", // ヘルプで表示されるコマンドの説明
            "longDescription": "", // ヘルプで表示されるコマンドの長い説明
            "beforeHelp": "", // ヘルプテキストの前に表示するコンテンツ
            "afterHelp": "", // ヘルプテキストの後に表示するコンテンツ
            "args": [], // コマンドの引数のリストで後で説明
            "subcommands": {
                "subcommand-name": {
                    // `./app subcommand-name --arg1 --arg2 --etc`で受け入れ可能なサブコマンドを構成
                    // configuration as above, with "description", "args", etc.
                }
            }
        }
    }
}
```

> ℹ️ メモ
>
> ここのJSONの構成は単なるサンプルで、明確にするために多くの他のフィールドは省略されています。

### 引数の追加

`args`配列は、そのコマンドが受け入れることができる引数のリストまたはサブコマンドを表現します。
それらを構成する方法の詳細については[ここ](https://tauri.app/v1/api/config#tauri)で見つけることができます。

#### 位置引数

位置引数は、引数のリスト内のそれ自身の位置によって識別されます。

```json
{
    "args": [
        {
            "name": "source",
            "index": 1,
            "takesValue": true
        },
        {
            "name": "destination",
            "index": 2,
            "takesValue": true
        }
    ]
}
```

ユーザーは`./app tauri.txt dest.txt`のようにアプリを実行でき、引数の一致は"tauri.txt"を`source`に、"dest.txt"を`destination`にマップします。

#### 名前付き引数

名前付き引数は、(キー、値)のペアで、キーは値を識別します。

```json
{
    "args": [
        {
            "name": "type",
            "short": "t",
            "takesValue": true,
            "multiple": true,
            "possibleValues": ["foo", "bar"]
        }
    ]
}
```

ユーザーは、`./app --type foo bar`、`./app -t foo -t bar`または`./app --type=foo,bar`でアプリを起動でき、引数の一致は`["foo", "bar"]`を`type`にマップします。

#### フラグ引数

フラグ引数は単独のキーで、存在するか存在しないかをアプリケーションに提供します。

```json
{
    "args": [
        {
            "name": "verbose",
            "short": "v",
            "multipleOccurrences": true
        }
    ]
}
```

ユーザーは、`./app -v -v -v`、`./app --verbose --verbose --verbose`または`./app -vvv`でアプリを起動でき、引数の一致は`true`として`verbose`、`occurrences = 3`でマップします。

#### サブコマンド

いくつかのCLIアプリケーションはサブコマンドとして追加のインターフェイスを持っています。
例えば、`git`CLIは`git branch`、`git commit`そして`git push`があります。
`subcommands`配列を使用して、追加のネストされたインターフェイスを定義できます。

```json
{
    "cli": {
        ...
        "subcommands": {
            "branch": {
                "args": []
            },
            "push": {
                "args": []
            }
        }
    }
}
```

その構成は、ルートアプリケーションの設定と同じで、`description`、`longDescription`、`args`などを持ちます。

### マッチの読み込み

#### Rust

```rust
fn main() {
    tauri::Builder::default()
        .setup(|app| {
            match app.get_cli_matches() {
                // ここの`matches`は{ args, subcommand }を持つ構造体です。
                // `args`は`HashMap<String, ArgData>`で、`ArgData`は{ value, occurrences }を持つ構造体です。
                // `subcommand`は`Option<Box<SubcommandMatches>>`で、`SubcommandMatches`は{ name, matches }を持つ構造体です。
                Ok(matches) => {
                  println!("{:?}", matches)
                }
                Err(_) => {}
            }
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

#### JavaScript

```js
import { getMatches } from '@tauri-apps/api/cli'

getMatches().then((matches) => {
    // { args, subcommand } `matches`を使用して何かします。
})
```

### 完全なドキュメント

CLI構成について詳細なドキュメントは[ここ](https://tauri.app/v1/api/config#tauri)にあります。

## フロントエンドからのRustの呼び出し

Tauriは、WebアプリからRust関数を呼び出すための、単純ですが強力な`コマンド`システムを提供しています。
コマンドは引数を受け取り、値を返すことができます。
それらはエラーを返すことや、`async`であることも可能です。

### 基本的な例

コマンドは`src-tauri/src/main.rs`ファイルに定義されます。
コマンドを作成するために、単に関数を追加して、それを`#[tauri::command]`で注釈します。

```rust
#[tauri::command]
fn my_custom_command() {
    println!("I was invoked from JS!");
}
```

次のようにビルダー関数にコマンドのリストを提供する必要があります:

```rust
// src-tauri/src/main.rs
fn main() {
    tauri::Builder::default()
        // これはコマンドを渡す場所です。
        .invoke_handler(tauri::generate_handler![my_custom_command])
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

これで、JSコードからコマンドを呼び出すことができます。

```js
// Tauri API npmパッケージを使用するとき
import { invoke } from '@tauri-apps/api/tauri'
// Tauriグローバルスクリプトを使用するとき（もしnpmパッケージを使用しない場合）
// `tauri.conf.json`内の`build.withGlobalTauri`がtrueに設定されていることを確認
const invoke = window.__TAURI__.invoke

// コマンドの呼び出し
invoke('my_custom_command')
```

### 引数のパース

コマンドハンドラは引数を受け取ることができます。

```rust
#[tauri::command]
fn my_custom_command(invoke_message: String) {
    println!("I was invoked from JS, with this message: {}", invoke_message);
}
```

引数はキャメルケースのキーを持ったJSONオブジェクトとしてパースされるべきです。

```js
invoke('my_custom_command', { invokeMessage: 'Hello!' })
```

引数は、それらが[serde::Deserialize](https://docs.serde.rs/serde/trait.Deserialize.html)を実装している限り、任意の型になれます。

Rustでスネークケースを使用して引数を定義したとき、JavaScriptのために引数はキャメルケースに変換されます。
JavaScriptにおいてスネークケースを使用するために、`tauri::command`文の中でそれを定義する必要があります。

```rust
#[tauri::command(rename_all = "snake_case")]
fn mu_custom_command(invoke_message: String) {
    println!("I was invoked from JS with this message: {}", invoke_message);
}
```

対応するJavaScriptのコードは次です。

```js
invoke('my_custom_command', { invoke_message: 'Hello!' })
```

### データを返す

コマンドハンドラは同様に値を返すことができます。

```rust
#[tauri::command]
fn my_custom_command() -> String {
    "Hello from Rust!".into()
}
```

`invoke`関数は返された値に解決する`Promise`を返します。

```js
invoke('my_custom_command').then((message) => console.log(message))
```

返されたデータは、それが[serde::Serialize](https://docs.serde.rs/serde/trait.Serialize.html)を実装している限り、任意の型になれます。

### エラー処理

もし、ハンドラが失敗する可能性があり、エラーを返すことができるようにする必要がある場合、`Result`を返す関数を使用します。

```rust
#[tauri::command]
fn my_custom_command() -> Result<String, String> {
    // 何かが失敗した場合
    Err("This failed!".into())
    // 機能した場合
    Ok("This worked!".into())
}
```

もし、コマンドがエラーを返した場合、`Promise`が`reject`して、そうでない場合は`resolve`します。

```js
invoke('my_custom_command')
    .then((message) => console.log(message))
    .catch((error) => console.error(error))
```

上記で言及したように、エラーを含むコマンドから返されるものはすべて`serde::Serialize`を実装しなければなりません。
Rust標準ライブラリまたは外部クレートから導入したほとんどのエラー型はそれを実装していないため、もしそれらのエラー型で機能させる場合、これは問題になる可能性があります。
単純なシナリオにおいて、これらのエラーを`String`に変換するために`map_err`を使用できます。

```rust
#[tauri::command]
fn my_custom_command() -> Result<(), String> {
    // これはエラーを返す
    std::fs::File::open("path/that/does/not/exist").map_err(|err| err.to_string())?;
    // 成功した場合は何も返さない
    Ok(())
}
```

これは理想的ではないため、`serde::Serialize`を実装する独自のエラー型を作成したいと望むかもしれません。
次の例において、エラー型を作成する支援をするために[thiserror](https://github.com/dtolnay/thiserror)クレートを使用します。
それは、`thiserror::Error`トレイトを導出することで、列挙型をエラー型に変換します。
詳細については(thiserrorの)ドキュメントを参照してください。

```rust
// プログラム内で表現されるすべてのエラーを表現するエラー型を作成
#[derive(Debug, thiserror::Error)]
enum Error {
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

// serde::Serializeを手動で実装する必要がある
impl serde::Serialize for Error {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        serializer.serialize_str(self.to_string().as_ref())
    }
}

#[tauri::command]
fn my_custom_command() -> Result<(), Error> {
    // これはエラーを返す
    std::fs::File::open("path/that/does/not/exist")?;
    // 成功した場合は何も返さない
    Ok(())
}
```

カスタムエラー型は発生しうるすべてのエラー型を明示的に作成できる利点があるため、読み手は何のエラーが発生する可能性があるかすぐに識別できます。
これは、後でコードをレビューしたりリファクタリングするときに、作成者以外の人にとって多くの時間を節約します。
また、それはエラー型がシリアライズされる方法を完全に制御する方法を与えます。
上記例において、単に文字列でエラーメッセージを返しましたが、Cに似たコードでそれぞれのエラーを割り当てることができるため、この方法は、例えば見た目が似ているTypeScriptのエラー列挙型にそれをより簡単にマップできる可能性があります。

### 非同期コマンド

Tauriにおいて、非同期関数は、UIのフリーズや速度低下を引き起こさない方法で重たい作業を実行することに役に立ちます。

> ℹ️ メモ
>
> 非同期コマンドは`async_runtime::spawn`を使用して別スレッドで実行されます。
> *async*キーワードのないコマンドは、*#[tauri::command(async)]*で定義されていない限り、メインスレッドで実行されます。

***もし、コマンドを非同期で実行する必要がある場合、単にそれを`async`で宣言してください。***

> ⚠️ 注意
>
> Tauriを使用して、非同期関数を作成するとき注意する必要があります。
> 現在、非同期関数のシグネチャ内に借用した引数を単に含めることができません。
> このような型の一般的な例として、`&str`または`State<'_, Data>`があります。
> この制限はここで追跡されています: <https://github.com/tauri-apps/tauri/issues/2533>そして、回避策を下に示します。

借用された型を機能させるとき、追加で変更する必要があります。
主要な2つの選択したあります:

#### 選択肢1

`&str`のような型から、`String`のような借用されない似たような型に変換します。
これは、例えば`State<'_, Data>`のように、すべての型で機能しません。

```rust
// &strは借用されているためサポートされていないため、&strの代わりにStringを使用した非同期関数を宣言
#[tauri::command]
async fn my_custom_command(value: String) -> String {
    // 他の非同期関数を呼び出し、それが終了するまで待機
    some_async_function().await;
    format!(value)
}
```

#### 選択肢2

`Result`で返す型をラップします。
これは実装することがすこす難しくなりますが、すべての型で機能します。

戻り値の型`Result<a, b>`を使用し、`a`を返したい型に置き換えるか、何も返さない場合は`()`を使用し、`b`を何か問題が発生した場合に返すエラー型に置き換え、またはオプションのエラーが返されないように`()`を返します。

* `Result<String, ()>`は、`String`を返して、そしてエラーはありません。
* `Result<(), ()>`は、何も返しません。
* `Result<bool, Error>`は、`bool`、または上記[エラー処理](https://tauri.app/v1/guides/features/command#error-handling)セクションで示したエラーを返します。

```rust
// 借用の問題を回避するためにResult<String, ()>を返す
#[tauri::command]
async fn my_custom_command(value: &str) -> Result<String, ()> {
    // 他の非同期関数を呼び出し、それが終了するまで待機
    some_async_function().await;
    // 戻り値が`Ok()`内にラップされなければならないことに注意してください
    Ok(format!(value))
}
```

#### JSからの呼び出し

JavaScriptからのコマンド呼び出しは`Promise`を返すため、ちょうどそれは任意の他コマンドのように機能します。

```js
invoke('my_custom_command', { value: 'Hello, Async!' }).then(() =>x
    console.log('Completed')
)
```

### コマンド内でWindowにアクセスする

コマンドはメッセージを呼び出した`Window`インスタンスにアクセスできます:

```rust
#[tauri::command]
async fn my_custom_command(window: tauri::Window) {
    println!("Window: {}", window.label());
}
```

### コマンド内でアプリケーションハンドルにアクセスする

コマンドは`AppHandle`インスタンスにアクセスできます:

```rust
#[tauri::command]
async fn my_custom_command(app_handle: tauri::AppHandle) {
    let app_dir = app_handle.path_resolver().app_dir();
    use tauri::GlobalShortcutManager;
    app_handle.global_shortcut_manager().register("CTRL + U", move || {});
}
```

### 管理された状態にアクセスする

Tauriは`tauri::Builder`で`manage`関数を使用することで、状態を管理できます。
状態は、コマンドで`tauri::State`を使用してアクセスされます。

```rust
struct MyState(String);

#[tauri::command]
fn my_custom_command(state: tauri::State<MyState>) {
    assert!(state.0 == "some state value");
}

fn main() {
    tauri::Builder::default() .manage(MyState("some state value".into()))
        .invoke_handler(tauri::generate_handler![my_custom_command])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 複数のコマンドを作成する

`tauri::generate_handler!`マクロは、コマンドの配列を受け取ります。
複数のコマンドを登録するために、複数回`invoke_handler`を呼び出すことはできません。
最後の呼び出しだけが有効になります。
1回の`tauri::generate_handler!`の呼び出しにそれぞれのコマンドを渡さなければなりません。

```rust
#[tauri::command]
fn cmd_a() -> String {
    "Command a".into()
}

#[tauri::command]
fn cmd_b() -> String {
    "Command b".into()
}

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![cmd_a, cmd_b])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 完全な例

上記機能の任意またはすべては組み合わせできます:

```rust
struct Database;

#[derive(serde::Serialize)]
struct CustomResponse {
    message: String,
    other_val: usize,
}

async fn some_other_function() -> Option<String> {
    Some("response".into())
}

#[tauri::command]
async fn my_custom_command(
    window: tauri::Window,
    number: usize,
    database: tauri::State<'_, Database>,
) -> Result<CustomResponse, String> {
    println!("Called from {}", window.label());
    let result: Option<String> = some_other_function().await;
    if let Some(message) = result {
        Ok(CustomResponse {
            message,
            other_val: 42 + number,
        })
    } else {
        Err("No result".into())
    }
}

fn main() {
    tauri::Builder::default()
        .manage(Database {})
        .invoke_handler(tauri::generate_handler![my_custom_command])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```js
// JSから呼び出し
invoke('my_custom_command', {
    number: 42,
}).then((res) => {
    console.log(`Message: ${res.message}, Other Val: ${res.other_val}`)
}).catch((err) => {
    console.error(err)
})
```

## イベント

Tauriイベントシステムは、フロントエンドとバックエンドの間でのメッセージの受け渡しを可能にする、マルチプロデューサーとマルチコンシューマーの通信プリミティブです。
それはコマンドシステムと似ていますが、ペイロードの型チェックはイベントハンドラに記述されなければならず、それはバックエンドからフロントエンドへの通信を単純にして、チャネルのように機能します。

Tauriアプリケーションは、グローバルと特定のウィンドウのイベントを受け取り(listen)、または発生(emit)できます。
フロントエンドとバックエンドからの利用は、次で説明します。

### フロントエンド

フロントエンドで、`@tauri-apis/api`パッケージの`event`と`window`モジュールを使用して、フロントエンドでイベントシステムはアクセスできます。

#### グローバルイベント

グローバルイベントチャネルを使用するために、`event`モジュールをインポートして、`emit`そして`listen`関数を使用します。

```js
import { emit, listen } from '@tauri-apps/api/event'

// `click`イベントをリッスンして、イベントリスナーから削除するために関数を取得
// イベントを購読誌絵t、最初のイベントで自動的にリスナーの購読を解除する`once`関数もあります。
const unlisten = await listen('click', (event) => {
    // event.eventはイベント名です（複数のイベントの型にひとつのコールバックを使用する場合に便利）。
    // event.payloadはペイロードオブジェクトです。
})

// オブジェクトのペイロードをつけて`click`イベントを発行
emit('click', {
    theMessage: 'Tauri is awesome!',
})
```

#### 特定ウィンドウイベント

特定のウィンドウのイベんtのは`window`モジュールで公開されます。

```js
import { appWindow, WebviewWindow } from '@tauri-apps/api/window'

// 現在のウィンドウのみ見えるイベントを発行
appWindow.emit('click', { message: 'Tauri is awesome!' })

// 新しいWebViewウィンドウを作成して、そのウィンドウのみにイベントを発行
const webView = new WebviewWindow('window')
webview.emit('click')
```

### バックエンド

バックエンドで、グローバルイベントチャンネルは`App`構造体によって公開され、特定のウィンドウイベントは`Window`トレイとを使用して発行されます。

#### グローバルイベント

```rust
use tauri::Manager;

// ペイロード型は`Serialize`と`Clone`を実装しなければならない。
#[derive(Clone, serde::Serialize)]
struct Payload {
    message: String,
}

fn main() {
    tauri::Builder::default()
        .setup(|app| {
            // `event-name`をリッスン（任意のウィンドウに発行された）
            let id = app.listen_global("event-name", |event| {
                println!("got event-name with payload {:?}", event.payload());
            });
            // `listen_global`関数の戻り値である`id`を使用して、イベントのリッスンを解除
            // `once_global`APIは`App`構造体で公開される
            app.unlisten(id);

            // フロントエンドのすべてのWebViewウィンドウに`event-name`イベントを発行
            app.emit_all("event-name", Payload { message: "Tauri is awesome!".into() }).unwrap();
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

### 特定ウィンドウのイベント

特定ウィンドウのチャンネルを使用するために、`Window`オブジェクトはコマンドハンドラまたは`get_window`関数で得られる。

```rust
use tauri::{Manager, Window};

// ペイロード型は`Serialize`と`Clone`を実装しなければならない。
#[derive(Clone, serde::Serialize)]
struct Payload {
    message: String,
}

// コマンドのバックグラウンドプロセスを初期化して、コマンドで使用されたウィンドウにのみ定期的にイベントを発行する
#[tauri::command]
fn init_process(window: Window) {
    std::thread::spawn(move || {
        loop {
            window.emit("event-name", Payload { message: "Tauri is awesome!".into() }).unwrap();
        }
    });
}

fn main() {
    tauri::Builder::default()
        .setup(|app| {
            // ここで`main`はウィンドウラベルです; それはウィンドウの作成または`tauri.conf.json`で定義されます。
            // デフォルト値は`main`です。それがユニークであることに注意してください。
            let main_window = app.get_window("main").unwrap();

            // `event-name`をリッスン（`main`ウィンドウに発行された）
            let id = main_window.listen("event-name", |event| {
                println!("got window event-name with payload {:?}", event.payload());
            });
            // `listen`関数で返された`id`を使用して、イベントのリッスンを解除
            main_window.unlisten(id);

            // `main`ウィンドウに`event-name`イベントを発行
            // `once`APIは`Window`構造体で公開される
            main_window.emit("event-name", Payload { message: Task::new("Tauri is awesome!".into()) }).unwrap();
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![init_process])
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

## アイコン

Tauriは、そのロゴに基づいたデフォルトのアイコンセットが同梱します。
これは、アプリケーションを出荷するときに望んでいることではありません。
この一般的な状況を改善するために、Tauriは入力ファイルを受け取り（デフォルトでは`./app-icon.png`）、さまざまなプラットフォームで必要になるすべてのアイコンを作成します。

> ℹ️ ファイルタイプメモ
>
> * `icon.icns` = macOS
> * `icon.ico` = Windows
> * `*.png` = Linus
> * `Square*Logo.png`と`StoreLogo.png` = 現在使用していませんが、AppX/MS Storeをターゲット用に意図しています
>
> アイコンタイプは、上記以外のプラットフォーム (特に`png``) でも使用できることに注意してください。
> したがって、プラットフォームのサブセットのみをビルドする場合でも、すべてのアイコンを含めることをお勧めします。

### コマンドの使用方法

`@tauri-apps/cli`/`tauri-cli`バージョン1.1から、`icon`サブコマンドはメインCLIの一部です。

```sh
cargo tauri icon
```

```text
> cargo tauri icon --help
cargo-tauri-icon 1.1.0

Generates various icons for all major platforms

USAGE:
    cargo tauri icon [OPTIONS] [INPUT]

ARGS:
    <INPUT>    Path to the source icon (png, 1024x1024px with transparency) [default: ./app-icon.png]

OPTIONS:
    -h, --help               Print help information
    -o, --output <OUTPUT>    Output directory. Default: 'icons' directory next to the tauri.conf.json file
    -v, --verbose            Enables verbose logging
    -V, --version            Print version information
```

デフォルトで、アイコンは`src-tauri/icons`フォルダに配置され、そこはビルドされたアプリに自動的に含まれています。
もし、異なる場所からアイコンを取得したい場合、`tauri.conf.json`ファイルのこの部分を編集できます。

```json
{
    "tauri": {
        "bundle": {
            "icon": [
                "icons/32x32.png",
                "icons/128x128.png",
                "icons/128x128@2x.png",
                "icons/icon.icns",
                "icons/icon.ico"
            ]
        }
    }
}
```

### 手動でアイコンを作成する

もし、これらのアイコンを自分で作成したい場合、例えば、小さいサイズに対してよりシンプルなデザインを使用したい場合、またはCLIの内部画像サイズ変更に依存したくない場合は、アイコンがいくつかの要件を満たしていることを確認する必要があります:

* `icon.icns`: [icns](https://en.wikipedia.org/wiki/Apple_Icon_Image_format)ファイルに必要なレイヤサイズと名前は、[Tauriリポジトリ](https://github.com/tauri-apps/tauri/blob/dev/tooling/cli/src/helpers/icns.json)に記載されています。
* `icon.ico`: [ico](https://en.wikipedia.org/wiki/ICO_(file_format))ファイルは、16、24、32、48、64及び256ピクセルのレイヤを含む必要があります。
  開発中のICO画像を最適に表示するには、32pxレイヤを最初のレイヤにする必要があります。
* `png`: pngアイコンの要件は、幅 == 高さ、RGBA (RGB + 透明度)、ピクセルあたり32ビット(チャネルあたり8ビット)です。
  一般に予想されるサイズは32、128、256及び512ピクセルです。
  少なくとも`tauri icon`: `32x32.png`、`128x128.png`、`128x128@2x.png`そして`icon.png`の出力と一致させることをお勧めします。

## ウィンドウメニュー

ネイティブなアプリケーションメニューは、ウィンドウに接続できます。

### メニューの作成

ネイティブなウィンドウメニューを作成するために、`Menu`、`Submenu`、`MenuItem`及び`CustomMenuItem`型をインポートします。
`MenuItem`列挙型はプラットフォーム固有のアイテムのコレクションを含みます(Windows用は現在実装されていません)。
`CustomMenuItem`は、独自のメニューアイテムを作成でき、それらに特別な機能を加えることができます。

```rust
use tauri::{CustomMenuItem, Menu, MenuItem, Submenu};
```

`Menu`インスタンスを作成します。

```rust
// ここで、`"quit".to_string()`は、メニューアイテムのIDを定義して、2番目のパラメーターはメニューアイテムのラベルです。
let quit = CustomMenuItem::new("quit".to_string(), "Quit");
let close = CustomMenuItem::new("close".to_string(), "Close");
let submenu = Submenu::new("File", Menu::new().add_item(quit).add_item(close));
let menu = Menu::new()
    .add_native_item(MenuItem::Copy)
    .add_item(CustomMenuItem::new("Hide", "Hide"))
    .add_submenu(submenu);
```

### すべてのウィンドウにメニューを追加する

定義されたメニューは、`tauri::Builder`構造体の`menu`APIを使用してすべてのウィンドウに設定されます。

```rust
use tauri::{CustomMenuItem, Menu, MenuItem, Submenu};

fn main() {
    let menu = Menu::new();     // メニューを構成
    tauri::Builder::default()
        .menu(menu)
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 特定のウィンドウにメニューを追加する

ウィンドウを作成して、使用されるためにメニューを設定できます。
これは、それぞれのアプリケーションウィンドウごとに特定のメニューセットを定義できます。

```rust
use tauri::{CustomMenuItem, Menu, MenuItem, Submenu};
use tauri::WindowBuilder;

fn main() {
    let menu = Menu::new();     // メニューを構成
    tauri::Builder::default()
        .setup(|app| {
            WindowBuilder::new(
                app,
                "main-window".to_string(),
                tauri::WindowUrl::App("index.html".into()),
            )
            .menu(menu)
            .build()?;
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### カスタムメニューアイテムのイベントのリスニング

それぞれの`CustomMenuItem`はクリックされたときにイベントをトリガーします。
それらを処理するために、グローバルな`tauri::Builder`または特定のウィンドウのどちらかで、`on_menu_event`APIを使用します。

#### グローバルメニューでイベントをリスニング

```rust
use tauri::{CustomMenuItem, Menu, MenuItem};

fn main() {
    let menu = Menu::new();    // メニューを構成
    tauri::Builder::default()
        .menu(menu)
        .on_menu_event(|event| {
            "quit" => {
                std::process:exit(0);
            }
            "close" => {
                event.window().close().unwrap();
            }
            _ => {}
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

#### ウィンドウメニュのイベントのリスニング

```rust
use tauri::{CustomMenuItem, Menu, MenuItem};
use tauri::{Manager, WindowBuilder};

fn main() {
    let menu = Menu::new();    // メニューを構成
    tauri::Builder::default()
        .setup(|app| {
            let window = WindowBuilder::new(
                app,
                "main-window".to_string(),
                tauri::WindowUrl::App("index.html".into()),
            )
            .menu(menu)
            .build()?;
            let window_ = window.clone();
            window.on_menu_event(move |event| {
                match event.menu_item_id() {
                    "quit" => {
                        std::process::exit(0);
                    }
                    "close" => {
                        window_.close().unwrap();
                    }
                    _ => {}
                }
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

#### メニューアイテムを更新する

`Window`構造体は`menu_handle`メソッドを持っていて、それはメニュアイテムの更新させます。

```rust
fn main() {
    let menu = Menu::new(); // メニューを構成
    tauri::Builder::default()
        .menu(menu)
        .setup(|app| {
            let main_window = app.get_window("main").unwrap();
            let menu_handle = main_window.menu_handle();
            std::thead::spawn(move || {
                // `set_selected`、`set_enable`そして`set_native_image`（macOSのみ）できます
                menu_handle.get_item("item_id").set_title("New Title");
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

## 複数ウィンドウ

ウィンドウはTauriの構成ファイルから静的に、もしくはランタイムで作成されます。

### 静的なウィンドウ

複数ウィンドウは、[tauri.windows]構成配列で作成されます。
次のJSONスニペットは、構成を介して、静的にいくつかのウィンドウを作成す方法を説明しています。

```json
{
    "tauri": {
        "windows": [
            {
                "label": "external",
                "title": "Tauri Docs",
                "url": "https://tauri.app"
            },
            {
                "label": "local",
                "title": "Tauri",
                "url": "index.html"
            }
        ]
    }
}
```

ウィンドウラベルがユニークでなければならないこと、そしてウィンドウインスタンスにアクセスするためにランタイムで使用できることに注意してください。
静的ウィンドウように利用可能な構成オプションのリストは[WindowConfig](https://tauri.app/v1/api/config#windowconfig)ドキュメントの中で見つけることができます。

### ランタイムウィンドウ

Rustレイヤを介してまたはTauri APIを経由して、ランタイムにウィンドウを作成することもできます。

#### Rustでウィンドを作成する

ウィンドウは、[WindowBuilder](https://docs.rs/tauri/1.0.0/tauri/window/struct.WindowBuilder.html)構造体を使用して、ランタイムに作成されます。

ウィンドウを作成するために、実行している[App](https://docs.rs/tauri/1.0.0/tauri/struct.App.html)または[AppHandle](https://docs.rs/tauri/1.0.0/tauri/struct.AppHandle.html)のインスタンスを持つ必要があります。

#### Appインスタンスを使用したウィンドウの作成

`App`インスタンスは準備フックまたは[Builder::build](https://docs.rs/tauri/1.0.0/tauri/struct.Builder.html#method.build)の呼び出し後に得ることができます。

```rust
tauri::Builder::default()
    .setup(|app| {
        let docs_window = tauri::WindowBuilder::new(
            app,
            "external",     // ユニークなウィンドウラベル
            tauri::WindowUrl::External("https://tauri.app/".parse().unwrap())
        ).build()?;
        let local_window = tauri::WindowBuilder::new(
                app,
                "local",
                tauri::WindowUrl::App("index.html".into())
        ).build()?;
        Ok(())
    })
    .run(tauri::generate_context!())
    .expect("error while running tauri application");
```

準備フックを使用することは、静的なウィンドウとTauriプラグインが初期化されていることを保証します。
代わりに、`App`の構築後にウィンドウを作成できます。

```rust
let app = tauri::Builder::default()
    .build(tauri::generate_context!())
    .expect("error while running tauri application");

let docs_window = tauri::WindowBuilder::new(
    &app,
    "external",     // ユニークなウィンドウラベル
    tauri::WindowUrl::External("https://tauri.app/".parse().unwrap())
).build().expect("failed to build window");

let local_window = tauri::WindowBuilder::new(
    &app,
    "local",
    tauri::WindowUrl::App("index.html".into())
).build()?;

// これはアプリの開始、これより下の他のコードを実行されません。
app.run(|_. _| {});
```

このメソッドは、値の所有権をセットアップクロージャに移動できない場合に便利です。

#### AppHandleインスタンスを使用したウィンドウの作成

`AppHandle`インスタンスは`App::handle`関数を使用するか、Tauriコマンドに直接挿入して得ることができます。

```rust
tauri::Builder::default()
    .setup(|app| {
        let handle = app.handle();
        std::thread::spawn(move || {
            let local_window = tauri::WindowBuilder::new(
                &handle,
                "local",
                tauri::WindowUrl::App("index.html".into())
            ).build()?;
        });
        Ok(())
    })
```

```rust
#[tauri::command]
async fn open_docs(handle: tauri::AppHandle) {
    let docs_window = tauri::WindowBuilder::new(
        &handle,
        "external", /* the unique window label */
        tauri::WindowUrl::External("https://tauri.app/".parse().unwrap())
    ).build().unwrap();
}
```

> ℹ️ 情報
>
> Tauriコマンド内でウィンドウを作成したとき、[wry#583](https://github.com/tauri-apps/wry/issues/583)イシューにより、Windowsにおけるデッドロックを避けるために、コマンド関数が`async`であることを確認してください。

#### JavaScriptでのウィンドウの作成

Tauri APIを使用して、[WebviewWindow](https://tauri.app/v1/api/js/window#webviewwindow)クラスをインポートすることにより、ランタイムで容易にウィンドウを作成できます。

```js
import { WebviewWindow } from '@tauri-apps/api/window'
const webview = new WebviewWindow('theUniqueLabel', {
    url: 'path/to/page.html',
})
// WebViewウィンドウを非同期で作成されるため、Tauriは作成レスポンスおw通知するために、`tauri://created`と`tauri://error`を発行します。
webview.once('tauri://created', function () {
    // WebViewウィンドウが成功裡に作成された。
})
webview.once('tauri://error', function (e) {
    // WebViewウィンドウの作成中にエラーが発生した。
})
```

### 追加のHTMLページを作成する

もし、`index.html`以外に追加でページを作成する場合、構築時にそれが`dist`ディレクトリに存在することを確認する必要があります。
これをする方法はフロントエンドの設定に依存します。
もし、Viteを使用している場合、2番目のHTMLページのために`vite.config.js`内に追加の[input](https://vitejs.dev/guide/build.html#multi-page-app)を作成してください。

### ランタイムでウィンドウにアクセスする

ウィンドウインスタンスはラベルとRustの[get_window](https://docs.rs/tauri/1.0.0/tauri/trait.Manager.html#method.get_window)メソッド、またはJavaScriptの[WebviewWindow.getByLabel](https://tauri.app/v1/api/js/window#getbylabel)を使用して問い合わせされます。

```rust
use tauri::Manager;
tauri::Builder::default()
    .setup(|app| {
        let main_window = app.get_window("main").unwrap();
        Ok(())
    })
```

[App](https://docs.rs/tauri/1.0.0/tauri/struct.App.html)または[AppHandle](https://docs.rs/tauri/1.0.0/tauri/struct.AppHandle.html)インスタンスの[get_window](https://docs.rs/tauri/1.0.0/tauri/trait.Manager.html#method.get_window)メソッドを使用するために、[tauri::Manager](https://docs.rs/tauri/1.0.0/tauri/trait.Manager.html)をインポートしなければなりません。

```js
import { WebviewWindow } from '@tauri-apps/api/window'
const mainWindow = WebviewWidow.getByLabel('main')
```

### 他のウィンドウとの通信

イベントシステムを使用して、ウィンドウは通信できます。
詳細は[イベントガイド](https://tauri.app/v1/guides/features/events)を参照してください。

## Tauriプラグイン

プラグインはTauriアプリケーションのライフサイクルをフックして、新しいコマンドを導入することができます。

### プラグインの使用

プラグインを使用するために、単にプラグインのインスタンスを`App`の`plugin`めそっぢに渡します。

```rust
fn main() {
    tauri::Builder::default()
        .plugin(my_awesome_plugin::init())
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

### プラグインの記述

プラグインは一般的な問題を解決するTauri APIの再利用可能な拡張です。。
これらはコードベースを構造化する非常に便利な方法でもあります。

もし、他とプラグインを共有する意図があるのであれば、準備済みのテンプレートを提供します！
インストールされたTauri [CLI]で実行します:

```sh
cargo tauri plugin init --name awesome
```

#### APIパッケージ

デフォルトでプラグインの利用者は、次のように提供されたコマンドを呼び出すことができます。

```js
import { invoke } from '@tauri-apps/api'
invoke('plugin:awesome|do_something')
```

`awesome`の場所は、プラグインの名前で置き換えされます。

これはあまり便利ではありませんが、プラグインはいわゆるAPIパッケージ、つまりコマンドへの便利なアクセスを提供するJavaScriptパッケージを提供するのが一般的です。

> この例は[tauri-plugin-store](https://github.com/tauri-apps/tauri-plugin-store)にあり、それはstoreにアクセスするための便利なクラス構造を提供します。
> このようにJavaScript APIパッケージでTauriプラグインの足場を作成できます。

```sh
cargo tauri plugin init --name awesome --api
```

### プラグインの記述

`tauri::plugin::Builder`を使用して、アプリを定義するのと似た方法でプラグインを定義できます。

```rust
use tauri::{plugin::{Builder, TauriPlugin},
    Runtime,
};

// もし、APIを拡張することを選択した場合は、プラグインのカスタムコマンドハンドラ

#[tauri::command]
// これは`invoke('plugin:awesome|initialize')`でアクセスされます。
// `awesome`はプラグインの名前です。
fn initialize() {}

#[tauri::command]
// これは`invoke('plugin:awesome|do_something')`でアクセスされます。
fn do_something() {}

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("awesome")
        .invoke_handler(tauri::generate_handler![initialize, do_something])
        .build()
}
```

プラグインは、ちょうどアプリができるように、状態を準備して管理します。

```rust
use tauri::{
    plugin::{Builder, TauriPlugin},
    AppHandle, Manager, Runtime, State,
};

#[derive(Default)]
struct MyState {}

#[tauri::command]
// これは`invoke('plugin:awesome|do_something')`でアクセスされます。
fn do_something<R: Runtime>(_app: AppHandle<R>, state: State<'_, MyState>) {
    // ここで、`MyState`にアクセスできます！

}

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("awesome")
        .invoke_handler(tauri::generate_handler![do_something])
        .setup(|app_handle| {
            // ここでプラグインの特定の状態を準備します。
            app_handle.manage(MyState::default());
            Ok(())
        })
        .build()
}
```

#### 慣例

* クレートはプラグインを作成するために`init`メソッドを公開します。
* プラグインは`tauri-plugin-`を前置詞にもつ明確な名前を持つべきです。
* `Cargo.toml`/`package.json`に`tauri-plugin`キーワードを含めてください。
* 英語でプラグインをドキュメント化してください。
* プラグインを紹介するアプリ例を追加してください。

#### 高度化

`tauri::plugin::Builder::build`によって返される`tauri::plugin::TauriPlugin`構造体に依存する代わりに、自身で`tauri::plugin::Plugin`を実装できます。
これは、関連するデータに対する完全な制御を持つことをを可能にします。

`name`関数を除いて、`Plugin`トレイトのそれぞれの関数はオプションであることに注意してください。

```rust
use tauri::plugin::{Plugin, Result as PluginResult};
use tauri::{Runtime, PageLoadPayload, Window, Invoke, AppHandle};

struct MyAwesomePlugin<R: Runtime> {
    invoke_handler: Box<dyn Fn(Invoke<R>) + Send + Sync>,
    // プラグインの状態、フィールド構成
}

#[tauri::command]
fn initialize() {}

#[tauri::command]
fn do_something() {}

impl<R: Runtime> MyAwesomePlugin<R> {
    // ここで、構成フィールドを追加できます。
    // 詳細は、https://doc.rust-lang.org/1.0.0/style/ownership/builders.htmlを参照してください。
    pub fn new() -> Self {
        Self {
            invoke_handler: Box::new(tauri::generate_handler![initialize, do_something]),
        }
    }
}

impl<R: Runtime> Plugin<R> for MyAwesomePlugIn<R> {
    // プラグイン名。定義されなければならず、`invoke`呼び出しで使用されます。
    fn name(&self) -> &'static str {
        "awesome"
    }

    // 初期化で評価するためのJSスクリプト。
    // プラグインが`window`を介してアクセスされるか、アプリの初期化でJSタスクを実行する必要がある場合に便利です。
    // 例えば、"window.awesomePlugin = { ... the plugin interface }"
    fn initialization_script(&self) -> Option<String> {
        None
    }

    // `tauri.conf.json > plugins > $yourPluginName`またはデフォルト値で提供される構成でプラグインを初期化する。
    fn initialize(&mut self, app: &AppHandle<R>, config: serde_json::Value) -> PluginResult<()> {
        Ok(())
    }

    /// Windowが作成されたときに呼び出されるコールバック
    fn created(&mut self, window: Window<R>) {}

    /// WebViewがナビゲーションを実行したときに呼び出されるコールバック
    fn on_page_load(&mut self, window: Window<R>, payload: PageLoadPayload) {}

    /// 呼び出しハンドラを拡張
    fn extend_api(&mut self, message: Invoke<R>) {
        (self.invoke_handler)(message)
    }
}
```

## スプラッシュスクリーン

もし、アプリがロードするために時間がかる可能性がある場合、またメインウィンドウを表示する前にRustで初期化プロシージャーを実行する必要がある場合、スプラッシュスクリーンはユーザーのためにロード中の体験を改善できる可能性があります。

### 準備

最初に、スプラッシュスクリーン用にHTMLコードを含んだ`splashscreen.html`を`distDir`に作成します。
そして、次のように`tauri.conf.json`を更新します。

```json
 "windows": [
    {
        "title": "Tauri App",
        "width": 800,
        "height": 600,
        "resizable": true,
        "fullscreen": false,
+       "visible": false // デフォルトでメインウィンドウを隠す
    },
    // スプラッシュスクリーンウィンドウを追加
+   {
+       "width": 400,
+       "height": 200,
+       "decorations": false,
+       "url": "splashscreen.html",
+       "label": "splashscreen"
+   }
 ]
```

これで、アプリが起動されたとき、メインウィンドウは隠されて、スプラッシュスクリーンウィンドウが表示されます。
次に、アプリが準備できたとき、スプラッシュスクリーンを閉じて、メインウィンドウを表示する方法が必要です。
これをする方法は、スプラッシュスクリーンを閉じる前に何を待っているかに依存します。

### Webページを待っている場合

Webコードを待っている場合、`close_splashscreen`[コマンド](https://tauri.app/v1/guides/features/command)を作成します。

```rust
use tauri::{Manager, Window};
// コマンドの作成:
// このコマンドは、メインスレッドで実行されないため、非同期でなければなりません。
#[tauri::command]
async fn close_splashscreen(window: Window) {
    // スプラッシュスクリーンを閉じる
    window.get_window("splashscreen").expect("no window labeled 'splashscreen' found").close().unwrap();
    // メインウィンドウを表示
    window.get_window("main").expect("no window labeled 'main' found").show().unwrap();

}

// コマンドを登録
fn main() {
    tauri::Builder::default()
        // この行を追加
        .invoke_handler(tauri::generate_handler![close_splashscreen])
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

2つの方法のうち1つでプロジェクトにそれをインポートできます。

```js
// Tauri API npmパッケージで:
import { invoke } from '@tauri-apps/api/tauri'
```

または

```js
// Tauriグローバルスクリプトで:
const invoke = window.__TAURI__.invoke
```

そして最後に、イベントリスナーを追加します（または希望するときはいつでも単に`invoke()`を呼び出します）。

```js
document.addEventListener('DOMContentLOaded', () => {
    // これはウィンドウがロードされるまで待ってますが、希望するトリガーでいつでもこの関数を実行できます。
    invoke('close_splashscreen')
})
```

### Rustを待っている場合

もし、Rustコードが実行されるまで待っている場合、`setup`関数ハンドラ内にそれを追加するため、`App`インスタンスへのアクセスを持ちます。

```rust
use tauri::Manager;

fn main() {
    tauri::Builder::default()
        .setup(|app| {
            let splashscreen_window = app.get_window("splashscreen").unwrap();
            let main_window = app.get_window("main").unwrap();
            // 新しいタスクで初期化コードを事項するため、アプリはフリーズしません。
            tauri::async_runtime::spawn(async move {
                // スリープする代わりに、ここでアプリを初期化します。
                println!("Initializing...");
                std::thread::sleep(std::time::Duration::from_secs(2));
                println!("Done initializing.");

                // それが終わった後、スプラッシュスクリーンを閉じて、メインウィンドウを表示します。
                splashscreen_window.close().unwrap();
                main_window.show().unwrap();
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("failed to run app");
}
```

## システムトレイ

### 準備

`tauri.conf.json`の`systemTray`オブジェクトを構成します。

```json
{
    "tauri": {
        "systemTray": {
            "iconPath": "icons/icon.png",
            "iconAsTemplate": true
        }
    }
}
```

`iconAsTemplate`は、macOSにおける[テンプレートイメージ](https://developer.apple.com/documentation/appkit/nsimage/1520017-template?language=objc)を表現するかどうかを決定する論理値です。

#### Linuxにおける準備

Linuxにおいて、`libayatana-appindicator`または`libappindicator3`パッケージをインストールする必要があります。
Tauriはランタイムでどちらのパッケージを使用するかを決定して、もし両方がインストールされている場合、`libayatana`を使用します。

デフォルトでは、Debianパッケージ(.deb)は`libayatana-appindicator3-1`への依存関係が追加されます。
`libappindicator3`をターゲットにするDebianパッケージを作成するために、`TAURI_TRAY`環境変数に`libappindicator3`を設定します。

アプリ画像のバンドルは自動的にトレイライブラリに埋め込まれ、それを手動で選択するために`TAURI_TRAY`環境変数を使用することもできます。

> ℹ️ 情報
>
> `libappindicator3`はメンテナンスされておらず、`debian11`のようないくつかのディストロは存在しませんが、`libayatana-appindicator`は古いリリースは存在しません。

### システムトレイの作成

ネイティブなシステムトレイを作成するために、`SystemTray`型をインポートします。

```rust
use tauri::SystemTray;
```

新しいトレイインスタンスを初期化します:

```rust
let tray = SystemTray::new();
```

### システムトレイコンテキストメニューの構成

オプションでコンテキストメニューを追加でき、それはトレイアイコンが右クリックされたときに表示されます。
`SystemTrayMenu`、`SystemTrayMenuItem`及び`CustomMenuItem`型をインポートします。

```rust
use tauri::{CustomMenuItem, SystemTrayMenu, SystemTrayMenuItem};
```

`SystemTrayMenu`を作成します。

```rust
// ここで、`"quit".to_string()`は、メニューアイテムのIDを定義して、2番目のパラメーターはメニューアイテムのラベルです。
let quit = CustomMenuItem::new("quit".to_string(), "Quit");
let hide = CustomMenuItem::New("hide".to_string(), "Hide");
let tray_menu = SystemTrayMenu::new()
    .add_item(quit)
    .add_native_item(SystemTrayMenuItem::Separator)
    .add_item(hide);
```

トレイメニューを`SystemTray`インスタンスに追加します。

```rust
let tray = SystemTray::new().with_menu(tray_menu);
```

### アプリのシステムトレイの構成

作成された`SystemTray`インスタンスは、`tauri::Builder`構造体の`system_tray`APIを使用して設定されます:

```rust
use tauri::{CustomMenuItem, SystemTray, SystemTrayMenu};

fn main() {
    let tray_menu = SystemTrayMenu::new();  // ここでメニューアイテムを挿入
    let system_tray = SystemTray::new().with_menu(tray_menu);
    tauri::Builder::default()
        .system_tray(system_tray)
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### システムトレイイベントのリスニング

クリックされたとき、それぞれの`CustomMenuItem`はイベントをトリガーします。
Tauriはトレイアイコンクリックイベントも発行します。
それらを処理するために`on_system_tray_event`APIを使用します:

```rust
use tauri::{CustomMenuItem, SystemTray, SystemTrayMenu, SystemTrayEvent};
use tauri::Manager;

fn main() {
    let tray_menu = SystemTrayMenu::new();      // ここでメニューアイテムを挿入
    tauri::Builder::default()
        .system_tray(SystemTray::new().with_menu(tray_menu))
        .on_system_tray_event(|app, event| match event {
            SystemTrayEvent::LeftClick {
                position: _,
                size: _,
                ..
            } => {
                println!("system tray received a left click");
            }
            SystemTrayEvent::RightClick {
                position: _,
                size: _,
                ..
            } => {
                println!("system tray received a right click");
            }
            SystemTrayEvent::DoubleClick {
                position: _,
                size: _,
                ..
            } => {
                println!("system tray received a double click");
            }
            SystemTrayEvent::MenuItemClick { id, .. }n => {
                match id.as_str() {
                    "quit" => {
                        std::process::exit(0);
                    }
                    "hide" => {
                        let window = app.get_window("main").unwrap();
                        window.hide().unwrap();
                    }
                    _ => {}
                }
            }
            _ => {}
        })
        .run(tauri::generate_context!())
        .expect("failed while running tauri application");
}
```

### システムトレイの更新

`AppHandle`構造体には`tray_handle`メソッドがあり、それは更新するトレイアイコンとコンテキストメニューの更新を許可するシステムトレイのハンドルを返します:

#### コンテキストメニューアイテムの更新

```rust
use tauri::{CustomMenuItem, SystemTray, SystemTrayMenu, SystemTrayEvent};
use tauri::Manager;

fn main() {
    let tray_menu = SystemTrayMenu::new(); // insert the menu items here
    tauri::Builder::default()
        .system_tray(SystemTray::new().with_menu(tray_menu))
        .on_system_tray_event(|app, event| match event {
            SystemTrayEvent::MenuItemClick { id, .. } => {
                // クリックされたメニューアイテムのハンドラを取得
                // `tray_handle`はどこでも呼び出すことができることに注意してください。
                // setupフックで`app.handle()`を使用して`AppHandle`インスタンスを得ると、他の関数またはスレッドにそれが移動します。
                let item_handle = app.tray_handle().get_item(&id);
                match id.as_str() {
                    "hide" => {
                        let window = app.get_window("main").unwrap();
                        window.hide().unwrap();

                        // you can also `set_selected`, `set_enabled` and `set_native_image` (macOS only).
                        // また`set_selected`、`set_enabled`そしえ`set_native_image`(macOSのみ)もできます。
                        item_handle.set_title("Show").unwrap();
                    }
                    _ => {}
                }
            }
            _ => {}
        })
        .run(tauri::generate_context!())
    .expect("error while running tauri application");
}
```

##### トレイアイコンの更新

`Icon::Raw`を使用できるようにするため、`Cargo.tom`内にTauriの依存関係の`icon-ico`または`icon-png`フィーチャフラグを追加する必要があることに注意してください。

```rust
app.tray_handle().set_icon(tauri::Icon::Raw(include_bytes!("../path/to/myicon.ico").to_vec())).unwrap();
```

#### アプリが閉じられることを回避

デフォルトで、最後のウィンドウが閉じられたとき、Tauriはアプリケーションを閉じます。
これを避けるために、単に`api.prevent_close`を呼び出せます。

必要に応じて次の2つの選択肢から1つを使用できます:

##### バックグラウンドでバックグラウンド実行を継続

もし、バックエンドがバックグラウンドで実行すべき場合、次のように`api.prevent_exit()`を呼び出せます。

```rust
tauri::Builder::default()
    .build(tauri::generate_context!())
    .expect("error while running tauri application")
    .run(|_app_handle, event| match event {
        tauri::RunEvent::ExitRequested { api, .. } => {
            api.prevent_exit();
        }
        _ => {}
    });
```

#### バックグラウンドでフロントエンドの実行を継続

もし、バックグラウンドでフロントエンドの実行を継続する必要がある場合、これは次のように達成されます:

```rust
tauri::Builder::default().on_window_event(|event| match event.event() {
    tauri::WindowEvent::CloseRequested { api, .. } => {
        event.window().hide().unwrap();
        api.prevent_close();
    }
    _ => {}
})
.run(tauri::generate_context!())
.expect("error while running tauri application");
```

## ウィンドウのカスタマイズ

Tauriは、アプリケーションのウィンドウの見た目をカスタマイズする多くのオプションを提供しています。
カスタムタイトルバーの作成、透明なウィンドウを持つ、制約したサイズに強制するなどできます。

### 構成

ウィンドウの構成を変更する3つの方法があります。

* [tauri.conf.jsonを経由](https://tauri.app/v1/api/config#tauri.windows)
* [JS APIを経由](https://tauri.app/v1/api/js/window#webviewwindow)
* [Rustのウィンドウを経由](https://docs.rs/tauri/1/tauri/window/struct.Window.html)

### カスタムタイトルバーを作成

これらのウィンドウの機能の一般的な使用は、カスタムタイトルバーを作成することです。
この短いチュートリアルはそのプロセスを一通り説明します。

#### CSS

タイトルバーをスクリーンの上部に固定して、ボタンのスタイルを設定するために、タイトルバーにいくつかのCSSを追加する必要があります。

```css
.titlebar {
    height: 30px;
    background: #329ea3;
    user-select: none;
    display: flex;
    justify-content: flex-end;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
}
.titlebar-button {
    display: inline-flex;
    justify-content: center;
    align-items: center;
    width: 30px;
    height: 30px;
}
.titlebar-button:hover {
    background: #5bbec3;
}
```

#### HTML

ここで、タイトルバーのHTMLを追加する必要があります。
次を`<body>`タグの最上部に配置します。

```html
<div data-tauri-drag-region class="titlebar">
    <div class="titlebar-button" id="titlebar-minimize">
        <img
          `src="https://api.iconify.design/mdi:window-minimize.svg"
          `alt="minimize"
        />
    </div>
    <div class="titlebar-button" id="titlebar-maximize">
        <img
            src="https://api.iconify.design/mdi:window-maximize.svg"
            alt="maximize"
        />
    </div>
    <div class="titlebar-button" id="titlebar-close">
        <img src="https://api.iconify.design/mdi:close.svg" alt="close" />
    </div>
</div>
```

タイトルバーが覆わないように、残りのコンテンツを下に移動する必要があることに注意してください。

#### JS

最後に、ボタンが機能するようにする必要があります。

```js
import { appWindow } from '@tauri-apps/api/window'
document
    .getElementById('titlebar-minimize')
    .addEventListener('click', () => appWindow.minimize())
document
    .getElementById('titlebar-maximize')
    .addEventListener('click', () => appWindow.toggleMaximize())
document
    .getElementById('titlebar-close')
    .addEventListener('click', () => appWindow.close())
```

#### tauri.conf.json

`appWindow`の呼び出しが機能するようにするために、`tauri.conf.json`に[window]パーミッションを追加することを忘れないでください。

```json
"tauri": {
    "allowList": {
        ...
        "window": {
            "all": false,
            "close": true,
            "hide": true,
            "show": true,
            "maximize": true,
            "minimize": true,
            "unmaximize": true,
            "unminimize": true,
            "startDragging": true
        }
    }
    ...

    "windows": [
        {
            "decorations": false,
            ...
        }
    ]
}
```
