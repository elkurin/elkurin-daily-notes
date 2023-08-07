# absl::flat_hash_map を読んでみよう

[absl::flat_hash_mapの概要](/docs/day72.md)の続編。  
今度は実装をみてみる。

## Swiss Tablesの構成
テーブルには`ctrl`と`slots`の2つがある。  
`slots`には本体のデータが格納されており、`ctrl`にはH2の7bitsもしくはempty/deletedを表す特殊値が格納されている。([absl::flat_hash_mapの概要](/docs/day72.md)で触れたとおり、flat_hash_mapにおけるハッシュ値は前57bitsからなるH1と後ろ7bitsからなるH2に分けて使用される。)  
両者とも各インデックスに対応するのは同じデータになっている。  
`ctrl`が`slots`と別々の配列に入っているのはパフォーマンス的にも良い。`slots`がキャッシュラインにもっと入るので。

このテーブルはGroupごとに分割されており、これはSSEでの最適化に活用されている。

## ハッシュと特殊値
[`H1`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=525;drc=df097ef9b722789243248f807ad66c7085a14099)は以下のような実装。
```cpp=
inline size_t PerTableSalt(const ctrl_t* ctrl) {
  return reinterpret_cast<uintptr_t>(ctrl) >> 12;
}

inline size_t H1(size_t hash, const ctrl_t* ctrl) {
  return (hash >> 7) ^ PerTableSalt(ctrl);
}
```
ハッシュ値をそのまま使用するのでなくソルトをかけている。  
PerTableSaltはイテレーションをランダムにするために使用される。

`ctrl_t`は以下のように定義されているenum型。
```cpp=
enum class ctrl_t : int8_t {
  kEmpty = -128,   // 0b10000000
  kDeleted = -2,   // 0b11111110
  kSentinel = -1,  // 0b11111111
};
```
この特殊値が最適化において重要な役割を占めている。

1byteからなるコントロール用の値。  
kEmptyはスロットが空の時に使われ、kDeletedはスロットが削除されたときに使われる。  
kSentinelはイテレーションをストップする場所を表す。イテレーション時にはempty,deletedのスロットはスキップし、sentinelで止まるようになっている。  
値が入っているときは先頭に1bitをたてて、残りの7bits分がハッシュ値になる。これがH2として使われるやつ。  
つまりこんな感じで詰め込まれている:
```
   empty: 1 0 0 0 0 0 0 0  
 deleted: 1 1 1 1 1 1 1 0  
    full: 0 h h h h h h h  // h represents the hash bits.  
sentinel: 1 1 1 1 1 1 1 1  
```
各特殊な値はSSE-flavered SIMDにチューニングされた値らしい。詳しくは後ろのセクションにて。



## keyの検索・挿入
アイテムをハッシュテーブルから検索したり挿入するのに[find_or_prepare_insert](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=2642;drc=df097ef9b722789243248f807ad66c7085a14099)が使われる。

ここでは[probe](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=1297;drc=df097ef9b722789243248f807ad66c7085a14099)で各スロットを変な順番でめぐる。

まずH2(hash(x))によってGroup内をスクリーニングして候補を探す。これは[Group::Match](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=593;drc=df097ef9b722789243248f807ad66c7085a14099)で行われる。  
もしその中にempty slotを見つけたらエラーを返す。  
もし値が同じだったら検索成功としそれを返す。  
それ以外の場合はprobe_seqの次へ移動する。  
probe_seqが一様でないことから[absl::flat_hash_mapの概要](/docs/day72.md)で言及した線形走査法ではない賢い感じのイテレーションをしている。

```cpp=
probe_seq(size_t hash, size_t mask) {
  assert(((mask + 1) & mask) == 0 && "not a mask");
  mask_ = mask;
  offset_ = hash & mask_;
}

void next() {
  index_ += Width;
  offset_ += index_;
  offset_ &= mask_;
}
```
`Width`は各グループの統一サイズ。Widthを`index_`に足すことで同じグループ内を探してしまわないようにしている。  
上記の式をまとめると
```
p(i) := Width * (i^2 + i)/2 + hash (mod mask + 1)
```
こんな感じでnextを決定し、次に探索するグループを返している。


## イテレーション
このハッシュテーブルにおけるiterationについて。
```cpp=
void skip_empty_or_deleted() {
  while (IsEmptyOrDeleted(*ctrl_)) {
    uint32_t shift = Group{ctrl_}.CountLeadingEmptyOrDeleted();
    ctrl_ += shift;
    slot_ += shift;
  }
  if (ABSL_PREDICT_FALSE(*ctrl_ == ctrl_t::kSentinel)) ctrl_ = nullptr;
}
```
IsEmptyOrDeletedで`ctrl_`にある値がempty/deleteかチェック。  
empty/deleteの場合はスキップしたいので1回イテレートする。  
[CountLeadingEmptyOrDeleted](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=620;drc=df097ef9b722789243248f807ad66c7085a14099)はkSentinentalと`ctrl`の">"をcmpgtで計算し、+1しておくことで0x00 or 0x01にしている。計算しやすそう。  
これを_mm_movemask_epi8でi8のレーンごとに計算しbitmaskを生成する。  
このbitmaskに対し[std::countr_zero](https://cpprefjp.github.io/reference/bit/countr_zero.html)で連続した0のビットを数える。連続した0とはつまりempty/deleteである要素の位置なので、この数分だけスキップすればいい。  
スキップする値が`shift`として返ってきているので、`ctrl_`, `slot_`の配列をshift分だけいてレートするとfilledまたはSentinelの値になるはず。  
TODO(elkurin): なぜ1発シフトで終了でなく、while文になっているの？


## SSE への最適化
SSE2以上のサポートがある場合、[GroupSse2Impl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=585;drc=df097ef9b722789243248f807ad66c7085a14099)を[Group](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/abseil-cpp/absl/container/internal/raw_hash_set.h;l=762;drc=df097ef9b722789243248f807ad66c7085a14099)の実装として使用する。  
kEmpty, kDelete, kSentinelの値はこの最適化のために定められている。  
面白そうなので読んでみる。

`kWidth=16`を各グループに割り振るスロットとしている。この16個分を
まずkEmpty, kDeleted, kSentinelはどれも0x80とのマスク（つまり最上位bit a.k.a. MSB)は1であるべき。各要素のMSBのみを使用するSSEの命令である[blendvpd](https://www.felixcloutier.com/x86/blendvpd)があるからだと思う。  

kEmpty, kDeletedよりkSentinelが大きくないといけない。これによりIsEmptyOrDeletedが効率化。多分MSBチェック＋">"演算(_mm_cmpgt_epi8)をしている。  

kSentinelは-1にすることでメモリからSIMDレジスタへの読み込みをはしょっているっぽい。[pcmpeqd xmm](https://www.felixcloutier.com/x86/pcmpeqb:pcmpeqw:pcmpeqd)を使って効率化。これは`=`を計算する命令。  

kEemptyは-128にすることでSIMDでのIsEmptyチェックを効率化している。これは否定を高速に計算する[psignb xmm](https://www.felixcloutier.com/x86/psignb:psignw:psignd)を使うため。psignbのイメージは[こんな感じ](https://www.officedaytime.com/simd512e/simdimg/binop.php?f=psignb)。もちろん最初から11..111にしたほうが速そうだが、それだとkEmptyがkSentinelより小さい条件を満たせないので00..にして否定を使用することにしたのだと思う。  

kEmptyとkDeletedは、kSentinelは持たないbitを両者とも持っておくとMaskEmptyOrDeletedを効率化。まあ確かに。  

kDeletedは-2とすることでConvertSpecialToEmptyAndFullToDeletedを効率化。
```cpp=
void ConvertSpecialToEmptyAndFullToDeleted(ctrl_t* dst) const {
  // 各レーンで同じi8 vectorをつくる。
  auto msbs = _mm_set1_epi8(static_cast<char>(-128));
  auto x126 = _mm_set1_epi8(126);

  // zero vector
  auto zero = _mm_setzero_si128();
  
  // 2つのi8 vectorの">"を計算して0xff/0x00を返す。
  auto special_mask = _mm_cmpgt_epi8_fixed(zero, ctrl);
  
  // 2つのi128のOR
  auto res = _mm_or_si128(msbs, _mm_andnot_si128(special_mask, x126));

  // STORE
  _mm_storeu_si128(reinterpret_cast<__m128i*>(dst), res);
}
```

むずい。
