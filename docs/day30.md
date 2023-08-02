# Chromium Component Build

Chromiumのビルドファイルを書いている時、なんかよくわからないが他のBUILD.gnの書き方を真似して書いている。[components/arc/common/BUILD.gn](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/BUILD.gn)は自分でいちから書いたが、正直何をしてるのかわからずに書いている。  
static_libraryって何？source_setって何？  
というわけで、今回はそれらの内容をまとめる。

参考資料：[GN reference](https://gn.googlesource.com/gn/+/main/docs/reference.md#func_shared_library), [The Chrome Component Build](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/component_build.md)

## Component Buildとは
Release buildsのChromiumは1つの実行ファイル（と0~2個の共有ライブラリ）になっている。これは実行時には効率的だがバイナリがでかい分コンパイル時のリンクに時間がかかる。  
ChromiumにはComponent buildというのがある。これは一つのでかい実行ファイルを作る代わりにたくさんの小さい共有ライブラリを生成して実行時に動的なリンクを行う。実行時のパフォーマンスは落ちるが、ちょっと変えてコンパイルする場合影響範囲が少なくコンパイルが早くなるのでdeveloperは嬉しい。実際debugビルドではデフォルトでenableされている。  
args.gnの中で`is_component_build = true`と書けばComponent buildできる。

Componentにしておくと分解してビルドできて嬉しいがいっぱいありすぎると各Componentごとにリンクされるコードがたくさん複製されて共有ライブラリを作ること自体に時間がかかってしまうし、実行時のロード時間もさらに長くなる。  
あんまり量産しないようにしよう。

## BUILD.gnの書き方
BUILD.gnはChromiumでビルドルールを指定するときに書き込むファイル。  
コンパイルされたいファイルはBUILD.gnの中に入れないといけないし、そのファイルのセットがコンパイルされるチェインにいないといけない。  
component buildの他にstatic library, source setなどがある。  
各々GN templateが用意されており、以下のようにBUILD.gnファイルでライブラリのあり方を指定できる。
```python
# 静的ライブラリ
static_library("exo") {
  sources = [
    "buffer.cc",
    ...
  ]
}

# source set. 次のセクションで説明
source_sets("waylnd") {
  sources = [
    "content_type.cc",
    ...
  ]
}

# component build でのcomponent
component("browser") {
  output_name = "chrome_browser"
  sources = ...
  ...
}

# テストセット
test("exo_unittests") {
  ...
}
```
`is_component_build`がtrueなら共有ライブラリとしてビルドし、falseなら静的ライブラリとしてビルドする。  
`output_name`で共有ライブラリの名前を選択できるが、何も指定しなければcomponent()に書かれる名前で作られる。上のケースだと"browser"になる。  

component以外にはstatic_library、source_setなどがある。この2つに依存しているcomponentはリンクされ、複数のcomponentから依存されている場合は複製される。なので複数から依存されるような書き方はあんまり良くない。基本的にはcomponentはcomponentにしか依存しないことが推奨されているとのこと。  

## シンボルのエクスポート
Chromiumのコード中でこんなのをよく見る。
```cpp=
class COMPONENT_EXPORT(UI_DATA_PACK) DataPack {
  ...
};

// Inside BUILD.gn
component("ui_data_pack") {
  defines = [ "IS_UI_DATA_PACK_IMPL" ]
}
```
これは関数やクラスにつけておくことで他の共有ライブラリから使えるようになり、シンボルの値はBUILDファイルのdefinesに記しておく。  
むかしこれをつけなかったことで`is_component_build=true`のテストに落ちてrevertされたことがある。[これ](https://chromium-review.googlesource.com/c/chromium/src/+/3592116/1..2)。  

また、どこからシンボルを使われるかを制限するには`visibility`の値に使用していいcomponentを指定できる。
```python=
component("ui_data_pack") {
  visibility = [
    ":base",
    "//chromeos/crosapi/mojom",
    ...
  ]
}
```

## source_set と static_library
source_setはコンパイルされるが特になんのライブラリも生成しない。そのsource_setに依存しているライブラリすべてに勝手にリンクされる。  
一方static_libraryは ".a" / ".lib" のライブラリファイルを生成する。  
でかい静的ライブラリの生成をスキップできるのでsource_setの方が良いことが多い。

この2つはシンボルの扱いで違いが出るらしい。  
シンボルは基本静的ライブラリからexportされることを想定している。リンカーはシンボルリンクを終えたあとexported関数から届かない不要なコードを消す機能がある。しかしsource setの場合ライブラリファイルを生成せずリンクをスキップするので不要なコード分を消すことはない。つまりシンボルをexportすることは中間状態のターゲットではなく完成形の共有ライブラリからexportされることを保証してくれるとのこと。


### 結局どちらを使うべき？
[参考](https://gn.googlesource.com/gn/+/main/docs/style_guide.md#Source-sets-versus-static-libraries)
よくわからないときはsource_setなら正しく動くのでこっちを選ぼう。  

exportするシンボルを持つ場合はstatic_libraryではなくsource_setを使おう。  
componentが依存する場合はsource_setを使おう。  
Unit testsはsource_setを使おう。  
場合に寄ってはstatic_libraryは全体を何度も複製しうる。これは多分unreachableなdead codeを消すから依存しているcomponentによって消すところがまちまちになるから？  

正しさ的にはsource setが選ばれがち。  
ただし上記のケースに当てはまらず、シンボルリンクがそんなに必要とされない場合、static_libraryを使うとかなりlinkが早くなるらしい。必要のないオブジェクトファイルは無視されるから。  
