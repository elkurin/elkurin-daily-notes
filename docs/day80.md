# absl::InlinedVector

標準ライブラリのパフォーマンス改善版をabseilがいくつか実装している。  
例えば[absl::flat_hash_map](/docs/day72.md)はstd::unordered_mapの改善。  

今回はstd::vectorの代替としてabseilが実装している[absl::InlinedVector](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/inlined_vector.h)を確認する。

## 概要
std::vectorの完全上位互換ではなく、サイズがある程度小さい場合に使えるデータ構造。  
ただし配列とは違ってサイズは動的に変更できる。  
宣言時にこれくらいやろっていうサイズ`N`を与え、そのサイズ以下のときはインラインの配列、それ以上になったらstd::vectorと同じ挙動になる。  
なのでほとんどの場合std::vectorよりパフォーマンスが良くなるが、サイズ`N`を超えたときにコストの重いアロケーションが発生する。またデフォルトで`N`分の領域を確保することになるので、大きい値を設定すると無駄領域ができることがある。

APIはstd::vectorが持っているものすべてに対応している。

## 実装
テンプレートの宣言はこれ
```cpp=
template <typename T, size_t N, typename A = std::allocator<T>>
class InlinedVector {
  using Storage = inlined_vector_internal::Storage<T, N, A>;
  ...
```
`T`型が中身の値の型で、`N`が想定される配列のサイズ。  
`Storage`はデータを格納している`storage_`の型で、その中のデータの状態はAllocatedとInlinedの2種類ある。
```cpp=
union Data {
  Allocated allocated;
  Inlined inlined;
};
```
よくみる`union`を使って内部実装を切り替えている。  
これらはそれぞれ[GetAllocatedData](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=387;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)を[GetInlinedData](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=399;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)によって取れる。  

Inlinedの中身を見てみる。
```cpp=
struct Inlined {
  alignas(ValueType<A>) char inlined_data[sizeof(
      ValueType<A>[kOptimalInlinedSize])];
};

static constexpr size_t kOptimalInlinedSize =
  (std::max)(N, sizeof(Allocated) / sizeof(ValueType<A>));
```
[alignas](https://cpprefjp.github.io/lang/cpp11/alignas.html)を用いて指定された値分の配列を確保している。  
サイズは、最初にテンプレートで指定した`N`か、もし`sizeof(Allocated)`が大きい場合はそのサイズ分だけ確保している。

データにアクセスする際は[`data()`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/inlined_vector.h;l=344;drc=ec360d3f928af69a303d413e806138b7624997e6)でアクセスし、AllocatedかInlinedかによって呼び出し方を変えている。

### 挿入
[push_back](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/inlined_vector.h;l=750;drc=5dd7dd46d5ae09b28f5553ca24221eea0072b66f)などの内部実装である[EmplaceBack](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=827;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)を見る。  

AllocatedでもInlinedでも同じように扱える[StorageView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=162;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)型に落とし込んでいる。
```cpp=
template <typename A>
struct StorageView {
  Pointer<A> data;
  SizeType<A> size;
  SizeType<A> capacity;
};
```

`capacity`は今のデータで持てる最大サイズを示している。これはAllocatedの方でも指定されている。  
もしEmplaceBackでcapacityを超えなければただ最後にくっつけるだけ。
```cpp=
if (ABSL_PREDICT_TRUE(n != storage_view.capacity)) {
  // Fast path; new element fits.
  Pointer<A> last_ptr = storage_view.data + n;
  AllocatorTraits<A>::construct(GetAllocator(), last_ptr,
                                std::forward<Args>(args)...);
  AddSize(1);
  return *last_ptr;
}
```
ほとんどの場合capacityを超えることはないのでABSL_PREDICT_TRUEのコンパイル用オプションがついている。  

もしちょうどcapacityサイズと同じなstorageに追加しようとした場合は[EmplaceBackSlow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=844;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)を行う。  

`capacity`は2倍に増やし、現在の`data`をMoveIteratorでmoveする。  
そして[SetIsAllocated](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/inlined_vector.h;l=456;drc=4763cf47e6f4919821efa6bc1ca3c7757125b54c)でAllocated状態にしておく。

### 削除
一方削除の[`pop_back`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/inlined_vector.h;l=761;drc=5dd7dd46d5ae09b28f5553ca24221eea0072b66f)の実装は以下。
```cpp=
void pop_back() noexcept {
  ABSL_HARDENING_ASSERT(!empty());

  AllocatorTraits<A>::destroy(storage_.GetAllocator(), data() + (size() - 1));
  storage_.SubtractSize(1);
}
```
そしてSubtractSizeは`GetSizeAndIsAllocated() -= count << static_cast<SizeType<A>>(1);`。  
これでわかるとおりサイズの変更をしているだけで、Nより小さくなったタイミングでinlinedに戻すなどは行っていない。

[resize](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/inlined_vector.h;l=600;drc=5dd7dd46d5ae09b28f5553ca24221eea0072b66f)もあるが、これは空っぽの余った分の要素を消すだけで、構造を変えてはいない。

### look up
[operator[]](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/inlined_vector.h;l=362;drc=5dd7dd46d5ae09b28f5553ca24221eea0072b66f)の実装は`data`を`storage_`から持ってくる。  
AllocatedもInlinedも`[i]`でアクセスできるので同様にアクセスできる。  
なお範囲外アクセスのチェックははいる。
```cpp=
reference operator[](size_type i) ABSL_ATTRIBUTE_LIFETIME_BOUND {
  ABSL_HARDENING_ASSERT(i < size());
  return data()[i];
}
```
[absl::optional](/docs/day65.md)の際にも触れたがABSL_HARDENING_ASSERTがデバッグビルド時のチェックをしてくれるマクロ。

## Chromiumでの扱い
absl::InlinedVectorは先月に許可されたところ([CL](https://chromium-review.googlesource.com/c/chromium/src/+/4692682))。  
それまではbase::StackVectorを自前でもっていたが、これはAPIがちょっと変だったらしい。([参考](https://groups.google.com/a/chromium.org/g/cxx/c/jTfqVfU-Ka0/m/caaal90NCgAJ))  

まさに3日前のAug 10にbase::StackVectorが[Remove](https://chromium-review.googlesource.com/c/chromium/src/+/4764912)された。  
[InlinedVectorを代わりに使おう](https://groups.google.com/a/chromium.org/g/chromium-dev/c/inB5QAClfcc)。
