# 文字のイテレータ in Chromium

文字型のイテレータ型がChromiumには実装されている。  
それらイテレータに何ができるかやどんなのがあるかを見てみる。

## CharIterator
[CharIterator](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/char_iterator.h)はCharのイテレータを実装している。

使用例はこんな感じ。
```cpp=
base::i18n::UTF8CharIterator iter(text_utf8);
while (iter.array_pos() < range.start()) {
  iter.Advance();
}
```

`UTF8CharIterator`はUTF8型用。  
コンストラクタにStringPieceとして文字列を渡し、その`str`を[`str_`](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/char_iterator.h;l=51;drc=e4622aaeccea84652488d1822c28c78b7115684f)に保存してイテレーションしてくれる。  
現在参照しているCharはint32_型の`char_`として保存されている。  

[Advance](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/char_iterator.h;l=47;drc=e4622aaeccea84652488d1822c28c78b7115684f)を呼ぶと次に進む。  
`array_pos_`に現在のポジションをいれ、そこから次の文字の位置を`next_pos_`に持つ。  
`char_pos_`はそれとは違い、現在何文字目を見ているかを保持しているので、Advanceのたびに1だけインクリメントされる。  
`next_pos_`の計算は[CBU8_NEXT](https://source.chromium.org/chromium/chromium/src/+/main:base/third_party/icu/icu_utf.h;l=237;drc=8c5d627af88158e0026d2e2d0eb0298194686523)というマクロで行われる。  
[ICU](https://icu.unicode.org/)というthid pirtyライブラリから持ってきている。  
> ICU is a mature, widely used set of C/C++ and Java libraries providing Unicode and Globalization support for software applications. 

文字や浮動小数点処理まわりはthird partyを使っているところが多い気がする。  

このイテレータにはAdvanceの他に`get`で現在の文字を取得することやポジションを取ることはできるが、BackIteratorはなく複数文字進むようなAPIはない。

### UTF16CharIterator
上記と近いイテレータで[UTF16CharIterator](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/char_iterator.h;l=66;drc=e4622aaeccea84652488d1822c28c78b7115684f)がある。  
名前の通り、UTF16版。

こちらはUTF8版と違い[Rewind](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/char_iterator.h;l=123;drc=e4622aaeccea84652488d1822c28c78b7115684f)というもとに戻るAPIもある。  
またLowerBound/UpperBoundというオフセットの場所から区切りの場所を返す返すメソッドもある。  
UTF16のほうがメソッドが多いのは、シンプルに必要に応じて実装されていきUTF8はまだサポートされていないだけっぽい。使用例を確認してもUTF16の方が圧倒的に多い。

どれも内部の実装は[ICU](https://icu.unicode.org/)を使っている。  
CharIteratorはICUのラッパーみたいなもの。

## BreakIterator
CharIteratorの他に、[BreakIterator](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/break_iterator.h)という区切りをいろいろ設定できるものがある。

区切りのルールは以下の通り。
```cpp=
enum BreakType {
  BREAK_WORD,
  BREAK_LINE,
  BREAK_SPACE = BREAK_LINE,
  BREAK_NEWLINE,
  BREAK_CHARACTER,
  RULE_BASED,
  BREAK_SENTENCE,
};
```
各BreakTypeによって[初期化](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/break_iterator.cc;l=125;drc=8c5d627af88158e0026d2e2d0eb0298194686523)時にキャッシュタイプを変えている。  
[`DefaultLocalBreakIteratorCache<UBRK_WORD>>`](https://source.chromium.org/chromium/chromium/src/+/main:base/i18n/break_iterator.cc;l=82-89;drc=8c5d627af88158e0026d2e2d0eb0298194686523)のようにBreakIterator用のキャッシュを作っている。

この中でもICUのライブラリの[ubrk_next](https://source.chromium.org/chromium/chromium/src/+/main:third_party/icu/source/common/ubrk.cpp;l=224;drc=8c5d627af88158e0026d2e2d0eb0298194686523)などを使用している。  

## StringPieceとは
StringPieceは[std::string_view](https://en.cppreference.com/w/cpp/string/basic_string_view)のこと。  
これは文字列の所有権を保持せず、文字列を参照して、参照先の文字列を加工してあつかうクラス。  
普通のstringが持つメンバ関数は同様に使用できる。
