# ObserverList in Chromium

オブザーバのデザインパターンでは多くの場合通知する側とされる側が1対多の構図になっている。
通知する側は複数のObserverを抱えられるようになっているべき。  
また通知される側が自由に追加する形になっていてほしいが、これは逆に言えば自由に消せるようになっていてほしいいうこと。invalidateされたObserverへのアクセスはUAFである。  
それをうまく処理してくれる[ObserverList](https://source.chromium.org/chromium/chromium/src/+/main:base/observer_list.h)をいうクラスがあり、ChromiumでObserverを複数持つ場合は基本これが使われる。  
このノートではその中身を確認する。

## Overview
[ObserverList](https://source.chromium.org/chromium/chromium/src/+/main:base/observer_list.h)は複数のオブザーバを使う際に使用されるコンテナ。  
以下のように使われる。

```cpp=
// Observer
class BrowserManagerObserver : public base::CheckedObserver {
  virtual void OnMojoDisconected() {}
  ...;
};  

// Observerを持つインスタンス
base::ObserverList<BrowserManagerObserver> observers_;

// `observers_`に追加
void BrowserManager::AddObserver(BrowserManagerObserver* observer) {
  observers_.AddObserver(observer);
}

// `observers_`に登録されたオブザーバに通知
void BrowserManager::OnLoadComplete(...) {
  ...;
  for (auto& observer : observers_) {
    observer.OnLoadComplete(success, version);
  }  
} 
```
イテレータを回してすべてのobserverに通知するというパターンでよく使用される。  
vectorやlistと異なり、iteration中にObserverListを変更しても問題なく作動する。

## 実装
```cpp=
template <class ObserverType,
          bool check_empty = false,
          bool allow_reentrancy = true,
          class ObserverStorageType = internal::CheckedObserverAdapter>
class ObserverList {
  ...;
}; 
```
ObserverTypeにObserverの型が代入される。上の例では[BrowserManagerObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager_observer.h;l=13;drc=5578b78366de32caf83044e5ea2268e6d91af766)。  
オブザーバ本体たちは[`observers_`](https://source.chromium.org/chromium/chromium/src/+/main:base/observer_list.h;l=367;drc=bc4629d204ad160b9ac06bb6257d7bf6531bbc5e)にvector型で保存されている。このなかではObserverTypeではなくObserverStorageTypeを使っている。このStorageについては次のセクションで。  
外からオブザーバを追加する際はObserverTypeを使用してよい。
```cpp=
void AddObserver(ObserverType* obs) {
  // nullのオブザーバは禁止
  DCHECK(obs);
  // すでに登録しているオブザーバもだめ
  if (HasObserver(obs)) {
    NOTREACHED();
    return;
  }
  ...;
  observers_.emplace_back(ObserverStorageType(obs));
}
```

イテレータは[ObserverList::Iter](https://source.chromium.org/chromium/chromium/src/+/main:base/observer_list.h;l=120;drc=bc4629d204ad160b9ac06bb6257d7bf6531bbc5e)で定義されている。  
IterはObserverListへのweak ptrを[WeakLinkNode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=129;drc=5578b78366de32caf83044e5ea2268e6d91af766)としてもつ。`index_`がそのIterインスタンスが指すObserverのvectorの位置。  
```cpp=
Iter& operator++() {
  if (list_) {
    ++index_;
    EnsureValidIndex();
  }
  return *this;
}
```

### iteration中の挙動
Overviewで言った通りiteration中にobserverが破壊されても大丈夫なようになっている。その仕組みを見る。  
[EnsureValidIndex](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list.h;l=215;drc=5578b78366de32caf83044e5ea2268e6d91af766)の中で以下のように`index_`をアジャストしている。  
```cpp=
while(index_ < max_inex &&
      list_->observers_[index_].IsMarkedForRemoval()) {
  ++index_;
}
```
[IsMarkedForRemoval](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=85;drc=5578b78366de32caf83044e5ea2268e6d91af766)はObserverがまだ生きているかを確認している。Removalフラグは[MarkForRemoval](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=79;drc=5578b78366de32caf83044e5ea2268e6d91af766)によってマークされ、これは[RemoveObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list.h;l=294;drc=5578b78366de32caf83044e5ea2268e6d91af766)でObserverが削除されたときにマークされている。  
EnsureValidIndexによって、RemoveされたObserverをスキップしてiterationすることができている。  


## Checked/Unchecked Observer
[CheckedObserver](https://source.chromium.org/chromium/chromium/src/+/main:base/observer_list_types.h;l=26;drc=e4622aaeccea84652488d1822c28c78b7115684f)はObserverListに登録する用オブザーバの基底クラス。use-after-freeを防ぐ。  
`class BrowserManagerObserver : public base::CheckedObserver`のようにCheckedObserverを継承しているObserverの実装が推奨  

CheckObserverはStorageTypeとして[CheckedObserverAdapter](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=68;drc=5578b78366de32caf83044e5ea2268e6d91af766)を使用する。デフォルトはこちらで推奨もされているが、Uncheckedを使う場合は型が用意されている。  
```cpp=
base::ObserverList<DisplayObserver>::Unchecked observers_;
```
やむなく使っているというよりかはhistorical reasonっぽい。  

Uncheckedでは[UncheckedObserverAdapter](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=27;drc=5578b78366de32caf83044e5ea2268e6d91af766)を使用しており、違いはObserverをWeakPtrとして管理するかraw_ptrとして管理するか。  

[IsMarkedForRemoval](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/observer_list_internal.h;l=85;drc=5578b78366de32caf83044e5ea2268e6d91af766)の中でCheckedの場合は`CHECK(!weak_ptr_.WasInvalidated());`のチェックを行っている。ここで分かる通りRemovalはObserverを破壊する前に明示的に行う必要がある。  
例えばWindowObserverでは[OnWindowDestroyed](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_observer.h;l=150-158;drc=e98d214c48dd94a2b4363b384e952a32258b9487)を呼ぶ前に自動的にObserverを消すような実装になっており、OnWindowDestroydの実装内でRemoveObserverしなくてよいようになっている。  

以上のようにWeakPtrの場合はポインタのinvalidationを正しく検知しているが、raw_ptrはDanglingマークがついておりuse-after-freeの可能性がある実装になっている。  
