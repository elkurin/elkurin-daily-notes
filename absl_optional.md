optional型とは有効な値Tまたは無効な値を指定できる型。
例えば、ある処理が成功したときはその値を、失敗したときはnullを返すといったことができる。
Chromiumでは昔Chromium独自の実装である[base::optional](https://chromium.googlesource.com/chromium/src/+/71.0.3578.80/docs/optional.md)を使用していたが、何年か前に一斉工事が行われ[absl::optional](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/optional.h)に置き換わった。

現在も[absl::optional](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/optional.h)が引き続き使われているが、標準ライブラリにも[std::optional](https://cpprefjp.github.io/reference/optional/optional.html)は実装されている。

両者を比較するためにまず[absl::optional](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/optional.h)を読んでみる。

## 使い方
std::optionalとほぼ同じ。
以下に個人的によく使う仕様例を示す。
```cpp=
std::optional<int> result = DoSomethingAndReply();

// null checkで値が有効か分かる
  std::cout << result.value() << std::endl;
}
// もちろん逆も同じ
if (!result) {
  // error
}

// optional::value_orで有効ならその値、無効なら指定した値を出してくれる
std::cout << result.value_or(0) << std::endl;

// -> でそのなかの要素に直接アクセスできる。ただしvalidチェックはしないとダメ
std::optional<ClassA> result_class = Hoge();
if (result_class.has_value()) {
  std::cout << result->member() << std::endl;
}

// 値へのreferenceをそのまま渡す。引数の型はClassAになっている。
if (result_class.has_value()) {
  Run(*result_class);
}
```

## 実装
## Constructor
まずは普通のやつをみる。
```cpp=
// 空のを作れる。仕様的にデフォルトは無効値
constexpr optional() noexcept = default;
constexpr optional(nullopt_t) noexcept {}

// copy/move もOK
optional(const optional&) = default;
optional(optional&&) = default;
```

ではここから有効な値で初期化するコンストラクタ。
```cpp=
template <typename InPlaceT, typename... Args,
          absl::enable_if_t<absl::conjunction<
              std::is_same<InPlaceT, in_place_t>,
              std::is_constructible<T, Args&&...> >::value>* = nullptr>
constexpr explicit optional(InPlaceT, Args&&... args)
    : data_base(in_place_t(), absl::forward<Args>(args)...) {}

template <
    typename U = T,
    typename std::enable_if<
        absl::conjunction<absl::negation<std::is_same<
                              in_place_t, typename std::decay<U>::type> >,
                          absl::negation<std::is_same<
                              optional<T>, typename std::decay<U>::type> >,
                          std::is_convertible<U&&, T>,
                          std::is_constructible<T, U&&> >::value,
        bool>::type = false>
constexpr optional(U&& v) : data_base(in_place_t(), absl::forward<U>(v)) {}
```
読めない。

(以下stdのリファレンスを読んでいるが、おおまかな内容は同じはずなので多分OK？)
[enable_if](https://cpprefjp.github.io/reference/type_traits/enable_if.html)型はコンパイル時条件式が真の場合のみ有効になり型Tをメンバ型`type`として定義する。そうでなければ`type`型をメンバに持たなくなる。なのでvalueを使用することでテンプレートの置き換えに成功・失敗の分岐ができSFINAEが使えるようになる。

上の例だとabsl::conjunction<...>がtrueならstd::enable_if::typeが存在し置き換えに成功してくれる。`=nullptr`や`=false`をつけてるのはデフォルト値をつけることでユーザーが無視できるようにする、いわばコンパイルを通すためで、特に意味はないはず。

なおこのようなbool値を返すメタ関数を"特性"と呼ぶ。

[conjunction](https://cpprefjp.github.io/reference/type_traits/conjunction.html)は複数の特性の論理積を計算してくれる。[negation]は特性の論理否定を計算する。
is_same, is_constructible, is_convertibleは名前の通り。

いろいろあったがこれらの条件を満たすやつらかをコンパイル時にチェックして、型がマッチしたら[optional_data](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=153-199;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)型にArgsやUをstd::forwardして突っ込む。なおここで使われている[in_place_t](https://cpprefjp.github.io/reference/utility/in_place_t.html)はオーバーロードするための型クラスで要素型のコンストラクタ引数を直接受け取って構築するための関数オーバーロードを定義するためにあるとのこと。まさに今回のケース。
optional_dataはコピーやアサイン、破壊を定義している。とりあえずもらった値を代入してるのは確か。


### アクセス
optional型は`has_value`で値が有効か確認し、有効な場合`value`で値を取れる。
```cpp=
constexpr bool has_value() const noexcept { return this->engaged_; }

// boolへの型変換演算もある
constexpr explicit operator bool() const noexcept { return this->engaged_; }
```

他にも`*`や`->`演算子でアクセスできる。
```cpp=
// -> で値を返す
T* operator->() ABSL_ATTRIBUTE_LIFETIME_BOUND {
  ABSL_HARDENING_ASSERT(this->engaged_);
  return std::addressof(this->data_);
}

// * で値へのreferenceを返す
T& operator*() & ABSL_ATTRIBUTE_LIFETIME_BOUND {
  ABSL_HARDENING_ASSERT(this->engaged_);
  return reference();
}
```
priveta関数の`reference`の実装は以下。
```cpp=
T& reference() { return this->data_; }
```

### value_or
```cpp=
template <typename U>
constexpr T value_or(U&& v) const& {
  static_assert(std::is_copy_constructible<value_type>::value,
                "optional<T>::value_or: T must be copy constructible");
  static_assert(std::is_convertible<U&&, value_type>::value,
                "optional<T>::value_or: U must be convertible to T");
  return static_cast<bool>(*this)
             ? **this
             : static_cast<T>(absl::forward<U>(v));
}
```
optionalの中の値とデフォルト値は型が違っても良いらしい。
が、U->Vが`std::is_converible`でないとダメ。
このチェックはコンパイル時に走る。

## abslのattibutes
[ABSL_ATTRIBUTE_LIFETIME_BOUND](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/base/attributes.h;l=769;drc=b5cd13bb6d5d157a5fbe3628b2dd1c1e106203c6) は参照したオブジェクトが戻り値でも参照されるかもしれない際につけるアトリビュート。[lifetimebounds](https://clang.llvm.org/docs/AttributeReference.html#lifetimebound)のセクションに詳細がある。
