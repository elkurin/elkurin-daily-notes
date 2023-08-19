# ScopedTempDir in Chromium

テストを書く際に度々ディレクトリやファイルを作りたくなる。  
例えば、User Dataが入っているディレクトリがプロダクションにはあるが、テスト環境でもダミーあるいは実際に利用するために用意しておきたい。  
しかしテストのたびにディレクトリを作ってそのままにしておくとゴミがパソコンにたくさん残ってしまうことになる。  
そんなときに使えるのが[base::ScopedTempDir](https://source.chromium.org/chromium/chromium/src/+/main:base/files/scoped_temp_dir.h;drc=e4622aaeccea84652488d1822c28c78b7115684f)。

## 概要
/tmpの下にtemp directoryを作ってくれるオブジェクト。  
スコープを出てオブジェクトが破壊されると、作成したtemp directoryも消してくれる。

こんな感じで使う。
```cpp=
{
  base::ScopedTempDir dir;

  CHECK(dir.CreateUniqueTempDir());
  const base::FilePath& dir_path = dir.GetPath();
} // dir is destructed, and the directory is removed.
```

コンストラクタは特に何もしない。  
`CreateUniqueTempDir`や`CreateUniqueTempDirUnderPath`でディレクトリを作る。

## 実装
### ディレクトリ作成
[CreateUniqueTempDir](https://source.chromium.org/chromium/chromium/src/+/main:base/files/scoped_temp_dir.cc;l=36;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)は、すでにディレクトリが作られている場合はfalseを返して終了。  
なければ[file_util::CreateNewTempDirectory](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_util.h;l=439;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)で作る。  
この関数はposix版とwin版で別々の実装がある。Posixを見てみよう。

まず[GetTempDir](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_util_posix.cc;l=634;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)で/tmpの場所を取る。  
正確には[getenv](https://www.ibm.com/docs/ja/zos/2.2.0?topic=functions-getenv-get-value-environment-variables)で[TMPDIR](https://www.ibm.com/docs/ja/idr/11.4.0?topic=windows-setting-tmpdir-environment-variable-linux-unix)環境変数の値を獲得しているので、もしこの環境変数に違う場所を指定していればそこを返す。デフォルトは/tmp。  
GetTempDirでとってきたパスに対し[CreateTemporaryDirInDirImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_util_posix.cc;l=715;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)を呼ぶ。  
この中では毎度おなじみ[ScopedBlockingCall](https://source.chromium.org/chromium/chromium/src/+/main:base/threading/scoped_blocking_call.h;l=92;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)でブロックする。これは[mkdtemp](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/mkdtemp.3.html)を呼ぶため。  
[mkdtemp](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/mkdtemp.3.html)とは他と重ならない一時的なディレクトリを作成するライブラリ関数(man3なのでね)。  
最後の6文字が`XXXXXX`になっているような文字列`template`を受け取り、最後の6文字は適宜他と重ならないように文字を当てはめてディレクトリを作る。成功すると`template`の値を実際のパス名に置き換えて返す。  
`template`の値の制約に合わせるため`sub_dir`の値は`GetTempDir() + GetTempTemplate()`になっている。ちなみにGetTempTemplateの中身は以下。
```cpp=
base::FilePath GetTempTemplate() {
  return FormatTemporaryFileName("XXXXXX");
}
```
ここまでやってスレッドのブロックを解除。

### ディレクトリの破壊
[~ScopedTempDir](https://source.chromium.org/chromium/chromium/src/+/main:base/files/scoped_temp_dir.cc;l=31;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)の中で破壊している。  
[Delete](https://source.chromium.org/chromium/chromium/src/+/main:base/files/scoped_temp_dir.h;l=48;drc=e4622aaeccea84652488d1822c28c78b7115684f)を明示的に呼ぶことで破壊もできる。  
Deleteはデストラクタの内部実装でも呼ばれている。

ディレクトリの破壊時は、recursivelyに破壊しないと行けないことに注意。  
[DoDeleteFile](https://source.chromium.org/chromium/chromium/src/+/main:base/files/file_util_posix.cc;l=279;drc=48290eaa6a80203b317bff0b5e6cf91005602c41)で破壊が行われるがここでもまたScopedBlockingCall。  
まずファイルの状態を[lstat](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/stat.2.html)システムコールで取得、もしなんらかのエラーがかえってきたら即終了。  
`S_ISDIR`で確認しもしパスがディレクトリでなければ[unlink](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/unlink.2.html)システムコールでファイルを削除。unlinkはもしその名前が最後のリンクでどのプロセスもファイルをopenしていなければ、ファイルを削除してくれるやつ。  
もしディレクトリであれば、recursivelyに消していかないといけないので子孫ファイル・ディレクトリたちを[File Enumerator](/docs/day52.md)を使って列挙 、各々ファイルであればunlinkを呼び、ディレクトリはスタックに放り込む。  
このスタック`directories`は全ファイルの削除を終えた後に順番に[rmdir](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/rmdir.2.html)を呼んで消している。  
ここまでやってスレッドのブロックを解除。

なお[Take](https://source.chromium.org/chromium/chromium/src/+/main:base/files/scoped_temp_dir.h;l=52;drc=e4622aaeccea84652488d1822c28c78b7115684f)を呼ぶとディレクトリのownershipを取ることができる。こうしておけば、ScopedTempDirのオブジェクトが破壊された後でもディレクトリが生き残る。  
Takeの実装では[std::exchange](https://en.cppreference.com/w/cpp/utility/exchange)再登場。
