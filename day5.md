# Task Runner in Chromium

非同期な処理を行いたい場合、例えば以下のようにスレッドに投げて走らせることができる。
```cpp=
base::ThreadPool::PostTaskAndReplyWithResult(
    FROM_HERE, {base::MayBlock()},
    base::BindOnce(&CheckForComponentToPreloadMayBlock),
    base::BindOnce(&PreloadComponent, component_manager_));
```
そのChromiumでの内部実装を確認する。

## Postの仕方
最もシンプルなタスクの投げ方は
```cpp=
bool PostTask(const Location& from_here, OnceClosure task);
```
Location型は関数名・ファイル名・行数を持つもので、ここでは関数がどこから来たかの情報を保存している。多くの場合[`FROM_HERE`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/location.h;l=104;drc=02199fc586b3af08fee6a7b188d731f381cd814c)というマクロを使ってbuiltin関数を呼び出す。  
OnceClosure型はChromiumで使われる関数ポインタの一種で返り値がないやつ。[Callback](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback.h;l=73;drc=02199fc586b3af08fee6a7b188d731f381cd814c)についてはまた今度。

他には
```cpp=
// `delay` にタスクを入らせるまでの遅延時間を指定できる
bool PostDelayedTask(const Location& from_here,
                     OnceClosure task,
                     TimeDelta delay);

// `task` のあとに続けて `reply` を走らせる
bool PostTaskAndReply(const Location& from_here,
                      OnceClosure task,
                      OnceClosure reply);

// `task` の結果を `reply` にバインドして走らせる
bool PostTaskAndReplyWithResult(const Location& from_here,
                                CallbackType<TaskReturnType()> task,
                                CallbackType<void(ReplyArgType)> reply);
```
とかがある。[TaskTraits](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/task_traits.h)というタスクのtraitsを指定することもできる。  

このPostTask関数を呼ぶと非同期処理をしてくれる。  

上記のようないろんなPostTask関数を呼ぶと[ThreadPoolImpl::PostTaskWithSequenceNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_pool_impl.cc;l=414;drc=02199fc586b3af08fee6a7b188d731f381cd814c)などにたどり着く。この中で[SequenceのqueueにTaskを放り投げる](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.cc;l=93;drc=02199fc586b3af08fee6a7b188d731f381cd814c0)。同期的な処理はここまで。  

queueに投げ入れられたTaskはフリーになった[WorkerThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/worker_thread.h;l=254;drc=f541f7780152ad2d639cbd8ed3f4a3afcd054276)が順々にTaskを取っていってRun。コードは[このあたり](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/worker_thread.cc;l=440-499;drc=02199fc586b3af08fee6a7b188d731f381cd814c)。  
このときTaskは今queueの中にある最も古いやつ(delayed taskは実行予定時間、普通のtaskはpostされた時間でsort)を[TakeEarliestTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.cc;l=173;drc=02199fc586b3af08fee6a7b188d731f381cd814c)から取得しているので、だいたい古い順に走ることは約束されているが、この状態だと複数のThreadが好きにTaskを取っていってRunするのでsequencialに走るわけではない。

### ところで
[WorkerThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/worker_thread.h;l=48-56;drc=02199fc586b3af08fee6a7b188d731f381cd814c)にはPOOLED、SHARED、DEDICATEDと3種類ある。  
これSharedWorkerとかいうインターンのときに実装していたやつじゃん！  
[HTML Worker Spec](https://html.spec.whatwg.org/multipage/workers.html)の[Shared workers and the SharedWorker interface](https://html.spec.whatwg.org/multipage/workers.html#shared-workers-and-the-sharedworker-interface)のどこか(error handlingらへん?)は僕が書きました。


## ThreadPoolとTaskRunner
Chromiumではタスクをスレッドに投げるのには大きく2つの方法がある。  
1. [base::ThreadPool](https://source.chromium.org/chromium/chromium/src/+/main:base/task/thread_pool.h;drc=63e1f9974bc57b0ca12d790b2a73e5ba7f5cec6e) に直接 [PostTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool.cc;l=48;drc=50061fdd60b62a7c022a12a04e14d6e10b81bf19) する
2. [base::TaskRunner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/task_runner.h) を作ってそこに [PostTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/task_runner.cc;l=46;drc=02199fc586b3af08fee6a7b188d731f381cd814c) する

base::ThreadPoolを使う場合は以下のように呼ぶ。
```cpp=
base::ThreadPool::PostTask(FROM_HERE, base::BindOnce(&Task));
```

一方TaskRunnerを使用する際は以下のようにTaskRunnerを作ってからそこにPostする
```cpp=
scoped_refptr<SequencedTaskRunner> sequenced_task_runner =
    base::ThreadPool::CreateSequencedTaskRunner(...);

// TaskB runs after TaskA completes.
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskA));
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskB));
```


どちらの方法でもだいたい[ThreadPoolImpl::PostTaskWithSequenceNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_pool_impl.cc;l=414;drc=02199fc586b3af08fee6a7b188d731f381cd814c)付近にたどり着くので大まかなPostのflowは変わらない。  
ただしThreadPoolを使用する際はPostTaskWithSequenceNowの引数 `scoped_refptr<Sequence> sequence`が常に [TaskSourceExecutionMode::kParallel](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_pool_impl.cc;l=251-255;drc=02199fc586b3af08fee6a7b188d731f381cd814c) に指定されており、[Sequence](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.h)は各タスクごとに作られている。  

一方TaskRunnerの一つである[SequencedTaskRunner](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequenced_task_runner.h;drc=86c1a4586c013fde3cf0c61c0459f4f3a597ae5c)では[TaskSourceExecutionMode::kSequenced](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/pooled_sequenced_task_runner.cc;l=18-20;drc=50061fdd60b62a7c022a12a04e14d6e10b81bf19)に指定されているし、SingleThreadTaskRunnerでは[kSingleThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/pooled_single_thread_task_runner_manager.cc;l=423-426;drc=02199fc586b3af08fee6a7b188d731f381cd814c)に指定されている。そして[Sequence](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.h)のインスタンスは各TaskRunnerの中で一貫している。

### 並列実行と逐次実行
TaskSourceExecutionModeはなにか？
- Paralell: タスクが並列に走る
- Sequenced: タスクは順に走るが同じスレッドとは限らない
- SingleThread: タスクは一つのスレッド上で順に走る

ところで上のセクションで述べたqueueは[Sequence](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.h)の中に保存されいるので、各SequenceごとにTask queueが存在するということ。  

各WorkerThreadは次に走らせるべきTaskSourceを[`delegate_->GetWork(this)`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/worker_thread.cc;l=451;drc=02199fc586b3af08fee6a7b188d731f381cd814c)で探している。  
GetWorkの実装はSingleThreadとそれ以外で大きく違う。  
SingleThreadでは[pooled_single_thread_task_runner_manager.cc](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/pooled_single_thread_task_runner_manager.cc;l=128;drc=02199fc586b3af08fee6a7b188d731f381cd814c)の中にある。シンプルに`priority_queue_`の順に取ってきているだけ。これはSingleThreadモードだとひとつのスレッドに対しひとつのsequenceがアサインされているため、ひとつのTaskが終わったら次とすればいいから。ひとつのWorkerThread[`worker_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/pooled_single_thread_task_runner_manager.cc;l=257;drc=02199fc586b3af08fee6a7b188d731f381cd814c)のみを所有。  

その他では[ThreadGroupImpl::WorkerThreadDelegateImpl::GetWork](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_group_impl.cc;l=564;drc=02199fc586b3af08fee6a7b188d731f381cd814c)でなにかしている。ThreadGroupは複数のWorkerThread[`workers_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_group_impl.h;l=283;drc=02199fc586b3af08fee6a7b188d731f381cd814c)を所有しておりThreadGroupが管理してタスクを投げる。SignleThreadと違い、いろんなSequenceのものから選んでいくが、Sequencedのタスクはどこかのスレッドで走っている直前のタスクが終わるまで次のものを走らせることができないので注意。  
RunWorkerの1ループの中で[TaskTracker::RunAndPopNextTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/task_tracker.cc;l=378;drc=02199fc586b3af08fee6a7b188d731f381cd814c)が一度呼ばれTaskをPopしてRunしている。WorkerThreadの中ではBlockingでTaskを走らせ、その間はThreadGroupからGetWorkで送られてきたTaskSourceは保持し続けている。そして[RanTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/task_tracker.cc;l=400;drc=02199fc586b3af08fee6a7b188d731f381cd814c)終了後[DidProcessTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/sequence.cc;l=227;drc=02199fc586b3af08fee6a7b188d731f381cd814c)を呼びSequenceのqueueの中にまだTaskがあればもう一度ThreadGroupに[返還](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/worker_thread.cc;l=486;drc=02199fc586b3af08fee6a7b188d731f381cd814c)し、もう一度Sequenceが[ReEnqueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_group_impl.cc;l=661-663;drc=02199fc586b3af08fee6a7b188d731f381cd814c)される。こうすることで直前のタスクが終わるまで次のタスクに行かないという挙動を担保している。

#### TL;DR; 使い方
並列実行で良い場合、単発のTaskを投げる場合は
```cpp=
base::ThreadPool::PostTask(FROM_HERE, base::BindOnce(&Task));
```
がベスト。

順に走ってほしい場合はSequencedTaskRunnerを作ってPostするのがベスト。
```cpp=
scoped_refptr<SequencedTaskRunner> sequenced_task_runner =
    base::ThreadPool::CreateSequencedTaskRunner(...);

// TaskB runs after TaskA completes.
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskA));
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskB));
```

SingleThreadTaskRunnerよりSequencedTaskRunnerを優先すべき。  
SequencedTaskRunnerはThreadGroupの中からフリーなものを使ってくれるので効率が良くSingleThreadと違って無駄に一つのスレッドを専有しないので。  

それでもSingleThreadを使いたいケースは存在する。  
例えばパフォーマンスシビアなCompositorはSingleThreadTaskRunnerを使用している（違う理由があるかも）
[ui/compositor/compositor.h](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/compositor.h;l=557;drc=aeccd60bbfdea43e400fa24fbb2080e9dc4e6a6b)

## Additional Note
### queue
Task
のqueueの実装にはThreadPool専用の[PriorityQueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/priority_queue.h;l=23;drc=02199fc586b3af08fee6a7b188d731f381cd814c)を用いている。

### CurrentDefaultHandle
CurrentDefaultHandleはそのlifetimeの間だけ`task_runner`を現在のスレッドにバインドしている。つまり[以下](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/task_tracker.cc;l=475-476;drc=02199fc586b3af08fee6a7b188d731f381cd814c)のように書くとそのtask sourceのTaskRunnerをバインドしており[GetCurrentDefault()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequenced_task_runner.cc;l=82;drc=02199fc586b3af08fee6a7b188d731f381cd814c)でスレッドを獲得できる。
```cpp=
sequenced_task_runner_current_default_handle.emplace(
    static_cast<SequencedTaskRunner*>(task_source->task_runner()));
```
[base::AutoReset](https://source.chromium.org/chromium/chromium/src/+/main:base/auto_reset.h)というお便利クラスを使っていた。