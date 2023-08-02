# raw_ptr in Chromium
メモリ保全は難しく、[Google Secutiry Blog](https://security.googleblog.com/2022/09/use-after-freedom-miracleptr.html)によると2022のQ1に発見されたChromiumのバグのうち50%以上のバグがUse-after-free起因だったらしい。  
[以前のノート](https://hackmd.io/@elkurin/SJCCRyJvn)で言及したMiracle Ptrはuse-after-freeを防ぐためいに導入されたChromium内でraw_ptrを乗っ取っているsmart pointerで、中身はBackupRefPtrという賢いポインタ。

BackupRefPtrについては[ドキュメント]((https://docs.google.com/document/d/1m0c63vXXLyGtIGBi9v6YFANum7-IRC3-dmiYBCWqkMk/edit))があり、これはMiracle Ptrの[Readme](https://chromium.googlesource.com/chromium/src/+/HEAD/base/memory/raw_ptr.md)のリンクから探せるpublicドキュメントです。

## 概要
raw_ptrはメモリの先頭にデータを書き込んでおくことでそのメモリの状態を他のところから参照したときでも取得でき、メモリの状態に応じた処理ができるという仕組み。

raw_ptrはallocateされているメモリの先頭にreference countなどのデータを持つヘッダーを書き込んでいる。以下のようにAllocateされているエリアのうちヘッダーのあとの部分が実際のメモリの保存したデータになっている。

![](https://hackmd.io/_uploads/S1NWVezv3.png)
([Google Security Blog](https://security.googleblog.com/2022/09/use-after-freedom-miracleptr.html)より引用)

この仕組みを[raw_ptr](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr.h)というクラスでラップしている。

もしreference countが0より大きいときにpointerがfree/deleteされたらメモリを即解放するのではなくヘッダーに情報を上書きして他のraw_ptrが参照できるようにしている。これによってuse-after-freeによる危険を取り除いている。  
メモリはreference countが0になったらようやく解放される。

ちなみに実装は[base/allocator/partition_allocator/pointers/raw_ptr.h]((https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr.h)にあるが長いので[`base/memory/raw_ptr.h`](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/raw_ptr.h)をincludeしよう。

## BackupRefPtrの実装
[`wrapped_ptr_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=1061;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)に実際のアドレスが保存されている。  
`raw_ptr`クラスは先頭のデータを追加したり処理したりしてくれている。

大まかな流れは、allocate時にallocateしたというフラグをtrueにし、破壊されたときにfalseにする。ポインタが生きているかはこのフラグを確認すればわかる。  
raw_ptrの作成時にref countをインクリメントし、破壊時にデクリメントする。  
もしref countが0より大きいなら参照しているやつがいるので残しておいて、0になったらフリー。
なのでallocateフラグがfalseになっていてもref countが正の数になっているケースがあり、このときはまだフリーせず、次に参照されたときにもuse-after-freeにならないようにする・あるいは違うメモリによって上書きされていないようにする。

### タグの実装
Allocatedされているエリアの先頭にタグを書くのがraw_ptrだが、そのタグは[PrtitionRefCount](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=39;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)型として処理している。  
[partition_ref_count.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h)のドキュメントを見ると、タグの中身はこんな感じ
- 0ビット目：`is_allocated`のフラグ。construct時に1になり、[ReleaseFromAllocator](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=186;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で0にんる
- 1-31ビット目：`ptr_count`。raw_ptrのreferenceが何個あるかを保存。[Aquire](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=110;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で++、[Release](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=141;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で--。
- 32ビット目：`dangling_detected`。Dangling Ptrを検知するためのフラグ。
- 33ビット目：`needs_mac11_malloc_size_hack`。なんですか？
- 34-63ビット目：`unprotected_ptr_count`。DisableDanglingPtrDetectioのraw_ptrのカウント。[AcquireFromUnprotectedPtr](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=127;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で++、[ReleaseFromUnprotectedPtr](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=169;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で--。

`is_allocated`、`ptr_count`、`unprotected_ptr_count`がすべて0のときallocationは開放されている。

### raw_ptrの値を変える時
以下Dangling Ptrについては無視する。

まずはraw_ptrをリセットする方を見ていく。以下のようなケース。
```cpp=
// dtor
~raw_ptr();

// `window` を null　でリセット
raw_ptr<aura::Window> window = nullptr;

// 自分が所有していない`new_window_owned_by_other`の生ポインタで上書き
window = new_window_owned_by_other.get();
```
[`constexpr ~raw_ptr() noexcept`](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=628;drc=4c540946223beb63061d2338af2b974d9cd2c7d8)はdesturct時に明らかにリセットしてる。  
[`constexpr raw_ptr& operator=(std::nullptr_t) noexcept`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=738;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)はraw_ptrにnullptrが代入されるときに呼ばれる関数で、すなわちリセット時の操作が実装されている。  
[`constexpr raw_ptr& operator=(T* p) noexcept`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=743;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)では新しいポインタが代入されるときに呼ばれる関数で、これもリセットである。  
両者の関数でImpl::ReleaseWrappedPtr()が呼ばれている。  
BackupRefPtrの場合の実装は[raw_ptr+backup_ref_impl.h](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=219;drc=dd71552024ce04579097f70325177fff5da1ecff)にあり、やっていることは大きく2つ。
```cpp=
// 簡略ver
template <typename T>
static constexpr void ReleaseWrappedPtr(T* wrapped_ptr) {
  uintptr_t address = partition_alloc::UntagPtr(UnpoisonPtr(wrapped_ptr));
  if (IsSupportedAndNotNull(address))
    ReleaseInternal(address);
}
```
UntagPtrの中身は`address & internal::kPtrUntagMask`で、以下のように定義されているkPtrUntagMaskをmaskしている
```cpp=
constexpr uint64_t kPtrTagMask = 0xff00000000000000uLL;
constexpr uint64_t kPtrUntagMask = ~kPtrTagMask;
```
ちなみに"~"という演算子はビットごとの補数。つまり`address & internal::kPtrUntagMask`はタグを取り除くという操作をしていることになる。

そののちに`address`が[`IsSupportedAndNotNull`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=95;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)ならば[ReleaseInternal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.cc;l=36;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)を呼ぶ。つまり上書きしたいポインタである`wrapper_ptr`がnullptrならUntagしたままで終わり。raw_ptrがnullptr状態ならタグは空っぽらしい。

一方[ReleaseInternal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.cc;l=36;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)が呼ばれるケース、すなわち新しいポインタを上書きしたい場合について。  
[PartitionAllocGetSlotStrtInBRPPool](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_root.h;l=955;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)で`address`に対応するallocated slotの先頭を獲得してるが、これは`adress`はslot内ならどこでもよいかららしい。マジで？  
そのアドレスを用いて[PositionRefCountPointer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=415;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)からslotの先頭に保存されているreference countを獲得する。  
簡略版は以下。
```cpp=
PrtitionRefCount* PartitionRefCountPointer(uintptr_t slot_start) {
  uintptr_t refcount_address = slot_start - sizeof(PartitionRefCount);
  return reinterpret_cast<PartitionRefCount*>(refcount_address);
}
```
このPartitionRefCountに対して[Release](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.cc;l=50;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)を呼んでおり、つまり`ptr_count`がデクリメントされた。Releaseの中では[ReleaseCommon](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=283;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)を呼んでおり、もし`count`が`kPtrCountMask`とのorでtrueならつまりまだそのアドレスを参照しているポインタがあるということなので保持したい。そうでない場合はfreeしたい。ここで`count`の値が活用されいてた。概要で書いたとおり、`ptr_count`が0に到達するまではフリーしていない。  
freeしたい場合は[Release()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.cc;l=50;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)がtrueを返すので、[PartitionAlocFreeForRefCounting](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_root.h;l=1018;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)が呼ばれてFreeされる。


逆にセットする側。  
`window = new_window_owned_by_other.get();`はポインタのセットもしている。  
[`constexpr raw_ptr& operator=(T* p) noexcept`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=743;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)では上述の[Impl::ReleaseWrappedPtr()](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=219;drc=dd71552024ce04579097f70325177fff5da1ecff)のあとに[Impl::WrapRawPtr](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=185;drc=dd71552024ce04579097f70325177fff5da1ecff)も呼んでいる。
```cpp=
PA_ALWAYS_INLINE constexpr raw_ptr& operator=(T* p) noexcept {
  Impl::ReleaseWrappedPtr(wrapped_ptr_);
  wrapped_ptr_ = Impl::WrapRawPtr(p);
  return *this;
}
```
[Impl::WrapRawPtr](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=185;drc=dd71552024ce04579097f70325177fff5da1ecff)では[Impl::ReleaseWrappedPtr()](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=219;drc=dd71552024ce04579097f70325177fff5da1ecff)とは逆に[AcquireInternal](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=194;drc=dd71552024ce04579097f70325177fff5da1ecff)を呼んでいて、[Acquire](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=110;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)から`ptr_count`をインクリメントしている。

以上がおおまかな流れ。

### raw_ptrを参照する時
[`constexpr T* operator->() const`](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=805;drc=4c540946223beb63061d2338af2b974d9cd2c7d8)が参照する関数。  
中身は[GetForDeference](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=1037;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)を呼んでいるだけでその中身は[SafelyUnwrapPtrForDereference](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.h;l=242;drc=dd71552024ce04579097f70325177fff5da1ecff)。

以下簡略版
```cpp=
static constexpr T* SafelyUnwrapPtrForDereference(T* wrapped_ptr) {
  uintptr_t address = partition_alloc::UntagPtr(wrapped_ptr);
  if (IsSupportedAndNotNull(address)) {
    PA_BASE_CHECK(wrapped_ptr != nullptr);
    PA_BASE_CHECK(IsPointeeAlive(address));
  }
  return wrapped_ptr;
}
```
[IsPointeeAlive()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr_backup_ref_impl.cc;l=93;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc) → [IsAlive](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/partition_ref_count.h;l=228;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)でタグと`kMemoryHeldByAllocatorBit`とのビット&から`is_allocated`を確認して、ポインタがまだ生きているかCheck。  
この確認をパスしたならまだ生きているということなので安全にアクセスできる。

### `is_allocated`はいつ0になる？
ところでcountが0になるまでまつことでuse-after-freeを防ぐのはわかったが、ここまででは`is_allocated`が0にならないのでずっと`IsPointeeAlive() = true`のままである。  
ownershipを持つやつが破壊された時に0にセットされるはず。  
追っていくと[allocator_shim::internal::PartitionBatchFree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/shim/allocator_shim_default_dispatch_to_partition_alloc.cc;l=729;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)がそんな感じのことをやっていそう。  
PartitionAllocのDispatcherの[Initialize](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/dispatcher/dispatcher.h;l=48;drc=4efed7a716e3166adaf984f0e31cb020bfe4e7dc)時にその関数が挿入されているっぽいが長くなるのでこのノートではここまで。

### unique_ptrは？
ここまでraw_ptrの中だけで話してきたが、Chromiumのポインタはraw_ptrだけではない。  
例えばunique_ptrがdestructしたときにraw_ptrが保持しっぱなしみたいなケースがuse-after-freeの典型例だが、unique_ptrのref countも一緒に行っていないとこのシステムは成り立っていないはず。
```cpp=
std::unique_ptr<A> a = std::make_unique<A>();
// BはAのraw_ptrを持つ
std::unique_ptr<B> b = std::make_unique<B>(a.get());

// a を破壊
a = nullptr;
// a を参照するような関数
b->ReferToA();
```
例えばこのケースではReferToA()という関数でdestructされたはずの`a`にアクセスしないように、`is_allocated`が`a=nullptr`によって0になっていてほしい。しかしaはunique_ptrなので？  
unique_ptrのref countはどうやっている？内部実装にraw_ptrを使用していたりしますか…？
これもPartitionAllocのDispatcherとかを読まないとわからなさそう。

## Partition Alloc
ところで上記の説明では`is_allocated`の値が〜とか言っているが、そもそもメモリをフリーして他の何かが上書きしたらallocationもなにもなく`is_allocated`の位置のビットが書き換えられている可能性もあるのでは？  
という疑問もあるが、それはChromiumが[PartitionAlloc](https://chromium.googlesource.com/chromium/src/+/HEAD/base/allocator/partition_allocator/PartitionAlloc.md)というデザインを採用していることで守られている。
かんたんに言うと、メモリ領域をオブジェクトの種類と確保するサイズによってわかることにして、allocationの区切り位置を毎回同じにしている。このデザインであれば、毎回`is_allocated`などタグの各値の位置は不変なので正しく処理できる。

Chromiumのコード内でもBUILDFLAG(USE_PARTITION_ALLOC*)でガードされているところが散見される。

各Partitionのサイズは1024byteに抑えれているっぽい？([kMaxMemoryTaggingSize = 1024](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/partition_alloc_constants.h;l=317;drc=c964de601a7fdd237e3df1fd99b96c94b2c4fb84))

## Note
std::memory_order_relaxed というやつを見かけた。  
[memory_order](https://cpprefjp.github.io/reference/atomic/memory_order.html)の説明によると、マルチスレッドでの最適化で置きがちなメモリアクセスの順序が起きると困る際に順番を入れ替えないように強制するためのフラグらしい。今日CPU入門で読んだやつだ。
