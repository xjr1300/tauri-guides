# 開発

## 開発サイクル

### 1. 開発サーバーの起動

これですべてが準備したため、UIフレームワークまたはバンドラー（もちろん、何か1つを使用していることを想定しています）によって提供されるアプリケーション開発サーバーを起動します。

> ℹ️ 注意
> すべてのフレームワークはそれ自身の開発設備(tooling)があります。
> それら全てをカバーしたり最新に保つことは、このドキュメントの範囲外です。

### 2. Tauri開発ウィンドウの開始

```sh
cargo tauri dev
```

最初にこのコマンドを実行した場合、Rustパッケージマネージャーは必要とされるパッケージのダウンロードとビルドに数分かかります。
それらはキャッシュされるため、コードは再ビルドのみを必要とするため、後続するビルドはより早くなります。

一度、Rustがビルドを終了したら、`webview`が開き、Webアプリを表示します。
Webアプリを変更することができ、もしツール(tooling)がそれを有効にするならば、ちょうどブラウザのように`webview`は自動的に更新します。
Rustのファイルを変更したとき、それらは自動的に再ビルドされて、アプリが自動的に再起動します。

> ℹ️ Cargo.tomlとソースコード管理について
>
> Cargoは決定的なビルドを提供するためにロックファイルを使用するため、プロジェクトリポジトリ内の`src-tauri/Cargo.toml`と一緒に`src-tauri/Cargo.lock`をgitにコミットするべきです。
> 結果として、すべてのアプリケーションがそれらのCargo.lockを確認することを推奨します。
> `src-tauri/target`フォルダまたはそのコンテンツの何かをコミットする**べきではありません**。

## 依存関係の更新

### npmパッケージの更新

もし`tauri`パッケージを使用している場合:

```sh
yarn upgrade @tauri-apps/cli @tauri-apps/api --latest
```

Tauriの最新バージョンを検出することもコマンドラインでできます:

```sh
yarn outdated @tauri-apps/cli
```

あるいは、もし`vue-cli-plugin-tauri`を使用している場合:

```sh
yarn upgrade vue-cli-plugin-tauri --latest
```

### Cargoパッケージの更新

[cargo outdated](https://github.com/kbknapp/cargo-outdated)を使用して、またはcrates.ioページの[tauri](https://crates.io/crates/tauri/versions)/[tauri-buid](https://crates.io/crates/tauri-build/versions)で古くなったパッケージを確認できます。

`src-tauri/Cargo.toml`を開き、`tauri`と`tauri-build`を次の通り変更します。

```toml
[build-dependencies]
tauri-build = "%version%"

[dependencies]
tauri = { version = "%version%"}
```

`%version%`は上のバージョン番号に対応します。

その後、次を実行します。

```sh
cd src-tauri
cargo update
```

あるいは、[cargo-edit](https://github.com/killercup/cargo-edit)で提供される`cargo upgrade`コマンドを実行でき、それはこの全てを自動で行います。
