# パソコンが壊れた

ぴえん

## 何が起きたか
[Ninja](https://ninja-build.org/manual.html)システムを使ってビルドをしていたら突然以下のようなエラーが出た。
```
ninja: error: rebuilding 'build.ninja': Error writing to build log: Read-only file system
```
ああ、稀によくあるディレクトリ内でなんか壊れたやつか、と思いとりあえず[`gn clean`](https://gn.googlesource.com/gn/+/master/docs/reference.md#cmd_clean)してみたが、なんかまたエラーを吐いている。  
英語が読めないので、じゃあディレクトリ消しちゃうかと思い `rm -rf out_linux_lacros/Release` を試したがまたエラーが出てきて消すことすらできない。
ここでついに開眼し英語が読めるようになる。  

**Read-only file system**

なんといつの間にかファイルシステムが読み込み専用になっていた。

## なぜ起きたか
まずdmesgを確認した  
```
$ dmesg
...
[366422.779286] I/O error, dev sda, sector 5651050888 op 0x0:(READ) flags 0x80700 phys_seg 1 prio class 2
...
[366422.829294] EXT4-fs (dm-3): Remounting filesystem read-only
...
```
なんらかのI/O errorが /dev/sda で起きたらしい。本環境ではSSDを指す。  
SSDとは**S**olid **S**tate **D**riveの略で、とりあえずここではデータを記録する外部ストレージという理解でOK。  
その結果エラーを解消すべくファイルシステムがread-onlyとしてremountedされたとのこと。

なぜread-onlyでremountするのか？  
`/etc/fstab`というstatic file system informationを持つファイルの中身を見てみた。  
```
$ cat /etc/fstab
...
# <file sys>    <mount point>   <type>  <options>       <dump>  <pass>
UUID={見せられないよ}       /       ext4    errors=remount-ro,iversion ...
```
ルートディレクトリのオプションに `errors=remount-ro` とある。  
LinuxのManPageの[`mount`](https://linuxjm.osdn.jp/html/util-linux/man8/mount.8.html)によると、これはエラーが起こったときの振る舞いを指定しており、この場合"ファイルシステムをリードオンリーでマウントしなおす"という挙動になる。  
なんらかのエラーがファイルシステムに起きた結果Read-onlyでマウントされてしまったということ。

なんのエラーかというと`I/O error, dev sda, sector 5651050888 op 0x0:(READ) flags 0x80700 phys_seg 1 prio class 2`らしい。  
ハードディスクの中にはデータを記録する部分がsectorと呼ばれる単位で細かく区切られている。  
この中でなんらかの理由でぶっ壊れているセクタが存在し、それを使っているからエラーになっているっぽい。

## どう対処したか
とりあえずマウントしてみた。  
[`mount`](https://man7.org/linux/man-pages/man8/mount.8.html)コマンドは文字通りマウントする。ディスク装置をLinuxのディレクトリ内に埋め込み使えるようにする。
```
$ mount
...
/dev/mapper/glinux_20210602-root on / type ext4 (ro,relatime,errors=remount-ro)
...
/dev/mapper/sda_crypt on /scratch type ext4 (ro,relatime)
...
```
右のカッコ内に状態が記述されているが、どちらも"ro"すなわちRead-onlyと書いてある。
だめだった。  

次に読み書き権限を指定してマウントしてみた。
```
$ sudo mount -o remount,rw / 
$ sudo mount -o remount,rw /scratch
```
ルートディレクトにマウントには成功した。"rw"になってくれた。  
しかし/scratch (/dev/sda のマウントポイント)には、write-protectedだからという理由で却下された。

次に[`fsck`](http://linuxjm.osdn.jp/html/e2fsprogs/man8/fsck.8.html)というファイルシステムのチェックと修復を行うコマンドを試してみた。  
```
$ sudo fsck -y /dev/sda
```
これも理由は忘れたが動かなかった（確かすでにmountしてるからダメとか言ってた）。このコマンドはマウント前のデバイスでないと動かない。[参考](https://atmarkit.itmedia.co.jp/ait/articles/1803/15/news040.html)  

次にLinuxのDisksアプリを立ち上げた。  
問題の/dev/sdaを触ろうとしたらbusyなのでなんとかしろと言われた。なぜbusyかわからなかったのでとりあえずrebootした。  
その時点で/scratchを探してみたらなんと`ls`で見つからない。消えちゃった？  
と思ったが、そこからアプリ内のUIで/dev/sdaを開き  
Unlock -> Repair Filesystem -> Mount をやった。  
`df -h`という空き領域を表示するコマンドを使ってディスク一覧を確認したら、/scratch が復活した。  
`mount`コマンドで確認したら`/dev/mapper/sda_crypt on /scratch type ext4 (rw,relatime)`になってた。"rw"つまり書き込み権限が復活。  
めでたし。  

とはいえ、一箇所イカれ始めたらまた何度も壊れるのでSSDを買い替えたほうが良いと同僚に勧められた。



## 学び
### 不良セクタとSMART
bad sector  
物理的な不良や論理的な不良の理由で利用できなくなったセクタのこと。  
これらの不良セクタは検出してマークしておく機能がOSに備わっており、マークされたセクタは利用できなくなってデータも消失することになる。Linuxでは`badblocks`ツールで管理。  

この不良セクタは**S**elf-**M**onitoring, **A**nalysis and **R**eporting **T**echnology 略して smart という技術で早期検出・予測ができるようになっている。  

不良セクタの検出をしてみた。  
```
// badblocks を使う
$ sudo badblocks -v /dev/sda > badsectors.txt
```
長い。飽きた



SMARTでself-testを走らせてみた。長いのでとりあえずshortというすぐ終わるテスト。
```
$ sudo smartctl -t  short /dev/sda
smartctl 7.3 2022-02-28 r5338 [x86_64-linux-6.3.7-1rodete2-amd64] (local build)
Copyright (C) 2002-22, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Short self-test routine immediately in off-line mode".
Drive command "Execute SMART Short self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 2 minutes for test to complete.
Test will complete after Mon Jul 24 21:30:47 2023 JST
Use smartctl -X to abort test.
```
2分待つ。
```
$ sudo smartctl -H /dev/sda
smartctl 7.3 2022-02-28 r5338 [x86_64-linux-6.3.7-1rodete2-amd64] (local build)
Copyright (C) 2002-22, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

$ sudo smartctl -l selftest /dev/sda
smartctl 7.3 2022-02-28 r5338 [x86_64-linux-6.3.7-1rodete2-amd64] (local build)
Copyright (C) 2002-22, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     16566         -
```
SMARTさんによると大丈夫らしい。

### /dev/*
[Linuxの装置ネーミング](https://www.ibm.com/docs/ja/ds8800?topic=host-linux-device-naming)を確認した。

Linux上で /dev はデバイスが割り当てられている特殊ファイルがいる場所。カーネルドライバは装置を制御するために特殊な装置ファイルを使用する。  
SCSIディスク装置の特殊な装置ファイルには接頭部として先頭に"sd"がつき、そこに続く文字で識別される。形式は /dev/sd[a-z][a-z][1-15]。

中でも /dev/sda とは最初に見つかったハードディスクに与えられる名前で、本環境ではそれがSSDだった。
