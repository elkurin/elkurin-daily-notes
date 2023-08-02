# Browser Test のマクロ

Chromiumでテストを書く時以下のように`IN_PROC_BROWSER_TEST_F`などのマクロを使用して宣言する。
```cpp=
IN_PROC_BROWSER_TEST_F(TooltipBrowserTest,
                       ShowTooltipFromWebContentWithCursor) {
  NavigateToURL("/tooltip.html");
  std::u16string expected_text = u"my tooltip";
  ...; // test content
}
```
特にテストの名前の先頭にDISABLEDとかいれるとテストが勝手にdisableされてくれる。  
一体何をしてる？

## マクロ内のしごと
マクロの定義は[browser_test.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test.h)の中にある。  

1つ目の引数(上の例では`TooltipBrowserTest`)が`test_case_name`に、2つ目の引数(上の例では`ShoTooltipFromWebContentWithCursor`)が`test_name`に入る。  

大きく分けて3つのことをしている。  
1. テストクラスの生成
2. TestInfoを生成して登録
3. Runするためのメソッドの定義


まず[GTEST_TEST_CLASS_NAME_](https://source.chromium.org/chromium/chromium/src/+/main:third_party/googletest/src/googletest/include/gtest/internal/gtest-internal.h;l=1533;drc=b007c54f2944e193ac44fba1bc997cb65826a0b9)マクロで`test_case_name`と`test_name`からクラス名を生成する。  
```cpp=
#define GTEST_TEST_CLASS_NAME_(test_suite_name, test_name) \
  test_suite_name##_##test_name##_Test
```
上の例の場合は`TooltipBrowserTest_ShowTooltipFromWebContentWithCursor_Test`になる。  
このクラスは`parent_class`を継承する。この場合[`TooltipBrowserTest`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/tooltip/tooltip_browsertest.cc;l=144;drc=e6361d070be0adc585ebbff89fec76e2df4ad768)クラスがこれに相当する。  
このマクロが正しく動くためにはInProcessBrowserTestを継承したクラスが親である必要があり、これらのクラスは基本テストのccファイルの中で宣言されている。  
InProcessBrowserTestであれば[RunTestOnMainThread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test.h;l=69;drc=a328097ef1d47f51e76917b80110863037f4f744)がoverrideできコンパイルが通るようになっている。  
生成したクラスの`test_info_`メンバにtesting::TestInfoを作って入れる。  
TestInfoはgoogletestライブラリの中にある[MakeAndRegisterTestInfo](https://source.chromium.org/chromium/chromium/src/+/main:third_party/googletest/src/googletest/include/gtest/internal/gtest-internal.h;l=586;drc=b007c54f2944e193ac44fba1bc997cb65826a0b9)から生成。  
この関数では生成したTestInfoをGoogle Testに登録している。
```cpp=
TestInfo* MakeAndRegisterTestInfo(
    const char* test_suite_name, const char* name, const char* type_param,
    const char* value_param, CodeLocation code_location,
    TypeId fixture_class_id, SetUpTestSuiteFunc set_up_tc,
    TearDownTestSuiteFunc tear_down_tc, TestFactoryBase* factory) {
  TestInfo* const test_info =
      new TestInfo(test_suite_name, name, type_param, value_param,
                   code_location, fixture_class_id, factory);
  GetUnitTestImpl()->AddTestInfo(set_up_tc, tear_down_tc, test_info);
  return test_info;
}
```
TestInfoにはtest suite name, test nameの他にいろいろなパラメータを設定することが出来る。  
IN_PROC_BROWSER_TEST_Fの場合は  
`code_location`に`::testing::internal::CodeLocation(__FILE__, __LINE__)`  
`set_up_tc`にparent_class::SetUpTestCase  
`tear_down_tc`にparent_class::TearDownTestCaseを代入している。  
SetUpTestCaseやTearDownTestCaseは[testing::Test](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/include/gtest/gtest.h;l=257-258;drc=a328097ef1d47f51e76917b80110863037f4f744)の中で定義されている。なおInProcessBrowserTestやcontent::BrowserTestなど各テストクラスはすべてtesting::Testを継承しているので、ちゃんと定義されている。  

クラスの定義、TestInfoの登録ができたら、定義したクラス`TooltipBrowserTest_ShowTooltipFromWebContentWithCursor_Test`に対するRunTestOnMainThread()の実装を与える。  
なので、このマクロでは`void GTEST_TEST_CLASS_NAME_(test_case_name, test_name)::RunTestOnMainThread()`までとなっており、上の例のようにIN_PROCESS_BROWSER_TEST_Fマクロに続けて`{ // test content }`と書けばRunTestOnMainThread()に対する実装を与えていることになる。  

ここで作ったメソッドはBrowserTestでは[BrowserTestBase::ProxyRunTestOnMainThreadLoop](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test_base.cc;l=914;drc=a328097ef1d47f51e76917b80110863037f4f744)で呼ばれる。この時テストが走り終わるまで`base::ScopedDisallowBlocking`でスレッドをブロックしており、終わったらTearDownOnMainThreadが呼ばれる。  
ではこの[BrowserTestBase::ProxyRunTestOnMainThreadLoop](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test_base.cc;l=914;drc=a328097ef1d47f51e76917b80110863037f4f744)がどう呼ばれるかというと、まず[BrowserTestBase::SetUp](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test_base.cc;l=297;drc=a328097ef1d47f51e76917b80110863037f4f744)で`ui_task`にbase::BindOnceして渡され、後で[Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:content/public/test/browser_test_base.cc;l=721;drc=a328097ef1d47f51e76917b80110863037f4f744)される。この時initializeを正しく待つためにNetableTasksAllowedなRunLoopを使っている。  

## Testはどうやって走るか？
[TestSuite](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/include/gtest/gtest.h;l=652;drc=a328097ef1d47f51e76917b80110863037f4f744)はTestInfoからなるvectorの[`test_info_list_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/include/gtest/gtest.h;l=835;drc=a328097ef1d47f51e76917b80110863037f4f744)を持つ。ここには上のセクションで言及した[MakeAndRegisterTestInfo](https://source.chromium.org/chromium/chromium/src/+/main:third_party/googletest/src/googletest/include/gtest/internal/gtest-internal.h;l=586;drc=b007c54f2944e193ac44fba1bc997cb65826a0b9)によって登録されている。  

この中のTestInfoたちは[TestSuite::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/src/gtest.cc;l=2981;drc=a328097ef1d47f51e76917b80110863037f4f744)によってtriggerされる。  
このRunは[TestSuite::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/test_suite.cc;l=419;drc=a328097ef1d47f51e76917b80110863037f4f744)->[RUN_ALL_TESTS()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/include/gtest/gtest.h;l=2284;drc=a328097ef1d47f51e76917b80110863037f4f744)マクロ->[UnitTest::Run](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/src/gtest.cc;l=5363;drc=a328097ef1d47f51e76917b80110863037f4f744)->[UnitTestImpl::RunAllTests](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/src/gtest.cc;l=5744;drc=a328097ef1d47f51e76917b80110863037f4f744)から[`test_suites_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/src/gtest-internal-inl.h;l=859;drc=a328097ef1d47f51e76917b80110863037f4f744)に存在するすべてのTestSuiteに対し発火されている。  

使用例を見てみる。  
これはash_crosapi_testで使われている。  
[`CrosapiTestSuite`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/test/ash_crosapi_tests_main.cc;l=20;drc=14b9b14728983afc9d06a5ab683c49ef7e56d753)はbase::TestSuiteを継承して作られているが、このRun関数を[LaunchUnitTestSerially](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/launcher/unit_test_launcher.cc;l=294;drc=a328097ef1d47f51e76917b80110863037f4f744)の`run_test_suite`コールバックとしてわたし、実際に[RunTestSuite](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/test/launcher/unit_test_launcher.cc;l=179;drc=a328097ef1d47f51e76917b80110863037f4f744)の中で走っている。  

このようにテストの実行ファイルにおけるmain関数からほぼ直接TestSuite::Runを呼んで、登録されたテストすべてを走らせている。

## DISABLEDマクロ
以下のようにテストをDisableすることができる。  
```cpp= 
#if BUILDFLAG(IS_CHROMEOS) || BUILDFLAG(IS_LINUX)
#define MAYBE_ShowTooltipFromWebContentWithKeyboard \
  DISABLED_ShowTooltipFromWebContentWithKeyboard
#else
#define MAYBE_ShowTooltipFromWebContentWithKeyboard \
  ShowTooltipFromWebContentWithKeyboard
#endif
IN_PROC_BROWSER_TEST_F(TooltipBrowserTest,
                       MAYBE_ShowTooltipFromWebContentWithKeyboard) {
  // test content
}
```
テストの名前を変えるだけでどうやってフィルターしているのか？  

このDISABLEDという名前はTestInfoの中に`is_disabled`メンバとして情報が格納され、Filterに使われている。  
[UnitTestImpl::FilterTests](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/googletest/src/googletest/src/gtest.cc;l=6038;drc=a328097ef1d47f51e76917b80110863037f4f744)でTestInfo::is_disabled_が以下のように更新されている。  
```cpp=
static const char kDisableTestFilter[] = "DISABLED_*:*/DISABLED_*";
...;
const UnitTestFilter disable_test_filter(kDisableTestFilter);;

...;
const std::string test_name(test_info->name());
const bool is_disabled =
    disable_test_filter.MatchesName(test_suite_name) ||
    disable_test_filter.MatchesName(test_name);
test_info->is_disabled_ = is_disabled;
```
