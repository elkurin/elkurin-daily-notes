# atexit in Chromium

テスト終了時にポインタチェックが走ってクラッシュすることがある。  
その時~AtExitManager()がStackTraceに出てきている。  
これは何？

## atexit
[atexit](https://en.cppreference.com/w/c/program/atexit)はプログラムが正常に終了したときに呼び出す関数を登録できる関数でCの標準ライブラリ stdlib.h に定義されている。  
正常に終了したとは、main関数を抜けた時またはexit関数が呼ばれた時を指す。  
呼ぶ順番は登録された逆順。デストラクタとかと同じ。  

`atexit`はもし登録が正しく行えれば`0`を返し、そうでない場合はnon-zeroを返す。  
登録できる関数の個数には制限があるが、これは各実装依存ということになっている。32以上であることは保証されているらしい。

## AtExitManager
Chromiumの中では`atexit`は使われておらずその代わりに自前実装の[AtExitManager](https://source.chromium.org/chromium/chromium/src/+/main:base/at_exit.h;l=32;drc=63e1f9974bc57b0ca12d790b2a73e5ba7f5cec6e)が使用されている。

[AtExitManager](https://source.chromium.org/chromium/chromium/src/+/main:base/at_exit.h;l=32;drc=63e1f9974bc57b0ca12d790b2a73e5ba7f5cec6e)はChromium内の`atexit`に値するクラス。  
違いは登録されたcallbackの呼ぶタイミングをコントロールできること。
```cpp=
int main(...) {
  base::AtExitManager exit_manger;
  ...;
}
```
AtExitManagerのインスタンスを最初に作り、そのスコープが切れたタイミングで[dtor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/at_exit.cc;l=37;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)が呼ばれ[ProcessCallbacksNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/at_exit.cc;l=70;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)から積まれてるタスクを走らせていく。  
なおこのインスタンスは`g_top_manager`というグローバルシングルトンで管理されており、AtExitManagerへの関数の登録はstatic関数になっている。  
`g_top_manager`はstaticで定義されており、必ず最後にディストラクタが呼ばれるのでbase::NoDestructorなどを使うシングルトンのデザインにはならない。

積まれているタスクは[RegisterTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/at_exit.cc;l=56;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)から[`stack_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/at_exit.h;l=69;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)にbase::stackとして保存されている。  
[ProcessCallbacksNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/at_exit.cc;l=70;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)はこの`stack_`からpopしてtaskを走らせている。

いつも見てたポインタチェックのやつは[これ](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_alloc_support.cc;l=619;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)っぽい。  
テスト終了後に[CheckDanglingRawPtrBufferEmpty](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_alloc_support.cc;l=579;drc=b86abfa407f3f35d3e5904e76b9a8d3a741bdc8b)を走らせて、最後までリークしているポインタを検出している。例えばグローバルインスタンスから参照されているとか。

