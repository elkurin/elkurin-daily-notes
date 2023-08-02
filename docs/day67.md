# ChromiumでBanされているやつら

[std::optional vs absl::optional](/day65.md)で言及したようにChromiumでは`std::optional`は使用禁止となっている。  
このような例は他にもあり、それがわかりやすくまとまっているのが[PRESUBMIT.py](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py)。

## PRESUBMIT.py とは
TL;DR; Submit前に行うスタイルなどのチェッカー。  

普段Chromiumに変更を入れる時、ローカルで変更を作ってそれを`git cl upload`コマンドでアップロードしている。この際`PRESUBMIT.py`という名前のつくあらゆるファイル(変更したファイルが含まれるディレクトリのみを走査)を実行している。  
このロジックはgit_cl.pyの[`_RunPresubmit`](https://source.chromium.org/chromium/chromium/tools/depot_tools/+/main:git_cl.py;l=1434;drc=deff9a27cc52499610d4e1f2d756f7bf54d40609)にtriggerされる[presubmit_support.py](https://ci.chromium.org/ui/p/chromium/builders/try/chromium_presubmit)に書いているが、その話はまた今度。  

PRESUBMIT.pyの役目に挙動チェックのテストなどは含まれず、表記や使ってはいけないもののチェックなどをする。  

### 具体例
例えば依存してはいけないライブラリがあった時に弾くコードは以下：  
```python=
# //chrome/browser/PRESUBMIT.pyより抜粋
def _CheckUnwantedDependencies(input_api, output_api):
  problems = []
  for f in input_api.AffectedFiles():
    if not f.LocalPath().endswith('DEPS'):
      continue

    for line_num, line in f.ChangedContents():
      if not line.strip().startswith('#'):
        m = re.search(r".*\/blink\/public\/web.*", line)
        if m:
          problems.append(m.group(0))

  if not problems:
    return []
  return [output_api.PresubmitPromptWarning(
      'chrome/browser cannot depend on blink/public/web interfaces. ' +
      'Use blink/public/common instead.',
      items=problems)]
```
変更の合ったファイルが`input_api.AffectedFiles`の中に登録されているのでそれぞれ確認。  

[`DEPS`](https://developer.chrome.com/blog/chromium-chronicle-15/)というのはChromiumの中で依存先を定めるファイルの名前として使われている。例えば[//chrome/browser/DEPS](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/DEPS)の中では`include_rules`として"+apps", "+cc/paint"などのように依存していい先を直接指定したり、あるいは"-content/public/test/test_browser_context.h", "-chrome/browser/ui/views",などのように依存してはいけない場所を覗いたりしている。  
また`specific_include_rules`では以下のようにファイルごとに指定することも出来る。  
```
"exo_parts\.cc": [
  "+ash/shell.h",
],
```
`DEPS`による依存関係チェック自体は別のパスで行われている。[Doc](https://chromium.googlesource.com/chromium/src/+/master/buildtools/checkdeps/README.md)  

ではPRESUBMIT.pyで何をしているかというと。DEPSが変更されていれば確認して、`ChangedContents`の中に`r".*\/blink\/public\/web.*"`という正規表現とマッチする文が追加されていないかを確認している。DEPSに正しくない変更がされないようガードしているようだ。  

## BanRule
今回は特に//chromium/src直下にある[PRESUBMIT.py](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py)を確認する。  
このファイルには[`BanRule`](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=125-138;drc=e97ba778462d69a36c71da18af8d9b08221957bf)というクラスが定義されており、このデータは[CheckNoBannedFunctions::CheckForMatch](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=2326;drc=e97ba778462d69a36c71da18af8d9b08221957bf)で禁止コールがないかの確認をする際に参照される。  

つまりここを見るとスタイル的に禁止されている関数群などを確認することが出来る。

[`_BANNED_JAVA_IMPORTS`](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=141;drc=e97ba778462d69a36c71da18af8d9b08221957bf), [`_BANNED_JAVA_FUNCTIONS`](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=2326;drc=e97ba778462d69a36c71da18af8d9b08221957bf), [`_BANNED_HAVASCRIPT_FUNCTIONS`](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=141;drc=e97ba778462d69a36c71da18af8d9b08221957bf)などといったように、各言語、各特性ごとにBanグループが作られている。  

ここはC++勉強会なので[`_BANNED_CPP_FUNCTIONS`](https://source.chromium.org/chromium/chromium/src/+/main:PRESUBMIT.py;l=476;drc=e97ba778462d69a36c71da18af8d9b08221957bf)を見てみる。  
目についたものをピックアップ

### ラップしてるマクロ使え系
直接呼ぶと間違えがちだったり読みにくかったりするし、そもそも中身の挙動や実装を変えたくなったときにいちいち全部巡回するのは面倒すぎるのでマクロやメソッドで内部実装をwrapすることが多い。そんなとき元のやつを使われると困るのでBan。  


* r'/base::SequenceChecker\b': 理由→'Consider using SEQUENCE_CHECKER macros instead of the class directly.',
* r'/base::ThreadChecker\b': 理由→'Consider using THREAD_CHECKER macros instead of the class directly.',

### 例外禁止
Chromiumでは例外の仕様を禁止している。  
そして、その結果例外を使って結果を返す関数も連鎖的にBan対象。  
stoiなどが対象。  
* `r'/#include <exception>'`: 理由→'Exceptions are banned and disabled in Chromium.'
* r'/\bstd::sto(i|l|ul|ll|ull)\b': 理由→
    'std::sto{i,l,ul,ll,ull}() use exceptions to communicate results. ',
    'Use base::StringTo[U]Int[64]() instead.'

* r'/\bstd::sto(f|d|ld)\b': 理由→
    'std::sto{f,d,ld}() use exceptions to communicate results. ',
    'For locale-independent values, e.g. reading numbers from disk',
    'profiles, use base::StringToDouble().',
    'For user-visible values, parse using ICU.'

### Chromium自前の実装を使え
* r'/\bstd::function\b': 理由→'std::function is banned. Use base::{Once,Repeating}Callback instead.'
* r'/\bstd::bind\b': 理由→'std::bind() is banned because of lifetime risks. Use base::Bind{Once,Repeating}() instead.',
* r'/\bstd::shared_ptr\b': 理由→'std::shared_ptr is banned. Use scoped_refptr instead.'
* r'/\bstd::weak_ptr\b': 理由→'std::weak_ptr is banned. Use base::WeakPtr instead.'

#### optionalを発見
```
BanRule(
  r'/\bstd::optional\b',
  (
    'std::optional is not allowed yet (https://crbug.com/1373619). Use ',
    'absl::optional instead.',
  ),
  True,
  [
      # Clang plugins have different build config.
      '^tools/clang/plugins/',
      # Not an error in third_party folders.
      _THIRD_PARTY_EXCEPT_BLINK,
  ],
),
```
なんかちょっとうれしい。  

### パフォーマンス
std::to_stringはlocale-dependentで遅い。  
なのでこれもChromium自前の実装を使え形式。代替実装としてbase::NumberToStringやbase::FormatNumberを使うらしい。  

* r'/\bstd::to_string\b': 理由→
    'std::to_string() is locale dependent and slower than alternatives.',
    'For locale-independent strings, e.g. writing numbers to disk',
    'profiles, use base::NumberToString().',
    'For user-visible strings, use base::FormatNumber() and',
    'the related functions in base/i18n/number_formatting.h.',
    
TODO(elkurin): 実装を見てみよう

## Note
### chromium_presubmit
ここまで触れてきたPRESUBMITはgit cl でトリガーされるやつ。その他にもpresubmitというやつを見る。  

Chromiumにコードをmergeしようとした時、いろいろなテストボットが走る。多くは実際に挙動が壊れていないかを各プラットフォーム(ChromeOS, MacOS, WindowsOS...)上でテストしているボットで、その中に[chromium_presubmit](https://ci.chromium.org/ui/p/chromium/builders/try/chromium_presubmit)というボットがある。このボットはフォーマットが正しいかなど形式的なcheckをしてくれる。  

Executing commandを見る限りこれも`depot_tools/presubmit_support.py`をしてそう。
