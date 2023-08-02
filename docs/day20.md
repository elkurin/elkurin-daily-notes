# base::Unretained

[WeakPtrのノート](https://hackmd.io/@elkurin/ByFl9TSPn)で以下のようなポインタの渡し方に触れた。
```cpp=
base::BinbOnce(&Buffer::ReleaseContents, base::Unretained(this));
```
以前こんなコードをWeakPtrFactoryを使ったweak ptrに変更する[CL](https://chromium-review.googlesource.com/c/chromium/src/+/4483446)をいれたことがあるが、正直ちゃんと理解しておらず、WeakPtrFactoryはちゃんとinvalidateしてくれるがbase::Unretainedはそういうことをしてくれないサボり用の書き方かな〜くらいに思っていた。

今回はちゃんと[base::Unretained](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind.h;l=173;drc=9869a11472e6678126cd490e6af781f90df16d32)について見てみる。

## base::Unretained の使い時
[base::Unretained](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind.h;l=173;drc=9869a11472e6678126cd490e6af781f90df16d32)は[base/functional/bind.h]の中で宣言されている通り、関数ポインタのバインド時にのみ使える。  
[callback.md](https://source.chromium.org/chromium/chromium/src/+/main:docs/callback.md)によるとbase::Unretainedを使用してよいのは cancel-on-destroy 仕様が自明に必要ない時のみで、`base::Unretained(this)`で渡したものがcallbackが走るときより先に走る可能性がある場合は壊れる。つまりdangling ptrに対しては走らせてはいけない。

## base::Unretainedの中身

***TL;DR; ただの生ポインタ。一応dangling報告だけしてくれる***

実装は以下。
```cpp=
template <typename T>
inline auto Unretained(T* o) {
  return internal::UnretainedWrapper<T, unretained_traits::MayNotDangle>(o);
}
```
[UnretainedWrapper](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=115;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)クラスは`raw_ptr<T>`または普通の生ポインタである[`ptr_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=191;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)を持つ。  
基本raw_ptrで持つが、もしテンプレートに渡ってきたraw ptr traitsがコンパイル不能またはnot safeだったら[IsSupportedType](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=416;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)で検知して生ポインタにfall backしている。

[UnretainedWapper::get()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=154;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)で`ptr_`自身を返す。[`GetInternal`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=158-171;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)という関数でラップしているがやっているのはcheckで普通にポインタを返している。なおここでもしDangling Ptrであれば[`ReportIfDangling`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=1027;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)が報告してくれる。

というわけで、普通にdestruct処理などをしてくれない生ポインタなので、使用する際はcallbackが必ずスコープ内に帰ってくる保証が必要。  
普通にあんまり使わないほうが良さそう？

### SFINAE + Unretainedから除外されるケース
ところで巷で噂を聞くSHINAEを発見した  
IsSupportedTypeはSubstitution Failure Is Not An Error a.k.a. [SFINAE](https://cpprefjp.github.io/lang/cpp11/sfinae_expressions.html)を使用している。SFINAEとはテンプレートの置き換えに失敗した際コンパイル失敗とせずオーバーロード解決の候補から外すだけにしているやつ。
```cpp=
template <typename T, typename SFINAE = void>
struct IsSupportedType {
  static constexpr bool value = true;
};

template <typename T>
struct IsSupportedType<T, std::enable_if_t<std::is_function<T>::value>> {
  static constexpr bool value = false;
};
```
`std::enable_if_t<std::is_function<T>::value>>`というテンプレート引数は`is_function<T>`に`value`という型がなければ置き換えができず外される。つまり`T*`型が関数ポインタならC++17から導入されたらしい`is_function<T>::value`値とマッチしてvalue = falseとなる。  

またそれとは別にいくつかのクラスが外されている。
```cpp=
template <>
struct IsSupportedType<cc::Scheduler> {
  static constexpr bool value = false;
};
template <>
struct IsSupportedType<base::internal::DelayTimerBase> {
  static constexpr bool value = false;
};
template <>
struct IsSupportedType<content::responsiveness::Calculator> {
  static constexpr bool value = false;
};
```
コメントによるとperformance sensitiveな場所で、base::Unretainedから使用されると良くないらしい。なぜbase::Unretainedを使うと遅くなるんだろう？  
こんな[記述](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=409-414;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)がある  
> Templates that may end up using `raw_ptr<T>` should
> use IsSupportedType to ensure that raw_ptr is not used with unsupported
> types.  As an example, see how base::internal::StorageTraits uses
> IsSupportedType as a condition for using base::internal::UnretainedWrapper
> (which has a `ptr_` field that will become `raw_ptr<T>` after the Big
> Rewrite).
 
Big Rewriteとか言ってるしそもそもraw_ptrを作るのが大変？  
実際書き換えている[CL](https://chromium-review.googlesource.com/c/chromium/src/+/3379601)を見るとそうっぽい？  
TODO(elkurin): invesigate further

### MayNotDangle
[UnretainedWrapper](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind_internal.h;l=115;drc=7ac4a606711a108763d6dedbc84e6fc32ccf821f)のテンプレート引数にRawPtrTraitsがついている。  
上記のUnretainedの場合は`unretained_traits::MayNotDangle`がついているのでDanglingを許さないが、[UnsafeDangling](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind.h;l=238;drc=9869a11472e6678126cd490e6af781f90df16d32)を使うとdanglingすら許すポインタが作れる。使わないほうが良さそう。  
実際Strongly prefer Unretained()と[コメント](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind.h;l=233;drc=9869a11472e6678126cd490e6af781f90df16d32)にも書いてある。ではいつなら使っていいか？  

例として[ipc/ipc_mojo_bootstrap.cc](https://source.chromium.org/chromium/chromium/src/+/main:ipc/ipc_mojo_bootstrap.cc;l=878;drc=525a01446ec26a96cc0b68baa8d18ec2c1a6c09c)があった。  
```cpp=
void NotifyEndpointOfError(Endpoint* endpoint) {
  ...
  endpoint->task_runner()->PostTask(
    FROM_HERE,
    base::BindOnce(&ChannelAssociatedGroupController::
                   NotifyEndpointOfErrorOnEndpointThread,
                   this, endpoint->id(),
                   // This is safe as `endpoint` is verified to be in
                   // `endpoints_` (a map with ownership) before use.
                   base::UnsafeDangling(endpoint)));
}

void NotifyEndpointOfErrorOnEndpointThread(mojo::InterfaceId id,
                                           MayBeDangling<Endpoint> endpoint) {
  base::AutoLock locker(lock_);
  auto iter = endpoints_.find(id);
  if (iter == endpoints_.end() || iter->second.get() != endpoint)
    return;
  // ここからendpoint解禁
  if (!endpoint->Client())
    return;
  ...
}
```

Unretainedで渡されている`endpoint`のidも渡しそれがregisterされていることを確認したあとに`endpoint`を使用するというコード。  
registerされいない、つまり`endpoint`がすでに破壊済みな可能性がある場合はアクセスしていないので安全ということ。  
だが開発者の良心に頼っている方法なのであんまりやらないほうが良さそう。頑張れば排除できそう。  
どうなんでしょ  
ipc_mojo_bootstrapとか見るからに触られる頻度低そうなのでセーフ？


## Note
[std::conditional](https://cpprefjp.github.io/reference/type_traits/conditional.html)というtrue/falseで型をコンパイル時に変えるテンプレートの仕様例を見た。  
```cpp=
using StorageType =
  std::conditionl_t<raw_ptr::traits::IsSupportedType<T>::value,
                    DanglingRawPtrType,
                    T*>;
```
やったー
