# CrashKey

以前どこでCrashしているかを検知するためにこんな感じのログを追加した。
```cpp=
if (active_index == TabStripModel::kNoTab) {
  base::debug::DumpWithoutCrashing();
} else {
  base::debug::DumpWithoutCrashing();
}
```
行数が違えばどこでCrashするかわかると思ってのコードだった。  
しかしこのコードはコンパイラ最適化がかかって同じやんって扱われ、スタックトレースの中から消滅してしまったため結局どっちでCrashしたかわからなかった。  
残念。。。

こんなときにCrashKeyというのが役に立つことを発見した。

### ログに出力すればいいやん？
ログにも出力していた([参考CL](https://chromium-review.googlesource.com/c/chromium/src/+/4623973))が、特定のプラットフォーム上ではログを回収することができずStackTraceしかない。  
なので、なんとかしてDumpしたものから判定したい。

## CrashKeyの概要
CrashDumpのログに残るようなkeyを追加してくれる。  
これがあればただCrashしたStackTraceだけでなく、何がなんの値でクラッシュしたかや、同じ場所でもその時の状態や条件を出力してくれるのでデバッグに有用なデータが集められる。  

書き方は以下：
[base/debug/crash_logging.h](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h)をincludeして以下のように書く。
```cpp=
SCOPED_CRASH_KEY_NUMBER("FUNCTION_NAME", "VAR_NAME", value);
base::debug::DumpWithoutCrashing();
```

他にも[`SCOPED_CRASH_KEY_STRING32`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=151;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)や[`SCOPED_CRASH_KEY_BOOL`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=167;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)など型ごとにマクロが用意されている。  

## CrashKeyの中身
[`SCOPED_CRASH_KEY_NUMBER`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=172;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)と[`SCOPED_CRASH_KEY_BOOL`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=167;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)はそれぞれ `data` を `base::NumberToString()` や"true"/"false"で文字列に変換しているだけで中身はみんな [`SCOPED_CRASH_KEY_STRING_INTERNAL`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=139;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)。  
これはそのまま[`SCOPED_CRASH_KEY_STRING_INTERNAL2`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=126;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)にパスされているが[`__COUNTER__`](https://www.ibm.com/docs/ja/xl-c-aix/13.1.0?topic=macros-general)マクロを展開するためである。  

[`SCOPED_CRASH_KEY_STRING_INTERNAL2`](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=126;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)の中では [ScopedCrashKeyString](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=102;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3) を生成している。これはkeyのライフタイムを管理するためのラッパー。  
keyが実際に生成されるのは[AllocateCrashKeyString](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h;l=84;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)で、ぞこから[`g_crash_key_impl`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/debug/crash_logging.cc;l=16;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)の実装を読んでいる。  
なお、[`g_crash_key_impl`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/debug/crash_logging.cc;l=16;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)は[CrashKeyImplementation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/debug/crash_logging.h;l=182;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)というabstract classになっていて、productionでは[CrashKeyBaseSupport](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/crash/core/common/crash_key_base_support.cc;l=113-114;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)が設定されている。  
ここから先は//comonents/の中。

[CrashKeyBaseSupport::Allocate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/crash/core/common/crash_key_base_support.cc;l=59;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)を見てみるとこんなマクロが
```cpp=
SIZE_CLASS_OPERATION(size, return new, (name, size));
```
キモすぎ。  
もっとキモいのがある  
```cpp=
SIZE_CLASS_OPERATION(crash_key->size,
                    　reinterpret_cast<, *>(crash_key)->impl.Set(value));
```
ちなみに[SIZE_CLASS_OPERATION](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/crash/core/common/crash_key_base_support.cc;l=34;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)の実装は以下。  
```cpp=
#define SIZE_CLASS_OPERATION(size_class, operation_prefix, operation_suffix)\
  switch (size_class) {                                                     \
    case base::debug::CrashKeySize::Size32:                                 \
      operation_prefix BaseCrashKeyString<32> operation_suffix;             \
      break;  ...                                                      
```

それはさておき[base::debug::CrashKeyString](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/debug/crash_logging.h;l=198;drc=3b80f0fe294d247d4662b6a4540d0f78f58f0ec3)を引き継ぐ[BaseCrashKeyString](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/crash/core/common/crash_key_base_support.cc;l=26;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)を生成して、scopeの範囲内生き残る。  

ここで保存されているkeyはCrash時に呼ばれる[~LogMessage()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/logging.cc;l=723;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)で呼ばれ、[`stack_trace.OutputToStream`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/logging.cc;l=731;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)でStackTraceを流したあと[`base::debug::OutputCrashKeysToStream`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/debug/crash_logging.cc;l=60;drc=5182321e3c14eace59150d02f6aa78d357d2c83d)　→ [`crash_report::OutputCrashKeysToStream`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/crash/core/common/crash_key_base_support.cc;l=77;drc=5182321e3c14eace59150d02f6aa78d357d2c83d) をパスされていき出力された。  

## ところで
読んでいる途中で気づいたが [crash_logging.h](https://source.chromium.org/chromium/chromium/src/+/main:base/debug/crash_logging.h)のCrashKeyは非推奨だった。  

最初から[crash_reporter::CrashKeyString](https://source.chromium.org/chromium/chromium/src/+/main:components/crash/core/common/crash_key.h)を呼ぶ方法が推奨。  
しかしこれは//components/の中にあって依存できない場所もあるので、その場合//baseの中にあるcrash_logging.hを使用しろとのこと。実際crash_logging.hの中身も事実上 //components/crash/core/common/crash_key_* であり、依存関係を切るためにdelegateのように登録されているだけ。(の割には結構書き方違うね)  

crash_reporterの方はこんな感じで書く
```cpp=
static crash_reporter::CrashKeyString<20> key("active_tab");
key.Set(base::NumberToString(active_index));
base::debug::DumpWithoutCrashing();
```

## Note

DumpWithoutCrashingはCrashせずにシミュレートだけするやつ。  
このときLogMessageってdestructされないはず。  
今後のネタとしておく。
