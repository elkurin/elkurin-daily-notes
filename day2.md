# Chromium の CHECK macro
休日に英語は書けないので日本語に…  

先日Chromium Code base/* Readingでテックリードパイセンが出してくださったネタ。雰囲気しか理解できなかったので再度チャレンジ。

### 本題
このコードを理解する。
```
#define EAT_CHECK_STREAM_PARAMS(expr) \
  true ? (void)0                      \
       : ::logging::VoidifyStream(expr) & (*::logging::g_swallow_stream)
```       

## CHECKとは
CHECKはコードの挙動を保証するために使う条件確認用のマクロ。実際のプロダクトコードにも入っている。例えば[こんなん](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/rootfs_lacros_loader.cc;l=106-108;drc=8f994f189193084b98766fb0f5969de4d028fb78)。仕事でめっちゃ頻繁に使う。  

`CHECK(object)`とか`CHECK(!list.empty())`のように確認したい条件式をいれて使う。
派生でCHECK_EQ, CHECK_NEやDCHECK, DCHECK_CALLED_ON_VALID_SEQUENCEなどいろいろある。  

もしconditionが想定どおりでないと即crash、その際んのコンデションがダメだったかとメッセージとstack traceを吐いて終了しがち（フラグ次第）。メッセージはこんな感じで決めれる。
```cpp
CHECK(object) << "object must be set";
```
[例](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/wayland_positioner.cc;l=329-331;drc=cee6ba358ac097a8361e7840443f61ce64eb54b3)



## CHECKの挙動の違い
DCHECKはDebugビルドでしか走らないCHECK。  
実はCHECKもOfficialビルドかどうかで挙動が変わる。  
`#if defined(OFFICIAL_BUILD) && !DCHECK_IS_ON()`のときはただCrashするだけでメッセージを出さない。

## CHECKの実装
### OFFICIAL_BUILD && !DCHECK_IS_ON のとき
```cpp
#define CHECK(condition) \
  UNLIKELY(!(condition)) ? logging::CheckFailure() : EAT_CHECK_STREAM_PARAMS()
```
~~`UNLIKELY`は名前の通り、引数がfalseなときにtrueを返すやつ~~`UNLIKELY`は中身の値をそのまま返すだけで結果は変わらないコンパイル用のアノテーションで[実装](https://source.chromium.org/chromium/chromium/src/+/main:base/compiler_specific.h;l=209;drc=f541f7780152ad2d639cbd8ed3f4a3afcd054276)
```
#define UNLIKELY(x) __builtin_expect(!!(x), 0)
```
`__builtin_expect`はexpect通りの値だとめっちゃ早くなるように最適化してくれる組み込み関数らしい。


`condition`がfalseのときは`logging:CheckFailure()`に入ってその中身の実装は以下。
```cpp
[[noreturn]] IMMEDIATE_CRASH_ALWAYS_INLINE void CheckFailure() {
  base::ImmediateCrash();
}
```
ImmediateCrashの中身はまた今度として要はその場でCrashするだけ。  

`condition`がtrueのときは[`EAT_CHECK_STREAM_PARAMS()`](https://source.chromium.org/chromium/chromium/src/+/main:base/check.h;l=55-59;drc=bf47582a0ac2724d1757c99aa13fbe9a44521427)で実装は以下。  
```cpp
// Macro which uses but does not evaluate expr and any stream parameters.
#define EAT_CHECK_STREAM_PARAMS(expr) \
  true ? (void)0                      \
       : ::logging::VoidifyStream(expr) & (*::logging::g_swallow_stream)
BASE_EXPORT extern std::ostream* g_swallow_stream;
```
一見なにこれ？って感じ。読めない。ただの(void)0やん。いらんコード書かないで。**ここが本日の本題**。  
ただの(void)0でも普通のケースは動くが、`CHECK() << "hoge"`のようにメッセージを追加しようとするとこうでないといけない。OfficialビルドでのCHECKはメッセージを出力しないのでなんとか捨てたいが(void)0のままだと壊れる。  

C++のマクロはコードを直接defineされたものに上書きするやつなので、もしCHECK()の中身が`UNLIKELY(!(condition)) ? logging::CheckFailure() : (void)0`だと`CHECK(object) << "object must be set"`のように<<が入ると
```cpp
UNLIKELY(!(condition)) ? logging::CheckFailure() : (void)0 << "hoge"
```
となり壊れる。  
なぜなら a?b:c の優先度より << の優先度のほうが高く([参考](https://en.cppreference.com/w/cpp/language/operator_precedence))、`(void)0 << "hoge"` のシフト演算をしようとしてこんなエラーになる
```
error: invalid operands to binary expression ('void' and 'const char[106]')
      CHECK(ignore_extra_runs_) << "Both OnceCallbacks returned by "
      ~~~~~~~~~~~~~~~~~~~~~~~~~ ^  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
1 error generated.
```
void型をconst char*型でシフトはできません。  

ということで `(*::logging::g_swallow_stream) << "hoge"` でメッセージを吸わせている。このstd::ostreamの出力はそのまま捨てる。  
あとは`(void)0`と右項の型合わせ。
~~ここで注意しないといけないのは右項は`(*::logging::g_swallow_stream) << "hoge"`のケースと`(*::logging::g_swallow_stream)`のケースとどちらも対応しないといけないこと。前者の型は`basic_ostream<char, char_traits<char>>`で後者の型は`basic_ostream<char>`。~~ と思ったが別に一緒だった。  
演算子の優先順位は 左シフト(<<) > ビットアンド(&) > 条件演算子(?) なので、`true ? (void)0 : a & *g_swallow_stream << "hoge"`とすれば&が後で計算されてくれるが？よりは先に処理されてちょうどよい。  
ここの`a`には`::logging::VoidifyStream(expr)`という人がいる。名前から察するとvoidにしようとしているっぽい。[実装](https://source.chromium.org/chromium/chromium/src/+/main:base/check.h;l=46-53;drc=bf47582a0ac2724d1757c99aa13fbe9a44521427)は
```cpp
class VoidifyStream {
 public:
  VoidifyStream() = default;
  explicit VoidifyStream(bool) {}

  // This operator has lower precedence than << but higher than ?:
  void operator&(std::ostream&) {}
};
```
operator& を void を返す空関数にoverrideしている。  
型があった。めでたし。

### !OFFICIAL_BUILD || DCHECK_IS_ON のとき
```cpp
#define CHECK(condition) \
  CHECK_FUNCTION_IMPL(::logging::CheckError::Check(#condition), condition)
```
`logging::CheckErrorCheck`の[コード](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/check.cc;l=156-162;drc=4baaf6a3dc1c91f711a17d8fb789ad0e71ab2a17)を見ると、"Check failed: {condition}."みたいなログメッセージをストリームに流している。  
loggingの中身はまた今度。  