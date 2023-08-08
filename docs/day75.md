# Curiously Recurring Template Pattern

[scoped_refptr in Chromium](/docs/day17.md)のノートで[base::scoped_refptr](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/scoped_refptr.h)という参照カウント式のポインタのChromium実装を確認した。  
その際、こんな書き方の使用例を見た。
```cpp=
class GL_EXPORT GLSurface : public base::RefCounted<GLSurface>,
                            public base::SupportsWeakPtr<GLSurface> {
  ...;                              
}
```
自分で自分を継承していてキモい。  
雰囲気だけ納得していたやつが[C++テンプレートテクニック](https://www.sbcr.jp/product/4797376685/)に出てきたのでまとめる。

## CRTP
結論から言うとこれは[**C**uriously **R**ecurring **T**emplate **P**attern](https://en.cppreference.com/w/cpp/language/crtp)という手法。  
コンパイル時に決定できる静的な多態性を実現するのが目的。

多態性(polymorphism)とは型の概念を持った要素が複数の型に属することができる性質のこと。  
よくある実現方法はvirtualメソッドをoverrideする方法。
```cpp=
class Base {
  virtual void foo() const = 0;
};

class Derived {
  void foo() const override {
    std::cout << "foo" << std::endl;
  }
};

Derived d;
d.foo();  // "foo"
```
仮想関数を使用した場合どの関数を実行するかは実行時に動的に決定される。  
仮想関数テーブルによるディスパッチのコストが発生してちょっと遅くなるらしい。

一方テンプレートを使用するとどうなるか？

以下に[C++テンプレートテクニック](https://www.sbcr.jp/product/4797376685/)での例文を引用。
```cpp=
template <class Derived>
struct base {
  void base_foo() const {
    static_cast<const Derived&>(*this).foo();
  }
};

struct derived : public base<derived> {
  void foo() const {
    std::cout << "foo" << std::endl;
  }
};

derived d;
d.base_foo(); // "foo"
```
`derived`は`base<derived>`を継承しているのでstatic_castでDerived型にすることが出来る。  
このDerived型にはfooの実装がないとダメ。これは多分置き換え失敗ではないのでコンパイルエラーになる。

こうすることで継承先の関数を呼び出せるので、実質多態性。  
継承先のクラスに`foo`の実装を強要することになる。

他のメリットとしては、いろんな演算子(など似た実装を持つやつ)を一括で実装できる。  
例えば`==`を各継承先で実装すれば、基底クラスの方で`!=`を`==`の逆として実装しておけばいちいち継承先で実装しなくてすむ。
`>`なんかもそれ。比較演算子系で使いやすそう。

ひとことでまとめると、クラス間の循環依存を許してくれる便利テクニック。

## scoped_refptr
では[base::scoped_refptr](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/scoped_refptr.h)で実例を見てみよう。
```cpp=
template <class T, typename Traits = DefaultRefCountedTraits<T>>
class RefCounted : public subtle::RefCountedBase {
  ...;
  void Release() const {
    if (subtle::RefCountedBase::Release()) {
      ...;
      Traits::Destruct(static_cast<const T*>(this));
    }
  }
  ...;
}
```
解放時にstatic_cast<>で破壊できるようになってるだけだった。  
しかもdefault traits使っているならただdeleteするだけ。delete演算子は`void operator delete (void* ptr)  noexcept` なので型情報いらなそう。
多分Traitsで自前のDestroyを書けるようにしているんだと思う。  
それだけ？
TODO(hidehiko): それ以外にメリットありますか？

(デフォルトのTraitsでないやつを使っている子が1人だけいた。: [InvalidationSetDeleter](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/blink/renderer/core/css/invalidation/invalidation_set.cc;l=74;drc=b537cbde0c7c42da2ee730b286bfb1cc7735eb1a))
