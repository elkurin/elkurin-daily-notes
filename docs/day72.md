# absl::flat_hash_mapの概要

mapのいろいろな実装について[Chromium Maps](/docs/day39.md)で見た。  
今日はそれらよりも更に良いという[absl::flat_hash_map](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/flat_hash_map.h)を学ぶ。


## Overview
[absl::flat_hash_map](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/flat_hash_map.h)はstd::unordered_mapの効率化バージョンとされているハッシュマップ。  
計算量はunordered_map同様に検索・挿入・削除すべてO(1)。  
そこからさらにメモリ・計算量的な最適化が行われているので、abslの推奨はunordered_mapを使いたいときはflat_hash_mapをデフォルトで使用すること。

```cpp=
// K := keyの型　CopyConstructibleであれ
// V := valueの型　MoveConstructibleであれ
absl::flat_hash_map<K, V> table;
```
各データは`std::pair<const K, V>`としてコンテナ内に保存されている。  
![](https://hackmd.io/_uploads/BJPEi0ssh.png)  
（引用：[abseilによる大変わかり易い図](https://abseil.io/tips/136))

## なぜunordered_mapより良い？
flat_hash_mapがunordered_mapより良いのはどうしてか？  
flat_hash_mapは[swiss tables](https://abseil.io/about/design/swisstables)というabseil独自のコンセプトを使用している。

ところでハッシュテーブルにはopen hashとclosed hashがあり、前者はハッシュ値が衝突した時同じハッシュ値をもつやつらをチェイン上につなげて保持する一方、後者はなんらかの法則に従い格納先を決め直して入れる。  
closed hashの方が構造がややこしくなるが連結リストがいらない分メモリ効率は良い。  
Note:ただし格納先を決め直すアルゴリズムは注意が必要で、1つ次のインデックスにいれる(線形走査法)みたいなことをすると使用済みエリアの偏りができやすくなってパフォーマンスが下がる。このアルゴリズム次第では時間的パフォーマンスも改善するはず。

unordered_mapはopen hash(Chaining)が採用されている。  
策定時にclosed hash(open addressing)との議論が行われていたようだがopen addressingはnot mature enoughということで不採用になったらしい。
> I'm not aware of any satisfactory implementation of open addressing in a generic framework. Open addressing presents a number of problems:
... (omitted) ...
Solving these problems could be an interesting research project, but, in the absence of implementation experience in the context of C++, it would be inappropriate to standardize an open-addressing container class.

([引用元](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2003/n1456.html))

一方flat_hash_mapが採用しているSwiss tablesはclosed hash。

### Swiss Tables
Swiss tables ではハッシュ値関数は64bits値を出力し、以下の2つの部位に分けて使用する。
* H1: 57bitsのハッシュ値。インデックスから計算される普通のやつに使われる。
* H2: 7bitsのハッシュ値。メタデータにアサインされているやつに使われる。

メタデータという概念が登場したが、Swiss tablesではハッシュテーブル本体とは別にメタデータのarrayを持っている。  
各メタデータは1byteからなり、最初のbitはis_empty()をいれ、残り7bitsにH2ハッシュをいれる。このH2部分はfullのときだけ意味のある値が入っており、emptyのときは0000000、deletedのときは1111111になっている。emptyとdeletedで扱いが違うことに注意。

このハッシュ値でどうやって検索するか？  
検索したいkeyに対して生成された64bitsハッシュ値について。まず最初の57bits分にあたるH1を使って"bucket chain"の先頭を探す。  
残りの7bitsからなるH2はmaskとする。  
**S**treaming **S**IMD **E**xtension a.k.a [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)というSIMD(**S**ingle **I**nstruction **M**ultiple **D**ata)命令セットの拡張を用いてめっちゃはやくスキャンし、まずはマスクにマッチするものをmetadata列を使ってスクリーニングする。~~SIMDは同一命令を複数CPUに投げる形式のマルチプロセッサなので、マスクして結果を返せっていう命令を各分割されたテーブルごとにやって集めるみたいな並列処理が行われているのだと思う~~(胡散臭いという指摘があったので消しました)。これで高速に候補を大きく絞ることができる。  
その候補たちの中からH1も一致するやつをさがし、あったら発見。なかった新たな候補たちをprobeする。  
... というアルゴリズムが[swiss tables desige notes](https://abseil.io/about/design/swisstables)に書いてあるが正直よくわからないので、今度コードを読んでみる。


## パフォーマンス
flat_hash_mapはかなり早く[benchmark](https://martin.ankerl.com/2022/08/27/hashmap-bench-01/#abslflat_hash_map-)ではデカいテーブルにおいて検索最速のデータ構造だった。  
一方コピーやイテレーションは比較的遅いらしい。

### absl::node_hash_mapとの比較
似た構造として[absl::node_hash_map](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/node_hash_map.h)というものもある。  
これはflat_hash_mapと近いコンセプトで、同様にunordered_mapの代替として機能する。なんならこっちのほうがそのまま置き換えやすい。  
flat_hash_mapはvalueをインラインに保持することで間接参照分のオーバーヘッドをなくしアクセス速度を上げている。  
一方[node_hash_map](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/node_hash_map.h)ではkeyとvalueのペアを"node"として外に持っており、コンテナには各nodeへのポインタを持っている。    
![](https://hackmd.io/_uploads/rkka5Ass3.png)  
（引用：[abseilによる大変わかり易い図](https://abseil.io/tips/136))

基本的にはflat_hash_mapが推奨されているようだが、valueがmove constructibleでない場合はこちらを使う。

この2つのパフォーマンス面の交換は  
* flat_hash_map はインライン分速い
* node_hash_map はコンテナ内の各スロットが小さい

となっている。  
空スロットが多くなりがちなハッシュテーブルという構造では各スロットが小さいメリットもある。  
もしvalueのサイズが大きいならflat_hash_mapのvalueとして`std::unique_ptr<V>`を使うことが推奨されている。  
しかしkeyのサイズまで大きい場合はnode_hash_mapを使用したほうがいい。

## サポート
### Chromiumでの扱い
現状テスト段階っぽい。  
[kSupportsUserDataFlatHashMap](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/features.cc;l=31;drc=1d33943ee5bbb63e8f0ef1d1ebf3a16705533ec4)というfeature flagがあり、これは disabled by default なので現在の既定値はdisabled.  
ユーザーデータについてflat_hash_mapのほうがいいから現在A/Bテスト中とのこと。→[CL](https://chromium-review.googlesource.com/c/chromium/src/+/4136838)

なおA/Bテストとかなしに使用している場所もごくわずか（テスト含め3箇所）あった。[例](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/chromeos/policy/dlp/dlp_copy_or_move_hook_delegate.h;l=57;drc=c4ed13bd93652fd14b2a7a6de222c92137f5ea84)

### 標準ライブラリでの扱い
現状flat_hash_mapのサポートについて言及は見つからなかった。  
しかし[Chromium Maps](/docs/day39.md)でも言及した[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h)というソートした配列として持つデータ構造についてはC++23から導入されるっぽい。  
[std::flat_map](https://cpprefjp.github.io/reference/flat_map.html)のリファレンスページがある。

導入理由は
> Overall, flat_map is meant to be a drop-in replacement for map, just with different time- and space-efficiency properties. Functionally it is not meant to do anything other than what we do with map now.

([引用](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0429r3.pdf))
機能変更はないがより時間・空間効率の良いデータ構造として置き換えたいとのこと。

## Notes
ポリシーを使ったお手本のような実装をしていそう。  
[FlatHashMapPolicy](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/flat_hash_map.h;l=564;drc=7a93809c02fc40a8c6cfccba456a974b92b4352e)と[NodeHashMapPolicy](https://source.chromium.org/chromium/chromium/src/+/main:third_party/abseil-cpp/absl/container/node_hash_map.h;l=548;drc=7a93809c02fc40a8c6cfccba456a974b92b4352e)のいれかえでflatとnodeの実装の切り替えをしていそう。  
今度ちゃんと読もう
