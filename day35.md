# IS_CHROME_BRANDED

`is_chrome_branded`というフラグがChromiumにはある。  
自分もなんかargs.gnにいれているが、結局コイツは何？

`is_chrome_branded`がtrueだとinternalのものまで含めたGoogle Chromeになり、falseだとOSSのChromiumになる。  
このバージョンではsrc-internal checkoutを使用する。[doc](https://chromium.googlesource.com/chromium/src/+/main/docs/google_chrome_branded_builds.md)。  
`#if BUILDFLAG(GOOGLE_CHROME_BRANDING)` というifdefでくくられたエリアが採用されるかというのがChromium内での差。

見た目的な違いは、Google Chromeのアイコンはよく見るGoogleカラーの4色のやつ。  
Chromiumはなんか青いやつ。
![](https://hackmd.io/_uploads/B17_V-j_h.png)
引用：https://source.chromium.org/chromium/chromium/src/+/main:chrome/app/theme/chromium/chromeos/chrome_app_icon_192.png

またGoogle Chromeではcrash reportが作成されるがChromiumではされない。User metricsも同様にGoogle Chromeでのみ記録される。  

## resources.grd
Chromiumのアイコンのようなデータはgrd拡張子を持つresourcesファイルで管理されている。  
このアイコンの場合は[`IDR_CHROME_APP_ICON_192`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/app/theme/chrome_unscaled_resources.grd;l=100;drc=cf58016125a32ea03dc8d2a37dfe872380e1155b)と名付けられて外から使われる。  
この中を見ると`_google_chrome`がtrueなら`google_chrome/chromeos/chrome_app_icon_192.png`、falseなら`chromium/chromeos/chrome_app_icon_192.png`をpng fileのロケーションに指定している
これは、各リソースがGoogle Chromeならgoogle_chrome/chromeos directoryに、Chromiumならchromium/chromeos directoryに置かれるから。  
このことからもresourceはruntimeにresource fileから取得している。

## おわり
今週めっちゃ忙しいねん…すまん
