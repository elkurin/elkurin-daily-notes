# BUILDFLAG

Chromiumはいろんなプラットフォーム上で動くように実装されている。  
中にはプラットフォームごとに実装を切りける必要がある。  

[BUILDFLAG](https://source.chromium.org/chromium/chromium/src/+/main:build/buildflag.h;l=45;drc=50487bd48e946a090749d6506318e4762f6c2a74)とはプラットフォームごとや各フラグごとに実装を変えたいときに使用されるマクロ。  
ファイルを変えてプラットフォームごとにincludeをかえることもでき、実際その手法もChromiumではよく出てくるが、ほぼ同じで一部変えたいときはクラスやファイルを分けるよりコード中にifdefフラグをいれるほうが良いケースが多い。

## 使用例
以下のような感じ。
```cpp=
#if BUILDFLAG(IS_CHROMEOS_ASH)
AshImpl();
#elif BUILDFLAG(IS_CHROMEOS_LACROS)
LacrosImpl;
#else
NOTREACHED();
#endif
```
BUILDFLAGに渡せる引数は決まっており、IS_MACやIS_WINなどOSなこともあるし、IS_CHROME_BRANDEDというChromiumなのかGoogle産のChromeなのかを切り替えるものもあるし、いろいろある。

## 内部実装
この内部実装は

```cpp=
#define BUILDFLAG_CAT_INDIRECT(a, b) a ## b
#define BUILDFLAG_CAT(a, b) BUILDFLAG_CAT_INDIRECT(a, b)

#define BUILDFLAG(flag) (BUILDFLAG_CAT(BUILDFLAG_INTERNAL_, flag)())
```
`##`はMACROで主に使われる、トークンを連結する演算子。  
なのでやっていることは`BUILDFLAG_INTERNAL_`に`flag`をひっつけているだけの関数を呼んでいる。  
`BUILDFLAG_CAT_INDIRECT`と`BUILDFLAG_CAT`が2段階ふまれているのは、`##`を直接呼ぶと引数が展開されてしまうから。

では`BUILDFLAG_INTERNAL_`とはなにか？  
これは[build_config.h](https://source.chromium.org/chromium/chromium/src/+/main:build/build_config.h;drc=c964de601a7fdd237e3df1fd99b96c94b2c4fb84)にある。  

```cpp=
#if defined(OS_CHROMEOS)
#define BUILDFLAG_INTERNAL_IS_CHROMEOS() (1)
#else
#define BUILDFLAG_INTERNAL_IS_CHROMEOS() (0)
#endif
```
こんな感じでflag名と合体して1or0を変える関数になる。  
つまり、BUILDFLAG(flag)を呼ぶとflagがあるかどうかで0,1が帰ってくるので期待通り動く。

なお[defined](https://gcc.gnu.org/onlinedocs/cpp/Defined.html)は引数がマクロでdefineされているかどうかを返す。  
OS_CHROMEOSがdefineされているのは[BUILD.gn](https://source.chromium.org/chromium/chromium/src/+/main:build/config/linux/BUILD.gn;l=40;drc=54d7c782852ccf1e1b38c4dbe90d419022753e1e)の中
```
if (is_chromeos) {
  defines = [ "OS_CHROMEOS" ]
}
```
