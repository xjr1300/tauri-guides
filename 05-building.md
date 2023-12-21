# ビルド

## 導入

Tauriビルダーはバイナリをコンパイル、アセットをパッケージそして最終的なバンドルを準備するRustのハーネス（落下制止用器具）です。

それはオペレーティングシステムを検出して、それに応じてバンドルをビルドします。
それは現在次をサポート指定m流。

* [Windows](https://tauri.app/v1/guides/building/windows): `-setup.ext`、`.msi`
* [macOS](https://tauri.app/v1/guides/building/macos): `.app`、`.dmg`
* [Linux](https://tauri.app/v1/guides/building/linux): `.deb`、`.appimage`

## Windowsインストーラー

Windows用に、Tauriアプリケーションは、[WiX Toolset v3](https://wixtoolset.org/documentation/manual/v3/)を使用してマイクロソフトインストーラー(`.msi`ファイル)として配布されるか、Tauri v1.3以降では[NSIS](https://nsis.sourceforge.io/Main_Page)を使用してセットアップ実行形式(`-setup.ext`ファイル)として配布されます。
Tauri CLIはバイナリと追加されたリソースをバンドルします。
`.msi`インストーラーは、いまだにクロスコンパイルが機能しないため、**Windows上でのみ作成できる**ことに注意してください。
NSISのためのクロスコンパイルは実験的な段階で、現在も作業を続けています。

このガイドは、インストーラーのために利用可能なカスタマイズオプションに関する情報を適用します。

ビルドして、1つのインストーラーにTauriアプリケーション全体をバンドルするためには、単に次のコマンドを実行します。

```sh
cargo tauri build
```

上記は、フロントエンドをビルドして、Rustバイナリにコンパイルして、すべての外部バイナリとリソースを集めて、最終的に適切なプラットフォーム固有のバンドルとインストーラーを生成します。

### 32ビットまたはARM用のビルド

Tauri CLIはデフォルトでマシンのアーキテクチャを使用して実行形式にコンパイルします。
64bitマシンで開発していることが想定された場合、CLIは64bitアプリケーションを生成します。

もし、**32bit**マシンをサポートする必要がある場合、`--target`フラグを使用して**異なる**Rustターゲットでアプリケーションをコンパイルできます。

```powershell
tauri build --target i686-pc-windows-msvc
```

デフォフトで、使用しているマシンのターゲット用にのみツールチェインをインストールするため、最初に32bitのWindowsツールチェインをインストールする必要があります: `rustup target add i686-pc-windows-msvc`。

もし、**ARM64**用にビルドする必要がある場合、最初に追加のビルドツールをインストールする必要があります。
これを行うために、`Visual Studio Installer`を開き、「修正」をクリックして、「個別のコンポーネント」タブで「C++ ARM64ビルドツール」をインストールします。
これを記述している時点では、VS2022における正確な名前は`MSVC v143 - VS 2022 C++ ARM64 build tools (Latest)`です。
ここで、`rustup target add aarch64-pc-windows-msvc`でRustターゲットをインストールして、アプリをコンパイルするために上記で言及した方法を使用できます。

```powershell
tauri build --target aarch64-pc-windows-msvc
```

> ℹ️ 情報
>
> NSISターゲットのみARM4ターゲットをサポートするため、すべてのバンドルの種類をコンパイルするようにTauriを構成した場合は、上記のコマンドを`tauri build --target aarch64-pc-windows-msvc --bundles nsis`に変更して、NSISインストーラーのみをビルドすることができます。
>
> インストーラー自体は、エミュレーションを介してARMマシン上で引き続きx86で実行されることに注意してください。
> アプリ自体はネイティブARM64バイナリになります。
