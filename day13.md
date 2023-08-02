# RunLoop

[base::RunLoop](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.h)は非同期の操作が終了するまで待つためのお便利クラス。  
RunLoopをRunするとQuitが呼ばれるまでブロックするので、QuitClosureを非同期メソッドのコールバックに渡すなどするとawait的な動きをしてくれる。

## 使い方
RunLoopインスタンスを作り非同期関数を呼んでからRunLoopをRun。
```cpp=
base::RunLoop run_loop;
SomethingAsyncMethod(/*callback=*/run_loop.QuitClosure());
run_loop.Run();
```
Run()が呼ばれる前にQuit()してしまっても問題なく動く。

[RunLoop::RunUntilIdle()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=139;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)を呼ぶこともできる。
```cpp=
SomethingAsyncMethod();
RunLoop().RunUntilIdle();
```
この場合QuitClosureを明示的に渡さなくても勝手に終了してくれる。

## RunLoop::Run/Quitの実装 (not-nested)
RunLoopのインスタンス作成時にSingleThreadTaskRunnerを取ってくる。  
RunLoopは単一スレッド内で完結しているっぽい。

[RunLoop::Run()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=106;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)が呼ばれるとまず[BeforeRun](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=316;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)を呼んですでにQuitしていれば[`quit_called_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=330;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)がフリップされているので即終了してくれる。  
呼ばれていない場合はSingleThreadTaskRunnerである`origing_task_runner_`にTimeout用のコールバックを投げて[RunLoop::Delegate::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=204;drc=d80a2f905f8a88e023af6dc3908a27c67de0427a)を呼ぶ。  
RunLoop::Delegateの実装は[ThreadControllerWithMessagePumpImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=621;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)にある。[MessagePumpDefault::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/message_loop/message_pump_default.cc;l=32;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)は[`keep_running_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/message_loop/message_pump_default.h;l=34;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)がfalseになるまでずっとforループを回すブロッキング関数なので、ここで止まる。

[RunLoop::Quit()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=153;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)が呼ばれると[RunLoop::Delegate::Quit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=680;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)を呼び、[MessagePumpDefault::Quit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/message_loop/message_pump_default.cc;l=65;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)が`keep_running_`をfalseにすることで[MessagePumpDefault::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/message_loop/message_pump_default.cc;l=32;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)のforループが終了しブロッキングタスクが終わり、RunLoop::Run()のあとの処理へ続く。

Quit()は[QuitClosure()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=199;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)でコールバックとして渡すことができる。この場合別スレッドから呼ばれる可能性もあるが、QuitClosureは[ProxyToTaskRunner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=28;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)を通って元スレッドのTaskRunnerに投げられるので大丈夫。

## Nested RunLoop
RunLoopの中で違うRunLoopが作られるときもあり、これは`type_`に[kNestableTasksAllowed](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=77;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)が指定されているときのみ可能。デフォルトの[コンストラクタ](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=80;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)ではNestedは使えないので、使いたい場合はNest内で走るRunLoopを`RunLoop(kNestableTasksAllowed)`で作る。  
NestedのRunLoopは内側から順に終了するようにガードされている。

NestedのときRunLoopは[`active_run_loops_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=234;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)で管理される。  
[RunLoop::Delegate](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.cc;l=22;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)はthread内で同じものが使用されるのでRunLoop同士で[`active_run_loops_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=234;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)も共有されており、新しいRunLoopが作られると[BeforeRun](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=316;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)の中で追加される。  
[AfterRun](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.cc;l=352;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)が呼ばれたときに[`active_run_loops_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.h;l=234;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)から[pop](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.cc;l=362;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)されるが、この際直近にRunされたRunLoopから順にQuitが呼ばれないといけない。Quit()の中で[if分岐](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.cc;l=172;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)されており、最も内側のRunLoopでなければ[RunLoop::Delegate::Quit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=680;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)は呼ばれない。


## RunUntilIdle / QuitWhenIdle
あるいは[RunLoop::RunUntilIdle()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=139;drc=5992439d25f71ce29efa8db1c699b99e8773d41f)を呼ぶこともできる。  これはRunLoop::DelegateがTaskをすべて終えるまで待つ関数。  
[RunLoop::QuitWhenIdle()](https://source.chromium.org/chromium/chromium/src/+/main:base/run_loop.cc;l=178;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)という関数もあり、これはIdleになったらQuitする。ほぼ一緒。

これらは同様に[RunLoop::Run()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=106;drc=960e2d3bc64d51c96c4fcea5bb1430ddab589ba4)を呼び[MessagePumpDefault::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/message_loop/message_pump_default.cc;l=32;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)が走るが、ところでこの関数では[DoIdleWork](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=544;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)というステップがありこれは`has_more_immediate_work`がfalseのときに呼ばれる。つまりもうTaskがないとき。  
上述のRunの際はこれが呼ばれてもそのまま次のメッセージをまつがRunUntilIdleやQuitWhenIdleのときはここで抜ける。  
`quit_when_idle_`をtrueにセットしているので、[ShouldQuitWhenIdle](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/run_loop.cc;l=66;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)がtrueを返し、DoIdleWorkが[Quit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=611;drc=25f2ea1a864270fef1c96c014f552f1459280ac1)を呼ぶ。

ここでわかるとおり、Idleになるとすぐ止まってしまうので、もしかしたらまだTaskが投げられていないためIdleになり止まることもある。またすべてのIdleタスクをまつのでめっちゃまつこともある（アニメーションとかが使われているときは要注意）  
可能であればRun-Quitを使うと良さそう。
