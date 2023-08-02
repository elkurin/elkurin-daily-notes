# WeakPtr in Chromium
C++の標準ライブラリでは、[std::shared_ptr](https://cpprefjp.github.io/reference/memory/shared_ptr.html)は参照カウントをしてくれるスマートポインタで、[std::weak_ptr](https://cpprefjp.github.io/reference/memory/weak_ptr.html)はshared_ptrに対してカウントをしない参照を持つことができ所有者がいなくなったら解放することになっている。

Chromiumにおける[weak_ptr](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/weak_ptr.h;l=227;drc=5eac04a1be3edecafb8061e56000d1d4c72c0f7b)も近い概念で、lifetimeに影響を与えず参照を持ち、所有権を持つobjectが破壊された時などに勝手にinvalidateされるポインタクラスとされている。

## WeakPtrの使い方
Chromiumでweak_ptrを使用するためには、参照したいクラスがWeakPtrFactoryを持つ。
```cpp=
class A {
  void func() {
    B::Run(weak_factory_.GetWeakPtr());
  }
 private:
  base::WeakPtrFactory<A> weak_factory_{this};
};
```
また、以下のようなclass Bがあった時、B::Run()を何度も走らせてBを作りA破壊時にみんなdeleteできる。
```cpp=
class B {
 public:
  static void Run(base::WeakPtr<A> a) {
    B* b = new B(std::move(a));
  }
  
 private:
  B(base::WeakPtr<A> a) : a_(std::move(a)) {}
  base::WeakPtr<A> a_;
};
```
weak_ptrをChromiumで最も見るのはBindOnceとの組み合わせ。こんな感じ。
```cpp=
base::BindOnce(&BrowserLoader::OnGetVersion, weak_factory_.GetWeakPtr(),
               LacrosSelection::kRootfs, barrier_callback);
```
weak_ptrはownerが破壊されるとinvalidateされるので、OnceCallbackが走るタイミングで破壊されていても正常に終了する。

## WeakPtrの仕組み

### 概要
[WeakPtrFactory::GetWeakPtr](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=383;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)でweak ptrを獲得する。

WeakPtrFactoryインスタンスが破壊されるとscoped_refptrがinvalidateされて[WeakPtr::get](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=262;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)がnullptrを返すようになる。

### 実装
まずはWeakPtr classを見る。[`ptr_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=326;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)に参照している生ポインタを保存している。[`ref_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=318;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)は[WeakReference](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=102;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)型で参照が有効かを判定している。  
[`ref_.IsValid()`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=143;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)がtrueの時のみweak_ptrは有効で、`ptr_`の値は`!ref_.IsValid()`の際undefined。  
[WeakPtr::get](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=262;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)では`!ref_.IsValid()`の時nullptrを返しているし、[operator bool()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=279;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)も`ptr_`ではなく`get()`のnull checkをしている。

次にどうやって参照のライフタイムをチェックしているかについて。  
使い方のセクションで言及したとおり、[WeakPtrFactory](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=363;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)をクラスのインスタンスとして持ち、その中の[WeakReferenceOwner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=157;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)の[`weak_reference_owner_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=352;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)が作られるとこれがweak referenceの管理をする。  
[WeakReferenceOwner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=157;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)は[WeakReference::Flag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=106;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)のscoped_refptrを持っており、このFlagが参照のinvalidateを担当している。  
[WeakPtrFactory::GetWeakPtr](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=383;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)がweak ptrを獲得するためのメソッドだが、呼ばれると[WeakReferenceOwner::GetRef](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.cc;l=89;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)から[WeakReference(flag_)](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.cc;l=60;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)をコンストラクトして返し、このWeakReferenceを`ptr_`から作成したWeakPtrを渡す。  
Ownerであるクラスが破壊されると所有している[WeakPtrFactory](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=363;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)も破壊されchainして[~WeakReferenceOwner()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.cc;l=85;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)が呼ばれる。WeakReferenceOwnerのデストラクタでは[flag_->Invalidate()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.cc;l=22;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)が呼ばれておりAtomicFlagの[`invalidated_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=126;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)がSetされる。以後このflagは[`IsValid`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.cc;l=36;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)がfalseを返すようになるので、残存のWeakPtrたちは自身のインスタンスは破壊されない一方で[get()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/memory/weak_ptr.h;l=262;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)しようとすると`ref_.IsValid()`チェックに引っかかりnullptrを返すようになる。



## WeakPtrFactory vs Unretained
WeakPtrの仕様例として[BindOnce](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=62;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)でコールバックの返り先を保存するケースを書いたが、ところでもう一つ変なのがある。
```cpp=
base::BinbOnce(&Buffer::ReleaseContents, base::Unretained(this));
```
この書き方がされているときはWeakPtrFactoryの宣言もない。  
[base::Unretained](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=173;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)は何？

### weak ptr ではなかった
特殊なweak ptrの作り方かと思っていたが違うっぽい。
```cpp=
template <typename T>
inline auto Unretained(T* o) {
  return internal::UnretainedWrapper<T, unretained_traits::MayNotDangle>(o);
}
```
### non-refcounted
[コメント](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h;l=119-123;drc=1486850541d21b2aab1739e5a1958aa95c67e15f)によると参照カウントを無視するクラスらしい。そして生ポインタとは異なりcallbackが走るタイミングでdangling referenceをチェックしてくれる。  
宣言されているファイルも[base/functional/bind.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/bind.h)なのでCallback専用で使えるやつ。  
これについてはまた今度。
