# ChromeOS device ownership

ChromeOSにはOwnershipというコンセプトがある。  
多くの場合ChromeOSのOwnerはログインしている自分自身だが、会社や学校など組織で使用している場合は挙動が違う。  

## ChromeOSのOwnershipとは
現在ログインしているアカウントとは別。  
初めてデバイスを使用する際OOBE(**o**ut-**o**f-the-**b**ox **e**xperience)と呼ばれる、基本設定やアカウント情報などの初期化を行う画面が出てくるが、ここでログインしたアカウントがデバイスのOwnerとなる。  
最初のログインアカウントと違うアカウントでログインしていても、現在のアカウントはOwnerにはならない。（ログイン中のアカウントはPrimaryと呼ばれる）  
Ownerの変更にはデバイスの初期化が必要。  


Ownershipの取得状態は[GetOwnershipStatus](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/settings/device_settings_service.cc;l=179;drc=0340dd86c6c1f01ccd1a0b983633cfef5fcf6697)で確認でき、取得が完了したら[`AccountId`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/login/users/chrome_user_manager_impl.cc;l=689;drc=0340dd86c6c1f01ccd1a0b983633cfef5fcf6697)としてOwnerが保存される。

LacrosのLaunchはPolicyを変える可能性があるOwnerが定まるまで待つ必要がある。

## private/public key
ChromeOSの設定には鍵が使われている。
例えばDeviceSettingsServiceは[`public_key_`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/settings/device_settings_service.cc;l=353;drc=a5901556113ab19f130a21acb64c5ec9cc749d98)をscoped_refptrとして持っている。

公開鍵はどのユーザーでもアクセスすることができ、デバイスの設定を認証するために使われる。  
[ValidateDeviceSettings](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/settings/session_manager_operation.cc;l=146;drc=0340dd86c6c1f01ccd1a0b983633cfef5fcf6697)の中で[ValidateSignature](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/settings/session_manager_operation.cc;l=187;drc=0340dd86c6c1f01ccd1a0b983633cfef5fcf6697)を呼んでいるコードが見つかった。  
[CouldPolicyValidatotrBase::CheckSignature](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/policy/core/common/cloud/cloud_policy_validator.cc;l=468;drc=0340dd86c6c1f01ccd1a0b983633cfef5fcf6697)の中では`em::PolicyFetchRequest::SHA1_RSA`が見つかった。暗号っぽい  

一方秘密鍵はデバイスの設定に署名するために使われる。なのでOwnerしかアクセスできない。

それを踏まえてユーザー向けと組織向けの違いを見る。

## Consumer用
一般的に使われているやつ。  
シングルユーザーにOwnされていることを指す。  
最初に作られたユーザーがOwnerになる。  
ChromeoSは初期化時に上記の公開鍵・秘密鍵のペアを[login_manager::keygen::GenerateKey](https://source.chromium.org/chromiumos/chromiumos/codesearch/+/main:src/platform2/login_manager/keygen_worker.cc;l=29;drc=3a446f27a00fba076a5a42d383ae6134c333ee57)などから生成する。  
Chromeは署名されたデバイス設定をsession managerに送ってディスク上に保存する。また公開鍵も別々にsession managerが保有し、すべてのユーザーが閲覧できるようになっている。  
公開鍵が消滅した場合はsession managerがrestoreしてくれるらしい。秘密鍵が消滅した場合は現在のユーザーをOwnerとし新しい鍵のペアと署名したデバイス設定を生成することができる。

## Enterprise用
Consumerとは対象に組織に所有されているデバイス。  
Enterprise enrollmentというOOBEパスを通る。  
ChromeOSの場合はログイン画面でCNTL+ALT+E とすると企業向けログインページに行ける。(UI上からでもいける)  
こちらでは鍵のペアはサーバー上で管理され、端末のユーザーが受け取れるのは公開鍵とデバイスポリシーのみ。  
session managerがサーバーから得たデバイスポリシーを元にcunsumerと同様にセッティングをしてくれる。
