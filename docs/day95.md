# SequenceTaskRunner の Delete task

[SequenceTaskRunner](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.h)は、タスクを別スレッドで行うためのクラス。

その中で、Delete object用のメソッドが使われているコードを見たので、内容を確認する。

## [DeleteSoon](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.h;l=230;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)

[DeleteSoon](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.h;l=230;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)は渡されたオブジェクトを壊すタスクを投げる。
```cpp=
template <class T>
bool DeleteSoon(const Location& from_here, const T* object) {
  return DeleteOrReleaseSoonInternal(from_here, &DeleteHelper<T>::DoDelete,
                                     object);
}

template <class T>
bool DeleteSoon(const Location& from_here, std::unique_ptr<T> object) {
  return DeleteOrReleaseSoonInternal(
      from_here, &DeleteUniquePtrHelper<T>::DoDelete, object.release());
}
```
オブジェクトの型が生ポインタかunique_ptrかでマッチング。  
[DeleteHelper](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner_helpers.h;l=23;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)か[DeleteUniquePtrHelper](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner_helpers.h;l=33;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)かの差しか無いので中身は基本同じで[DeleteOrReleaseSoonInternal](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.cc;l=107;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)が呼ばれる。

デフォルトの実装だと[SequenceTaskRunner](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.cc;l=107;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)の実装を使用する。  
これはNon Nestable taskとして投げる実装で、Taskはデフォルト値である`CONTINUE_ON_SHUTDOWN`を取り、シャットダウン前に走ることは保証されていない。  
なので、先にShutdownしてしまうコードではオブジェクトはリークする。

実装を変えたい場合はDeleteOrReleaseSoonInternalをOverrideすればよい。  
実際に[BlinkSchedulerSingleThreadTaskRunner](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/scheduler/common/blink_scheduler_single_thread_task_runner.cc;l=107;drc=7e1a8277afbc15fc0871a64b378d7a7956dc9471)では実装を変えている。  

## 使用例
DeleteSoonで自身をdelayして破壊しているコードがあった。
```cpp=
void LoginDisplayHostCommon::ShutdownDisplayHost() {
  ...;
  base::SingleThreadTaskRunner::GetCurrentDefault()->DeleteSoon(FROM_HERE,
                                                                this);
}
```

static関数でもないのに自身を破壊するのはやめたほうがいいと思う。

## Note
特に目新しい発見はなかったがこういう日もある。
