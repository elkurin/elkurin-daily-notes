# ChromiumのCallback
これまたテックリード氏の話を聞いたもののちんぷんかんぷんだったので自分でコードを読むことに。

## 概要
Callbackは関数ポインタのようなもの。  
ChromiumのCallbackでは [base::BindOnce()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=62;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d) または [base::BindRepeating](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=82;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d) のどちらかで作られる。  
BindOnceで作られるOnceCallbackはたかだか一度しか走らせないコールバックで、BindRepeatingで作られるRepeatingCallbackは何度も走って良いという違いがあるが、コンセプトはだいたい一緒。
Bindでは引数を一部バインドすることができ、例えば `func(int a, int b) { return a + b; }` という関数があったとして `BindOnce(&func, 10)`とすると`b => 10 + b`みたいな関数になる。

## Callbackの使い方
[OnceCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback.h;l=73;drc=f42b3c2b9eeb6f3fc1bd672cada21aa36eeed0c7)も[RepeatingCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback.h;l=265;drc=f42b3c2b9eeb6f3fc1bd672cada21aa36eeed0c7)も以下のようなテンプレートで定義されている。
```cpp=
template <typename R, typename... Args>
class OnceCallback<R(Args...)> {
  using ResultType = R;
  using RunType = R(Args...);
  ...
};
```
使う際は以下のような感じ
```cpp=
// base::Version と const base::FilePath& を引数に受け取ってvoidを返す
base::OnceCallback<void(base::Version, const base::FilePath&)>
```

Bindは以下のように行う。
```cpp=
// weak_factory_.GetWeakPtr() は ClassName のインスタンスへのweak ptrをとる
base::BindOnce(&ClassName::Func, weak_factory_.GetWeakPtr(),
               param1, param2);
```

作ったCallbackは以下のように走らせる。
```cpp=
std::move(callback).Run(param1, param2);

// `callback` が RepeatingCallback の場合のみ可
callback.Run(param1, param2);
```

### Bindする関数について
一番良く見られるのは `void ClassName::Func(P1 param1, P2 param2);`のように関数を実装してその参照を渡す方法だが、ラムダ式を書いてつっこむことも可能。
```cpp=
base::OnceCallback<int(int, int)> callback =
    base::BindOnce([](int a, int b) { return a + b; }, 10);
```
ただしcapture(ラムダ式から外側の変数を参照するやつ、`[]`の中に書く)はプロダクションコードでは禁止されている（テストコードではOK）。コンパイラで弾かれると教わったが中身は忘れちゃった。  
TODO(elkurin): キャプチャってどうやって弾かれる？

## BindOnce / OnceCallbackの実装
[base::BindOnce()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=62;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)の中身のコンストラクタは[これ](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback.h;l=238;drc=f42b3c2b9eeb6f3fc1bd672cada21aa36eeed0c7)で[internal::BindImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=1642-1644;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)で呼ばれている。  
OnceCallback版で書くと以下のようになる
```cpp=
explicit OnceCallback(internal::BindStateBase* bind_state)
    : holder_(bind_state) {}

OnceCallback(BindState::Create(
    reinterpret_cast<InvokeFuncStorage>(Invoker::RunOnce),
    std::forward<Functor>(functor), std::forward<Args>(args)...));
```
TODO(elkurin): std::forwardって何

[`Invoker::RunOnce`](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind_internal.h;l=968;drc=64b025b7b6de9963d71438a99a5a624e0d63ca44)はBindStateBase*とPassingTypeを受け取って[RunImpl()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=996;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)を呼び、[InvokeHelper](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=915-958;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)を呼ぶ。ちなみに[internal::BindStateBase](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_internal.h;l=46;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)とはfunction objectとbound argumentsを保持するクラス。  
このInvoker::RunOnceが[BindStateBase]((https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_internal.h;l=46;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d))の[`polymorphic_invoke_`にSet](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_internal.cc;l=44;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)されている。  
なお`polymorphic_invoke_`は [`void (*)()`型](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_internal.h;l=56;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)になっているが、曰く
> In C++, it is safe to case function pointers to function pointers of another type. It is not okay to use void*. We create a InvokeFuncStorage that that can store our function pointer, and then cast it back to the original type on usage.  [引用](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_internal.h;l=89-92;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)  
ということで、型を無視して関数ポインタのキャストを行っているコードで良いらしい。  

作られたコールバックを走らせる関数は
```cpp=
using PolymorphicInvoke = R (*)(internal::BindStateBase*,
                                internal::PassingType<Args>...);

R Run(Args... arg) && {
    internal::BindStateHold holder = std::move(holder_);
    PolymorphicInvoke f =
        reinterpret_cast<PolymorphicInvoke>(holder.polymorphic_invoke());
    return f(holder.bind_state().get(), std::forward<Args>(args)...);
}
```
PolymorphicInvokeの雰囲気は関数オブジェクトそのものとバインドされた引数の組であるBindStateBaseポインタと、パスされる引数を合わせて引数と取ってResultTypeを返すような関数ポインタの型。

`holder.polymorphic_invoke()`は先述の[`Invoker::RunOnce`](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind_internal.h;l=968;drc=64b025b7b6de9963d71438a99a5a624e0d63ca44)を呼ぶのでそこからはいい感じにInvokeされてハッピー。



## Once と Repeating の実装的な違い
上述の通り、[base::OnceCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback.h;l=73;drc=f42b3c2b9eeb6f3fc1bd672cada21aa36eeed0c7)はたかだか一度しか走らず、[base::RepeatingCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback.h;l=265;drc=f42b3c2b9eeb6f3fc1bd672cada21aa36eeed0c7)は何度でも走らせてよい。  

これは関数を走らせるRun()関数の実装で実現。
```cpp=
// OnceCallback
R Run(Args... args) const& { /* static_assert(); NOTREACHED(); */ }
R Run(Args... args) && { invoke_callback(); }

// RepeatingCallback
R Run(Args... args) const& { inboke_callback(); }
R Run(Args... args) && { invoke_callback(); }
```
このようにOnceCallbackでは const& を禁止する。

### メンバ関数の左辺値右辺値装飾とstd::move
[参考](https://cpprefjp.github.io/lang/cpp11/ref_qualifier_for_this.html)  
上記のRun関数は以下のメンバ関数をオーバーロードしている。  
- int f() &; → *thisが非constな左辺値のときに呼び出される
- int f() const&; → *thisがconstな左辺値のときに呼び出される
- int f() &&; →  *thisが右辺値であるときに呼び出される

OnceCallbackではconst&がinvokeしないようオーバーロードされているため、左辺値に対してRunすることができない。&&は使えるので右辺値としてRunすることになり、std::move(callback)とすればcallbackを右辺値として扱えるのでOK。moveを使用して走らせているのでたかだか１回しか走らせられない（と明示している）。  
一方RepeatingCallbackは左辺値の制約がないので同じcallbackを何度もRunできる。  

## Receiver Object が破壊済みのとき
weak pointerがinvalidateされた場合はRun()はno-opになる。このcheckは[InvokeHelper](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=915-958;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)の中を確認するとわかる。  

[InvokeHelper](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=915-958;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d)は最初の型引数によって分岐していて、最初の引数は
```cpp=
static constexpr bool is_weak_call =
    IsWeakMethod<is_method,
                 std::tuple_element_t<indices, DecayedArgsTuple>...>();
```
つまりweak ptrがバインドされてるタイプかどうかで分岐しているっぽい。  
`is_weak_call`がtrueの方([こっち](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=932-958;drc=7825e28a1c167e3ef7da2f1ec48dfa4be818945d))では以下のコードでポインタが破壊されていたらコールバックを終了している。
```cpp=
const auto& target = Unwrap(std::get<0>(bound));
if (!target) {
    return;
}
```
というわけで、途中で破壊されても変なことをせずに宙に消えてくれる。  
Callbackが返ってこないと挙動がおかしくなるような実装の場合は壊れるが、これはCallbackを利用するコード側で気をつけるべきこと。

## お便利Callbackたち
以下お役立ちツール。力尽きたので概要だけ
- [base::DoNothing](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback_helpers.h;l=178;drc=1e19bc33707960385f0541a5288176af528c1bbc): なんもせん
- [base::DoNothingWithBoundArgs](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/callback_helpers.h;l=210;drc=1e19bc33707960385f0541a5288176af528c1bbc): 引数のbindだけする。objectのライフタイムで管理する系に役に立つ
- [base::BarrierCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/barrier_callback.h): いくつかの結果が返ってくるのを待ってからRun。RepeatingCallbackの使いどころ