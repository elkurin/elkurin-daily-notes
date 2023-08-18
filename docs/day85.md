# base::expected in Chromium

*今日は軽めです*
std::optionalは成功値またはエラー値が返せる。  
しかしstd::optionalではエラーとしてnullptrしか返せない。  
それの進化版としてエラー値を好きな型でいれることができるexpectedという型がある。  
その内容を確認する。
 
## 概要
[std::expected](https://en.cppreference.com/w/cpp/utility/expected)はC++23から採用された型で期待される型`T`とエラーの型`E`のどちらかを返す型を定義できる。

C++のコンパイラではまだほとんど実装されていないし少なくともChromiumではサポートされていないので、その代用品としてChromium独自の実装[base::expected](https://source.chromium.org/chromium/chromium/src/+/main:base/types/expected.h)がある。

```cpp=
base::expected<QueryResult, std::string> RunQuery(const std::string& query) {
  auto result_or_error = test_trace_processor_.ExecuteQuery(query);
  if (!result_or_error.ok()) {
    return base::unexpected(result_or_error.error());
  }
  return base::ok(result_or_error.result());
}
```

`base::ok`は成功時、`base::unexpected`はエラー時に値をラップする。  

## stdとの仕様の違い
成功時の値が
```cpp=
// std
return 1;

// base
return base::ok(1);
```
という表記の差がある。

また例外も返さない。Chromiumには必須の条件。

他にもimplicit conversionを許さないなどの差はあるが、多分stdで実装されたら変換できる程度の差分っぽい。
