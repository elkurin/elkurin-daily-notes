# TaskEnvironment in Chromium

以前 [Task Runner in Chromium](/V20PsjhqT7a7AJ1_zE5kMg) のノートでTaskをPostする方法について言及した。  
これをテストでも使用したい場合もある。  
例えばテスト対象が非同期な操作を行っており、その結果をチェックしたいときなど、[RunLoop](/3nWtoOH2QW28njOfcbvSyw) を使うことがよくある。  
この時、何も設定しないでunit testを走らせるとThreadPoolやTaskRunnerがなくてRunLoopなどは走らない。  
それを簡単にセットしてくれるのが[TaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h)。

## Threadについて
その前にChromeが持つスレッドについてまとめる。  
大きく分けてメインスレッド、I/Oスレッド、ThreadPoolがある。

Browserプロセス上だとメインスレッドはUIスレッドで、RendererプロセスだとBlinkを走らせる。

[BrowserThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/browser/browser_thread.h;l=53;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)には BrowserThread:UIとBrowserThread::IOがある。  
UIスレッドはbrowserにおけるメインスレッド。  
IOスレッドはIPCやnetworkなどnon-blockingなTaskを行うスレッド。  
その他のスレッドは[base::ThreadPool](https://source.chromium.org/chromium/chromium/src/+/main:base/task/thread_pool.h;l=68;drc=63e1f9974bc57b0ca12d790b2a73e5ba7f5cec6e)にあるプールされたスレッドを持ってきて好きにTaskを投げる。IO系のイベントでもblockingなものはThreadPoolに投げる。  


## TaskEnvironment の用法
[TaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h)はmain threadから呼ばれないといけない。  
以下のように作る。  

```cpp=
// クラス内で作る
class AuraTestBase {
 public:
  AuraTestBase() : task_environment_(
    base::test::TaskEnvironment::MainThreadType::UI) {}
 private:
  base::test::TaskEnvironment task_environment_;
};

// 各テストごとに作ることもできる。
TEST(InvitationCppTest_...) {
  base::test::TaskEnvironment task_environment;
  ... // Do something such as RunLoop.
}
```
[TaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h)は各テストに付き1つまで。  
Instanceを作るだけでOK。

TaskEnvironmentの[ctor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=204;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)には他にもいくつかのTraitsを与えることができる。  
- [TimeSource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=99;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758): `MOCK_TIME`に設定するとDelayed tasksがmock clockを使うようになり、もっとも早いdelayまで時間をジャンプするらしい。[FastForwardBy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=794;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を見ると書いてそう。
- [ThreadPoolExecutionMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=141;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758): `QUEUE`に設定すると[RunUntilIde()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=253;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)が呼ばれるまでただTaskをqueueして進まないようにする。デフォルトである`ASYNC`モードでも[RunUntilIde()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=253;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)でブロックすることは可能。
- [ThreadingMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=155;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758): `MAIN_THREAD_ONLY`を指定するとThreadPoolは初期化されずsingle threadで走るようになる。

また簡易版として[SingleThreadTaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=531;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)もある。  
これも[TaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h)とほぼ同じで、違いは[ThreadingMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=155;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)が[`MAIN_THREAD_ONLY`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h;l=535;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)にしばれているところ。  
これがあればSingleThreadTaskRunnerやSequencedTaskRunnerは使用できるし、RunLoopもいけるので推奨されている。  
[TaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.h)を用意しないといけないのはbase::ThreadPoolを使用するときのみで、そういうケースで使うのは全然OK。

## TaskEnvironmentの中身
TaskEnvironmentのコンストラクタで`task_queue_`, `task_runner_`を生成し、[InitializeThreadPool](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=512;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)でThreadPoolを初期化する。  
ここで、もしsubclassがすでにTaskRunnerを作っていた場合`subclass_creates_default_taskrunner`をenableしてTaskQueue, TaskRunnerの生成はスキップする。  
またThreadModeが`ThreadingMode::MAIN_THREAD_ONLY`のときはThreadPoolが必要ない（メインスレッドしか必要ない）のでInitializeThreadPoolはスキップ。  
[TaskQueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/task_queue.cc;l=45;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)はSingleThreadTaskRunnerへのscoped_refptrを保有しており、[CreateTaskRunner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/task_queue_impl.cc;l=269;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)からMakeRefCountedしてくれる。  
ここまででTaskRunnerは用意できたので、その後[InitializeThreadPool](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=512;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)が呼ばれThreadPoolを作る処理が始まる。  
[CreateThreadPool](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=492;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)の中を見ると [Background threads はdisable](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=502-505;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)されているようだ。ここで作ったThreadPoolはプロダクションと同様に[ThreadPoolInstanceにSet](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/thread_pool_instance.cc;l=110;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)されて使えるようになる。プロダクションの例を見ると例えば[CreateChildThreadPool](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/app/content_main_runner_impl.cc;l=521;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)のように明示的にCreateを呼んでThreadPoolを作っており、パスはテストと同じ。

### RunUntilIdleの仕様
TaskEnvironmentのインスタンスは[RunUntilIdle](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=692;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)や[RunUntilQuit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=671;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を自前で持っており、TaskEnvironment内のThreadがIdleになるまでトラックするという書き方がよくされる。中身の実装はRunLoop。  
ここで、もし`ThreadingMode::MAIN_THREAD_ONLY`が指定されていた場合は今いるスレッドのみを監視するだけで全タスクをトラックできるのでOK。ただRunLoopを読んでいるだけの実装。  
`MULTI_THREAD`だった場合少し複雑。[AllowRunTasks](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=965;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)でTaskを許可する。その後一回[RunLoop::RunUntilIdle](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=739;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を予備、事実上のFlushをする。どうやらThreadPoolInstance::FlushForTesting()は使えないらしい。その後[DisallowRunTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/task_environment.cc;l=978;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を読んでThreadPoolを止める。止まったらもう一度RunUntilIdleを走らせ残っているメインスレッド上のタスクの終了まで待つ。ここでメインスレッド上のタスクはすべて完了したことが保証される。[HasIncompleteTaskSourcesForTesting](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/task_tracker.cc;l=497;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)で他にタスクの残っているTaskSourceが無いことを確認できたら無限ループから抜けて終了。この中で確認する[`num_incomplete_task_sources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool/task_tracker.h;l=243;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)はatomic変数で各TaskSourceの総数をカウントしているので、稼働中の他スレッドを確認することができるということ。  
つまりTaskEnvironmentのRunUntilIdleは全スレッドのタスクが終了してくれるまで待ってくれるいい子。

## BrowserTaskEnvironment
上記のTaskEnvironmentはunit test用。  
browser testでは以下の[BrowserTaskEnvironment](https://source.chromium.org/chromium/chromium/src/+/main:content/public/test/browser_task_environment.cc;drc=7f5dfb90a2d1b68efbfcde627b3a0da99079f200)を使う。  
こちらでは[BrowserThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/browser/browser_thread.h;l=53;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を使用することができる。  
デフォルトではUIスレッドとIOスレッドがどちらもメインスレッドのshared message loopで走るようになっていることに注意。もしリアルなIOスレッドを用意したければ[`real_io_thread_`](https://source.chromium.org/chromium/chromium/src/+/main:content/public/test/browser_task_environment.h;l=199;drc=72d89b0bb832f2a7539a2469c592cdf589db8466)をenableしてあげないといけない。  
real_io_threadを使用する際は[RunIOThreadUntilIdle](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_task_environment.cc;l=226;drc=afec9eaf1d11cc77e8e06f06cb026fadf0dbf758)を使おう。
