# デフォルト引数のoverride

C++にはデフォルト引数という概念が存在し、以下のように使う。
```cpp=
void f(int args = 0) {
  std::cout << args << std::endl;
}

// 0 を出力
f();
// 1 を出力
f(1);
```

ところでデフォルト引数を持つ関数をoverrideするときどうなるかについて確認していたら面白い挙動を見つけた。  
（仕事が終わらないのでちょっと手抜きで…）

## override元の引数
以下のようなサンプルを考える  
```cpp=
class Base {
  virtual ~Base() = default;
  virtual void f(std::string text = "Base") = 0;
};

class WithDefault : public Base {
  WithDefault() = default;
  ~WithDefault() override = default;
  void f(std::string text = "WithDefault") override {
    std::cout << text << std::endl;
  }
};

class WithoutDefault : public Base {
  WithoutDefault() = default;
  ~WithoutDefault() override = default;
  void f(std::string text) override {
    std::cout << "WithoutDefault" << std::endl;
  }
};
```
この場合の出力結果は以下  
```cpp=
// WithDefault を出力
WithDefault().f();

// コンパイルエラー
// candidate expects 1 argument, 0 provided
WithoutDefault().f();
```
ここからわかるとおりoverride先でデフォルト引数は適応されない。  
継承先の関数でそれぞれデフォルトを指定しないと基本は使用できない。

## 静的な型でデフォルト引数は決まる
一方型をBaseとして持ってf()を走らせてみる。  
```cpp=
// Base を出力
std::unique_ptr<Base> with_default =
  std::make_unique<WithDefault>();
with_default.f();

// Base を出力
std::unique_ptr<Base> without_default =
  std::make_unique<WithoutDefault>();
```
するとこの場合Base型のデフォルト引数を参照しており、そのため`WithoutDefault`の方もコンパイルエラーとならなかった。

デフォルト引数は静的な型から取得される。しかし実際に走るものは動的な型に依存する。  
上の例では`Base::f()`は未定義だが、仮に中身が実装されていたとしても呼ばれるのは継承先の`f()`になる。  
必ずBase型にキャストして使用するのであればBaseクラスにデフォルト引数を書いてもよいがかなりわかりにくいのでやめたほうが良さそう。  

ちなみにvirtual関数になっていない場合
```cpp=
class Base {
  virtual ~Base() = default;
  void f(std::string text = "Base") {
    std::cout << text << "!!!" << std::endl;
  }
};

class WithDefault : public Base {
  WithDefault() = default;
  ~WithDefault() override = default;
  void f(std::string text = "WithDefault") {
    std::cout << text << std::endl;
  }
};

std::unique_ptr<Base> with_default =
  std::make_unique<WithDefault>();
// Base!!! を出力
with_default.f();
```
この場合は`Base::f()`が走る。virtual装飾子がついていれば継承先の実装を参照し、ついていなければ普通に自分を使う。意外とわかんなくなりがち。

## オーバーロードとの競合
他にこんなケースもある。
```cpp=
int f(int a = 1) {
  return a;
}

int f() {
  return 1;
}

f();
```
こうするとf()は`int f(int a)`のデフォルト引数を使用した呼び方なのか`int f()`を呼びたいのか定義できないのでコンパイルエラー。  
call of overloaded 'f()'' is ambiguous
