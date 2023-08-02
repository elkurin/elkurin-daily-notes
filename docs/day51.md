# Version in Chromium

BrowserLoaderの実装でVersionに触れる機会が多かったので、これを機に読み込んでみる。

## Chromiumでのバージョン
[chrome://version](chrome://version)へアクセスするとChromeのバージョンが確認できる。
例えば 106.0.5249.119 みたいな感じ。  
このようにChromiumでのVersionは4つの数字からなる。  
それぞれの値の意味は以下  
* 1つ目: Major -> マイルストーン。M117とか言われてるやつ。
* 2つ目: Minor -> 何これ。だいたいいつも0
* 3つ目: Build -> 通算でインクリメントされている値。1日に1,2個できるはず。
* 4つ目: Patch -> 基本新しいチェンジはBuildをインクリメントして追加されるが、バグフィックスなど同じBuildの中で更新したい場合ここをインクリメントする。

## base::Version
[base::Version](https://source.chromium.org/chromium/chromium/src/+/main:base/version.h)がVersionを持つ型。  

実際の値は [components_](https://source.chromium.org/chromium/chromium/src/+/main:base/version.h;l=64;drc=075b4951b25833945242e9c0ba96d59cd8cc7c79)にstd::vector<uint32_t>として保存している。  
要素数には1以上なら特に縛りがない。[base::Version::IsValid](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/version.cc;l=98;drc=547c1644798f475a0cafc54253dcfa052e6946c6)は`components_.empty()`のみのチェックとなっており、invalidなVersionをいい感じに処理したい場合はbase::Version()でempty componentsのインスタンスを作ればよい。  

[ParseVersionNumbers](https://source.chromium.org/chromium/chromium/src/+/main:base/version.cc;l=26;drc=e4622aaeccea84652488d1822c28c78b7115684f)の中でも`version_str`を`.`でスプリットして、その要素の分while文を回しており、Uintに変換できる数字を`.`で分割していればOK。  

Versionの比較は[CompareVersionComponents](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/version.cc;l=56;drc=547c1644798f475a0cafc54253dcfa052e6946c6)でやっており、`components_`のindexが小さい順に比較していっている。  
ChromiumのVersionの場合では`components_[0]`から順にMajor, Minor, Build, Patchの値が入ることになる。

### Note
[DataPackの実装](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/base/resource/data_pack_with_resource_sharing_lacros.h;l=47-57;drc=547c1644798f475a0cafc54253dcfa052e6946c6)でVersionを使用する際は、base::Version型ではなくuint32_tのサイズ4固定長配列として使っている  
これはストラクトのサイズを切り詰めるため。  
```cpp=
struct LacrosVersionData {
  static constexpr size_t kVersionComponentsSize = 4;
  uint32_t version[kVersionComponentsSize];
  ...;
};
```

## ChromeのVersion情報
実際のVersionの値はversion_info名前空間の中で管理されている。  
[version_info::GetVersion](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/version_info/version_info.cc;l=31;drc=547c1644798f475a0cafc54253dcfa052e6946c6)からbase::Version型のバージョンを得られる。PRODUCT_VERSIONのマクロ定数を使用して作られる。  
ちなみに[version_info::GetVersion](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/version_info/version_info.cc;l=31;drc=547c1644798f475a0cafc54253dcfa052e6946c6)は`static base::NoDestructor<base::Version>`でVersionを保存している。Versionの値は不変なのでこれでよい。時間節約。  

またMajor Versionは[GetMajorVersionNumberAsInt](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/version_info/version_info.cc;l=22;drc=547c1644798f475a0cafc54253dcfa052e6946c6)で得られる。

### version_info_values
Versionの値はコンパイル時に生成され`PRODUCT_VERSION`の値に代入される。  
これは components/version_info/version_info_values.h という自動生成ファイルで定義される。  
生成ファイルの中身は以下のような感じ。
```cpp=
#define PRODUCT_NAME "Chromium"
#define PRODUCT_VERSION "117.0.5891.0"
#define LAST_CHANGE "9bdb694ddfaf18188925746b1f071df76ea70069-refs/heads/main@{#1170861}"
#define IS_OFFICIAL_BUILD 0"
```

中身の値は[chrome/VERSION](https://source.chromium.org/chromium/chromium/src/+/main:chrome/VERSION)に書き込んでいるので、上記の生成ファイルはこのデータを基にしている。  
```
// chrome/VERSION の中身
MAJOR=117
MINOR=0
BUILD=5891
PATCH=0
```
chrome/VERSION は Chrome Release Bot によってインクリメントされている。[例](https://chromium-review.googlesource.com/c/chromium/src/+/4685138)  

ファイルの生成ルールは[BUILD.gn](https://source.chromium.org/chromium/chromium/src/+/main:components/version_info/BUILD.gn)に書かれている。
```python=
process_version("generate_version_info") {
  template_file = "version_info_values.h.version"
  sources = [
    "//chrome/VERSION",
    branding_file_path,
    lastchange_file,
  ]
  output = "$target_gen_dir/version_info_values.h"
}
```
process_version テンプレートは [build/util/process_version.gni](https://source.chromium.org/chromium/chromium/src/+/main:build/util/process_version.gni)にある。その中で使用しているスクリプトは[version.py](https://source.chromium.org/chromium/chromium/src/+/main:build/util/version.py)で、`@ + key + @`のような[変換](https://source.chromium.org/chromium/chromium/src/+/main:build/util/version.py;l=70;drc=df413c2e2be0cc7d011e25c05525f87d97b5e6bf)ルールで必要なキーワードを抽出している。実際生成用のファイルである [version_info_values.h.version](https://source.chromium.org/chromium/chromium/src/+/main:components/version_info/version_info_values.h.version)には以下のような感じ。  
```cpp=
#define PRODUCT_VERSION "@MAJOR@.@MINOR@.@BUILD@.@PATCH@"
```
置き換える値は[Fetchvalues](https://source.chromium.org/chromium/chromium/src/+/main:build/util/version.py;l=34;drc=df413c2e2be0cc7d011e25c05525f87d97b5e6bf)->[FetchValuesFromFile](https://source.chromium.org/chromium/chromium/src/+/main:build/util/version.py;l=18;drc=df413c2e2be0cc7d011e25c05525f87d97b5e6bf)でファイルから読み取っている。ここの`file_name`には上述の[chrome/VERSION](https://source.chromium.org/chromium/chromium/src/+/main:chrome/VERSION)が来ており、行ごとに取っているいるので、改行している各値がFetchされ、無事version_info_values.hが作られる。  
