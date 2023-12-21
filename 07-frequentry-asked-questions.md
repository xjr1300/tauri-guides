# よくある質問と回答

## どのように公開されていないTauriの変更を使用するのでしょうか？

GitHub(最新バージョン)からTauriを使用するには、Cargo.tomlファイルを変更し、CLIとAPIを更新する必要があります。

### Rustクレートをソースから引き出す

次を`Cargo.toml`ファイルに追加します。

```toml
[patch.crates-io]
tauri = { git = "https://github.com/tauri-apps/tauri", branch = "1.x" }
tauri-build = { git = "https://github.com/tauri-apps/tauri", branch = "1.x" }
```

これにより、すべての依存関係でcrates.ioの代わりにGit(Hub)からの`tauri`と`tauri-build`が使用されるようになります。

### ソースからTauri CLIを使用する

もし、Cargo CLIを使用している場合、GitHubからそれを直接インストールできます。

```sh
cargo install --git https://github.com/tauri-apps/tauri --branch 1.x tauri-cli
```

もし、`@tauri-apps/cli`パッケージを使用している場合、リポジトリをクローンして、それをビルドする必要があります。

```sh
git clone https://github.com/tauri-apps/tauri
cd tauri
git checkout 1.x
cd tauri/tooling/cli/node
yarn
yarn build
```

それを使用するために、nodeで直接実行します。

```sh
node /path/to/tauri/tooling/cli/node/tauri.js dev
node /path/to/tauri/tooling/cli/node/tauri.js build
```

代わりに、Cargoで直接アプリを実行できます。

```sh
cd src-tauri
cargo run --no-default-features # tauri devの代わりに
cargo build # tauri buildの代わりに - ただし、アプリはバンドルされません
```

#### ソースからTauri APIを使用する

GitHubからのTauriクレートを使用しているとき(必要でないかもしれません)、ソースからTauri APIパッケージを使用することを推奨します。
ソースからそれをビルドするために、次のスクリプトを実行します。

```sh
git clone https://github.com/tauri-apps/tauri
cd tauri
git checkout 1.x
cd tauri/tooling/api
yarn
yarn build
```

これで、yarnを使用してそれとリンクできます。

```sh
cd dist
yarn link
cd /path/to/your/project
yarn link @tauri-apps/api
```

または、distフォルダーを直接指すようにpackage.jsonを変更することもできます。

```json
{
    "dependencies": {
        "@tauri-apps/api": "/path/to/tauri/tooling/api/dist"
    }
}
```

## NodeまたはCargoを使用するべきでしょうか？

Cargoを経由してCLIをインストールすることが推奨されるオプションですが、インストール時にバイナリ全体を最初からコンパイルする必要があります。
CI環境または非常に遅いマシンを使用している場合は、別のインストール方法を選択することをお勧めします。

CLIはRustで書かれているため、当然のことながら[crates.io](https://crates.io/crates/tauri-cli)から入手でき、Cargoでインストールできます。

また、CLIをネイティブNode.jsアドオンとしてコンパイルし、[npm経由](https://www.npmjs.com/package/@tauri-apps/cli)でそれを配布します。
これには、Cargoインストール方法と比較していくつかの利点があります。

1. CLIは事前にコンパイルされているため、インストール時間が大幅に短縮されます
2. package.jsonファイルで特定のバージョンを固定できます。
3. Tauriを中心にカスタムツールを開発する場合は、CLIを通常のJavaScriptモジュールとしてインポートできます。
4. JavaScriptマネージャーを使用してCLIをインストールできます

## 推奨されるブラウザのリスト

ブラウザリストとビルドターゲットには、`es2021`、`最新の3つのChromeバージョン`及び`safari 13`を使用することをお勧めします。
Tauriは、OSのネイティブレンダリングエンジン (macOSではWebKit、WindowsではWebView2、LinuxではWebKitGTK)を利用します。

## Linux上のHomebrewとのビルドの競合

Linux上のHomebrewには、独自のpkg-config(システム上のライブラリを検索するユーティリティ)が含まれています。
これにより、Tauri用の同じpkg-configパッケージをインストールするときに競合が発生する可能性があります(通常、aptなどのパッケージマネージャーを通じてインストールされます)。
Tauriアプリを構築しようとすると、pkg-configを呼び出そうとしますが、最終的にはHomebrewからのアプリを呼び出すことになります。
Tauriの依存関係のインストールにHomebrewが使用されていない場合、エラーが発生する可能性があります。

通常、エラーには次のようなメッセージが含まれます: Xのカスタムビルドコマンドの実行に失敗しました - パッケージYがpkg-config検索パスに見つかりませんでした。
要な依存関係がまったくインストールされていない場合、同様のエラーが表示される可能性があることに注意してください。

この問題には2つの解決策があります。

1. [Homebrewをアンインストール](https://docs.brew.sh/FAQ#how-do-i-uninstall-homebrew)する
2. Tauriアプリをビルドする前に、正しいpkg-configを指すようにPKG_CONFIG_PATH環境変数を設定します。
   例: export PKG_CONFIG_PATH=/usr/lib/pkgconfig:/usr/share/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig
