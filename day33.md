# pragma pack

構造体のサイズは必ずしもメンバのサイズの合計とは限らない。  
例えば以下の`Int_Char`という構造体のサイズはいくつだろうか？
```cpp=
struct Int_Char {
  int a;
  char b;
};
```
int型は4bytesでchar型は1byteなので足したら5bytes?  
とおもいきや
```cpp=
std::cout << sizeof(Int_Char) << std::endl;

// 出力
8
```
これは自然な境界に位置合わせされるから。この場合、int型の4byptes区切りに合わされている。  

普通はこれでいいが、こちらで指定したいケースもある。そんなときに使えるのが`pragma pack`。

## pragma packとは
実際[ResourceFileの生成](https://source.chromium.org/chromium/chromium/src/+/main:ui/base/resource/data_pack_with_resource_sharing_lacros.cc;l=27-34;drc=c149c1657efb82884f21a3deea7678e66cfc422a)を昔書いていたときに、fileサイズを小さくしたくてheaderで勝手にでかい方に合わせてほしくなかったことがあった。  
その際以下のように書いた。

```cpp=
#pragma pack(push, 2)
struct FileHeaderV1 {
  uint32_t version;
  uint16_t mapping_count;
  uint16_t fallback_resource_count;
  uint16_t fallback_alias_count;
};
#pragma pack(pop)
```

`pragma pack(2)`とすると2bytes境界になる。  
なので上記の例だと4+2+2+2=10bytesから最大サイズの4bytesにアラインメントされ`sizeof(FileHeaderV1)`は12となるはずが、pragma packのおかげで2bytes境界になり`sizeof(FileHeaderV1)`は10となる。

## 構造体リサイズの仕様
ちょっと試してみた。

```cpp=
#include <iostream>

struct Normal {
  int a;
  char b;
};

#pragma pack(push, 1)
struct PragmaPacked {
  int a;
  char b;
};
#pragma pack(pop)

struct Sum {
  int a;
  PragmaPacked p;
};

struct Sum_Char {
  char c;
  PragmaPacked p;
};

int main(void) {
  std::cout << sizeof(Normal) << std::endl;
  std::cout << sizeof(PragmaPacked) << std::endl;
  std::cout << sizeof(Sum) << std::endl;
  std::cout << sizeof(Sum_Char) << std::endl;
  return 0;
}

// 出力
8  // Normal
5  // PragmaPacked
12 // Sum
6  // Sum_Char
```
Sumは5bytes + 4bytesだが12bytesになるらしい。  
Sum_Charは5bytes + 1byteで6bytesになるらしい。  
多分構造体のサイズではなく、その他のtrivialなメンバのサイズを基準にしている？  
では構造体しか無いときは？

```cpp=
struct PragmaPacked2 {
  PragmaPacked p1;
  PragmaPacked p2;
};

struct PragmaPacked3 {
  PragmaPacked p1;
  PragmaPacked p2;
  PragmaPacked p3;
};

// 出力
10 // PragmaPacked2
15 // PragmaPacked3
```
普通に境界とか知らん。になってた。  

ところで構造体の中身とか見てるのかな？

```cpp=
struct Double_Int {
  double d;
  int i;
};

struct Double_Int_and_Int {
  Double_Int d;
  int i;
};

struct Double_Int2 {
  double d;
  int i1;
  int i2;
};

int main(void) {
  std::cout << sizeof(Double_Int) << std::endl;
  std::cout << sizeof(Double_Int_and_Int) << std::endl;
  std::cout << sizeof(Double_Int2) << std::endl;
  return 0;
}

// 出力
16 // Double_Int
24 // Double_Int_and_Int
16 // Double_Int2
```

`Double_Int2` と `Double_Int_and_Int` は中身が同じだがサイズは別。  
構造体から別の構造体を作る際、その中身まで見て賢くするみたいなことはないらしい。
