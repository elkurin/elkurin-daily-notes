# C++ のキーワードたち

今日はなんとなく知っているがちゃんとわかってないC++のキーワードを確認していく。


## Explicit/implicit ctor
explicitキーワードをつけると、暗黙の型変換を禁止する。  
以下のような感じ。
```cpp=
struct Implicit {
  Implicit(int) {}
  operator bool() const { return true; }
};

struct Explicit {
  explicit Explicit(int) {}
  explicit operator bool() const { return true; }
};

Implicit i = 1; // OK
Explicit e = 1; // コンパイルエラー：コピーコンストラクタはない

i + 1; // OK：boolへの型変換演算子があるのでintとの足し算ができる
e + 1; // コンパイルエラー：明示的な変換出ないとダメ
if (e) {} // OK：if文によるboolへの型変換なので
```
Google Coding Styleの[Implicit Conversions](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions)の章では以下のように書いてある。  
> Type conversion operators, and constructors that are callable with a single argument, must be marked explicit in the class definition. As an exception, copy and move constructors should not be explicit, since they do not perform type conversion.  

一変数の型変換演算子・コンストラクタはexplicitをつけるべし。  

## constexpr
constexprをつけるとコンパイル時に値が決定する定数、実行される関数、リテラルとして振る舞うクラスを定義できる。  
```cpp=
constexpr int square(int x) {
  return x * x;
}

constexpr int c = square(3); // コンパイル時
int r = square(3); // 実行時
```
constexprをつけた関数は左辺がconstexprでもそうでなくても使うことが出来、そうでない場合は普通の関数と同様実行時に走る。  

constexpr関数はvoid型を返せない。  
return文はひとつじゃないとダメ。  
if文のかわりに条件演算子、while/for文の変わりに再帰を利用する。ちなみにこの再帰回数は高々512回になるようコンパイラに推奨されるらしい。あるいは`g++ main.cpp -fconstexpr-depth=1024`をいうオプションで指定する。  

## noexcept
こんな感じでよく見る。
```cpp=
constexpr optional() noexcept = default;
```
名前の通り例外を送出する可能性がない関数につける。  
例外を考慮しなくて良い保証があることで、コンパイラは例外送出によるスタック巻き戻しのためのスタックを確保するひつようがなくなるのでパフォーマンスが向上するとのこと。  

Google Coding Styleの[noexcept](https://google.github.io/styleguide/cppguide.html#noexcept)のセクションには、noexceptがパフォーマンスを向上させる場合は使用しても良いと書いているが特にルールはなさそう。  
そもそも多くのGoogle C++環境では例外が完全にdisableされているらしい。[Exceptions](https://google.github.io/styleguide/cppguide.html#Exceptions)のセクションの冒頭文は  
> We do not use C++ exceptions.

## (ちょっと脇道にそれるけど)__func__
printfデバッグするときにめっちゃよく使うやつ。  
現在いる関数名が文字列として格納されている。  
以下のように関数のローカル変数として暗黙に定義される。  
```cpp=
static const char __func__[] = "function-name";
```
なお`__PRETTY_FUNCTION__`を使えば名前空間やクラス名も取得できたらしい。知らなかった…  
プロダクションコードでもDCHECKがONだと例えばNOTIMPLEMENTEDのメッセージとして使われている。  
```cpp=
#if DCHECK_IS_ON()
#define NOTIMPLEMENTED() \
  ::logging::CheckError::NotImplemented(__PRETTY_FUNCTION__)
#else
#define NOTIMPLEMENTED() EAT_CHECK_STREAM_PARAMS()
#endif
```
