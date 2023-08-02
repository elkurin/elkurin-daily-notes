# Chromium Maps

C++の標準ライブラリには [`std::map`](https://cpprefjp.github.io/reference/map/map.html) と [`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html) というマップの実装がある。  
前者は二分木、後者はハッシュテーブルを使用しており、それぞれtradeoffがある。  
それに加え、Chromiumには [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h) と [`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h) というstd::map-likeなインターフェースが実装されている。  
これらの違いをまとめる。

## 比較
[参考](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/README.md#Map%20and%20set%20details)

各実装を一言でいうと
- [`std::map`](https://cpprefjp.github.io/reference/map/map.html)　→　赤黒木
- [`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)　→　ハッシュテーブル
- [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h)　→　ソートした配列
- [`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h)　→　インラインバッファー？（map,unordered_mapにオーバーフローする）。コードを読まないとわからない…　→読みました [ノートはこちら](/docs/day44.md)

### Speed Performance
[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)は insert/delete がO(1)でできるが、それほど大きくないテーブルでは [`std::map`](https://cpprefjp.github.io/reference/map/map.html) と大差ない。  
[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h) は中身は`std::vector`なのでO(n)かかってしまう。ある程度大きいテーブルでは避けたほうが良さそう。なおsortedなためlook upはO(logN)でできる。  
計算量の麺から見ると[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)が最速だが、実行時間は必ずしもそうではない。  
[参考](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/README.md#std::unordered_map%20and%20std::unordered_set)に書いてあるベンチー膜によると、1Mのintegerを挿入するのに `std::unordered_set` の方が `std::set` の1.07xかかったらしい。query全般には0.67xしかかからず早かったそうだが、他のシナリオとして 4-entry set についてはqueryのパフォーマンスは`std::unordered_set`, `std::set`, `base::flat_set`すべて同じだったそうだ。  
さらにARMデバイスでは割り算が遅いのでハッシュキーの計算にちょっと時間がかかり[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)のパフォーマンスはより悪くなりがちらしい。  
早いので[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)！とやると罠にハマる。

### Memory Performance
以下64-bit platform。各データのoverheadについて考える。  
- [`std::map`](https://cpprefjp.github.io/reference/map/map.html) は赤黒木で各ノードは隣接する3点のポインタと色の情報を持つので、overhead は32bytes。
- [`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html) は std::vector + std::forward_list で実装されている。初期は8entriesあって、8 item追加されると64 entriesに拡張され、それ以後は倍に増えていく。各ノードはlibc++ではリストのポインタひとつを持ち、それに加え使用されていないテーブルエリアがoverheadとなっているので平均16bytes。
- [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h) は 内部の構造は`std::vector`なので各データのoverheadはない。
- [`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h) は [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h) に比べてmutationのパフォーマンス分ランタイムのメモリ消費効率がいいらしいがよくわからん。また今度コードを読みます。

### コードサイズ
[`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h) 以外はだいたい同じ。[`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h)だけ他の1.5倍程度あり、これは[`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h)がmapとunordered_map両方の実装をバックに持つから。


## 結局どれを使えばいい？
* データの数が常に少ない場合なら [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/flat_map.h) が良いことが多い。
  * ただしオブジェクトをたくさん作る場合は [`base::small_map`](https://source.chromium.org/chromium/chromium/src/+/main:base/containers/small_map.h) の方が良い
* データテーブルがめっっちゃでかいときは [`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)
* よくわからんときは [`std::map`](https://cpprefjp.github.io/reference/map/map.html)。これを使っておけば酷いことにはならない

## 使用例
体感 [`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/intent_helper/arc_intent_helper_mojo_delegate.h;l=44;drc=12be03159fe22cd4ef291e9561762531c2589539) が使用されていることが多いように感じる。  
[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)の使用例はテストを除くとかなり少なかった。

- Mojo の map は　[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/intent_helper/arc_intent_helper_mojo_delegate.h;l=44;drc=12be03159fe22cd4ef291e9561762531c2589539) を採用している。Mojoはプロセス間通信のためのプロトコルで、当然プロセス間ででかいデータをやりとりしたくはないので、渡すパラメータは小さいことを想定して良い。そのため[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/intent_helper/arc_intent_helper_mojo_delegate.h;l=44;drc=12be03159fe22cd4ef291e9561762531c2589539)を選択していると思う。([MojoのIDL解説書](https://chromium.googlesource.com/chromium/src/mojo/+/HEAD/public/tools/bindings/README.md)を読んだが内部の生成については見つからなかった。また生成コードを読んで見る)
以前Lacros用intent helperの実装をしていたときに[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/intent_helper/arc_intent_helper_mojo_delegate.h;l=44;drc=12be03159fe22cd4ef291e9561762531c2589539)を使ったことがある。
- `wl_client` とその `wl_resource` のマップ[`output_ids_`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/wayland_display_output.h;l=61;drc=ae0e89174260e68c0cbddca80dc9535624a9eeed)も　[`base::flat_map`](https://source.chromium.org/chromium/chromium/src/+/main:components/arc/common/intent_helper/arc_intent_helper_mojo_delegate.h;l=44;drc=12be03159fe22cd4ef291e9561762531c2589539)だった。大量のクライアントが生成されることはないと想定していそう。

- [`pending_request_ids_`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/zcr_ui_controls.cc;l=40;drc=a046ef85758c98df4e939b3060566e1604fb504a)は[`std::map`](https://cpprefjp.github.io/reference/map/map.html)を使っていた。特に理由はなさそう？
- [arc::OpenWithMenu::HandlerMap](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/chromeos/arc/open_with_menu.h;l=32;drc=4a8573cb240df29b0e4d9820303538fb28e31d84)は[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)を使用していた。このMapは各アプリごとのintent handlerをまとめており、アプリ数はUserによってはかなり多くなりがちなので[`std::unordered_map`](https://cpprefjp.github.io/reference/unordered_map/unordered_map.html)なのだと思う。


## Note
`base::fixed_flat_map` というやつもいた。これは`std::vector`の代わりに`std::array`を使っている。  
今度flat_map,small_mapを読む回をやります。
