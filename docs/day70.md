# 完☆全☆転☆送

Perfect Forwardingは日本語では「完全転送」と呼ぶらしい。


## 導入：T&&
今日のChromium Code Reading会で出た話題。  
先日読んだ[absl::optional](/G7kaKvL5R8C4M9Cuqy2Cgg)の中のコードで以下のような形が合った。
```cpp=
template <typename T>
constexpr optional<typename std::decay<T>::type> make_optional(T&& v) {
  return optional<typename std::decay<T>::type>(absl::forward<T>(v));
}
```
この`T&&`は実は右辺値参照ではない。じゃあ何？

## 右辺値参照
まずは前提のおさらい。

右辺値(rvalue)・左辺値(lvalue)とは、プログラムでいう `auto a = A();` のような式における左と右の概念と対応している。  
右辺値は一時的なオブジェクトで、変数に代入する前の式。  
左辺値は明示的に実体のある名前付きオブジェクト。

```cpp=
int i = 0;

i; // 左辺値
0; // 右辺値（リテラル）
i + i; // 右辺値

A a;

a; // 左辺値
A(); // 右辺値

f(); // 右辺値
```

右辺値参照は今説明した右辺値をその値のみ束縛する参照。  
`T&&`という書き方をされるやつ。対してよく見る`T&`は左辺値参照。  

```cpp=
int x = 0;

// 左辺値参照。ｘは左辺値
int& lvalue = x;

// 右辺値参照。0は右辺値
int&& rvalue = 0;
```

なお`rvalue`は左辺値である。なので`int&& hoge = rvalue` は無理。

## テンプレートにおける T&&
上では普通の`T&&`の用法を見た。  
これがテンプレートになると全く異なる挙動をするようになる。

[Universal Reference](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)とはテンプレートの型Tや`auto`に&&をつけて宣言したやつの名前。  
```cpp=
template <typename T>
void f(T&& x) {}
```
どのへんがUniversalかと言うと、右辺値が渡ってきたときは右辺値参照になり、左辺値が渡ってきたときは左辺値参照になってくれるところ。  
これはなぜ？とかでなくそういう仕様。

この特性を使用することで今日のお題が出来るようになる。

**完☆全☆転☆送**

目標はパラメータをそのままの形で転送すること。  
ここでの「そのままの形」とは右辺値は右辺値のまま、左辺値は左辺値のままということ。  
これを完☆全☆転☆送と呼ぶ。

[Universal Reference](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)を使用すると右辺値・左辺値を維持したまま渡すことができ、受け取った先で型情報マッチが出来る。

しかしこれだけだと、右辺値を右辺値参照で受け取った後そのまま引数を使えばその引数は左辺値になってしまっている。右辺値をキープしたい。  
これは[`std::forward`](https://en.cppreference.com/w/cpp/utility/forward)を使用する。

```cpp=
template<class T>
void wrapper(T&& arg)
{
  // argが左辺値参照なら左辺値に変換
  // argが右辺値参照なら右辺値に変換
  foo(std::forward<T>(arg)); 
}
```

このとき右辺値が渡されたならmoveがおきる。  
したがって以下のようなコードはアウト。
```cpp=
template <class T>
void bad(T&& x) {
  // xが右辺値の時moveされる
  f(std::forward<T>(x));
  // もうmoveされてるからアカン
  g(std::forward<T>(x)); 
}
```


## 疲れた
今日は疲れたので仕事をサボってお掃除してた。  
//chrome/browser/lacrosからdangling ptrがいなくなりました。  
//chrome/browser/ashは魔境。
