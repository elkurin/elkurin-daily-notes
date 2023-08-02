# File Enumerator

[StatefulLacrosLoader](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/stateful_lacros_loader.cc)の実装中に[base::FileEnumerator](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator.h)を使った。

```cpp=
bool IsInstalledMayBlock(const std::string& name) {
  ...;
  base::FilePath component_root =
    root.Append(component_updater::kComponentsRootPath).Append(name);
  ...;
  base::FileEnumerator enumerator(component_root, /*recursive=*/false,
                                  base::FileEnumerator::DIRECTORIES);
  base::FilePath path = enumerator.Next();
  return !path.empty();
}
```
FileEnumeratorはディレクトリの中身を確認するために使っているが、もともとあった実装を流用したのであまりよくわかっていない。

## Overview
[base::FileEnumerator](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator.h)はファイルを順不同に列挙するクラス。Blockingコールなので注意。  
コンストラクタは以下。
```cpp=
FileEnumerator(const FilePath& root_path,
               bool recursive,
               int file_type,
               const FilePath::StringType& pattern,
               FolderSearchPolicy folder_search_policy,
               ErrorPolicy error_policy);
```
各パラメータは検索条件の指定に用いられている。  
`root_path`の中でファイルを検索する。  
`recursive`はtrueになってるとサブディレクトリまで全部探索する。  
`file_type`はenum [FileType](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator.h;l=87;drc=28f386520cb1b5cd2b545f0e06f4961678bdbc9e)からなるビットマップ。`FILE`, `DIRECTORIES`, `INCLUDE_DOT_DOT`, `NAME_ONLY` and `SHOW_SYM_LINKS`がある。`INCLUDE_DOT_DOT`は`../`みたいなやつ。このビットマップにマッチするやつだけ検索する。  
`pattern`もlsを走らせるときにのような正規表現をいれることができ、例えば`FILE_PATH_LITERAL("*.txt")`とすれば`.txt`で終わるファイルを検索する。

## 実装
ファイル検索はプラットフォーム依存なので、[posixバージョン](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator_posix.cc)と[windowsバージョン](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator_win.cc)がある。  
Posixの実装のみ見ていく。  
[Stat](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/stat.2.html)のsyscallを使っており、ディレクトリかのチェックは[S_ISDIR](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_enumerator_posix.cc;l=62;drc=4e86105c35896cab92cb6c6792bc5a6f56726cab)  
またファイルマッチはFileEnumeratorの中でやるのではなく[FNMATCH](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/fnmatch.3.html)という~~syscall~~libcのAPIで計算している。  

各ファイルの検索結果は[FileInfo](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=48;drc=547c1644798f475a0cafc54253dcfa052e6946c6)型のインスタンスとして保持され、[`stat_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=82;drc=547c1644798f475a0cafc54253dcfa052e6946c6)と[`filename_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=83;drc=547c1644798f475a0cafc54253dcfa052e6946c6)で持つ。  
`stat_`はManPageで確認できる[Stat](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/stat.2.html)。  

検索はディレクトリごとに行われる。現在のディレクトリ内のファイル情報はvectorとして[`directory_entries_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=220;drc=547c1644798f475a0cafc54253dcfa052e6946c6)として列挙しており、まだ検索していないサブディレクトリを[`pending_paths_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=241;drc=547c1644798f475a0cafc54253dcfa052e6946c6)で持っておく。  
さらに[`visited_directories_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=226;drc=547c1644798f475a0cafc54253dcfa052e6946c6)に`std::unordered_set`としてすでに調べたディレクトリを保存しているが、これは循環symlinksによる無限ループを回避するため。  

実際のイテレーションをみてみる。  
[FileEnumerator::Next](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator_posix.cc;l=144;drc=547c1644798f475a0cafc54253dcfa052e6946c6)で次のファイルを見る。この間[ScopedBlockingCall](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/scoped_blocking_call.h;l=92;drc=547c1644798f475a0cafc54253dcfa052e6946c6)を使ってブロックしている。  
`current_directory_entry_`をインクリメントして[`directory_entries_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=220;drc=547c1644798f475a0cafc54253dcfa052e6946c6)を確認。まだ要素があれば`root_path_.Append(directory_entries_[current_directory_entry_].filename_)`をすぐ返す。  
もし現ディレクトリを走査し終えていたら[`pending_paths_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=241;drc=547c1644798f475a0cafc54253dcfa052e6946c6)を確認。それも空ならemptyのFilePath()を返す。  
まだあれば`root_path_`を次に走査するサブディレクトリに更新しここで[`directory_entries_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.h;l=220;drc=547c1644798f475a0cafc54253dcfa052e6946c6)を作成する。  
[OPENDIR](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/opendir.3.html)でDIRをゲットし、[READDIR](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/readdir.3.html)で読み込む。その結果からこのタイミングで[ShouldSkip](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator.cc;l=14;drc=547c1644798f475a0cafc54253dcfa052e6946c6)や[IsPatternMatched](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/files/file_enumerator_posix.cc;l=256;drc=547c1644798f475a0cafc54253dcfa052e6946c6)でフィルターをかけている。  
新しいサブディレクトリを開くときは結構コストの大きいタスクになるが、それ以外のタイミングはvectorのindexを一つ動かすだけなので一瞬で終わる。  

## Note
Man Pageのなかでman2のやつはsyscall、man3のやつはlibcライブラリらしい。  
それ以外のセクションは以下のように慣習的に決まっている。  

> 1 ユーザーコマンド (プログラム)
Commands that can be executed by the user from within a shell.
2 システムコール
Functions which wrap operations performed by the kernel.
3 ライブラリコール
All library functions excluding the system call wrappers (Most of the libc functions).
4 スペシャルファイル (デバイス)
Files found in /dev which allow to access to devices through the kernel.
5 ファイルのフォーマットと設定ファイル
Describes various human-readable file formats and configuration files.
6 ゲーム
Games and funny little programs available on the system.
7 概要、約束事、その他
Overviews or descriptions of various topics, conventions and protocols, character set standards, the standard filesystem layout, and miscellaneous other things.
8 システム管理コマンド
mount(8) のような root のみが実行可能なコマンド。

（引用: [MAN-PAGES](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/man-pages.7.html))
