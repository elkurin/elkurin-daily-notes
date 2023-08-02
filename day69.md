# テンプレート勉強会 Part1

[base/functional/invoke.h](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/invoke.h)がいいお勉強素材っぽいので読む。  
このファイルはいろんなヘルパーが実装されているっぽい。

## メンバ関数ポインタからクラスの型をゲット
以下は`member_pointer_class_t<T>`でT型がいるクラスを取るためのテンプレート
```cpp=
template <typename DecayedF>
struct member_pointer_class {};

template <typename ReturnT, typename ClassT>
struct member_pointer_class<ReturnT ClassT::*> {
  using type = ClassT;
};

template <typename DecayedF>
using member_pointer_class_t = typename member_pointer_class<DecayedF>::type;
```

`member_pointer_class`は2つの定義を持つ。  
2つ目の方は`ReturnT ClassT::*`でオーバーロードしている。  
この表記は馴染みがなかったが、`ReturnT`型メンバ変数へのポインタである。  
なおメンバ変数へのポインタはメンバ関数ポインタもOK。  
[member access operator](https://en.cppreference.com/w/cpp/language/operator_member_access)を確認すると以下のような例がある。
```cpp=
struct A { int i; };
struct B { void f(); };

int A::* mp = &A::i;
void (B::*mpf)() = &B::f;
```
pointer-to-memberアクセス演算子という名前で紹介されていた。  

これと踏まえた上でもう一度2つ目の`member_pointer_class`を見てみる。
```cpp=
template <typename ReturnT, typename ClassT>
struct member_pointer_class<ReturnT ClassT::*> {
  using type = ClassT;
};
```
`member_pointer_class<F>`のように渡したF型は`ReturnT ClassT::*`になり`ReturnT`と`ClassT`の置き換えにトライする。  
置き換えができたなら`ClassT`が`F`を含むクラスになるので`type`を`ClassT`として宣言し、`::type`からクラス型を取れるようになった。  

一方で`ReturnT`と`ClassT`の置き換えが出来ないケースもある。  
メンバ変数ポインタはクラスにしか定義されない演算子なので、もし`F`がクラスのメンバでなければ置き換え失敗し、SFINAEのやつになる。
```cpp=
template <typename DecayedF>
struct member_pointer_class {};
```
フォールバック先は空構造体。

そして`typename member_pointer_class<DecayedF>::type`と`type`値にアクセスする`member_pointer_class_t`を使うと`type`が存在しているときにしかオーバーロードされなくなる。  
この場合`type`はクラスである。

### IsMemFunPtr
では上で言及した`member_pointer_class_t`の使用例を確認する。
```cpp=
template <typename F,
          typename T,
          typename MemPtrClass = member_pointer_class_t<std::decay_t<F>>>
const bool& IsMemPtrToBaseOf =
    std::is_base_of<MemPtrClass, std::decay_t<T>>::value;
```

[`std::is_base_of<Base, Derived>`](https://en.cppreference.com/w/cpp/types/is_base_of)は既出だがBase型がDerived型のBaseかを調べ、std:true_typeかstd::false_typeを返す。  

T型は[`std::decay_t`](https://en.cppreference.com/w/cpp/types/decay)を通ることでなんかいい感じの型になる。関数型は関数ポインタに変換されている。  
というわけで型を渡すとT型の継承元に関数型Fがいるかを調べている。  

`IsMemPtrToBaseOf`の使用例は以下。
```cpp=
template <typename F,
          typename T1,
          typename... Args,
          EnableIf<IsMemFunPtr<F> && IsMemPtrToBaseOf<F, T1>> = true>
constexpr decltype(auto) InvokeImpl(F&& f, T1&& t1, Args&&... args) {
  return (std::forward<T1>(t1).*f)(std::forward<Args>(args)...);
}
```
EnableIfの文で呼び出し先として適切か確認していそう。  

## Reference wrapper
```cpp=
template <typename T>
struct is_reference_wrapper : std::false_type {};

template <typename T>
struct is_reference_wrapper<std::reference_wrapper<T>> : std::true_type {};

template <typename T>
const bool& IsRefWrapper = is_reference_wrapper<std::decay_t<T>>::value;
```
だいぶ読みやすくなった。  
[`std::reference_wrapper`](https://en.cppreference.com/w/cpp/utility/functional/reference_wrapper)は名前の通り参照オブジェクトにラップしてくれる。  
参照オブジェクトに置き換え可能だったら`std::true_type`を返す、つまり渡した型が参照かを確認しているだけ。  

## decltype(auto)
こんな`invoke`テンプレートがあった。
```cpp=
template <typename F, typename... Args>
constexpr decltype(auto) invoke(F&& f, Args&&... args) {
  return internal::InvokeImpl(std::forward<F>(f), std::forward<Args>(args)...);
}
```
`decltype(auto)`というのは初めて見た。  
[`decltype`](https://cpprefjp.github.io/lang/cpp11/decltype.html)はオペランドで指定した式の型を取得する。
```cpp=
int i = 0;
decltype(i) j = 0; 
```
このように`j`の型がintだと指定することができる。

ではautoをいれるとどうなる？  
これ専用のreferenceがあった。-> [decltype(auto)](https://cpprefjp.github.io/lang/cpp14/decltype_auto.html)によると
```cpp=
int a = 3;
int b = 2;

decltype(auto)  d = a + b;
```
こんな感じで右辺の式で置き換えて型推論する機能だった。  
[通常関数の戻り値推論](https://cpprefjp.github.io/lang/cpp14/return_type_deduction_for_normal_functions.html)に使用できる。


## Note
### is_member_object_pointer
IsMemPtrToBaseOfの他に以下のようなヘルパー関数もある。
```cpp=
template <typename F>
const bool& IsMemFunPtr =
    std::is_member_function_pointer<std::decay_t<F>>::value;

template <typename F>
const bool& IsMemObjPtr = std::is_member_object_pointer<std::decay_t<F>>::value;
```
[`is_member_object_pointer`](https://en.cppreference.com/w/cpp/types/is_member_object_pointer)は型Tがメンバ変数へのポインタ型かを調べ、std:true_typeかstd::false_typeを返す。  
[`std::is_base_of`](https://en.cppreference.com/w/cpp/types/is_base_of)でも出てきたが、std:true_typeやstd::false_typeはコンパイル時の条件式を扱うために導入されたヘルパー型。  
