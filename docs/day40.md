昨日のノート[Chromium Maps](/szPe4BDiSAqq2Lk1DSycHw)で各Mapの実装についてまとめた。  
今日はその中で[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h)を読んでみる。

## flat_map クラス
[`flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=178-182;drc=e4622aaeccea84652488d1822c28c78b7115684f)クラスのテンプレートは以下の通り。
```cpp=
template <class Key,
          class Mapped,
          class Compare = std::less<>,
          class Container = std::vector<std::pair<Key, Mapped>>>
class flat_map : public ::base::internal::
                     flat_tree<Key, internal::GetFirst, Compare, Container>
```

keyとvalueはそれぞれ`Key`, `Mapped`タイプで指定できる。  
そしてflat_mapはソートされたvectorなので、Compare演算が必要になり、これはデフォルトでは[`std::less`](https://cpprefjp.github.io/reference/functional/less.html)が指定されている。  
Containerは実際にkeyとvalueを保持する型であり、デフォルトでは`std::vector<std::pair<Key, Mapped>>`。std::pairのstd::vectorである。

flat_mapは[`flat_tree`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=148;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)を継承しており、本体の実装はほぼ[`flat_tree`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=148;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)の中にあるので以下[`flat_tree`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=148;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)のコードも読みながら解説する。  

flat_map固有の関数について見ていく。

### Look Up
以下のように実装されており、登録されていない`key`を探そうとしたらCHECKで落ちてクラッシュする。
```cpp=
template <class Key, class Mapped, class Compare, class Container>
template <class K>
auto flat_map<Key, Mapped, Compare, Container>::at(const K& key)
    -> mapped_type& {
  iterator found = tree::find(key);
  CHECK(found != tree::end());
  return found->second;
}
```
templateが2行あってキモい。  
これはclassでのtemplate引数および関数[at](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=216;drc=e4622aaeccea84652488d1822c28c78b7115684f)に対するtemplate引数それぞれあるから出現している。  
こういう場合外側で定義されているtemplete引数から順に複数行書いていく。上の例ではclass→methodの順。


中身の実装は[tree:find](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=978;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)。

```cpp=
template <class Key, class GetKeyFromValue, class KeyCompare, class Container>
auto flat_tree<Key, GetKeyFromValue, KeyCompare, Container>::find(
    const Key& key) const -> const_iterator {
  auto eq_range = equal_range(key);
  return (eq_range.first == eq_range.second) ? end() : eq_range.first;
}
```

[tree::equal_range](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=1028;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)とかいうやつがでてきた。  
これは[KeyValueCompare](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=461;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)型の比較演算を使用して`key`と同じになる範囲を探しているが、ここではもし一致するkeyが見つからなければ`{lower, lower}`という同じイテレータを返し、見つかった場合は`{lower, std::next(lower)}`というように1つ分のエリアを返している。おそらくflat_treeでは各Keyはたかだか一個しか登録されていないのでstd::next(イテレータを1個進める関数)でいいのだろうと思う。  
標準ライブラリにも[`std::ranges::equal_range`](https://cpprefjp.github.io/reference/algorithm/ranges_equal_range.html)というのがいて、これの戻り値は`{ranges::lower_bound(), ranges::upper_bounds()}`と書いてある。  

その[tree::equal_range](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=1028;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)から返ってきたやつがfirst, secondと同じ値なら見つからなかったってことなので`end()`を返し、同じでなければそのイテレータの左端、つまり`eq_range.first`を返す。  

ところで、iteratorとconst_iteratorはどちらもiteratorだが、const_iteratorは`iter++`ができない。ただしイテレータ自身がconstなだけでその値はconstではないので`*iter=value`はできる。  

### 挿入
以下のコードは `m[key]`みたいに書くやつ。
```cpp=
// m[key] = value ってやつ
template <class Key, class Mapped, class Compare, class Container>
auto flat_map<Key, Mapped, Compare, Container>::operator[](const key_type& key)
    -> mapped_type& {
  iterator found = tree::lower_bound(key);
  if (found == tree::end() || tree::key_comp()(key, found->first))
    found = tree::unsafe_emplace(found, key, mapped_type());
  return found->second;
}
```
# base::flat_map + 戻り値の型推論

[tree::lower_bound](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=1067;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)でkeyに対応するところを二分探索で見つける。  
この実装は[`ranges::lower_bound(*this, key, comp);`](https://source.chromium.org/chromium/chromium/src/+/main:base/ranges/algorithm.h;l=3106;drc=6d1cf699abe0ca8158015acc39b77606f327f972)で、その中身は普通に[std::lower_bound](https://cpprefjp.github.io/reference/algorithm/lower_bound.html)っぽい？
読めないのでまた今度。  

[tree::key_comp](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/containers/flat_tree.h;l=567;drc=dc11a1d175fad2a88df80a83b1b0cf2f20d348bd)はflat_mapに渡したKeyCompare型の値[`comp`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=375;drc=e4622aaeccea84652488d1822c28c78b7115684f)が保存されている。
つまり`key`が見つからないか、`key`＜見つかったやつの`key`が`comp`ならemplaceする。（key_compがtrueの条件の意味がよくわからない…）。よくわからんが見つからなかった場合mapped_typeのデフォルトコンストラクタを呼んでvalueとして代入して終わり。  
tree::unsafe_emplaceの中身は`body_.emplace`。ここで`body_`はコンテナのことなので、flat_treeにおいてはvector。  

ところで添え字に対する演算子のオーバーロードはなんか分かりにくい（当社比）。  
`m[key]`とすると`key`を右辺のオペランドとして受け取る。なので`operator[]()`が呼ばれることになる。この時`mapped_type&`と指定されているように返り値は参照渡し。なので、`m[key] = value`と書けばそのポインタに値が上書きされることになる。  


他にも[insert_or_assign](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=299;drc=e4622aaeccea84652488d1822c28c78b7115684f), [try_emplace](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=324;drc=e4622aaeccea84652488d1822c28c78b7115684f)も定義されている。  


ところで[`value_type`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h;l=191;drc=e4622aaeccea84652488d1822c28c78b7115684f)とかいらないusing宣言してね？消していい？  

### なぜContainer？
flat_mapは中身の実装はvectorということになっていたはずだが、この部分はContainerというtemplate引数によって指定できるようになっている。  
これは、Containerの中身がstd::arrayになっている[`base::fixed_flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/fixed_flat_map.h;l=89;drc=7ca1c1faf3ec5ff1c1faba93f7260d0c71c38d5e)を実装するためっぽい。実際[Containerが導入されて](https://chromium-review.googlesource.com/c/chromium/src/+/2510249)から10日後に[fixed_flat_mapが実装](https://chromium-review.googlesource.com/c/chromium/src/+/2532247)されいている。  

ただこの場合、flat_mapのContainerにオレオレ実装を挿入して計算量やメモリ使用量など[Chromium Maps](/szPe4BDiSAqq2Lk1DSycHw)で確認した性能から破壊したりできちゃうのでは？


## flat_set
[`flat_set`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_set.h;l=157;drc=e4622aaeccea84652488d1822c28c78b7115684f)はstd::set-likeなインターフェースで、flat_mapと同様にsortedなvectorを持つ。  
flat_treeは[flat_map](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h)と[flat_set](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_set.h)両方の実装になっている。  
Containerに`std::vector<Key>`を持つ。

## 脱線：戻り値の型推論について
上のセクションで以下のような矢印を含むtemplateが登場した。  
```cpp=
template <class Key, class Mapped, class Compare, class Container>
template <class K>
auto flat_map<Key, Mapped, Compare, Container>::at(const K& key)
    -> mapped_type& {
  iterator found = tree::find(key);
  CHECK(found != tree::end());
  return found->second;
}
```
 `-> 型名` みたいなのを引数の後にくっつけると戻り値の型推論ができる。このケースでは推論はしておらずtemplate引数から定まっている`mapped_type&`を使っているので推論は役に立っていないが、この[後置する関数宣言構文](https://cpprefjp.github.io/lang/cpp11/trailing_return_types.html)はC++11で導入された。  
しかしこの書き方では関数宣言の戻り値の型としてもう一度型を書かないといけない。上でもflat_mapの前にautoと指定していて二度手間。  

それを解決するためにC++14からは[通常関数の戻り値推論](https://cpprefjp.github.io/lang/cpp14/return_type_deduction_for_normal_functions.html)の仕様が追加された。以下のように戻り値にautoって書くだけで戻り値の型が関数のreturn文から推論されるようになった。
```cpp=
auto& flat_map<Key, Mapped, Compare, Container>::at(const K& key)
```
多分flat_treeのコードもこの形式にアップデートできる？でも返り値が非自明になって読みにくい可能性もある。  
と思いGoogleのCoding Styleを見に行ったら[Trailing Return Type Syntax](https://google.github.io/styleguide/cppguide.html#trailing_return)というセクションが合った。結論読みやすい方にしようって感じで、Trailing Return Typeは普通の書き方だと読みにくい場合だけ使用しろとのこと。  

TODO(elkurin):上の例ではなぜ`mapped_type& flat_map<...>`みたいに書いてないの？
