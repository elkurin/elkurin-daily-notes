# Dangling Pointer

Chromiumでは最近[BackupRefPtr](https://chromium.googlesource.com/chromium/src/+/ddc017f9569973a731a574be4199d8400616f5a5/base/memory/raw_ptr.md)(MiraclePtr)という進化版raw ptrが導入され、正しくポインタを使おう運動が起きている。  
実際自分も存在しないかもしれないポインタにアクセスするカスコードを埋め込んでいたことに気づかず1年以上経ち、最近の正しくポインタを使おう運動で検知されて直したりした([CL](https://chromium-review.googlesource.com/c/chromium/src/+/4159743))。  
このCLはかなりしょうもないミスだが、非自明な不正アクセスもあり、これらは無効な領域を指すポインタ Dangling Ptr と呼ばれる。

## Chromium での Dangling Ptr の扱い
Dangling Ptrを消していこうとはしているものの様々な事情によりすぐには取り除けないケースもある。そのようなポインタは以下のようなTraitsがつけられている。
```cpp=
raw_ptr<Window, DanglingUntriaged> window_;
```
DanglingUntriagedがついているものは、Dangling Ptrであることが知られているがまだ直されてないやつ。  
Dangling PtrはBuild flagに `enable_dangling_raw_ptr_checks = true` を追加しておくとチェックできる。

我々Chromium Contributorの推奨行動は[docs/dangling_ptr_guide.md](https://source.chromium.org/chromium/chromium/src/+/main:docs/dangling_ptr_guide.md)に書いてあった。  
1. 問題のDangling PtrのComponents Ownerではない場合
→直さなくてもいいけどDanglingUntriagedというフラグをつけてね、直してくれると嬉しいよ（逆に言うとOwnerの人は直しましょう!）
2. Dangling Ptr がobjectの所有権を持っていない場合
→ 破壊の順番を確認・Observer callbackを確認・循環ポインタは互いにnullをセット・lifespanが難しい場合はptrをそもそも消す・最終手段はDisableDanglingPtrDetectionをセット
3. Dangling Ptr がobjectの所有権を持っている場合
→スマートポインタを使おう・fileなどC APIなら実質new/deleteになるものも使用可・最終手段はClearAndDeletなどのraw_ptrメソッドを使う

## Dangling Ptr あるあるパターン
[参考](https://docs.google.com/document/d/11YYsyPF9rQv_QFf982Khie3YuNPXV0NdhzJPojpZfco/edit?usp=sharing&resourcekey=0-h1dr1uDzZGU7YWHth5TRAQ)　一般公開されているDocumentです。

たくさんのDangling Ptrが排除されていく中でそのノウハウをまとめているDocがあった。  
これからのクラッシュを避けるために学んでおく。  
以下にDangling Ptrをどうやって直すかの方法をグループに分けて紹介。

### raw_ptr 以外のポインタを使うべき
占有率24%。

Chromiumはstd::unique_ptrかscoped_refptrのスマートポインタを[strongly encorage](https://chromium.googlesource.com/chromium/src/+/main/docs/dangling_ptr_guide.md#the-pointer-manages-ownership-over-the-object)している。それはそう。  
Onwershipを持って良いポインタの場合はraw_ptrをunique_ptrに書き換えることができる。実際これで15%弱のDangling Ptrが直ったとのこと。


### 初期化・リセットの順番を正しくする
占有率25%。

依存関係にあるオブジェクトのdestruction順序がおかしいケースでもDangling Ptrが発生する。  
以下[Doc](https://docs.google.com/document/d/11YYsyPF9rQv_QFf982Khie3YuNPXV0NdhzJPojpZfco/edit?resourcekey=0-h1dr1uDzZGU7YWHth5TRAQ#bookmark=id.jgjtzldk9pvc)を引用。
```cpp=
class A {
 public:
  void Action() {
    b_ = std::make_unique<B>();
    c_ = std::make_unique<C>(b.get());
  }

 private:
  std::unique_ptr<C> c_;
  std::unique_ptr<B> b_;
}

class C {
 public:
  C(B* b) : b_(b) = default;

 private:
  raw_ptr<B, DanglingUntriaged> b_;
}
```
このケースでは、`b_`は`c_`に参照されているので、`b_`が先にdestructionされて次に`c_`のdestructorを走らせるとその中で`c_`を参照する可能性がある。  
ところで、クラスの各インスタンスはdefaultではクラス生成時には宣言順にconstructorが呼ばれ、クラス破壊時には宣言の逆順にconstructorが呼ばれる。この例の場合は`b_`を破壊してから`c_`を破壊しているのでdangling ptrの危険がある。  
似たようなケースでTearDown()関数などの中で`reset`を明示的に呼んでいくコードがあるが、同様にreset順に気をつけないといけない。

上述のコード例では`b_`と`c_`の宣言順番を変えるだけで良い。  
ただしこれ系統で難しいことも多い。例えばBrowserのスタートアップやWindow生成など階層が深いケースではもはや誰が誰に依存してるのかよくわからない。  
[WindowTreeHostの`window_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=451;drc=6f29856c2e4ca895819f2f5a72b7da905aec1f19)の例ではwindowのdestructorがまたwindowにreach backしてくるケースがあるのでその間活かしておくためにDanglingUntriagedをそのままにしている。  
どうしてもDanglingのほうが良いという最終形態にいくと
```cpp=
raw_ptr<T, DisableDanglingPtrDetection> dangling_ptr;
``` 
になるらしい。  
例えば[bind_internal.h](https://source.chromium.org/chromium/chromium/src/+/main:base/functional/bind_internal.h;l=267;drc=64b025b7b6de9963d71438a99a5a624e0d63ca44)の中で使われている。これはweak ptrを`Unretained()`で獲得する際に通るパス。

### raw_ptr をリセットする
占有率39%。

std::unique_ptrを使えばそもそもリセットしてくれるので理想だが様々な事情によりunique_ptrにはできないこともある。以下のようにメンバ変数のインスタンスが所有権を持っているポインタを親クラスも持ちたいときなどは明示的にresetしないといけない(nullptrをいれないといけない)が、それをしないとDangling Ptrになる。  
[Doc](https://docs.google.com/document/d/11YYsyPF9rQv_QFf982Khie3YuNPXV0NdhzJPojpZfco/edit?resourcekey=0-h1dr1uDzZGU7YWHth5TRAQ#bookmark=id.afszjd68fk4t)から引用
```cpp=
class A {
 public:
  void Initialize() {
    b_.Initialize();
    c_ = b_->GetC();
  }

  void Shutdown() {
    b_.Shutdown(); // `c_` is dangling
  }

 private:
  B b_;
  raw_ptr<C, DanglingUntriaged> c_;
}
```

またよくあるパターンとしてObserverの中でobjectをresetするべきなケース。  
Chromiumには`OnDestroyed()`とか`OnClosed()`のようなdestruction時に呼ぶobserverが多用されている。例えば[WindowDelegate::OnWindowDestroyed](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_delegate.h;l=81;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)、[DesktopNativeWidgetAura::OnHostClosed](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h;l=80;drc=3a06e758a5820009ee0f2590b08a311f00d76163)。  
これらのObserverをoverrideして親クラスを作り、親クラスはraw_ptrとしてObserveしたいものを持ちつつDestructionされたときには`OnDestroyed`などから通知を受け取るというデザインだが、このときにnullptrをセットし忘れているケースも散見されている。  
そもそもObserverを追加してリセットしなければならないのにしていないケースもありそう。

## Note
気になる[kExperimentalAsh](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=114;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)というTraitsがある。  
最近よくみるようになったやつだが、BackupRefPtrをChromeOSでenableするための準備らしい。
自分の理解ではBackupRefPtrはPartitionAllocでないと動かないはずだからChromiumのコードベース以外だと無理なはず。どうやってサポートするんだろう
