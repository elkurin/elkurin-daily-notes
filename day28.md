# Sequence Checker in Chromium

TaskをSequentialに走らせるのはとてもむずかしい。  
普通にスレッドとか意識できないしぶっ壊してしまう。  
そんなエンジニアのお供に[SequenceChecker](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h)というお助けツールがある。

## SequenceCheckerとは
[SequenceChecker](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h)は同じクラスのメソッドがsequentialに走っているか確認してくれるhelper classで以下のように使う。

```cpp=
class StatefulLacrosLoader {
  ...
 private:
  SEQUENCE_CHECKER(sequence_checker_);
};

void StatefulLacrosLoader::OnLoad(...) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  ...
}
```

Class内で`sequence_checker_`をマクロを使って宣言し、実際に確認したい関数のなかでまたマクロを使ってDCHECKする。  

他にも以下のようなマクロがある。
```cpp=
class SequenceAffine {
 public:
  void Increment() VALID_CONTEXT_REQUIRED(sequence_checker_) {
    ++counter_;
  }

 private:
  int counter_ GUARDED_BY_CONTEXT(sequence_checker_);

  SEQUENCE_CHECKER(sequence_checker_);
};
```
[`GUARDED_BY_CONTEXT`](https://source.chromium.org/chromium/chromium/src/+/main:base/thread_annotations.h;l=251;drc=10d865767e72f494da1e4e868eb6ae9befe87422)は thread_annotations.h の中で定義されているマクロで、~~これがついている変数は特定のmutexからしかアクセスされないことを保証してくれる。  
[`VALID_CONTEXT_REQUIRED`](https://source.chromium.org/chromium/chromium/src/+/main:base/thread_annotations.h;l=254;drc=10d865767e72f494da1e4e868eb6ae9befe87422)がついている関数はmutexを確認してくれるが、多分中で `DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_)` を呼ぶのと同じ？~~  
訂正(2023/07/07, thanks to dancerj@): DCHECK_CALLED_ON_VALID_SEQUENCEをつけているかをコンパイル時に保証し、その後はDCHECK_CALLED_ON_VALID_SEQUENCEのメカニズムで正しいシーケンスで呼ばれているかを実行時に確認している。

## 中身
SequenceCheckerはDCHECKなので DCHECK_IS_ON() のときしかコンパイルされない。

[`SEQUENCE_CHECKER`](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h;l=78;drc=e4622aaeccea84652488d1822c28c78b7115684f)は[base::SequenceCheckerImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker_impl.h;l=29;drc=03c6f32a08c8f176fefc2198846d89cb0f4dab6f)のインスタンスを作る。  

[`DCHECK_CALLED_ON_VALID_SEQUENCE`](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h;l=79;drc=e4622aaeccea84652488d1822c28c78b7115684f)はScopedValidateSequenceCheckerを作る。  
ctorでは`__attribute__(exclusive_lock_function())`attributeがついており、
dtorでは`__attribute__(unlock_function())`attributeがついている。  
あんまり読めないが[Clang document](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html)によると`EXCLUSIVE_LOCK_FUNCTION`と`UNLOCK_FUNCTION`はそれぞれacquireのみ、releaseのみっできるよう縛るらしい。  
clang以外ではno-op

ctorの中では[CalledOnvalidSequence](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_checker_impl.cc;l=32;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)を呼び実際のCHECKはすべてこの中で行われている。  
Sequenceの一致は[SequenceToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_token.h;l=15;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)の比較で確認する。  
[SequenceToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_token.h;l=15;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)はsequenced taskに対応するように作られ、実体はint型のid[`token_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_token.h;l=49;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)。Tokenは[AtomicSequenceNumber](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_token.cc;l=14;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)というgeneratorによって作られているので、名前からしてthread-safe。[GetNext](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/atomic_sequence_num.h;l=23;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)は値を1ずつincrementしており、すべてのthread上でsequentialになる値を渡している。  

`thread_checker_.sequence_token_`が`SequenceToken::GetForCurrentThread()`と一致しているかで確認。  
この`thread_checker_`は[base::SequenceCheckerImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker_impl.h;l=29;drc=03c6f32a08c8f176fefc2198846d89cb0f4dab6f)のインスタンス生成時に一緒に作られている。  
[ThreadCheckerImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.h;l=31;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)で実装されており、こちらもctorの中で[EnsureAssigned](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.cc;l=131;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)を呼び、[SequenceToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.h;l=103;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)に`SequenceToken::GetForCurrentThread()`をセットしている。  
`mutable SequenceToken sequence_token_ GUARDED_BY(lock_);` ← GUARDED_BYマクロを発見した。  
こうして各Sequenceごとにtokenがアサインされている。

逆に[DetachFromThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.cc;l=117;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)を呼ぶとResetされる。これは[`DETACH_FROM_SEQUENCE`](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h;l=82;drc=e4622aaeccea84652488d1822c28c78b7115684f)マクロからも呼べる。

## [`DETACH_FROM_SEQUENCE`](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h;l=82;drc=e4622aaeccea84652488d1822c28c78b7115684f) はいつ使う？

ところでexample codeにはこんな感じで[`DETACH_FROM_SEQUENCE`](https://source.chromium.org/chromium/chromium/src/+/main:base/sequence_checker.h;l=82;drc=e4622aaeccea84652488d1822c28c78b7115684f)が呼ばれている。
```cpp=
class MyClass {
  public:
  MyClass() {
    DETACH_FROM_SEQUENCE(sequence_checker_);
  }
  
  ~MyClass() {
    DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  }
 private:
  SEQUENCE_CHECKER(my_sequence_checker_);
};
```
上記のコード上だとSequenceCheckerインスタンスが作られたときに[ThreadCheckerImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.h;l=31;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)のctorも呼ばれて[SequenceToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.h;l=103;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)に`SequenceToken::GetForCurrentThread()`がセットされていた。  
メンバインスタンスが先に作られてからctorが走るので`sequence_token_`を[kInvalidSequenceToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_token.h;l=48;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)に上書きしている。  
これは初回の`DCHECK_CALLED_ON_VALID_SEQUENCE(name, ...)`のコールで[ここ](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/sequence_checker_impl.cc;l=71;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)にたどり着くルートで、[CalledOnValidThreadInternal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.cc;l=77;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)を呼び、[EnsureAssigned](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/thread_checker_impl.cc;l=131;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)している。その後`task_token_ == TaskToken::GetForCurrentThread()`のチェックが入る。

ここまでの流れを見ると、ctorと実際に呼ぶ際のsequenceが異なる場合に必要なやつっぽい。


## おまけ：PostTaskAndReply
[以前のノート](https://hackmd.io/@elkurin/SJrQhI7L3)で Task Runner について書いた。そこでSequenceTaskRunnerというものを紹介したが、これを直接呼ばずにTaskをSequentialに走らせることもできる。  
[PostTaskAndReply](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool.h;l=135;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)や[PostTaskAndReplyWithResult](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/task/thread_pool.h;l=155;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)のように、1つ目のTaskのあとに続けて次に呼ぶコールバックをRunできるような関数がそれにあたる。

[PostTaskAndReply](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/threading/post_task_and_reply_impl.cc;l=138;drc=22201befd595c4413fdfa85df57cb5e839a42ac3)の中身を見てみると

```cpp=
const bool has_sequenced_context = SequencedTaskRunner::HasCurrentDefault();

const bool post_task_success = PostTask(
    from_here, BindOnce(&PostTaskAndReplyRelay::RunTaskAndPostReply,
                        PostTaskAndReplyRelay(
                            from_here, std::move(task), std::move(reply),
                            has_sequenced_context
                                ? SequencedTaskRunner::GetCurrentDefault()
                                : nullptr)));
```

PostTaskAndReplyの中身は実質SequencedTaskRunnerだった。  
> PostTaskAndReply() requires a SequencedTaskRunner::CurrentDefaultHandle

という記述もある。

## Note
ところで[stateful_lacros_loader.cc](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/stateful_lacros_loader.cc;drc=aaf1daff68251fd30fd6ca57cff1fe3f4a5ab54a)の中の`DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_)`を書く場所足りてなくないですか？  
誰ですかこんなコード書いたの
