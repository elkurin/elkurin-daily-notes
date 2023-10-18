# Chromium Daily Blog

**Migrated from https://hackmd.io/@elkurin/BkpclhzOh**

This is a study blog updated on daily-basis written by elkurin.  
Most of them are based on [Chromium](https://source.chromium.org/chromium/chromium/src), and there are some random off-topic notes.  
Notes are my own, and do not represent any organization.  
Some of the posts are in English and others are in Japanese.  
I appreciate any feedback or corrections that you may have.

Chromiumブログです。主にChromiumについて話し、たまにC++の仕様について、稀に僕の趣味について書きます。  
日本語と英語が混在しています。間違いを見つけたりフィードバックがあったりする場合は言っていただけると喜びます。

## About me
Hi, I'm elkurin, a ChromeOS engineer creating browser.  
I had been working on: Graphics, Window Manager, Browser Loading/Launching, Memory Performance, Text Input, WebWorker.

## Table of Contents
Day 1: [Compositor Lock](/docs/day1.md)  
Day 2: [Chromium の CHECK macro](/docs/day2.md)  
Day 3: [ChromeOS compositing - How Frame is Sent](/docs/day3.md)  
Day 4: [ChromeOS compositing - How Frame is Produced](/docs/day4.md)  
Day 5: [Task Runner in Chromium](/docs/day5.md)  
Day 6: [Recreating Layer](/docs/day6.md)  
Day 7: [Configure-Ack](/docs/day7.md)  
Day 8: [Exo Surface Hierarchy](/docs/day8.md)  
Day 9: [Chromium の singleton](/docs/day9.md)  
Day 10: [ChromiumのCallback](/docs/day10.md)  
Day 11: [Hardware Overlay and Overlay Processor](/docs/day11.md)  
Day 12: [Delegated Compositing](/docs/day12.md)  
Day 13: [RunLoop](/docs/day13.md)  
Day 14: [Dangling Pointer](/docs/day14.md)  
Day 15: [Exo Window Coordinates](/docs/day15.md)  
Day 16: [raw_ptr in Chromium](/docs/day16.md)  
Day 17: [scoped_refptr in Chromium](/docs/day17.md)  
Day 18: [Chromium Window Destruction Flow](/docs/day18.md)  
Day 19: [WeakPtr in Chromium](/docs/day19.md)  
Day 20: [base::Unretained](/docs/day20.md)  
Day 21: [Widget to FrameSink](/docs/day21.md)  
Day 22: [デフォルト引数のoverride](/docs/day22.md)  
Day 23: [Display in Chromium](/docs/day23.md)  
Day 24: [Wayland Library - How message is called by Client](/docs/day24.md)  
Day 25: [Wayland Library - How message is sent from Server + alpha](/docs/day25.md)  
Day 26: [CrashKey](/docs/day26.md)  
Day 27: [Client and Non-Client views](/docs/day27.md)  
Day 28: [Sequence Checker in Chromium](/docs/day28.md)  
Day 29: [TaskEnvironment in Chromium](/docs/day29.md)  
Day 30: [Chromium Component Build](/docs/day30.md)  
Day 31: [Shadow and gfx library](/docs/day31.md)  
Day 32: [Immersive Mode on ChromeOS](/docs/day32.md)  
Day 33: [pragma pack](/docs/day33.md)  
Day 34: [Mariage Frères の Tea Club に参加してきたよ](https://elkurin.hatenablog.com/entry/2023/06/28/235656)  
Day 35: [IS_CHROME_BRANDED](/docs/day35.md)  
Day 36: [Views Hierarchy and Ownership](/docs/day36.md)  
Day 37: [Frame Eviction](/docs/day37.md)  
Day 38: [Surface Augmenter](/docs/day38.md)  
Day 39: [Chromium Maps](/docs/day39.md)  
Day 40: [base::flat_map + 戻り値の型推論](/docs/day40.md)  
Day 41: [はじめましての標準ライブラリたち Part1](/docs/day41.md)  
Day 42: [特殊化templateの順序](/docs/day42.md)  
Day 43: [Graphic Pipeline on Chrome Compositor (cc)](/docs/day43.md)  
Day 44: [base::small_map](/docs/day44.md)  
Day 45: [Resource Pool](/docs/day45.md)  
Day 46: [Scoped Observer](/docs/day46.md)  
Day 47: [LRU Cache in Chromium](/docs/day47.md)  
Day 48: [cc layers](/docs/day48.md)  
Day 49: [Clipping in cc](/docs/day49.md)  
Day 50: [ObserverList in Chromium](/docs/day50.md)  
Day 51: [Version in Chromium](/docs/day51.md)  
Day 52: [File Enumerator](/docs/day52.md)  
Day 53: [FF16をクリアした感想](https://elkurin.hatenablog.com/entry/2023/07/17/232558)  
Day 54: [Dragging in Chrome](/docs/day54.md)  
Day 55: [Pixel and dip coordinates](/docs/day55.md)  
Day 56: [Browser Test のマクロ](/docs/day56.md)  
Day 57: [Browser Manager](/docs/day57.md)  
Day 58: [Wayland version](/docs/day58.md)  
Day 59: [C++ のキーワードたち](/docs/day59.md)  
Day 60: [パソコンが壊れた](/docs/day60.md)  
Day 61: [またパソコンが壊れた](/docs/day61.md)  
Day 62: [Draw Properties in cc](/docs/day62.md)  
Day 63: [absl::optional](/docs/day63.md)  
Day 64: [absl::optional をもっと読もう](/docs/day64.md)  
Day 65: [std::optional vs absl::optional](/docs/day65.md)  
Day 66: [Context Menu View](/docs/day66.md)  
Day 67: [ChromiumでBanされているやつら](/docs/day67.md)  
Day 68: [ChromeOS device ownership](/docs/day68.md)  
Day 69: [テンプレート勉強会 Part1](/docs/day69.md)  
Day 70: [完☆全☆転☆送](/docs/day70.md)  
Day 71: [Graphics Overview on Chromium](/docs/day71.md)  
Day 72: [absl::flat_hash_mapの概要](/docs/day72.md)  
Day 73: [SurfaceId and synchronization](/docs/day73.md)  
Day 74: [absl::flat_hash_map を読んでみよう](/docs/day74.md)  
Day 75: [Curiously Recurring Template Pattern](/docs/day75.md)  
Day 76: [BrowserManager state transition](/docs/day76.md)  
Day 77: [Wayland messages and Flush](/docs/day77.md)  
Day 78: [atexit in Chromium](/docs/day78.md)  
Day 79: [TabletState in Chromium](/docs/day79.md)  
Day 80: [absl::InlinedVector](/docs/day80.md)  
Day 81: [String Conversion in Chromium](/docs/day81.md)  
Day 82: [WindowTargeter](/docs/day82.md)  
Day 83: [PathService in Chromium](/docs/day83.md)  
Day 84: [Decoration in Chromium Wayland](/docs/day84.md)  
Day 85: [base::expected in Chromium](/docs/day85.md)  
Day 86: [ScopedTempDir in Chromium](/docs/day86.md)  
Day 87: [Surface Origin during Resizing](/docs/day87.md)  
Day 88: [CallbackList in Chromium](/docs/day88.md)  
Day 89: [Frame Activation and Deadline](/docs/day89.md)  
Day 90: [アーキテクチャごとのコンパイラ最適化フラグ](/docs/day90.md)  
Day 91: [グノーシアをクリアした感想](https://elkurin.hatenablog.com/entry/2023/08/25/010617)  
Day 92: [Window Tree in Views](/docs/day92.md)  
Day 93: [文字のイテレータ in Chromium](/docs/day93.md)  
Day 94: [BUILDFLAG](/docs/day94.md)  
Day 95: [SequenceTaskRunner の Delete task](/docs/day95.md)  
Day 96: [Decoration Insets in Linux](/docs/day96.md)  
Day 97: [配列とポインタの違い](/docs/day97.md)  
Day 98: [Hit Testing in Exo](/docs/day98.md)  
Day 99: [Coordinates Conversion in Layer Tree](/docs/day99.md)  
Day 100: [CHECK_IS_TEST](/docs/day100.md)  
Day 101: [Input Event path](/docs/day101.md)  
Day 102: [Ash Debug Shortcuts](/docs/day102.md)  
Day 103: [人狼TLPTを布教したい](https://elkurin.hatenablog.com/entry/2023/09/06/004514)  
Day 104: [CommandLine in Chromium](/docs/day104.md)  
Day 105: [LinkedList in Chromium](/docs/day105.md)  
Day 106: [はじめましての標準ライブラリたちPart2](/docs/day106.md)  
Day 107: [WaylandEventWatcher](/docs/day107.md)  
Day 108: [DeviceOwnership](/docs/day108.md)  
Day 109: [波括弧初期化](/docs/day109.md)  
Day 110: [LayoutManager](/docs/day110.md)  
Day 111: [Even in Chromium](docs/day111.md)  
Day 112: [User Type in ChromeOS and Lacros Launching](/docs/day112.md)  
Day 113: [HUD Display](/docs/day113.md)  
Day 114: [errno を ChromiumのLOGで出力するには](/docs/day114.md)  
Day 115: [base::debug::Alias](/docs/day115.md)  
Day 116: [ebuild dependency](/docs/day116.md)  
Day 117: [ProfileManager and UserManager in Ash](/docs/day117.md)  
Day 118: [StrCat in Chromium](/docs/day118.md)  
Day 119: [BrowserMain Initialization](/docs/day119.md)  
Day 120: [KeyedService and KeyedServiceFactory](/docs/day120.md)  
Day 121: [Q3に読んだ本](https://elkurin.hatenablog.com/entry/2023/10/05/213823)  
Day 122: [Immersive Mode Handling](/docs/day122.md)  
Day 123: [UI Property](/docs/day123.md)  
Day 124: [GURL](/docs/day124.md)  
Day 125: [Wayland Server Initialization](/docs/day125.md)  
Day 126: [LTOでのインライン展開](/docs/day126.md)  
Day 127: [Border Paint on Linux](/docs/day127.md)  
Day 128: [Button view](/docs/day128.md)  
Day 129: [SSE](/docs/day129.md)  

