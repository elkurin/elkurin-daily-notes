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

