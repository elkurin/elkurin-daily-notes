# Chromium の singleton
Singletonとはクラスのインスタンスを単一にするデザインパターン。  
Chromium上ではいくつかSignletonを達成する方法がある。  

現在は[`base::NoDestructor`]((https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h))を使用することが推奨されている([参考](https://www.chromium.org/developers/coding-style/important-abstractions-and-data-structures/))。

## base::Singleton(非推奨)
[base::Singleton](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/singleton.h)は名前の通りSingletonをつくるやつで以下のようにSingletonを作ってそのインスタンスを取ることができる。
```cpp=
static T* T::GetInstance() {
    return base::Singleton<T>::get();
}
```
そのクラスのmethodを呼びたい場合は
```cpp=
T::GetInstance()->Method();
```
とすればよい。

しかし、現在はいろいろな問題([参考](https://bugs.chromium.org/p/chromium/issues/detail?id=925323))が解決され基本必要ないのでNoDestructorを使用してほしいらしい。

## base::NoDestructorとstaticローカル変数(推奨)
[base::NoDestructor](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h)はstaticローカル変数と組み合わせて使うことでシングルトンのように使えるクラス。

staticローカル変数は、最初に呼び出されたときに初期化され、スコープを抜けても生き残り、プログラムの終了時に破壊される。

以下のように使用される。
```cpp=
static T* GetInstance() {
    static base::NoDestructor<T> instance;
    return instance.get();
}
```
これはInstanceがなければNoDestructorで作って渡し、すでにあるならstatic装飾子のおかげで再初期化は行われずスコープを抜けて保持されているinstanceを渡すように動く。  
普通なら関数スコープを出たときにデストラクタが呼ばれるがstaticがついているためインスタンスが生き残る。つまりNoDestructorはSingletonのように扱える。

関数内のstaticローカル変数はスレッドセーフ([参考](https://cpprefjp.github.io/lang/cpp11/static_initialization_thread_safely.html))なので、NoDestructorで作られるSingletonもスレッドセーフ。

### base::NoDestructorはなぜ必要か
以上の話を見ると、別にNoDestructorで囲わずともそのままstatic装飾子をつけてTのインスタンスを作れば問題なさそう。  
実際自明なデストラクタを持つものは以下のように書いて良いらしい（[参考](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h;l=35-42;drc=117c6a9fb059a603e61dbdc958ba305056a9ee82)）
```cpp=
const uint64_t GetUnstableSessionSeed() {
    static const uint64_t kSessionSeed = base::RandUint64();
    return kSessionSeed;
}
```
ではなぜbase::NoDestructorというクラスがあるか。  
Chromiumではglobal constructor / destructor が許されていないからだそう([参考](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h;l=17-21;drc=117c6a9fb059a603e61dbdc958ba305056a9ee82))。
これはプログラムのスタートアップ・シャットダウンを早くしたいというモチベーションから。
**追記**：bashi@さんより[参考](https://neugierig.org/software/chromium/notes/2011/08/static-initializers.html)

...
~~よくわからないということがわかりました！いかがでしたか！
TODO(elkurin): なぜstd::stringでは必要か調べる← which prevents destructor invocation without requiring heap allocation~~  

~~インスタンスができるタイミングを実際に必要とされるタイミングまで送らせたい(lazy construction)のでstaticローカル変数とし、またデストラクタが呼ばれる順序をwell-definedにしたいとのこと。
でもstatic変数のデストラクタって宣言の逆順に呼ばれるby definitionじゃない？([参考](https://cpprefjp.github.io/lang/cpp11/static_initialization_thread_safely.html))~~  

**追記**
スタートアップ：static装飾子をつけることでコンストラクタの呼び出しを必要になったタイミングまで遅延  
シャットダウン：base::NoDestructorの中身ではインスタンスをchar*型で持っているのでデストラクタが呼ばれずに終了  

そしてconst uint64_tはtrivially destructible and does not require a global destructorだがconst std::stringはrequires a global destructorだから、const uint64_tのケースではNoDestructorがなくてもすでにデストラクタが呼ばれないためstatic装飾子をつけるだけで良い。(espresso3389@さんより[参考](https://twitter.com/espresso3389/status/1665234696396046337?s=20))

### base::NoDestructorの実装
NoDestructor()が呼ばれたときに[`storage_`](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h;l=111;drc=117c6a9fb059a603e61dbdc958ba305056a9ee82)にTのインスタンスが配置される。
```cpp=
template <typename... Args>
explicit NoDestructor(Args&&... args) {
　new (storage_) T(std::forward<Args>(args)...);
}
```
これは[placement new](https://en.cppreference.com/w/cpp/language/new#Placement_new)という文法で、あらかじめメモリを用意しておくと `new (placement-params) (type) initializer` という表記でインスタンスを割り当てることができる。そしてplacement newで作られたものは明示的にデストラクタを呼び出さないと破壊されない。  
[`storage_`](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h;l=111;drc=117c6a9fb059a603e61dbdc958ba305056a9ee82)は以下のように用意されており、[`alignas`](https://en.cppreference.com/w/cpp/language/alignas)はコンパイラに対し変数をメモリ上にアラインメントするためのキーワードで引数として渡された型のサイズ倍数のアドレスに配置される。  
```cpp=
alignas(T) char storage_[sizeof(T)];
// alignas(T) は alignas(alignof(T)) と同じ
```
これはコンパイル時からインスタンスを保存するための領域を確保しているので、そんなに必要じゃないインスタンスの場合はNoDestructorを使用しないほうがいい([参考](https://source.chromium.org/chromium/chromium/src/+/main:base/no_destructor.h;l=30-33;drc=117c6a9fb059a603e61dbdc958ba305056a9ee82))。

## 余談 base::LasyInstance(deprecated)
[LasyIntance](https://source.chromium.org/chromium/chromium/src/+/main:base/lazy_instance.h)も同様にSingletonを作ることができるinstanceだがこれはdeprecated.  
どうやらstaticローカル変数がC++11まではスレッドセーフではなかった([参考](https://cpprefjp.github.io/lang/cpp11/static_initialization_thread_safely.html))なのでその解決策のために導入されたクラスがLasyInstanceらしく、もう必要ないらしい([参考](https://chromium.googlesource.com/chromium/src/+/HEAD/styleguide/c++/c++-dos-and-donts.md#static-variables))。