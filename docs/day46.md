# Scoped Observer

以前[TabletState](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/ui/base/tablet_state.h)のコードスタックを呼んでいたときに、observerがどうやって登録されているのかわからなくて困ったことがあった。  
ちゃんと読む。

## Observerのデザインパターン
ObserverとはSubjectの状態が変化したことを通知するデザイン。Subjectから通知を受けたいクラスをオブザーバとして登録する。  
例えば、Networkのコネクション(IPアドレス、wifiから有線へ etc...)が変わったら、その変化を認知している。  
[NetworkChangeManagerClient](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/network_change_manager_client.cc;l=213;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)が[`lacros_network_change_observers_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/network_change_manager_client.h;l=126;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)に登録されているobserverたちに対し[OnNetworkChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/network_change_manager_client.cc;l=246;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)を呼ぶ。すると各Observerたちは各々のoverrideした実装に応じてNetworkChangeに対するアクションを取れる。この場合は[NetworkChangeManagerBridge::OnNetworkChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/lacros/net/network_change_manager_bridge.cc;l=84;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)が呼ばれることになる。  

このデザインは直接的な依存関係をなくすことができる。通知側はされる側の挙動を追う必要はなく、その実装はobserverを継承したクラス側に委ねることが出来る。  
通知する側と通知される側の関係が一対多のときに有効。

## Observerの書き方 in Chromium
よくみるobserverのデザインパターンでは、登録先のインスタンスをどうにかこうにか獲得して`instance->AddObserver(this)`みたいにするか、登録先の中でobserverになるインスタンスを生成して`AddObserver(std::make_unique<Observer>())`みたいにキープするかだった。  

```cpp=
// 登録先のインスタンスを獲得して登録するパターン。以下の2ケースが多い
// 1. staticのゲット関数からシングルトンを取ってくる
// 2. コンストラクタ引数でreferenceをもらう
ShellSurfaceBase::ShellSurfaceBase(...) {
  ...;
  WMHelper::GetInstance()->AddTooltipObserver(this);
  ...;
}
...
void TooltipLacros::AddObserver(wm::TooltipObserver* observer) {
  observers_.AddObserver(observer);
}

// 登録先でインスタンスを生成して登録するパターン
ArcIdleManager::ArcIdleManager(...) {
  AddObserver(std::make_unique<ArcCpuThrottleObserver>());
  AddObserver(std::make_unique<ArcBackgroundServiceObserver>());
  AddObserver(std::make_unique<ArcWindowObserver>());
  ...;
}
```

TabletStateのクラスではちょっと違う書き方をしていた。  
```cpp=
display::ScopedDisplayObserver display_observer_{this};
```
これは何？

## ScopedDisplayObserver

[ScopedDisplayObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.h;l=94;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)は[ScopedOptionalDisplayObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.h;l=83;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)のコンストラクタに`observer`をわたしているだけ。  
[ScopedOptionalDisplayObserverのctor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.cc;l=32;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)では`display::Screen::GetScreen()`でobserverの登録先インスタンスである`screen`を取ってきており、`screen`に対して`AddObserver(observer_)`している。  
`observer`はScopedOptionalDisplayObserverのprivateメンバ`observer_`としてreferenceを保存しておく。このおかげで[dtor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.cc;l=40;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)で`screen->RemoveObserver(observer_)`でobserverを正しく消すことが出来る。  
デザインパターンとしては登録先のインスタンスを獲得して登録するパターンに似ている。

では何が嬉しいか？  
Observerを勝手に追加・削除してくれる！  
observerの消し忘れによるDangling Ptrは頻出でみんな忘れがちなのでありがたい。  
またctor/dtorに余計なAdd/Removeが多すぎると読みにくい。  
こういうとき、ScopedObserverをクラスの中で作ってしまうことでdesturct時に自動で寿命が切れるようになっていると便利。

## ScopedObservation
ところで今日のお題はもともと[ScopedObservation](https://source.chromium.org/chromium/chromium/src/+/main:base/scoped_observation.h)にしようと思っていた。  
ノートを書くために自分が以前見た実用例を見に行ったら、ScopedObservationは使われておらず自前で作っていたので[ScopedDisplayObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.h;l=94;drc=9d4eb7ed25296abba8fd525a6bdd0fdbf4bcdd9f)単体について書くことになった。  

[ScopedObservation](https://source.chromium.org/chromium/chromium/src/+/main:base/scoped_observation.h)もコンセプトはほぼ同じ。  
```cpp=
template <class Source, class Observer>
class ScopedObservation {
...
```
Source型が登録先、Observer型がオブザーバ。  
DisplayObserverの例では、Sourceに`display::Screen`、Observerに`display::DisplayObserver`となる。  
やっていることはScopedDisplayObserverと同じだが、ではなぜScopedObservationを使っていないか？  
それはDisplayObserverにおいてはAdd時とRemove時に同じscreenとは限らないため、毎回GetScreen()で会得する必要があるから。
