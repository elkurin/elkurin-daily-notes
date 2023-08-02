# std::optional vs absl::optional

[absl::optional](/docs/day63.md)と[absl::optional をもっと読もう](/docs/day64.md)のノートでoptionalの実装を読んできた。  
Chromiumでは以前base::optionalを使っていたところから2年前くらいに移行が起きたが、その際std::optionalではなくabslライブラリを選択している。  
その後現在も[std::optionalはban](https://chromium.googlesource.com/chromium/src/+/HEAD/styleguide/c++/c++-features.md#std_optional-banned)されている。  
まだ実装されていないライブラリをChromium独自の実装でなんとかするみたいなことはあるが、どちらも選べるのに標準ライブラリを使っていないのはなぜだろう？  

## 主な理由：チェックの有無
[Chromium の CHECK macro](/docs/day2.md)について触れたが、同様のチェック機能がabslライブラリにもある。  
例えば
```cpp=
const T* operator->() const ABSL_ATTRIBUTE_LIFETIME_BOUND {
  ABSL_HARDENING_ASSERT(this->engaged_);
  return std::addressof(this->data_);
}
```
[`ABSL_HARDENING_ASSERT`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/macros.h;l=124;drc=194f3ad7f8c450799f678b273a5bbf50bbb25d23)はChromiumにおけるCHECKみたいなもの。このチェックは`ABSL_OPTION_HARDENED`のマクロによって制御されており、Chromiumでは[常にenabled](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/options.h;l=230;drc=1b23de37a63fada7b318b1a006480d92819106b8)されている。  
なので、`operator->`演算子では`engaged_`のチェックをしてから`data_`にアクセスしている。
同様に`operator*`でも`engaged_`のtrueチェックをしている。  

一方[std::optional](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/optional)にはなさそう。
```cpp=
constexpr const _Tp*
operator->() noexcept
{ return std::__addressof(this->_M_get()); }
```
`_M_get`は直接valueにアクセスするメソッド。  

base::optionalからabsl::optionalへ移行する際にもチェックができるかの[議論](https://groups.google.com/a/chromium.org/g/cxx/c/ucsnOkZhS2c/m/uj-yLnr7AQAJ)が行われており、多分ここが一番大きな差。  
現状標準ライブラリにはそういうフラグは存在しなさそう([参考](https://groups.google.com/a/chromium.org/g/chromium-dev/c/Ik3uKthTPa4))。

## その他実装の違い
また`constexpr`装飾子関連で違いがある。  
### operator->, *
例えば`operator->`の実装
```cpp=
// absl
T* operator->() ABSL_ATTRIBUTE_LIFETIME_BOUND {
  ABSL_HARDENING_ASSERT(this->engaged_);
  return std::addressof(this->data_);
}

// std
constexpr const _Tp*
operator->() noexcept
{ return std::__addressof(this->_M_get()); }
```
non-constのメンバ関数に標準ライブラリでは`constexpr`があり、abslにはない。（これはC++11とC++14の差に由来するらしい）  


### make_optional
また`make_optional`という、値からoptional型を生成するメソッドがある。  
このメソッドについて、`constexpr auto o = absl::make_optional(v)`が出来ない。  
より正しくはtrivially copyableな型Tに対しては可能だがnon-trivialの場合[copy elision](https://cpprefjp.github.io/lang/cpp17/guaranteed_copy_elision.html)サポートがいる。  

[copy elision](https://cpprefjp.github.io/lang/cpp17/guaranteed_copy_elision.html)とはオブジェクトの初期化のために使用する値のコピーを省略する仕様。  
例えば以下の時
```cpp=
T Func() {return T();} 
T t = Func(); 
```
Func()の中でT型のオブジェクトを作って、`t`にコピーされるように見えるが、このコピーをスキップして直接`t`を初期化してくれる(= コピーコンストラクタが呼ばれない)。  
[copy elision](https://cpprefjp.github.io/lang/cpp17/guaranteed_copy_elision.html)はC++17からサポートされている。ところで[std::optional](https://cpprefjp.github.io/reference/optional/optional.html)もC++17で実装されている。なのでstdでのoptionalはcopy alisionの保証を仮定できるが、abslでは無理っぽい。でnon-trivialな型のコピーはconstexprには許されていないので使えないとのこと。  
 
内部実装の差も見てみる。  
```cpp= 
// absl
template <typename T>
constexpr optional<typename std::decay<T>::type> make_optional(T&& v) {
  return optional<typename std::decay<T>::type>(absl::forward<T>(v));
}

// std
template<typename _Tp>
  constexpr
  enable_if_t<is_constructible_v<decay_t<_Tp>, _Tp>,
      optional<decay_t<_Tp>>>
  make_optional(_Tp&& __t)
  noexcept(is_nothrow_constructible_v<optional<decay_t<_Tp>>, _Tp>)
  { return optional<decay_t<_Tp>>{ std::forward<_Tp>(__t) }; }
```
標準ライブラリの方では[is_constructible](https://cpprefjp.github.io/reference/type_traits/is_constructible.html)でコンストラクタ呼び出しが適格であることを保証している。  
TODO: これはなんで違うんだろう？  

あとはmoveコンストラクタの制約が少し違ったり、throw型がabsl版だったりと差異はある。  
このセクションで言及した違いについてはabslを選択した理由ではなく、ただ存在している実装上のdiffって感じだと思う。  

## Notes
## [`ABSL_HARDENING_ASSERT`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/macros.h;l=124;drc=194f3ad7f8c450799f678b273a5bbf50bbb25d23)の実装
実装は以下。  
```cpp=
#if ABSL_OPTION_HARDENED == 1 && defined(NDEBUG)
#define ABSL_HARDENING_ASSERT(expr)                 \
  (ABSL_PREDICT_TRUE((expr)) ? static_cast<void>(0) \
                             : [] { ABSL_INTERNAL_HARDENING_ABORT(); }())
#else
#define ABSL_HARDENING_ASSERT(expr) ABSL_ASSERT(expr)
#endif
```
[ABSL_OPTION_HARDENED](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/options.h;l=230;drc=1b23de37a63fada7b318b1a006480d92819106b8)はChromiumでは常にenableなのでなので`NDEBUG`フラグ次第。  
Releaseビルドなら[`ABSL_ASSERT`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/macros.h;l=81;drc=194f3ad7f8c450799f678b273a5bbf50bbb25d23)と同じで`(false ? static_cast<void>(expr) : static_cast<void>(0))`が中に入る。つまり何もしない。  
デバッグビルドなら[`ABSL_INTERNAL_HARDENING_ABORT`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/macros.h;l=106;drc=194f3ad7f8c450799f678b273a5bbf50bbb25d23)でabortする。  

[`ABSL_PREDICT_TRUE`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/base/optimization.h;l=178;drc=65174ec3c5de38fb72fae00cdb1091789056b1b1)は`(__builtin_expect(false || (x), true))`でコンパイル最適化用のフラグ。CHECKのときに見た。分岐予測先をコンパイラに明示的に与えることができる。  

### std::optionalへの移行？
ドキュメントには  
> Will be allowed soon; for now, use absl::optional

とある。  
将来的には移行するらしい->[tracking bug](https://bugs.chromium.org/p/chromium/issues/detail?id=1373619)。  
ただし現状はCheckがないので移行できないらしく、we are sure will work in the same wayらしい。
