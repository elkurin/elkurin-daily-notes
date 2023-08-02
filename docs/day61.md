
# またパソコンが壊れた

[パソコンが壊れた](/day60.md)。  
昨日直した（？）がまた1時間位ビルドしてたら壊れた。同じことをして問題は隠せたがまたビルドしたら壊れた。  
明日SSDが届くのでそれまで折角の機会なので調べてみた。[参考](https://www.smartmontools.org/wiki/BadBlockHowto)

## 不良セクタを発見
昨日はSMARTで2分で終わる短いテストを走らせて問題が見つからなかったので、昨晩320分かかるというテストを走らせておいた。
```
$ sudo smartctl -t long /dev/sda

# ログ抜粋
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed: read failure       80%     16570         1852424
# 2  Short offline       Completed without error       00%     16566         -
```
Read failureになった。正しい。  
その横に書いている`LBA_of_first_error`は最初に見つかったread failureのセクタ番号である。  
今回の場合は1852424セクタが壊れてるらしい。  

さらにログを読んでみると
```
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  5 Reallocated_Sector_Ct   0x0033   024   024   010    Pre-fail  Always       -       3570

183 Runtime_Bad_Block       0x0013   024   024   010    Pre-fail  Always       -       3570
```
Reallocated_Sector_CtをRuntime_Bad_Blockの値が同じ→Bad blockにあたったら自動でreallocateしてくれてるっぽい？  
3570回失敗というのがどれくらいかわからないが、再起動してからほぼテスト走らせる以外指定ない状態なので多いかも？

### ログたち
このときの状態のログを見てみる。  
まずdmesg(抜粋)
```
$ sudo dmesg
[ 1988.642306] I/O error, dev sda, sector 1852424 op 0x0:(READ) flags 0x0 phys_seg 1 prio class 2
[ 1988.642317] Buffer I/O error on dev sda, logical block 231553, async page read
[ 1988.642337] ata10: EH complete
[ 1988.642678] ata10.00: Enabling discard_zeroes_data
[ 1988.848063] ata10.00: exception Emask 0x0 SAct 0x10 SErr 0x0 action 0x0
[ 1988.848075] ata10.00: irq_stat 0x40000008
[ 1988.848081] ata10.00: failed command: READ FPDMA QUEUED
[ 1988.848085] ata10.00: cmd 60/08:20:08:44:1c/00:00:00:00:00/40 tag 4 ncq dma 4096 in
                        res 41/40:08:08:44:1c/00:00:00:00:00/00 Emask 0x409 (media error) <F>
[ 1988.848101] ata10.00: status: { DRDY ERR }
[ 1988.848106] ata10.00: error: { UNC }
[ 1988.848350] ata10.00: supports DRM functions and may not be fully accessible
[ 1988.851467] ata10.00: supports DRM functions and may not be fully accessible
[ 1988.854226] ata10.00: configured for UDMA/133
[ 1988.854249] sd 9:0:0:0: [sda] tag#4 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_OK cmd_age=0s
[ 1988.854256] sd 9:0:0:0: [sda] tag#4 Sense Key : Medium Error [current] 
[ 1988.854261] sd 9:0:0:0: [sda] tag#4 Add. Sense: Unrecovered read error - auto reallocate failed
[ 1988.854267] sd 9:0:0:0: [sda] tag#4 CDB: Read(16) 88 00 00 00 00 00 00 1c 44 08 00 00 00 08 00 00
```

次にfdisk
```
$ sudo fdisk -lu /dev/sda
Disk /dev/sda: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: Samsung SSD 870 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

最後にfstab
```
$ cat /etc/fstab 
# /etc/fstab: static file system information.
#
# <file sys>	<mount point>	<type>	<options>	<dump>	<pass>

# device during installation: /dev/glinux_20210602/root
UUID=851605f5-1579-4af3-bd5c-d5f34f374be7	/	ext4	errors=remount-ro,iversion	0	1

# device during installation: /dev/nvme0n1p2
UUID=27e5a6f6-7d52-4e36-b432-2932a8937002	/boot	ext2	iversion	0	2

# device during installation: /dev/nvme0n1p1
UUID=6A46-AF35	/boot/efi	vfat	iversion,umask=0077	0	2

# device during installation: /dev/glinux_20210602/swap
UUID=37936a61-2171-4cdf-a6d7-4c83f9dc5400	none	swap	sw	0	0
UUID=4ac76c58-fc47-4e35-b319-ad8ef81543e0 /scratch ext4 defaults, nofail 0 1
UUID=4ac76c58-fc47-4e35-b319-ad8ef81543e0 /scratch ext4 defaults,nofail 0 1
```


## 不良セクタを取り除きたい
[debugfs](https://man7.org/linux/man-pages/man8/debugfs.8.html)はext2, ext3, ext4のファイルシステムのデバッガ。  
ext2~4とはextended file systemと呼ばれるLinuxでのファイルシステムのこと、4が一番新しい。

使ってみる。
```
$ sudo debugfs
debugfs: Bad magic number in super-block while trying to open /dev/sda
/dev/sda contains a crypto_LUKS file system
```

どうやらsudo debugfsだとcrypt_LUKSは読めない  
[参考](https://superuser.com/questions/1592447/can-i-open-a-file-by-its-inode-in-a-ext4-luks-partition)

以下でmapされた先を渡してみるとうまくいった。  
このmapperはLVM(**l**ogical **v**olume **m**anager)を使用したときに見られるもの。LVMは複数のハードディスクやパーティションにまたがった記憶領域をまとめて単一の論理ボリューム（LV）として扱うことのできるディスク管理機能。[参考](https://www.pmi-sfbac.org/linux-lvm/)  
```
$ sudo debugfs /dev/mapper/sda_crypt
debugfs 1.47.0 (5-Feb-2023)
debugfs:  testb 1852424
Block 1852424 not in use
debugfs:  
```
bad sector の1852424 はnot in useだった。あれ？  
ここでdmesgを見てみる  
```
[ 1988.854270] I/O error, dev sda, sector 1852424 op 0x0:(READ) flags 0x0 phys_seg 1 prio class 2
[ 1988.854281] Buffer I/O error on dev sda, logical block 231553, async page read
```
logical block の231553を見てみると
```
debugfs:  testb 231553
Block 231553 marked in use
```
marked in use.だった。発見。

この値から逆算してみると`231553 = 1852424 * 512 / 4096`

[参考](https://www.smartmontools.org/wiki/BadBlockHowto)によると以下がlogical sectorを探す式  
> b = (int)((L-S)*512/B)
where:
b = File System block number
B = File system block size in bytes
L = LBA of bad sector
S = Starting sector of partition as shown by fdisk -lu
and (int) denotes the integer part.

つまり B = File system block size in bytes は4096。

脱線したが、次にこの壊れている不良セクタが何に使われているか引きつづきdebugfsの中で確認してみた。
```
debugfs:  icheck 231553
Block	Inode number
231553	169349969
debugfs:  ncheck 169349969
Inode	Pathname
169349969	/4-chromium/src/out_arm-generic/Lacros/pnacl/pnacl_public_arm_ld_nexe
```
    
普段自分が使っているビルドディレクトリの中のなんかのartifactだった。  
壊れているところにzerosを上書きして強制的にreallocateする。
```
$ sudo dd if=/dev/zero of=/dev/sda bs=4096 count=1 seek=231553
1+0 records in
1+0 records out
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.000259245 s, 15.8 MB/s
```

０で埋められたことを確認
```
$ sudo hdparm --read-sector 1852424 /dev/sda

/dev/sda:
reading sector 1852424: succeeded
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
```

[参考](https://www.smartmontools.org/wiki/BadBlockHowto)によるとそもそもreadできなくなっているはずだがうまくいっていない？  
selftestを走らせても以前read failureのままだった。

仕方ないので、allocateされているファイルを消してみた。  
すると見たところdmesgから`I/O error, dev sda, sector 1852424`がどかどか出てこなくなった。

治したのでもう一度selftestを走らせてみた

```
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed: read failure       90%     16582         2036544
```
次の壊れてるセクターが出てきた...  
もうええわ

## なんで壊れた？
Wear_Leveling_Countの値は99%なので使いすぎによる消耗ではなさそう。  
それもそのはず、まだ2年しか経ってない。  
異常なことが起きたって感じ。  
未だわかりません。  
