# absl::optional をもっと読もう

昨日の[absl::optional](/docs/day63.md)ではctor、アクセスなどのメソッドの実装テンプレートを読んだ。  
今日はさらにoptional_internalの中身と、データをどう格納しているかを確認する。

## optional_dataの型
実際のデータは[optional_data_dtor_base](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=48;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)のクラスで定義されている。  
[optional_data_dtor_base](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=48;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)は[optional_data_base](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=117;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)->[optional_data](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=153;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)と継承され、absl::optionalの中では[`data_base`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/optional.h;l=124;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)として使われている。  
なぜこんな継承の仕方をしているかというと、高階メタ関数みたいなやつ。それぞれの型の特性に寄って実装の中身を変えるために階層化されている。  

まず[optional_data_dtor_base](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/types/internal/optional.h;l=48;drc=714e1e34e823b7b4bc172fa4bdefee5aaf5e5fff)は2つの実装を持っていて`bool unused = std::is_trivially_destructible<T>::value`がマッチするかどうかで分岐する。  [is_trivially_destructible](https://cpprefjp.github.io/reference/type_traits/is_trivially_destructible.html)は名前の通りT型がtrivialに破壊できる(ユーザー定義されないデストラクタがあるか)か調べており、可能な場合`::value`がtrueになりそうでないとfalseになる。  
`optional_data_dtor_base<T>`をまず
```cpp=
template <typename T>
class optional_data_dtor_base<T, true> {
```
に当てはめると二引数のoptional_data_dtor_baseになる。つまり
```cpp=
template <typename T, bool unused = std::is_trivially_destructible<T>::value>
class optional_data_dtor_base {
```
ここになり、もし`std::is_trivially_destructible<T>::value`がtrueならより特殊化されている`<T, true>`の方が選ばれ、そうでなかったら`bool unused = std::is_trivially_destructible<T>::value`が使われる。(参照：[特殊化templateの順序](/docs/day42.md))  

この階層を設けることでTがtrivially destructibleかどうかで実装を分けることが出来た。
では１つ上の`optional_data`階層では何で分岐しているかというと  
```cpp=
template <typename T,
          bool unused = absl::is_trivially_copy_constructible<T>::value&&
              absl::is_trivially_copy_assignable<typename std::remove_cv<
                  T>::type>::value&& std::is_trivially_destructible<T>::value>
class optional_data;
```
[`is_trivially_copy_constructible`](https://cpprefjp.github.io/reference/type_traits/is_trivially_copy_constructible.html)かつ[`absl::is_trivially_copy_assignable`](https://cpprefjp.github.io/reference/type_traits/is_trivially_copy_assignable.html)。  
optionl_data_dtor_baseと同様に`class optional_data<T, true>`と`class optional_data<T, false>`がありそれぞれの実装が書いてある。  
ただ、`optional_data_dtor_base`と異なり、2変数テンプレートを分離して宣言して`<T, true>`用と`<T, false>`をどちらも宣言して実装している。ただの表記ブレですか？<- TODO(hidehiko)  

## データの格納の仕方
その中でどうデータが定義されているか。
```cpp=
bool engaged_;
union {
  T data_;
  dummy_type dummy_;
}
```
`engaged_`は以下の使用例からもわかるとおり、値が有効かどうかをbool値でもっている。
```cpp=
constexpr explicit operator bool() const noexcept { return this->engaged_; }
constexpr bool has_value() const noexcept { return this->engaged_; }
```

ではその下にあるunionは何？  
まずここでの目標はoptional型のコンストラクタでT型のオブジェクトをconstructしてしまいたくない。なので以下のようなナイーブな実装だと駄目。
```cpp=
bool engaged_;
T data_;
```

constructしたくないならplacement new([Chromium の singleton](/docs/day9.md)で言及)の出番では？  
実際[boost::optional](https://www.boost.org/doc/libs/1_34_1/boost/optional/optional.hpp)では[aligned_storage](https://cpprefjp.github.io/reference/type_traits/aligned_storage.html)タイプとともにこの方法が取られている。  
```cpp=
template<class T>
class optional_base : public optional_tag {
  typedef aligned_storage<T> storage_type ;
  ...
  void construct ( argument_type val )
  {
    ::new (m_storage.address()) value_type(val) ;
    m_initialized = true ;
  }
  ...
  bool m_initialized ;
  storage_type m_storage ; 
}
```
boostのコーディングスタイルなんかきしょいね。  

しかしabsl::optionalやstd::optionalでもこの書き方はされずunionを使っている。  
[stack overflow](https://stackoverflow.com/questions/52338255/stdoptional-implemented-as-union-vs-char-aligned-storage)の記事を参考にすると  
* Tがtrivially copyableのとき`optional<T>`もtrivially copyableでないといけないから 
* constexpr はnewやreinterpret_castを使ってはいけないが、constexpr optionalを宣言する時newでオブジェクトを作ってreinterpret_castでもとに戻すみたいなことを外側でしないといけない

らしい。(constexprでoptional宣言したいことある？)  
いずれにしてもplacement newのコードなんか読みにくくて汚い。  
一方unionだとどうなるか。[`union`](https://en.cppreference.com/w/cpp/language/union)は[base::small_map](/docs/day44.md)のノートで言及したが複数の型のどちらかを持つ型。今回の使用例だと有効な値`data_`か空っぽの値`dummy_`かのどちらかとなっている。サイズはT型の領域とっている。  
```cpp=
struct empty_struct {};

struct dummy_type {
  empty_struct data[sizeof(T) / sizeof(empty_struct)];
};

// 再掲
union {
  T data_;
  dummy_type dummy_;
}
```

初手は`dummy_`がコンストラクトされる。
```cpp=
// unionはplacement newで定義される。
// dummy_のアドレスを使用することでcv-qualified(const/volatile装飾子がついている)
// T* を void* へキャストできる
template <typename... Args>
void construct(Args&&... args) {
  ::new (static_cast<void*>(&this->dummy_)) T(std::forward<Args>(args)...);
  this->engaged_ = true;
}

// constexprでは特殊にcontructしてあげとく
// dummy_は空っぽクラスなのでコンストラクタ呼んでもOK
constexpr optional_data_dtor_base() noexcept : engaged_(false), dummy_{{}} {}
```
こうして最初にからっぽの`dummy_`を身代わりでplacement newし、Tを作ってしまうのを避ける。  

破壊時も`dummy_`は破壊する必要がなく、`engaged_`がtrueなら`data_.~T();`で破壊しておく。trivially destructibleならこれすらいらん。

なお`dummy_type`で配列を使っているが、これはGCC6だとplcement-newでwarningが出るかららしい。

## Note
[aligned_storage](https://cpprefjp.github.io/reference/type_traits/aligned_storage.html)はC++23以降非推奨らしい。  
理由は未定義動作を引き起こす、保証が正しくない、APIがいろいろ無茶苦茶、とのこと。

TODO(elkurin): constexprについて最新情報で再度確認。
