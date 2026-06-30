# fd

[![CICD](https://github.com/sharkdp/fd/actions/workflows/CICD.yml/badge.svg)](https://github.com/sharkdp/fd/actions/workflows/CICD.yml)
[![Version info](https://img.shields.io/crates/v/fd-find.svg)](https://crates.io/crates/fd-find)
[[English (Original)](https://github.com/sharkdp/fd)]
[[中文](https://github.com/cha0ran/fd-zh)]
[[한국어](https://github.com/spearkkk/fd-kor)]

`fd` はファイルシステム上のエントリを検索するプログラムです。
[`find`](https://www.gnu.org/software/findutils/) のシンプルで高速、かつユーザーフレンドリーな代替ツールです。
`find` の強力な機能すべてをサポートすることを目的とはしていませんが、多くのユースケースで合理的な（意見を反映した）デフォルト動作を提供します。

[インストール](#インストール) • [使い方](#使い方) • [トラブルシューティング](#トラブルシューティング)

## 特徴

* 直感的な構文: `find -iname '*PATTERN*'` の代わりに `fd PATTERN`。
* 正規表現（デフォルト）とグロブパターンに対応。
* ディレクトリの並列探索による[高速な検索](#ベンチマーク)。
* `ls` と同様にファイルタイプを色分けして表示。
* [並列コマンド実行](#コマンド実行)をサポート。
* スマートケース: デフォルトでは大文字・小文字を区別しない検索を行い、パターンに大文字が含まれる場合は自動的に大文字・小文字を区別する検索に切り替わります[\*](http://vimdoc.sourceforge.net/htmldoc/options.html#'smartcase')。
* デフォルトで隠しディレクトリとファイルを無視。
* デフォルトで `.gitignore` のパターンを無視。
* コマンド名が `find` より *50%* 短い[\*](https://github.com/ggreer/the_silver_searcher) :-)。

## デモ

![Demo](doc/screencast.svg)

## 使い方

まず、利用可能なすべてのコマンドラインオプションの概要を確認するには、
[`fd -h`](#コマンドラインオプション) で簡潔なヘルプを表示するか、`fd --help` でより詳細なバージョンを表示します。

### シンプルな検索

*fd* はファイルシステム上のエントリを検索するために設計されています。最も基本的な検索は、
引数を1つ指定して *fd* を実行することです（その引数が検索パターンになります）。
たとえば、`netflix` を含む名前の古いスクリプトを探したい場合:
``` bash
> fd netfl
Software/python/imdb-ratings/netflix-details.py
```
このように引数を1つだけ指定すると、*fd* はカレントディレクトリを再帰的に検索し、
パターン `netfl` を *含む* エントリをすべて表示します。

### 正規表現による検索

検索パターンは正規表現として扱われます。ここでは、`x` で始まり `rc` で終わるエントリを検索します:
``` bash
> cd /etc
> fd '^x.*rc$'
X11/xinit/xinitrc
X11/xinit/xserverrc
```

`fd` が使用する正規表現の構文は[こちらに記載されています](https://docs.rs/regex/latest/regex/#syntax)。

### ルートディレクトリの指定

特定のディレクトリを検索したい場合は、*fd* の第2引数として指定できます:
``` bash
> fd passwd /etc
/etc/default/passwd
/etc/pam.d/passwd
/etc/passwd
```

### すべてのファイルを再帰的にリストアップ

*fd* は引数なしで呼び出すこともできます。これはカレントディレクトリ内のすべてのエントリを
再帰的に（`ls -R` に似た形で）素早く確認するのに非常に便利です:
``` bash
> cd fd/tests
> fd
testenv
testenv/mod.rs
tests.rs
```

指定したディレクトリ内のすべてのファイルをリストアップするには、
`.` や `^` などのキャッチオールパターンを使用する必要があります:
``` bash
> fd . fd/tests/
testenv
testenv/mod.rs
tests.rs
```

### 特定の拡張子のファイルを検索

特定のタイプのファイルをすべて検索したい場合、`-e`（または `--extension`）オプションを使用します。
ここでは、fd リポジトリ内のすべての Markdown ファイルを検索します:
``` bash
> cd fd
> fd -e md
CONTRIBUTING.md
README.md
```

`-e` オプションは検索パターンと組み合わせて使用できます:
``` bash
> fd -e rs mod
src/fshelper/mod.rs
src/lscolors/mod.rs
tests/testenv/mod.rs
```

### 特定のファイル名を検索

指定した検索パターンに完全一致するファイルを検索するには、`-g`（または `--glob`）オプションを使用します:
``` bash
> fd -g libc.so /usr
/usr/lib32/libc.so
/usr/lib/libc.so
```

### 隠しファイルと無視されるファイル

デフォルトでは、*fd* は隠しディレクトリを検索せず、検索結果に隠しファイルを表示しません。
この動作を無効にするには、`-H`（または `--hidden`）オプションを使用します:
``` bash
> fd pre-commit
> fd -H pre-commit
.git/hooks/pre-commit.sample
```

Git リポジトリ（またはGitリポジトリを含むディレクトリ）で作業している場合、*fd* は
`.gitignore` のパターンに一致するフォルダ（およびファイル）を検索・表示しません。
この動作を無効にするには、`-I`（または `--no-ignore`）オプションを使用します:
``` bash
> fd num_cpu
> fd -I num_cpu
target/debug/deps/libnum_cpus-f5ce7ef99006aa05.rlib
```

*すべての* ファイルとディレクトリを本当に検索するには、`-HI` で hidden と ignore の両機能を
組み合わせるか、`-u`/`--unrestricted` を使用します。

### フルパスのマッチ

デフォルトでは、*fd* は各ファイルのファイル名のみでマッチングします。ただし、
`--full-path` または `-p` オプションを使うと、フルパス全体に対してマッチングできます。

```bash
> fd -p -g '**/.git/config'
> fd -p '.*/lesson-\d+/[a-z]+.(jpg|png)'
```

### コマンド実行

検索結果を表示するだけでなく、それらに対して何か処理を行いたい場合、`fd` は
各検索結果に対して外部コマンドを実行する2つの方法を提供します:

* `-x`/`--exec` オプションは、*各検索結果に対して*（並列で）外部コマンドを実行します。
* `-X`/`--exec-batch` オプションは、*すべての検索結果を引数として*外部コマンドを1度だけ実行します。

#### 使用例

すべての zip アーカイブを再帰的に検索して解凍する:
``` bash
fd -e zip -x unzip
```
`file1.zip` と `backup/file2.zip` の2つのファイルがある場合、
`unzip file1.zip` と `unzip backup/file2.zip` が実行されます。
2つの `unzip` プロセスは（ファイルが十分に速く見つかれば）並列で実行されます。

すべての `*.h` と `*.cpp` ファイルを検索して `clang-format -i` で自動フォーマットする:
``` bash
fd -e h -e cpp -x clang-format -i
```
`clang-format` の `-i` オプションは別の引数として渡すことができます。
そのため、`-x` オプションを最後に置きます。

`-x` の後の位置引数はコマンドテンプレートに属し、`fd` 自体には属しません。
パターンや検索パスも渡したい場合は、`-x` を最後に置きます:
``` bash
fd pattern path -x echo
```

すべての `test_*.py` ファイルを検索してエディタで開く:
``` bash
fd -g 'test_*.py' -X vim
```
ここでは `-X`（大文字）を使用して、1つの `vim` インスタンスを開きます。
`test_basic.py` と `lib/test_advanced.py` の2つのファイルがある場合、
`vim test_basic.py lib/test_advanced.py` が実行されます。

ファイルのパーミッション、オーナー、サイズなどの詳細を確認するには、
各結果に対して `ls` を実行するように `fd` に指示します:
``` bash
fd … -X ls -lhd --color=always
```
このパターンは非常に便利なため、`fd` にはショートカットが用意されています。
`-l`/`--list-details` オプションでこの方法で `ls` を実行できます: `fd … -l`。

`-X` オプションは、`fd` と [ripgrep](https://github.com/BurntSushi/ripgrep/) (`rg`) を組み合わせて
特定のクラスのファイル（例: すべてのC++ソースファイル）内を検索する際にも便利です:
```bash
fd -e cpp -e cxx -e h -e hpp -X rg 'std::cout'
```

すべての `*.jpg` ファイルを `*.png` ファイルに変換する:
``` bash
fd -e jpg -x convert {} {.}.png
```
ここで、`{}` は検索結果のプレースホルダーです。`{.}` は同じですが、ファイル拡張子を除いたものです。
プレースホルダー構文の詳細については以下を参照してください。

`-x` を使って並列スレッドから実行されたコマンドのターミナル出力は混在や文字化けしないため、
`fd -x` は多くのファイルに対するタスクを大まかに並列化するために使用できます。
これの例として、ディレクトリ内の各ファイルのチェックサムを計算することが挙げられます:
```
fd -tf -x md5sum > file_checksums.txt
```

#### プレースホルダー構文

`-x` と `-X` オプションは、（単一の文字列ではなく）一連の引数として *コマンドテンプレート* を受け取ります。
コマンドテンプレートの後に `fd` の追加オプションを渡したい場合は、`\;` で終了させることができます。

たとえば、`fd -x echo \; pattern path` は `pattern path` を `echo` へ渡すのではなく、
`fd` の引数として扱います。実際には、`fd pattern path -x echo` と書く方が分かりやすいことが多いです。

コマンドを生成するための構文は [GNU Parallel](https://www.gnu.org/software/parallel/) に似ています:

- `{}`: 検索結果のパス（`documents/images/party.jpg`）に置き換えられるプレースホルダートークン。
- `{.}`: `{}` と同じですが、ファイル拡張子なし（`documents/images/party`）。
- `{/}`: 検索結果のベース名（`party.jpg`）に置き換えられるプレースホルダー。
- `{//}`: 検出されたパスの親ディレクトリ（`documents/images`）。
- `{/.}`: 拡張子を除いたベース名（`party`）。

プレースホルダーを含めない場合、*fd* は自動的に末尾に `{}` を追加します。

#### 並列実行と逐次実行

`-x`/`--exec` では、`-j`/`--threads` オプションを使って並列ジョブ数を制御できます。
逐次実行には `--threads=1` を使用します。

### 特定のファイルやディレクトリの除外

特定のサブディレクトリの検索結果を無視したい場合があります。たとえば、
すべての隠しファイルとディレクトリ（`-H`）を検索しつつ、`.git` ディレクトリからのマッチは
除外したい場合、`-E`（または `--exclude`）オプションを使用します。
このオプションは任意のグロブパターンを引数として受け取ります:
``` bash
> fd -H -E .git …
```

マウントされたディレクトリをスキップする場合にも使用できます:
``` bash
> fd -E /mnt/external-drive …
```

特定のファイルタイプをスキップする場合:
``` bash
> fd -E '*.bak' …
```

このような除外パターンを永続的にするには、`.fdignore` ファイルを作成します。
`.gitignore` ファイルと同様に機能しますが、`fd` 専用です。例:
``` bash
> cat ~/.fdignore
/mnt/external-drive
*.bak
```

> [!NOTE]
> `fd` は `rg` や `ag` などの他のプログラムが使用する `.ignore` ファイルもサポートしています。

これらのパターンを `fd` でグローバルに無視したい場合は、`fd` のグローバル無視ファイルに記載します。
通常、macOS や Linux では `~/.config/fd/ignore`、Windows では `%APPDATA%\fd\ignore` に配置されています。

`--hidden` オプションを使用する際に `.git` ディレクトリとその内容を出力に含めないようにするには、
`.git/` を `fd/ignore` ファイルに追加することをお勧めします。

### ファイルの削除

`fd` を使って、検索パターンに一致するすべてのファイルとディレクトリを削除できます。
ファイルのみを削除したい場合は、`--exec-batch`/`-X` オプションを使って `rm` を呼び出します。
たとえば、すべての `.DS_Store` ファイルを再帰的に削除するには:
``` bash
> fd -H '^\.DS_Store$' -tf -X rm
```
不確かな場合は、まず `-X rm` なしで `fd` を実行してください。または `rm` の "interactive" オプションを使用します:
``` bash
> fd -H '^\.DS_Store$' -tf -X rm -i
```

特定のクラスのディレクトリも削除したい場合は、同じ方法を使用できます。
ディレクトリを削除するには `rm` の `--recursive`/`-r` フラグが必要です。

> [!NOTE]
> `fd … -X rm -r` を使用すると競合状態が発生するシナリオがあります: `…/foo/bar/foo/…` のような
> パスがあり、`foo` という名前のすべてのディレクトリを削除しようとする場合、外側の `foo`
> ディレクトリが先に削除され、`rm` の呼び出しで（無害な）*"'foo/bar/foo': そのようなファイルまたはディレクトリはありません"*
> エラーが発生する状況になることがあります。

### コマンドラインオプション

以下は `fd -h` の出力です。完全なコマンドラインオプションのセットを確認するには、
`fd --help` を使用してください（より詳細なヘルプテキストも含まれています）。

```
Usage: fd [OPTIONS] [pattern [path]...]

Arguments:
  [pattern]  検索パターン（'--glob' を使用しない場合は正規表現; 省略可能）
  [path]...  ファイルシステム検索のルートディレクトリ（省略可能）

Options:
  -H, --hidden                     隠しファイルとディレクトリを検索
  -I, --no-ignore                  .(git|fd)ignore ファイルを無視しない
  -s, --case-sensitive             大文字・小文字を区別する検索（デフォルト: スマートケース）
  -i, --ignore-case                大文字・小文字を区別しない検索（デフォルト: スマートケース）
  -g, --glob                       グロブベースの検索（デフォルト: 正規表現）
  -a, --absolute-path              相対パスの代わりに絶対パスを表示
  -l, --list-details               ファイルメタデータを含む長い形式で表示
  -L, --follow                     シンボリックリンクをたどる
  -p, --full-path                  フル絶対パスで検索（デフォルト: ファイル名のみ）
  -d, --max-depth <depth>          最大検索深度を設定（デフォルト: なし）
  -E, --exclude <glob>             指定したグロブパターンに一致するエントリを除外
  -t, --type <filetype>            タイプでフィルタ: ファイル (f), ディレクトリ (d/dir), シンボリックリンク (l),
                                   実行可能 (x), 空 (e), ソケット (s), パイプ (p), キャラクターデバイス
                                   (c), ブロックデバイス (b)
  -e, --extension <ext>            拡張子でフィルタ
  -S, --size <size>                ファイルサイズに基づいて結果を制限
      --changed-within <date|dur>  ファイル更新時刻でフィルタ（指定より新しい）
      --changed-before <date|dur>  ファイル更新時刻でフィルタ（指定より古い）
  -o, --owner <user:group>         所有ユーザーおよび/またはグループでフィルタ
      --format <fmt>               テンプレートに従って結果を表示
  -x, --exec <cmd>...              各検索結果に対してコマンドを実行
  -X, --exec-batch <cmd>...        すべての検索結果を一度にコマンドの引数として実行
  -c, --color <when>               色を使用するタイミング [デフォルト: auto] [使用可能な値: auto,
                                   always, never]
      --hyperlink[=<when>]         出力パスにハイパーリンクを追加 [デフォルト: never] [使用可能な値: auto, always, never]
      --ignore-contain <name>      指定した名前のエントリを含むディレクトリを無視
  -h, --help                       ヘルプを表示（'--help' でより詳細）
  -V, --version                    バージョンを表示
```

オプションはパターンやパスの後に指定することもできます。

## ベンチマーク

ホームフォルダ内で `[0-9].jpg` で終わるファイルを検索してみましょう。
約750,000のサブディレクトリと約400万のファイルが含まれています。
平均化と統計分析には [hyperfine](https://github.com/sharkdp/hyperfine) を使用します。
以下のベンチマークは「ウォーム」（事前に満たされたディスクキャッシュ）で実行されています
（「コールド」ディスクキャッシュでも同様の傾向が見られます）。

まず `find` から始めます:
```
Benchmark 1: find ~ -iregex '.*[0-9]\.jpg$'
  Time (mean ± σ):     19.922 s ±  0.109 s
  Range (min … max):   19.765 s … 20.065 s
```

`find` は正規表現検索を行わない場合はかなり高速です:
```
Benchmark 2: find ~ -iname '*[0-9].jpg'
  Time (mean ± σ):     11.226 s ±  0.104 s
  Range (min … max):   11.119 s … 11.466 s
```

次に `fd` で同じことを試してみましょう。`fd` はデフォルトで正規表現検索を行います。
公平な比較のために `-u`/`--unrestricted` オプションが必要です。
そうしないと `fd` は隠しフォルダや無視されるパスを探索しなくて済む分有利になってしまいます（下記参照）:
```
Benchmark 3: fd -u '[0-9]\.jpg$' ~
  Time (mean ± σ):     854.8 ms ±  10.0 ms
  Range (min … max):   839.2 ms … 868.9 ms
```
この例では、`fd` は `find -iregex` より約 **23倍高速** で、`find -iname` より約 **13倍高速** です。
ちなみに、両ツールはまったく同じ546ファイルを見つけました :smile:。

**注記**: これは *特定のマシン* での *特定のベンチマーク* です。さまざまなテストを実施し（一貫した結果が得られました）、
あなたの環境では異なる結果になる可能性があります！ご自身でも試してみることをお勧めします。
必要なスクリプトはすべて[このリポジトリ](https://github.com/sharkdp/fd-benchmarks)にあります。

*fd* の速度については、[ripgrep](https://github.com/BurntSushi/ripgrep) でも使用されている
`regex` と `ignore` クレートに多くの功績があります（ぜひチェックしてみてください！）。

## トラブルシューティング

### `fd` がファイルを見つけられない!

`fd` はデフォルトで隠しディレクトリとファイルを無視します。また、`.gitignore` ファイルのパターンも無視します。
すべてのファイルを確実に見つけるには、常に `-u`/`--unrestricted` オプション（または
隠しファイルと無視されたファイルを有効にする `-HI`）を使用してください:
``` bash
> fd -u …
```

また、デフォルトでは `fd` はファイル名のみで検索し、フルパスとパターンを比較しません。
フルパスに基づいて検索したい場合（`find` の `-path` オプションに似た動作）は、
`--full-path`（または `-p`）オプションを使用する必要があります。

### カラー出力

`fd` は `ls` と同様に、拡張子によってファイルを色分けできます。これが機能するためには、
環境変数 [`LS_COLORS`](https://linux.die.net/man/5/dir_colors) が設定されている必要があります。
通常、この変数の値は `dircolors` コマンドによって設定されます。このコマンドは
異なるファイル形式に色を定義するための便利な設定形式を提供します。
ほとんどのディストリビューションでは、`LS_COLORS` はすでに設定されているはずです。
Windows を使用している場合や、より完全な（またはよりカラフルな）代替手段を探している場合は、
[こちら](https://github.com/sharkdp/vivid)、[こちら](https://github.com/seebi/dircolors-solarized)、
または[こちら](https://github.com/trapd00r/LS_COLORS)を参照してください。

`fd` は [`NO_COLOR`](https://no-color.org/) 環境変数もサポートしています。

### `fd` が正規表現パターンを正しく解釈していないようだ

多くの特殊な正規表現文字（`[]`、`^`、`$` など）はシェルでも特殊文字として扱われます。
不明な場合は、正規表現パターンを必ずシングルクォートで囲んでください:

``` bash
> fd '^[A-Z][0-9]+$'
```

パターンがダッシュで始まる場合は、コマンドラインオプションの終わりを示すために `--` を追加する必要があります。
そうしないと、パターンがコマンドラインオプションとして解釈されます。
代わりに、ハイフン1文字のキャラクタークラスを使用することもできます:

``` bash
> fd -- '-pattern'
> fd '[-]pattern'
```

### `alias` やシェル関数で "Command not found" エラーが出る

シェルの `alias` やシェル関数は、`fd -x` や `fd -X` を使ったコマンド実行には使用できません。
`zsh` では `alias -g myalias="…"` でエイリアスをグローバルにすることができます。
`bash` では `export -f my_function` を使ってサブプロセスから利用可能にすることができます。
その場合でも `fd -x bash -c 'my_function "$1"' bash` と呼び出す必要があります。
他のユースケースやシェルでは、（一時的な）シェルスクリプトを使用してください。

### `-x`/`-X` のプレースホルダー

シェルによっては、プレースホルダー（`{}`、`{/}`、`{//}`、`{.}`、`{/.}`）を
シェルが解釈する前に `fd` が受け取れるよう、クォートする必要がある場合があります。

## 他のプログラムとの連携

### fd と `fzf` の連携

*fd* を使って、コマンドラインのファジーファインダー [fzf](https://github.com/junegunn/fzf) の
入力を生成できます:
``` bash
export FZF_DEFAULT_COMMAND='fd --type file'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
```

その後、ターミナルで `vim <Ctrl-T>` と入力することで fzf を開き、fd の結果から検索できます。

または、シンボリックリンクをたどり、隠しファイルを含める（ただし `.git` フォルダは除外する）ように設定することもできます:
``` bash
export FZF_DEFAULT_COMMAND='fd --type file --follow --hidden --exclude .git'
```

fzf の中で fd のカラー出力を使用することもできます:
``` bash
export FZF_DEFAULT_COMMAND="fd --type file --color=always"
export FZF_DEFAULT_OPTS="--ansi"
```

詳細については、fzf の README の [Tips セクション](https://github.com/junegunn/fzf#tips)を参照してください。

### fd と `rofi` の連携

[*rofi*](https://github.com/davatorium/rofi) は標準入力から読み取ってメニューを作成できるグラフィカルなランチメニューアプリケーションです。
`fd` の出力を rofi の `-dmenu` モードにパイプすることで、ファイルやディレクトリのファジー検索可能なリストを作成できます。

#### 使用例

`$HOME` ディレクトリ以下の *PDF* ファイルの大文字・小文字を区別しない検索可能な複数選択リストを作成し、
選択をPDFビューアで開きます。すべてのファイルタイプをリストアップするには `-e pdf` 引数を削除します。

``` bash
fd --type f -e pdf . $HOME | rofi -keep-right -dmenu -i -p FILES -multi-select | xargs -I {} xdg-open {}
```

rofi が表示するリストを変更するには `fd` コマンドに引数を追加します。
rofi の検索動作を変更するには `rofi` コマンドに引数を追加します。

### fd と `emacs` の連携

emacs パッケージ [find-file-in-project](https://github.com/technomancy/find-file-in-project) は
*fd* を使ってファイルを検索できます。

`find-file-in-project` をインストールした後、`~/.emacs` または `~/.emacs.d/init.el` ファイルに
`(setq ffip-use-rust-fd t)` の行を追加します。

emacs で `M-x find-file-in-project-by-selected` を実行して一致するファイルを検索するか、
`M-x find-file-in-project` を実行してプロジェクト内のすべての利用可能なファイルをリストアップします。

### ツリー形式で出力

`fd` の出力をファイルツリー形式にフォーマットするには、`tree` コマンドを
`--fromfile` と共に使用します:
```bash
❯ fd | tree --fromfile
```

これは `tree` を単独で実行するよりも便利です。なぜなら `tree` はデフォルトでどのファイルも
無視せず、何を表示するかを制御するための `fd` ほど豊富なオプションセットをサポートしていないからです:
```bash
❯ fd --extension rs | tree --fromfile
.
├── build.rs
└── src
    ├── app.rs
    └── error.rs
```

bash や類似のシェルでは、シンプルにエイリアスを作成できます:
```bash
❯ alias as-tree='tree --fromfile'
```

### fd と `xargs` または `parallel` の連携

`fd` には `-x`/`--exec` と `-X`/`--exec-batch` オプションによる[コマンド実行](#コマンド実行)機能が
内蔵されていることに注意してください。必要に応じて、`xargs` と組み合わせて使用することもできます:
``` bash
> fd -0 -e rs | xargs -0 wc -l
```
ここで、`-0` オプションは *fd* に検索結果を改行の代わりに NULL 文字で区切るよう指示します。
同様に、`xargs` の `-0` オプションはこの形式で入力を読み取るよう指示します。

## インストール

[![Packaging status](https://repology.org/badge/vertical-allrepos/fd-find.svg)](https://repology.org/project/fd-find/versions)

### Ubuntu の場合

*...および他の Debian ベースの Linux ディストリビューション。*

Ubuntu 19.04 (Disco Dingo) 以降を使用している場合は、
[公式メンテナンスパッケージ](https://packages.ubuntu.com/fd-find)をインストールできます:
```
apt install fd-find
```
バイナリは `fdfind` と呼ばれることに注意してください。`fd` というバイナリ名は別のパッケージで
すでに使用されているためです。インストール後、このドキュメントと同じ方法で `fd` を使用できるよう、
`ln -s $(which fdfind) ~/.local/bin/fd` コマンドを実行して `fd` へのリンクを追加することをお勧めします。
`$HOME/.local/bin` が `$PATH` に含まれていることを確認してください。

古いバージョンの Ubuntu を使用している場合は、[リリースページ](https://github.com/sharkdp/fd/releases)から
最新の `.deb` パッケージをダウンロードし、次のコマンドでインストールできます:
``` bash
dpkg -i fd_9.0.0_amd64.deb # バージョン番号とアーキテクチャを適宜変更
```

このプロジェクトのリリースページの .deb パッケージは実行ファイルを `fd` という名前のままにしています。

### Debian の場合

Debian Buster 以降を使用している場合は、
[公式メンテナンス Debian パッケージ](https://tracker.debian.org/pkg/rust-fd-find)をインストールできます:
```
apt-get install fd-find
```
バイナリは `fdfind` と呼ばれることに注意してください。`fd` というバイナリ名は別のパッケージで
すでに使用されているためです。インストール後、このドキュメントと同じ方法で `fd` を使用できるよう、
`ln -s $(which fdfind) ~/.local/bin/fd` コマンドを実行して `fd` へのリンクを追加することをお勧めします。
`$HOME/.local/bin` が `$PATH` に含まれていることを確認してください。

このプロジェクトのリリースページの .deb パッケージは実行ファイルを `fd` という名前のままにしています。

### Fedora の場合

Fedora 28 以降では、公式パッケージソースから `fd` をインストールできます:
``` bash
dnf install fd-find
```

### Alpine Linux の場合

適切なリポジトリが有効になっていれば、公式ソースから [fd パッケージ](https://pkgs.alpinelinux.org/packages?name=fd)をインストールできます:
```
apk add fd
```

### Arch Linux の場合

公式リポジトリから [fd パッケージ](https://www.archlinux.org/packages/extra/x86_64/fd/)をインストールできます:
```
pacman -S fd
```
[AUR から fd をインストール](https://aur.archlinux.org/packages/fd-git)することもできます。

### Gentoo Linux の場合

公式リポジトリから [fd ebuild](https://packages.gentoo.org/packages/sys-apps/fd) を使用できます:
```
emerge -av fd
```

### openSUSE Linux の場合

公式リポジトリから [fd パッケージ](https://software.opensuse.org/package/fd)をインストールできます:
```
zypper in fd
```

### Void Linux の場合

xbps-install で `fd` をインストールできます:
```
xbps-install -S fd
```

### ALT Linux の場合

公式リポジトリから [fd パッケージ](https://packages.altlinux.org/en/sisyphus/srpms/fd/)をインストールできます:
```
apt-get install fd
```

### Solus の場合

公式リポジトリから [fd パッケージ](https://github.com/getsolus/packages/tree/main/packages/f/fd)をインストールできます:
```
eopkg install fd
```

### RedHat Enterprise Linux (RHEL) 8/9/10、Almalinux 8/9/10、EuroLinux 8/9、Rocky Linux 8/9/10 の場合

Fedora Copr から [`fd` パッケージ](https://copr.fedorainfracloud.org/coprs/tkbcopr/fd/)をインストールできます。

```bash
dnf copr enable tkbcopr/fd
dnf install fd
```

jemalloc の代わりに[より低速な](https://github.com/sharkdp/fd/pull/481#issuecomment-534494592) malloc を使用した別バージョンも、
`fd-find` パッケージとして EPEL8/9 リポジトリから利用可能です。

### macOS の場合

[Homebrew](https://formulae.brew.sh/formula/fd) で `fd` をインストールできます:
```
brew install fd
```

... または MacPorts で:
```
port install fd
```

### Windows の場合

[リリースページ](https://github.com/sharkdp/fd/releases)からビルド済みバイナリをダウンロードできます。

または [Scoop](http://scoop.sh) で `fd` をインストールできます:
```
scoop install fd
```

または [Chocolatey](https://chocolatey.org) で:
```
choco install fd
```

または [Winget](https://learn.microsoft.com/en-us/windows/package-manager/) で:
```
winget install sharkdp.fd
```

### GuixOS の場合

公式リポジトリから [fd パッケージ](https://guix.gnu.org/en/packages/fd-8.1.1/)をインストールできます:
```
guix install fd
```

### Mise の場合

[mise](https://github.com/jdx/mise) を使って次のようなコマンドで `fd` をインストールできます:
```
mise use -g fd@latest
```

### NixOS / Nix の場合

[Nix パッケージマネージャー](https://nixos.org/nix/)を使って `fd` をインストールできます:
```
nix-env -i fd
```

### Flox の場合

[Flox](https://flox.dev) を使って Flox 環境に `fd` をインストールできます:
```
flox install fd
```

### FreeBSD の場合

公式リポジトリから [fd-find パッケージ](https://www.freshports.org/sysutils/fd)をインストールできます:
```
pkg install fd-find
```

### npm から

Linux と macOS では、[fd-find](https://npm.im/fd-find) パッケージをインストールできます:

```
npm install -g fd-find
```

### ソースから

Rust のパッケージマネージャー [cargo](https://github.com/rust-lang/cargo) を使って *fd* をインストールできます:
```
cargo install fd-find
```
Rust バージョン *1.77.2* 以降が必要です。

ビルドには `make` も必要です。

### バイナリから

[リリースページ](https://github.com/sharkdp/fd/releases)には Linux、macOS、Windows 向けのコンパイル済みバイナリが含まれています。
静的リンクされたバイナリも利用可能です: ファイル名に `musl` が含まれるアーカイブを探してください。

## 開発

```bash
git clone https://github.com/sharkdp/fd

# ビルド
cd fd
cargo build

# ユニットテストと統合テストの実行
cargo test

# インストール
cargo install --path .
```

### シェル補完

#### リリースアーカイブから

ビルド済みの補完ファイルは、[リリースページ](https://github.com/sharkdp/fd/releases)の
リリースアーカイブ（`.tar.gz`/`.zip`）の `autocomplete` ディレクトリに含まれています。
これらの補完を使用するには:

- **bash**: `fd.bash` ファイルを `~/.bashrc` でソースするか、自動的にソースされるディレクトリに配置します。
- **zsh**: `_fd` を `fpath` 内のディレクトリ（例: `~/.zfunc`）に移動します。
- **fish**: `fd.fish` を `~/.config/fish/completions/` にコピーします。
- **powershell**: `_fd.ps1` を[プロファイルスクリプト](https://learn.microsoft.com/en-us/powershell/scripting/learn/shell/creating-profiles?view=powershell-7.5)の1つからソースします。

#### fd から生成

`fd --gen-completions <shell>` を使って補完を直接生成することもできます:

```bash
# Bash
fd --gen-completions bash > ~/.local/share/bash-completion/completions/fd

# Zsh (ensure ~/.zfunc is in your fpath)
fd --gen-completions zsh > ~/.zfunc/_fd

# Fish
fd --gen-completions fish > ~/.config/fish/completions/fd.fish

# PowerShell
fd --gen-completions powershell >> $PROFILE
```

## メンテナー

- [sharkdp](https://github.com/sharkdp)
- [tmccombs](https://github.com/tmccombs)
- [tavianator](https://github.com/tavianator)

## ライセンス

`fd` は MIT ライセンスと Apache License 2.0 の両方の条件のもとで配布されています。

ライセンスの詳細については [LICENSE-APACHE](LICENSE-APACHE) と [LICENSE-MIT](LICENSE-MIT) ファイルを参照してください。
