# String Conversion in Chromium

PRESUBMIT.pyを読んだ回([ChromiumでBanされているやつら](/docs/day67.md))で触れたstring変換系のメソッドについて  
stringをintに変換する[std::stoi](https://en.cppreference.com/w/cpp/string/basic_string/stol)やstringをdoubleに変換する[std::stod](https://en.cppreference.com/w/cpp/string/basic_string/stof)が標準ライブラリに実装されているが、Chromiumでは自前の実装である[base::StringToInt](https://source.chromium.org/chromium/chromium/src/+/main:base/strings/string_number_conversions.cc;l=73;drc=e4622aaeccea84652488d1822c28c78b7115684f)や[base::StringToDouble](https://source.chromium.org/chromium/chromium/src/+/main:base/strings/string_number_conversions.cc;l=113;drc=e4622aaeccea84652488d1822c28c78b7115684f)を使用することになっている。  
Chromiumでの実装や標準ライブラリとの差分を見ていく。

## std::stoi と StringToInt
[std::stoi](https://en.cppreference.com/w/cpp/string/basic_string/stol)はstd::stringまたはstd::wstringの文字列`str`を受け取って、int型に変換する関数。  
ここまでは[base::StringToInt](https://source.chromium.org/chromium/chromium/src/+/main:base/strings/string_number_conversions.cc;l=73;drc=e4622aaeccea84652488d1822c28c78b7115684f)と同じ。  
違いはAPIの呼び方と変換に失敗した際の挙動。  

それぞれAPIは以下。
```cpp=
// 結果は返り値で返す
int stoi(const std::string& str, std::size_t* idx = nullptr, int base = 10);

// 結果は渡したポインタ`output`に代入
// 返り値は成功・失敗のbool値
bool StringToInt(StringPiece input, int* output);
```
StringToIntは返り値のboolがtrueの場合のみ`output`が保証される。falseの場合もなにか値が入っているが途中結果だったり0だったりする。
  

また失敗時の挙動を見てみる。  
std::stoiは失敗すると例外を送出する。  
* 変換が起きなかった場合→std::invalid_argument
* 変換結果がintの範囲外だった場合→std::out_of_range

が送られる。

一方[base::StringToInt](https://source.chromium.org/chromium/chromium/src/+/main:base/strings/string_number_conversions.cc;l=73;drc=e4622aaeccea84652488d1822c28c78b7115684f)は上でも触れたように成功・失敗は戻り値のbool値から判断でき、例外を出さない。  

Chromiumでは[例外を使用しない](https://google.github.io/styleguide/cppguide.html#Exceptions)ことになっているので、エラーを例外で受け取るstd::stoiは使用禁止になっている。

## StringToDouble
上記のIntとは異なり、doubleは[StringToDoubleConverter](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/third_party/double_conversion/double-conversion/string-to-double.h;l=35;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)で変換する。  
小数のいろいろな表記を許容するために`Flags`のenumのbit組でどの記法を許すか決めている。  
基本のStringToDoubleは`ALLOW_LEADING_SPACES`と`ALLOW_TRAILING_JUNK`をセットする。  
`ALLOW_LEADING_SPACES`はスペースを無視する e.g. "-  123.2" -> -123.2  
`ALLOW_TRAILING_JUNK`は変なcharがはいっていることを許容する。入っている場合はjunk_string_valueを返す。  
空の場合は0.0で、junkは0、NanやInfinityのシンボルはnullptrとし定めていない。  

この実装はbase/third_partyの中にあり、[double-conversion](https://github.com/google/double-conversion) projectの効率良い小数変換を使用している。  

## StringToNumberParser
Parserの中身で何をしているか追ってみる。

[base::StringToInt](https://source.chromium.org/chromium/chromium/src/+/main:base/strings/string_number_conversions.cc;l=73;drc=e4622aaeccea84652488d1822c28c78b7115684f)では[StringToNumber](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_number_conversions_internal.h;l=155;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)のテンプレートで変換を行う。この返り値はvalueと成功・失敗のbool値をあわらすvalidの組、[Result](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_number_conversions_internal.h;l=76;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)。

まずwhitespaceがあれば[IsAsciiWhitespace](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_util.h;l=389;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)で確認し空白をスキップする。なおこの空白があった場合はvalidが必ずfalseになるが、一応valueは計算されるようになっている。  
最初の文字が`-`である場合これは負の数になる。型によっては負の数を持てないので、`std::numeric_limits<Number>::is_signed`を確認しfalseなら失敗なのでvalidにfalseをいれて終了。  
OKなら`-`をのぞいた`begin+1`~`end`の範囲に対してParser::Negative::Invokeを呼ぶ。  
同様に`+`が先頭にある場合もスキップして今度はParser::Positive::Invokeを呼ぶ。

Parser::Negative/Parser::Positiveは[Curiously Recurring Template Pattern](/docs/day75.md)のテクニックを使っている。  
[Base](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_number_conversions_internal.h;l=90;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)は`Sign`のtypeを受けとる。各Signの`CheckBounds`と`Increment`の実装を参照している。

[Invoke](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_number_conversions_internal.h;l=93;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)はまず`x`や先頭の`0`などをスキップする。
```cpp=
if (kBase == 16 && end - begin > 2 && *begin == '0' &&
    (*(begin + 1) == 'x' || *(begin + 1) == 'X')) {
  begin += 2;
}
```
その後一文字ずつ[CharToDigit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/strings/string_number_conversions_internal.h;l=59;drc=ee37f51db66140a21ee2c9e88f967bd59c473f91)で変換していく。この返還に失敗したらvalidをfalseにして終了。  
大丈夫ならCheckBoundsで範囲内におさまっているかチェック。進数を表す`kBase`で、`td::numeric_limits<Number>::max()`から得た最大の数`kMax`を超えないか確認している。  
大丈夫ならIncrementを使ってvalueに新しい`new_digit`を足す。これはNegtiveの場合は引くことになる。  
この流れをすべての数字に対して行う。

最後まで問題なく完了したら`valid`がtrueをいれてここまでで計算した数を返す。
