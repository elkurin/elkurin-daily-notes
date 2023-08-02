# scoped_refptr in Chromium

Chromiumでコードを読んでいるとたまにこういうクラス宣言を見かける。  
このような宣言をしているクラスのインスタンスを作る際は以下のように`scoped_refptr`型で`MakeRefCounted`を使って初期化している。
```cpp=
class CrOSComponentManager
    : public base::RefCountedThreadSafe<CrOSComponentManager> {
};

scoped_refptr<CrOSComponentInstaller> cros_component_manager =
  base::MakeRefCounted<CrOSComponentInstaller>(nullptr, nullptr);
```

この`base::RefCounted`や`base::RefCountedThreadSafe`は何？

## scoped_refptr の概要
一言でいうと std::shared_ptr。  
scoped_refptr は参照カウントをしてくれるスマートポインタで、不要になったらdelete/freeしてくれる。  
RefCountを継承しているクラスに対して使用できる。  
class宣言時に`base::RefCounted*<T>` を継承することでRefCountをしてくれるようになる。

書き方はこんな感じ。
```cpp=
class MyFoo : public RefCount<MyFoo> {
  ...
 private:
  friend class RefCounted<MyFoo>;
  ~MyFoo();
};

void some_function() {
  scoped_refptr<MyFoo> foo = MakeRefCounted<MyFoo>();
  foo->Method(param);
  // `foo` is released when this function returns
}
```
`base::RefCount*`クラスがreference countをして適切なときにdestructをしてくれるので、Tのdestructorはprivateまたはprotectedとし勝手にdestructできないようにする。  
そして`base::RefCount*`をfriend classとして宣言することで`base::RefCount*`のみからdestructできるようになっている。

### 余談：friend class の ctor/dtor
ctor / dtorの正しくない呼び方を防ぐために、もはやprivate関数にしてしまってどうしても使いたい場合はfriend classとして追加してもらうというテクニックがChromiumに存在している。  
例えば前回のコード読み会で言及されたbase/rand_util.hの[InsecureRandomGenerator](https://source.chromium.org/chromium/chromium/src/+/main:base/rand_util.h;l=167;drc=87e35a8b1e7ab08439f4a72e3b9e6bff3467b912)は危ないrandom生成関数なので理解した上で使えとある。しかしコメントでそういうことを書いても開発者はあんまり言うことを聞かないので、より強い強制力としてfriend classをたさなきゃ使えないようにしているというわけ。  
さすがにfriend classを追加するのは気が引けるし、またreviewerも絶対確認するので有効っぽい。
#### 余談の余談：PassKey
似たようなガードとして[PassKey](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/patterns/passkey.md)というものもある。  
これは以下のようにPassKeyクラスを作ってコンストラクタをprivateにいれfriend classをつけておく。このPassKeyをつくらないとPassKeyを引数に取る関数を呼ぶことができないのでガードになる。
```cpp=
class Foo {
  class BarPasskey {
   private:
    friend class Bar;
    BarPasskey() = default;
    ~BarPasskey() = default;
  };
  void HelpBarOut(BarPasskey, ...);
};

void Bar::DoStuff() {
  foo->HelpBarOut(Foo::BarPasskey(), ...);
}
```
## RefCountedの中身

[`base::RefCounted`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=335;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)や[`base::RefCountedThreadSafe`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=402;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)、[`base::RefCountedDeleteOnSequence`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted_delete_on_sequence.h;l=33;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)がある。とりあえず一番シンプルな[`base::RefCounted`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=335;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)を見る。

[`MakeRefCounted`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/scoped_refptr.h;l=154;drc=8a7e15d1d2f4f74b9f737e3684c477f15fd012c9)は`scoped_refptr<T>(obj)`を作って返しているだけで、ref countを0からはじめるのか1から始めるのかだけチェックしている。  
`scoped_refptr<T>(obj)`の内部実装は
```cpp=
scoped_refptr(T* p) : ptr_(p) {
  if (ptr_)
    AddRef(ptr_);
}
```
`ptr_`は`T*`の生ポインタ。

以下RefCountedについて追う。  
AddRef()は`ptr->AddRef()`を呼ぶが、`T*`はRefCounted型なので[RefCounted::AddRef](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=344;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)が呼ばれ[`++ref_count_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted.h;l=129;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)している。  
逆に破壊時は
```cpp=
~scoped_refptr() {
  if (ptr_)
    Release(ptr_);
}
```
で、同様なパスを通って[`--ref_count_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted.h;l=130;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)。

`ref_count_`が0になるとdestruct。[Release](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted.h;l=348;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)の中で[`Traits::Destruct(static_cast<const T*>(this))`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted.h;l=329;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)が呼ばれている。内部実装はシンプルに[`delete x;`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/ref_counted.h;l=364-367;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)。

### その他コンストラクタ
コピーコンストラクタは
```cpp=
scoped_refptr(const scoped_refptr& r) : scoped_refptr(r.ptr_) {}
```
r.ptr_はRefCountedなので、すでに存在している`ptr_`に対して再度[RefCounted::AddRef](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=344;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)が呼ばれ`ref_count++``。

Moveコンストラクタは
```cpp=
scoped_refptr(scoped_refptr&& r) noexcept : ptr_(r.ptr_) {
  r.ptr_ = nullptr;
}
```
コピーされて`ref_count++`され、`r.ptr_`の方はリセットされている。安心。

## ThreadSafe な RefCounted
上のセクションでは[`base::RefCounted`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=335;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)について見たが、[`base::RefCountedThreadSafe`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=402;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)というthread-safeなreference countもある。

[RefCountedThreadSafeBase::AddRefImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ref_counted.h;l=207;drc=4b5e28be48f7b6d85f6b05d7e53bcd5591ac7665)ではthread-safeではないものと違って`++ref_count_`とシンプルにするのではなく[AtomicRefCount::Increment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/atomic_ref_count.h;l=31-33;drc=b0e89488c257ea10c5b433b7ff037150bacbade6)でインクリメントしている。中身は
```cpp=
int Increment(int increment) {
  ref_count_.fetch_add(increment, std::memory_order_relaxed);
}
```
`ref_count_`はstd::atomic_int型で、Incrementの中ではatomicライブラリの[std::atomic::fetch_add](https://cpprefjp.github.io/reference/atomic/atomic/fetch_add.html)を呼んでいる。  
atomic変数はthread-safeに操作できる。  

## Note
[std::is_base_of_v<Base, Derived>](https://en.cppreference.com/w/cpp/types/is_base_of)というお便利テンプレートを見つけた。BaseがDerivedの基底クラスかを確認してbool値を返してくれる。  

`AssertRefCountBaseMatches`というメソッドが使用していた。
```cpp=
AssertRefCountBaseMatches(ptr, ptr);

// 両方に`ptr`が渡されてRefCounted<T>のように作られていればマッチする
constexpr void AssertRefCountBaseMatches(const T*, const RefCounted<U, V>*) {
  static_assert(std::is_base_of_v<U, T>,
                "T implements RefCounted<U>, but U is not a base of T.");
}

// これ、failしないとダメじゃないのか？と思ったが
// Ref*で書かれていなければそもそもここに到達しないので型合わせているだけっぽい
constexpr void AssertRefCountBaseMatches(...) {}
```
