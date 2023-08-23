# アーキテクチャごとのコンパイラ最適化フラグ

CFLAGS と CXXFLAGS はC/C++のコンパイル時に慣例的に使われるオプションの環境変数。  
Makefile内で指定するか、ない場合は環境変数から読むことが多い。

CFLAGSやCXXFLAGSに含まれるオプションの使用目的は、各システム向けに可能な限り早くて小さく完全に動作するコードを生成すること。

いくつかよく使われるフラグを見ていく。

## -march
システムプロセッサアーキテクチャを指定するオプション。  
コンパイラにどんな命令が使用できるかを指定する。  
つまり`-march=cpu-type`はオプションの集合みたいなもの。  
ちなみに今どんなCPUを使っているかは
```shell=
$ cat /proc/cpuinfo
```
で確認できる。  
また指定するcpu-typeは
```shell=
$ /lib64/ld-linux-x86-64.so.2 --help
...
Subdirectories of glibc-hwcaps directories, in priority order:
 x86-64-v4
 x86-64-v3 (supported, searched)
 x86-64-v2 (supported, searched)
```
みたいに探せた。x86-64-v3が指定できるらしい。

基本特定のCPUタイプを指定するが、autoみたいな感じで`-march=native`とするとGCCがプロセッサを判別してフラグを設定してくれる。  
しかしあまり望ましくない。コンパイルしているマシンに特化しているため、例えばしょぼいパソコン用にハイパフォーマンスなパソコンでコンパイルすると実行できないファイルになったりする。  
クロスコンパイル時はちゃんと正しいcpu-typeを設定してあげよう。

各アーキテクチャに対して使用できるオプションの[リスト](https://gcc.gnu.org/onlinedocs/gcc/Submodel-Options.html#Submodel-Options)がある。  
例えば[x86 Options](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html)。  
同じx86の中にもめっちゃいっぱいある。  
e.g. x86-64, i386, corei7, skylake, goldmont ...

それぞれCPUの名前。  
例えばskylakeは
> Intel Skylake CPU with 64-bit extensions, MOVBE, MMX, SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2, POPCNT, CX16, SAHF, FXSR, AVX, XSAVE, PCLMUL, FSGSBASE, RDRND, F16C, AVX2, BMI, BMI2, LZCNT, FMA, MOVBE, HLE, RDSEED, ADCX, PREFETCHW, AES, CLFLUSHOPT, XSAVEC, XSAVES and SGX instruction set support.

サポートしている命令のリストが書いてある。  
`-march=skylake`とすると上記の命令セットが有効になる。

一方、各命令ごとに手動で有効にすることもできる。  
-msse、-msse2、-msse3、-mmmx、-m3dnow はそれぞれ Streaming SIMD Extentions (SSE)、SSE2、SSE3、MMX、3DNow!の命令セットを有効にする。  
基本正しい-marchを使っていればこれらの命令セットは含まれているはずだが、新しいVIAとAMD64はその限りではないらしい。  
cpuinfoの内容から正しいフラグをセットしよう。


## -mtune
marchと似てcpu-typeを指定する最適化のオプションだが、違いはmarchは可能な限りcpu-typeに最適化し他のプラットフォームでは動かなくなることも許容するが、mtuneはほどほどに最適化する。  
このオプションの用途や役割はアーキテクチャによってちょっと違うっぽい。

[x86-Options](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html)ではデフォルトのCPU、あるいは-marchが設定されているならそのcpu-typeでも動くようにしたまま、mtuneで指定されたCPUに可能な限り最適化しているらしい。

ARM系のmarchだとmtuneに[cortex-a53](https://en.wikipedia.org/wiki/ARM_Cortex-A53)などプロセッサを指定することができる。

## その他オプション
いろいろみていく


### [-mfpu](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html#index-mfpu-1)
ARMで使われる。  
FPUを指定する。  
`-mfpu=auto`で自動に設定できる。

### [-mfloat-abi](https://developer.arm.com/documentation/dui0774/b/compiler-command-line-options/-mfloat-abi)
soft, hardのいずれかを指定する。これは浮動小数点の演算の仕方は変えずいずれもFPUを使用して実行されるらしい。その違いは引数と戻り値の入る場所。softなら普通にARMの整数レジスタを使用して渡すが、hardはFPUレジスタを使う。

### [-ftree-vectorize](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html#index-ftree-vectorize)
ループのベクトル化を頑張る。ループをベクトル演算命令に変換する。
