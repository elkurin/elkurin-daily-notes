# CallbackList in Chromium

複数のCallbackを保持するデータ構造として[base::CallbackList](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h)が用意されている。  
普通に`std::vector<base::OnceClosure>`のように書くこともできるのにCallback専用のデータ構造が用意されている。  
何ができるんだろう？


## APIs
以下がCallbackListに対してできること。
* [Add](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=152;drc=c8c3f3293e3bd132660e8f3058392fd49371515f): callbackを追加
* [set_removal_callback](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=172;drc=c8c3f3293e3bd132660e8f3058392fd49371515f): リストにあるcallbackが消されたときに走る`removal_callback_`を設定できる。RepeatingCallbackなので何度も呼べる
* [empty](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=178;drc=c8c3f3293e3bd132660e8f3058392fd49371515f): リストが空か返す
* [Notify](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=200;drc=c8c3f3293e3bd132660e8f3058392fd49371515f): リストにあるcallback全てをRunする。

## 実装のポイント
一度しか呼べないOnceCallbackと何度でも呼べるRepeatingCallbackどちらに対してもCallbackListがある。  
一つのリストにごちゃまぜにすることはできない。  
いずれも[CallbackListBase](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=131;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)を継承している。

### コンテナ
コールバックを持つ[`callback_`](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=249;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)は`typename CallbackListTraits<CallbackListImpl>::Callbacks`型。

Traitsは以下。
```cpp=
template <typename Signature>
struct CallbackListTraits<OnceCallbackList<Signature>> {
  using CallbackType = OnceCallback<Signature>;
  using Callbacks = std::list<CallbackType>;
};
template <typename Signature>
struct CallbackListTraits<RepeatingCallbackList<Signature>> {
  using CallbackType = RepeatingCallback<Signature>;
  using Callbacks = std::list<CallbackType>;
};
```
Callbackのコンテナとしては[std::list](https://en.cppreference.com/w/cpp/container/list)が採用されている。  
これはアイテム追加時にイテレータがinvalidateされない構造でないといけないから。  
iteration中にも空チェックやcallbackへの通知が出来るようになる。

[std::vector](https://en.cppreference.com/w/cpp/container/vector)ではアイテムの挿入時、キャパが変わったらすべてのイテレータがinvalidateされる。
> push_back, emplace_back:	If the vector changed capacity, all of them. If not, only end().

一方std::listはされないので適切。  

### Notify内のイテレーション
Notify時はすべてのCallbackに対してRunを走らせることになるのでイテレーションを行う。  
その際`it++`みたいな感じではなく、以下の`next_valid`関数でincrementしている。
```cpp=
const auto next_valid = [this](const auto it) {
  return std::find_if_not(it, callbacks_.end(), [](const auto& callback) {
    return callback.is_null();
  });
};
```
nullなcallbackはスキップしている。  
Addが途中で起きるかもしれないから`callbacks_.end()`は変わりうるのでキャッシュして使ってはいけない。

### イテレーション中の挙動
[`iterating_`](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=272;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)フラグがイテレーション中にセットされるようになっている。  
セットされているのは上述の[Notify](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=200;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)中で、`AutoReset<bool> iterating(&iterating_, true);`のみでtrueにフリップされている。  
[AutoReset](https://source.chromium.org/chromium/chromium/src/+/main:base/auto_reset.h;l=25;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)はスコープ内だけ特定の値にし抜けたらもとの値に戻すやつ。

`iterating_`がセットされている間はCallbackListのデストラクタが呼ばれないようにしないといけない。呼ぶとCHECKでクラッシュする。  
またNotify中にもう一度Notifyを呼ぶことはできて、その場合先に走ったNotifyは`removal_callback_`を呼ばないようにしている。  
また、登録されているCallbackが破壊されるときに走るよう設定されている[CancelCallback](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=253;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)の中でも挙動が変わり、it->Resetになる。これでイテレーション中でも壊れないようになっている。  
なおCancelCallbackは[CallbackListSubscription](https://source.chromium.org/chromium/chromium/src/+/main:base/callback_list.h;l=90;drc=c8c3f3293e3bd132660e8f3058392fd49371515f)に渡され、登録されたCancelCallbackは破壊時または再代入が起きたときに走る。

なお[CL](https://chromium-review.googlesource.com/c/chromium/src/+/2343954)前はemptyやNotifyはイテレーション中ブロックされていたが改善された。


## Note
また[Curiously Recurring Template Pattern](/docs/day75.md)が現れた。  
```cpp=
template <typename Signature>
class OnceCallbackList
    : public internal::CallbackListBase<OnceCallbackList<Signature>> {
  ...;
```
これはコンテナのCallbackのtypeを知る必要があるから。  
相互依存出来るの書きやすくていいね。
