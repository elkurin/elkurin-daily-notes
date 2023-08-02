# はじめましての標準ライブラリたち Part1

コードを呼んでいて出会った標準ライブラリたちをメモっておいたので、まとめてみた。  
オムニバス形式です。

## [`std::exchange`](https://cpprefjp.github.io/reference/utility/exchange.html)
Dangling ptr回避のCLのために発見した。  
値を書き換えて書き換え前の値を返すライブラリで、以下のようなケースで使った。

```cpp=
std::unique_ptr<View> RemoveView(View* view) {
  ...;
  // raw_ptr を unique_ptr にラップする関数
  return base::WrapUnique(view);
}

RemoveView(view_);
view_ = nullptr;
```

上のような書き方をすると、RemoveView()の返り値でunique_ptrが返ってきておりそれを捨てるとその瞬間`view_`をラップしたunique_ptrが破壊されてしまいこの瞬間だけdanglingしてしまう。  
以下のような書き方をすれば回避は可能。  
```cpp=
auto temp = RemoveView(view_);
view_ = nullptr;
// temp destructed when scoped out
```
ただなんかかっこ悪い。  

というわけで、こう書く。
```cpp=
RemoveView(std::exchange(view_));
```
std::exchange()の評価の段階で`view_`にはnullptrが入っているので、返り値を投げ捨ててもDanglingにならない。  

## [`std::tie`](https://cpprefjp.github.io/reference/tuple/tie.html)
std::tie自体ははじめましてではなかったが知らない用法が合ったので。  

以下のようにtuple型のオブジェクトを生成するやつ。
```cpp=
std::tuple<A, B, C> t = std::tie(a, b, c);
```

それだけでなく以下のように要素を取り出すことにも使える。

```cpp=
std::tie(flags, deadline) = watch_state->GetFlagsAndDeadline();
```
make_pair的なものかと思っていたが違った。

いらない要素がいるなら[`std::ignore`](https://cpprefjp.github.io/reference/tuple/ignore.html)で捨てられる。
```cpp=
// (a, _) = t みたいな感じ
std::tie(a, std::ignore) = t;
```

### [`std::ignore`](https://cpprefjp.github.io/reference/tuple/ignore.html)について
C++日本語リファレンスでは[`std::tie`](https://cpprefjp.github.io/reference/tuple/tie.html)に使うためのものとあるが、別にtupleでなくても使える。  
Chromiumでも以下のような使用例を見つけた。
```cpp=
std::ignore = fd.release();
```
しかし[cppreference.com](https://en.cppreference.com/w/cpp/utility/tuple/ignore)にはこう書いてある。  
> While the behavior of std::ignore outside of std::tie is not formally specified, some code guides recommend using std::ignore to avoid warnings from unused return values of [[nodiscard]] functions.

std::tie以外での使用は未定義らしいが、上の例の使用例はまさに`[[nodiscard]]`回避用。  
[ScopedGeneric::release](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/scoped_generic.h;l=148;drc=e510fa298843c8ff43b538742e4fa16306da265f)には`[[nodiscard]]`マークがついている。

## [`std::next`](https://cpprefjp.github.io/reference/iterator/next.html)
[`flat_tree`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=1036;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)の実装の中で出会った。  
イテレータをn進めたやつを返す。引数なしなら1個だけ進める。
```cpp=
std::vector<int> v = {1, 2, 3};

// 2
auto it1 = std::next(v.begin());
std::cout << *it1;

// 3
auto it2 = std::nect(v.begin(), 2);
std::cout << *it2;
```

## [`std::as_const`](https://cpprefjp.github.io/reference/utility/as_const.html)
constにしてくれるよ！  
```cpp=
iterator itr = const_cast_it(std::as_const(*this).find(key));
```

右辺値参照をconst左辺値参照にする。
```cpp=
T* foo::GetT() const {
  // very complicated implementation
}
T* foo::GetT() {
  // return GetT(); <- compile error
  return std::as_const(*this).GetT();
}
```
この書き方は[Using Const Correctly](https://source.chromium.org/chromium/chromium/src/+/main:styleguide/c++/const.md)というChromium C++ Styleguide でも推奨されている。  

### 余談：返り値・引数const
返り値や引数がconst/constなしのときは`const_cast<T*>`を使えば良い。`  
```cpp=
aura::Window* GetToplevelWindow(aura::Window* window) {
  return const_cast<aura::Window*>(
      GetToplevelWindow(const_cast<const aura::Window*>(window)));
}

const aura::Window* GetToplevelWindow(const aura::Window* window) {
  // Do something
}
```
`const_cast<T*>(hoge)`は`hoge`のconstを外してくれる。  
`const_cast<const T*>(hlge)`は`hoge`にconstをつけてくれる。  

## [`std::less`](https://cpprefjp.github.io/reference/functional/less.html)
`std::less<T>()(a, b)`で `a < b` の結果を返してくれる。  
メンバ関数は operator()だけ。  
特殊化することで好きに順序付けを定義でき、任意のポインタに対する特殊化は全順序を定義することになる。
