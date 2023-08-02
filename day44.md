# base::small_map

[Chromium Maps](/szPe4BDiSAqq2Lk1DSycHw)でいろいろなmapの比較を確認した。  
その中で[base::small_map](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h)だけよくわからなかったので、読んでみる。

## コンセプト
[base::small_map](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h)は、要素数が少ない間はarrayでもって、上限を超えたら他のmap型にフォールバックするバランス型のクラス。  
なんのmap型にフォールバックするかは宣言時に選択し、arrayで持つ上限の要素数も指定できる。

## クラス宣言
small_mapクラスのテンプレートは以下。
```cpp=
template <typename NormalMap,
          size_t kArraySize = 4,
          typename EqualKey = typename internal::select_equal_key<
              NormalMap,
              internal::has_key_equal<NormalMap>::value>::equal_key,
          typename MapInit = internal::small_map_default_init<NormalMap>>
class small_map {
  ...;
};
```

１つ目のテンプレート引数`NormalMap`はフォールバック用のmapの型を指定している。  
またmap型で指定された `iterator::value_type` を値の方としてarrayでも使用する。  
それ以外はデフォルト値を持っている。  
`kArraySize`はarrayの上限を指定しており、この値を書き換えている使用例もある。  
```cpp=
// kMaxNumSpatialLayersPlusOne を指定している
using InputSurfaceMap = base::small_map<
    std::map<gfx::Size, std::unique_ptr<ScopedVASurface>, SizeComparator>,
    kMaxNumSpatialLayersPlusOne>;
```

## 実装
デフォルトで使用するarrayとフォールバック用のmapは以下のように宣言されている。  
```cpp=
union {
  value_type array_[kArraySize];
  NormalMap map_;
};
```
`array_`はサイズ`kArraySize`の固定長配列。  
`map_`はNormalMap型で宣言されたなんかのマップ。

ところで[`union`](https://en.cppreference.com/w/cpp/language/union)という型は共用体と呼ばれるもので、複数の方のどれかを格納したい際に用いる。  
初期化時は最初のメンバを持ち、その後は別のデータメンバに値を代入するとそのメンバがactivateされ、もとのやつは寿命が尽きる。なので、共用体のサイズは、メンバの中でもっとも大きいやつのサイズになっている。宣言は`struct`と同じように書ける。細かい仕様はまた調べてみる。

ここではまず`array_`がサイズ`kArraySize`で初期化される。  
要素数が`kArraySize`を超えると、例えば以下のように`map_`がアクティブになる。
```cpp=
data_type& operator[](const key_type& key) {
  ...;
  if (size_ == kArraySize) {
    ConvertToRealMap();
    return map_[key];
  }
}
```
`map_`への切り替えは[ConvertToRealMap()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h;l=551;drc=255b4e7036f1326f2219bd547d3d6dcf76064870)が行う。  
`array_`の中身を`map_`に移し替えないといけないのでいきなりdeactivateするのはダメ。
まず`array_`から`temp`にすべて要素を移し替え`map_`をactivateしたら`temp`の値を挿入していっている。  
この実装の中では`temp`を意図的にunionとして宣言している。これは`kArraySize`分の要素のコンストラクタを呼ぶコストを避けて、ただコピーするだけにしたいから。  
その後以下の処理を行う。
```cpp=
size_ = kUsingFullMapSentinel;
// `map_`を初期化しておく。
functor_(&map_);
```

unionのどちらがアクティブかのフラグは[`size_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h;l=539;drc=255b4e7036f1326f2219bd547d3d6dcf76064870)で管理している。  
`array_`使用時は`size_`はkArraySize以下の値で、`map_`を使用し始めたら[`kUsingFullMapSentinel`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h;l=19;drc=255b4e7036f1326f2219bd547d3d6dcf76064870)という`size_t`型の最大値を代入して、`map_`を使用するかのフラグは `size_ == kUsingFullMapSentinel` で管理する。  

### Look up
keyをもらってdataを返す関数。

Arrayのときとmapのときで挙動が変わる。  
mapのときは普通にmapのそれぞれの型のlook upを使用する。  
arrayの際はイテレータを後ろから探している。つまり最近追加された要素を早く探索しようとしている。
```cpp=
data_type& operator[](const key_type& key) {
  // mapのとき
  if (UsingFullMap()) {
    return map_[key];
  }

  // arrayのとき
  for (size_t i = size_; i > 0; --i) {
    const size_t index = i - 1;
    if (compare(array_[index].first, key)) {
      return array_[index].second;
    }
  }
}
```

### 挿入
key-valueのペアであるvalue_typeを挿入する関数。

Loop upと同様にmapのときはその型のinsertを参照するだけ。  
arrayのときは、初期化時に必要な要素分静的な配列を作っているので、適切なイテレータにつっこむだけ。  
すでに存在する場合は挿入は行わないで対応するイテレータをそのまま返す。
```cpp=
std::pair<iterator, bool> insert(const value_type& x) {
  // mapのとき
  if (UsingFullMap()) {
    std::pair<typename NormalMap::iterator, bool> ret = map_.insert(x);
    return std::make_pair(iterator(ret.first), ret.second);
  }
  
  // すでに登録済みな時
  for (size_t i = 0; i < size_; ++i) {
    if (compare(array_[i].first, x.first)) {
      return std::make_pair(iterator(array_ + i), false);
    }
  }
  
  // arrayのとき
  new (&array_[size_]) value_type(x);
  return std::make_pair(iterator(array_ + size_++), true);
}
```

## おまけ
### サンプルコード
いろんなmapの宝庫みたいなファイルを見つけた。  
[wayland_buffer_manager_gpu.h](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/gpu/wayland_buffer_manager_gpu.h;l=306-325;drc=15246b4169c99b4eb14176e2d20ad33553fdc7b7):
```cpp=
std::map<gfx::AcceleratedWidget, WaylandSurfaceGpu*>
    widget_to_surface_map_;  // Guarded by |lock_|.

base::flat_map<gfx::BufferFormat, std::vector<uint64_t>>
    supported_buffer_formats_with_modifiers_;

base::small_map<std::map<gfx::AcceleratedWidget,
                         scoped_refptr<base::SingleThreadTaskRunner>>>
    commit_thread_runners_;
```

### numeric_limits
[`kUsingFullMapSentinel`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/small_map.h;l=19;drc=255b4e7036f1326f2219bd547d3d6dcf76064870)の初期化に `std::numeric_limits<size_t>::max()` という標準ライブラリを使っていた。  
これは実装が提供する算術型の性質を提供するライブラリで、min, max, lowest,is_exactなどいろいろな特徴を返してくれる。
