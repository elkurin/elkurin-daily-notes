# PathService in Chromium

Chromiumの中でファイルシステムにアクセスしたいケースがある。  
しかしファイルのパスはプラットフォーム依存で変わるし、毎回文字列を打つのはデザイン的に良くない。
そんなときに使えるのが[PathService](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.h)。

## 概要
[PathService](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.h)はファイルシステム上のパスをキーワードから得られるサービス。  
スレッドセーフ。  

[PathData](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=139;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63) をシングルトンとして持ち、ここに登録されたパスを保存している。
```cpp=
// 親の顔よりは見てないstatic変数によるsingleton
static PathData* GetPathData() {
  static auto* path_data = new PathData();
  return path_data;
}
```
[PathData](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=139;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)の中では複数の[Provider](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=52;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)を持つ。  
各Providerはそれぞれkeyとpathのペアの集合を持っている。  
[Provider](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=52;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)はlinked listで管理されており、次のproviderが`next`ポインタとしてstructの中にある。  

## パスの検索
パスの検索をする関数は[PathService::Get](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=202;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)  
概要は以下のような感じ。
```cpp=
bool PathService::Get(int key, FilePath* result) {
  PathData* path_data = GetPathData();
  ...;
  while (provider) {
    if (provider->func(key, &path))
      break;
    provider = provider->next;
  }
  ...;
  *result = path;
  return true;
}
```
検索のkeyはint型で、その結果を`result`に代入し成功したらtrueを返すAPI。

上述の通り複数のProviderが合体してPathの集合になっている。  
それらすべてから検索しているのが`provider->func`のところ。  
`func`はkeyに対応するパスを探してくれる関数だが、それらを横断して検索できる。  
これはkeyがすべてのProviderの中の唯一の値になるように割り振られているから。  

Providerの中身をみてみよう。
```cpp=
struct Provider {
  PathService::ProviderFunc func;
  RAW_PTR_EXCLUSION struct Provider* next;
#ifndef NDEBUG
  int key_start;
  int key_end;
#endif
  bool is_static;
};
```
この`func`に検索するための関数が入っているわけだが、たいていkeyにたいする返り値を登録しているだけ。  
どうやってこのProviderが登録されているかを次のセクションで見る。


## パスの登録
static関数である[RegisterProvider](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=334;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)からkeyとpathのペア集合を登録できる。  
GetPathData()からPathDataのシングルトンを獲得し、そこに新しく作ったProviderを登録する。  
このProviderを作るためにはProviderFuncとパスのkey集合が必要になる。  
ProviderFuncにはintのkeyに対してそれに対応するFilePathを返すメソッドを作って渡す。  
例えば[chrome_paths.ccのPathProvider]みたいな感じ。  
`switch`して各enumに対するパスを返している。

そして肝心のkey。  
このkeyは先述の検索がうまく動くようにChromium全体で唯一の値になるようにしないといけない。  
これはenumの最初の値に決まった定数を与えることで実現している。  
例えば[chrome_paths.h](https://source.chromium.org/chromium/chromium/src/+/main:chrome/common/chrome_paths.h;l=23;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)には`PATH_START = 1000`, [ash_paths.h](https://source.chromium.org/chromium/chromium/src/+/main:ash/constants/ash_paths.h;l=20;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)には`PATH_START = 7000`, [dbus_paths.h](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/dbus/constants/dbus_paths.h)には`PATH_START = 7200`, [lacros_paths.h](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/lacros/lacros_paths.h)には`PATH_START = 13000`といったようにSTARTの値を指定してなんとかしている。  
なので法外な量のパスを適当につっこむとenumがconflictして壊れる。  
このバグを検出するためのデバッグコードもちゃんとある。  
[RegisterProvider](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=334;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)ではkeyの最初と最後を渡すことになっているが、実はこれはプロダクションでは使われていない。どのProviderに含まれているかの確認はkeyの検索だけで行っているから。  
ただしNDEBUGのフラグがenableされているときだけ保存され、RegisterProviderの中で[coflict check](https://source.chromium.org/chromium/chromium/src/+/main:base/path_service.cc;l=355;drc=0d6e097c1e1aeaa0c4200d8e2b1d887d14f74a63)が行われる。
```cpp=
#ifndef NDEBUG
  Provider *iter = path_data->providers;
  while (iter) {
    DCHECK(key_start >= iter->key_end || key_end <= iter->key_start) <<
      "path provider collision";
    iter = iter->next;
  }
#endif
```
