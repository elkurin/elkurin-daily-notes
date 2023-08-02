# LRU Cache in Chromium

baseディレクトリの中を散歩していたら[lru_cache.h](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/lru_cache.h)というファイルを見つけたので読んでみる。

## LRU Cache とは
**L**east **R**ecently **U**sed **Cache**。  
キャッシュ容量がパンパンになったときに、使用履歴が一番古いものを消す方式のキャッシュのこと。
いろんなところで使われている有効なコンセプト。  
最近だと[Frame Eviction](/docs/day37.md)のノートで読んだ[FrameEvictionManager](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/client/frame_eviction_manager.h)で`unlocked_frames_`のリストを管理するために使用しているのを見た。  
上の例では自前で実装しているが、Chromiumには[base::LRUCache](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h)というデータ構造も実装されている。  
以下のように書くと使える。
```cpp=
base::LRUCache<GURL, std::unique_ptr<CacheEntry>> cache_;
```

## LRUCacheの実装
```cpp=
template <class ValueType, class GetKeyFromValue, class KeyIndexTemplate>
class LRUCacheBase {
```
ValueType型でストアするキャッシュの値。  
データは`std::list<value_type>`型の`ordering_`に保存される。  
キーはKeyIndex型の`index_`に保存されている。  
KeyIndexTemplateにはstd::unordered_mapの[HashingLRUCacheKeyIndex](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=237;drc=f09c12c84b39d13189a7039a05253ca3766d4751)か、std::mapの[LRUCacheKeyIndex](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=231;drc=f09c12c84b39d13189a7039a05253ca3766d4751)が使用される。  

`index_`がkeyとvalueの組み合わせをハッシュテーブルまたはマップで持ち、`ordering_`でLRUキャッシュとしての順番を保存している。  
LRUキャッシュの最大サイズは[コンストラクタ](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=78;drc=f09c12c84b39d13189a7039a05253ca3766d4751)で設定される。

### Look up
[Get](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=130;drc=f09c12c84b39d13189a7039a05253ca3766d4751)というAPIがやる。  

普通に`key`を受け取ってそれに対応するものを`index_`から探し返す。  
そして`ordering_.splice(ordering_.begin(), ordering_, iter);`で対応する`iter`を`ordering_`の先頭に持ってくる。

また[Peek](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=143;drc=f09c12c84b39d13189a7039a05253ca3766d4751)では順序に影響を与えずに値だけ参照することができる。そのため`const_iterator Peek(const key_type& key) const`の関数ではconst装飾子がついている。  

### 挿入
[Put](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/lru_cache.h;l=96;drc=2efb465248fe03de205fdfea00a01b88e0665c07)というAPIがやる。  

`value`を代入すると以下の実装が呼ばれる。
```cpp=
iterator Put(value_type&& value) {
  key_type key = GetKeyFromValue{}(value);
  typename KeyIndex::iterator index_iter = index_.find(key);
  if (index_iter != index_.end()) {
    Erase(index_iter->second);
  } else if (max_size_ != NO_AUTO_EVICT) {
    ShrinkToSize(max_size_ - 1);
  }
  ordering_.push_front(std::move(value));
  index_.emplace(std::move(key), ordering_.begin());
  return ordering_.begin();
}
```
もしvalueに対応するkeyがあるなら[Erase](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=166;drc=f09c12c84b39d13189a7039a05253ca3766d4751)を呼んで`index_`と`ordering_`からデータを消して`ordering_`にpush_frontする。  
新しいvalueなら[ShrinkToSize](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=182;drc=f09c12c84b39d13189a7039a05253ca3766d4751)を呼んで、もしサイズが`max_size_`を超えたらその分うしろから消し、やはり`ordering_`にpush_front。  

## Note
KeyIndexの型定義がこうなっていた。  
```cpp=
typename KeyIndexTemplate::template Type<typename ValueList::iterator>;
```
全然読めない。

まず`<typename ValueList::iterator>`はなにか？  
これはテンプレートパラメータ内部にある型はそのままだと型名として解釈されないので、typenameをつけてやる必要がる。`using size_type = typename ValueList::size_type;`などでも、ValueListというテンプレートパラメータの内部にあるsize_type型を型名だと認識させるために`typename`装飾子をつけている。  
前半の`typename KeyIndexTemplate::template`でもKeyIndexTemplateの内部にある`template`を型名と認識させるためにつけている。  

次に`Type`は紛らわしいがただのテンプレート。[HashingLRUCacheKeyIndex](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=237;drc=f09c12c84b39d13189a7039a05253ca3766d4751)か[LRUCacheKeyIndex](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/lru_cache.h;l=231;drc=f09c12c84b39d13189a7039a05253ca3766d4751)の中で定義されている。  

最後に`::template`。これはテンプレートパラメータに突っ込まれている型のメンバtemplateを取り出す際にそれがテンプレートだと示すためにつけるキーワードらしい。  

というわけで、この一行は`KeyIndexTemplate`型に突っ込まれたtemplateである`Type`のテンプレート引数に`ValueList::iterator`型を渡した型だよってこと。
