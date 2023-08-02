# 特殊化templateの順序

今日のTGIFでtemplateについておしゃべりしていた時に出た話。  
複数の特殊化されたtemplateのうちより当てはまるやつが適用される→「より当てはまる」ってなんだ？  

## 特殊化とは
一番オーソドックスなtemplateは以下のようなやつ。
```cpp=
template<typename T>
void f(T) {
  std::cout << "普通のやつ\n";
}
```
これをプライマリテンプレートと呼ぶ。これでも便利だが、与える型によって実装を変えたいときもある。そんなときに使えるのが完全特殊化と部分特殊化というテクニック。  


templateは引数に当てはめる具体的な型に応じて関数を生成し、コンパイル時にインライン展開される。この関数の生成を「特殊化」という。  

templateは同じ関数名に対して複数の実装を書くことができ、マッチするやつを選択する機能がある。  
例えば以下のように型の中身を書かず`template<>`として`f<型名>`で実装すれば引数の型が完全に一致した時のみ使われる実装を定義できる。これを完全特殊化という。  
```cpp=
template<typename T>
void f(T) {
  std::cout << "普通のやつ\n";
}

template<>
void f<int>(int) {
  std::cout << "完全特殊化\n"
}

f("hello"); // 普通のやつ
f(1); // 完全特殊化
```

一方で完全一致しか無いと面倒なことがある。例えばtemplate引数が複数あって１つ一致したときに使用したいときや、`std::vector<T>`のように構造だけ一致してればOKみたいにしたいこともある。  
そんなときに使えるのが部分特殊化。以下のように引数の部分でマッチングできる。
```cpp=
template<typename T>
void f(T) {
  std::cout << "普通のやつ\n";
}

template<typename T>
void f(std::vector<T>) {
  std::cout << "部分特殊化\n";
}

f("1"); // 普通のやつ
f(std::vector<int>()); // 部分特殊化
```

これらのテクニックを使っている箇所はChromiumの中にもたくさんある。  
例えばraw_ptrのサポート確認をしている。  
以下[base/allocator/partition_allocator/pointers/raw_ptr.h](https://source.chromium.org/chromium/chromium/src/+/main:base/allocator/partition_allocator/pointers/raw_ptr.h;l=405-438;drc=69f4da820e296d4424ab6a3efa1f99488b65c768)から抜粋したサンプル。  
```cpp=
template <typename T, typename SFINAE = void>
struct IsSupportedType {
  static constexpr bool value = true;
};

// 部分特殊化
template <typename T>
struct IsSupportedType<T, std::enable_if_t<std::is_function<T>::value>> {
  static constexpr bool value = false;
};

// 完全特殊化
template <>
struct IsSupportedType<cc::Scheduler> {
  static constexpr bool value = false;
};
```

## どの特殊化を使用する？
ところで上述の部分特殊化や完全特殊化の実装は一つの関数に対し複数実装することができる。  
上の例だとわかりやすくどれが適応されるかわかる気がするが、複数マッチする場合はどれを使えばいいのか？  
`f(std::vector<int>())` の時点で`f(T)`にも`f(std::vector<T>)`にもマッチしてしまうので、何かしら優先順位が定まっているはず。  

「T1がT2より特殊化されている」というのは、直感的には「T1が受理する型はT2が受理する型の部分集合である」と言える。しかしコンパイラに数学は多分できない（？）  
何かしら半順序のルール付が必要になる。  

### 実験
まず実際に挙動を確認してみる。  
例えば以下の例を考える。
```cpp=
template<typename T>
void f(T) {
  std::cout << "Level 1\n";
}

template<typename T>
void f(T*) {
  std::cout << "Level 2\n";
}

template<typename T>
void f(const T*) {
  std::cout << "Level 3\n";
}

int i = 0;
const int ci = 0;
int* pi = &i;
const int* cpi = &ci;

f(i); // Level 1
f(pi); // Level 2
f(cpi); // Level 3
```
直感通りな感じに `T` < `T*` < `const T*` の順になっている。  

順序付けができなそうな例も考えてみる。

```cpp=
template<typename T1, typename T2>
void f(T1, T2) {
  std::cout << "Level 1, 1\n";
}

template<typename T1, typename T2>
void f(std::vector<T1>, T2) {
  std::cout << "Level 2, 1\n";
}

template<typename T1, typename T2>
void f(T1, std::vector<T2>) {
  std::cout << "Level 1, 2\n";
}


f(0, 0); // Level 1, 1
f(std::vector<int>(), 0); // Level 2, 1
f(0, std::vector<int>()); // Level 1, 2
f(std::vector<int>(), std::vector<float>()); // コンパイルエラー
```
最後の例以外でコンパイルすると問題なく通る。  
予想はつくが、受理される型同士の包含関係が順序付けてきていなくてもそのケースを踏む使用例が無い限りコンパイルはできるらしい。

### 関数テンプレートの仕様
[cppreference.com](https://en.cppreference.com/w/cpp/language/partial_specialization)によるとtemplateは以下のルールで適応される。  
1. マッチする特殊化がひとつだけの場合はそれを使う
2. 複数の特殊化がマッチする場合はどの特殊化がより特殊化されているか半順序のルールで探す。最も特殊化されているものが一意に定まればそれを使用、定まらなければコンパイル失敗
3. マッチする特殊化がなければプライマリテンプレートを使用

*note: プライマリテンプレートは特殊化に含まれない*  

この半順序のルールがミソなので、[cppreference.com](https://ja.cppreference.com/w/cpp/language/function_template#Function_template_overloading)を読もうと思ったが…長すぎ  

まずマッチする特殊化をすべて見つける。その後マッチする集合の中で任意の組み合わせに対して順序付けを行う。（コンパイラがすべての組み合わせ分計算してるわけではないかもしれない、推移束とかは使ってるかも？TODO）  
２つのテンプレートに対しては以下のように順序を考える。  
１つ目のテンプレートを実引数テンプレートとして、他方のテンプレートの元のテンプレート型を仮引数テンプレートとして用いて[テンプレートの実引数推定](https://ja.cppreference.com/w/cpp/language/template_argument_deduction)を行う。逆も行う。どちらかのみ成功すれば順序付け可能。  

例えば`T` < `T*` < `const T*`の例を考える。
```cpp=
const int* cpi;
f(cpi);

Level 2 -> Level 1 : void(U1*) -> void(T) : A=U1*, P=T 推定成功 T=U1*
Level 1 -> Level 2 : void(U1) -> void(T*) : A=U1, p=T* 推定失敗
Level 2のほうが特殊化されている。

Level 3 -> Level 1 : void(const U1*) -> void(T) : A=const U1*, P=T 成功 t+const U1*
Level 1 -> Level 3 : void(U1) -> void(const T*) : A=U1, P=const T* 失敗

Level 3 -> Level 2 : void(const U1*) -> void(T*) : A=const U1*, P=T* 成功 T=const U1
Level 2 -> Level 3 : void(U1*) -> void(const T*) : A=U1*, P=const T* 失敗
Level 3のほうが特殊化されている。
```
まあなんとなくわかった（？）  
[テンプレートの実引数推定](https://ja.cppreference.com/w/cpp/language/template_argument_deduction)というやつを理解しないといけないが、なんかここまで落とし込むとできそうな気がする。けど長すぎ。
パット見た感じ、一貫したルールで一発解決というわけではなく、それぞれのケースについて個別にルールを作っているっぽい。  

### クラステンプレートは？
以下のような架空の関数テンプレートに変換する。  
1. １つ目の部分特殊かと同じテンプレート仮引数と１つ目の部分特殊化の実引数を用いたクラステンプレートの特殊化を型とする１個の関数引数を持つテンプレート
2. ２つ目の部分特殊化と同じテンプレート引数と、２つ目の部分特殊化の実引数を用いたクラステンプレートの特殊化を型とする１個の関数引数を持つテンプレート

```cpp=
// プライマリテンプレート
template<int I1, int I2, typename T>
struct X {};

// 部分特殊化 Level 1
template<int I1, int I2>
struct X<I1, I2, int> {};
// Level 1 に対する架空の関数テンプレートは以下
template<int I1, int I2> void f(X<I1, I2, int>);

// 部分特殊化 Level 2
template<int I>
struct X<I, I, int> {};
// Level 2 に対する架空の関数テンプレートは以下
template<int I> void f(X<I, I, int>);
```
すると関数テンプレートのオーバーロードを同様に適用できるようになる。
