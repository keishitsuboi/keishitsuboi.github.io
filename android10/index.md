# ファイルベースの暗号化（FBE）されたAndroid10のエミュレータの/data/data/領域をPCで復号してみる

[ファイルベースの暗号化](https://source.android.com/security/encryption/file-based?hl=ja)の説明にある通り、Android7.0以降ではファイルベースの暗号化（FBE）がサポートされています。Android10の/data/data/領域（最初のユーザーのアプリストレージ）もFBEで暗号化されています。FBEによる暗号化はスクリーンロック設定の有無に関わらず行われていますが、スクリーンロック設定が無い方が復号に必要な情報が少なくなります。

スクリーンロック設定が無いAndroid10のエミュレータから/data/data/に該当する領域をダンプし、PC（Linux）で復号を試みたところ成功しました。本稿では使用した環境とその手順について記載します。
<!--more-->
## 1.手順の概要
大きく分けて３つの手順を踏みます。

1. Android10のエミュレータから、復号対象のデータをダンプ
2. Android10のエミュレータから、復号用の鍵をダンプ
3. ダンプしたデータと鍵を使用して、PCで復号

2.の手順については、以下の２つの方法があります。詳細については後述します。

  * Androidのソースコードをダウンロードし、その中のvoldのソースコードの一部を改変した上でビルドして実行する方法
  * [Frida](https://frida.re/docs/android/)を使う方法

## 2.使用した環境

  ### 2.1 PC
Arch Linux (5.10.50-1-lts)

  ### 2.2 Androidのエミュレータ
[Android Studio](https://developer.android.com/studio?hl=ja)のAVD Managerから作成しました。

Release Name: Q
API Level: 29
ABI: x86_64
Target: Android10.0(Google APIs)
Hardware: root権限が取得可能なもの（＝Play Storeにチェックが入っていないもの）
スクリーンロック設定: 無し

  ### ビルド環境
DockerのUbuntu 18.04

## コマンドの実行例の表記について
本稿ではいくつかコマンドの実行例を記載します。
複数の環境での実行例を記載するため、実行環境の区別を以下の表記で行います。

|先頭部分の表記|意味|
|----|----|
|$|PC上での実行|
|generic_x86_64:/ #|Android10のエミュレータ上でのroot権限での実行|
|root@263eeed4469e:|DockerのUbuntu18.04での実行|

## 手順1. Android10のエミュレータから、復号対象のデータをダンプ

  ### FBEで暗号化されているか確認

  [ro.crypto.stateとro.crypto.typeの内容をチェックします。](https://source.android.google.cn/security/encryption/file-based?hl=ja#validation)

```
generic_x86_64:/ # getprop | grep crypto
[ro.crypto.state]: [encrypted]
[ro.crypto.type]: [file]
[ro.crypto.volume.filenames_mode]: [aes-256-cts]

```
  ### ダンプ対象の確認
```
generic_x86_64:/ # mount | grep ' /data '                                                                                                                                        
/dev/block/vdc on /data type ext4 (rw,seclabel,nosuid,nodev,noatime,resgid=1065,errors=panic,data=ordered)
```

/dev/block/vdcをダンプすることになります。

  ### データのダンプ
- PCの受け側
```
$ nc -l -p 9999 | gzip -d > ret
```
- Android側
```
generic_x86_64:/ # dd if=/dev/block/vdc | gzip | nc 10.0.2.2 9999

```
[10.0.2.2はPCのIPです。](https://developer.android.com/studio/run/emulator-networking?hl=ja#networkaddresses)

  ### ダンプしたデータの確認
  成功するとretというファイル名のファイルがPC側にできます。サイズは800MBでした。
  ```
  $ file ret
ret: Linux rev 1.0 ext4 filesystem data, UUID=57f8f4bc-ffffabf4-655f-ffffbf67-946fc0f9fffff25b (needs journal recovery) (extents) (large files)
  ```

  ### ダンプしたデータのマウント
  FBEはファイル単位の暗号化のためか、復号用の鍵を知らなくてもマウントすることができました。

  ```
  $ mkdir tmp
$ sudo mount -oloop,ro ret tmp
$ ls tmp/
adb   app-asec       app-staging  dalvik-cache  local       mediadrm  nfc          property           server_configurable_flags  system_de    user_de
anr   app-ephemeral  backup       data          local.prop  misc      ota          resource-cache     ss                         tombstones   vendor
apex  app-lib        bootchart    drm           lost+found  misc_ce   ota_package  rollback           system                     unencrypted  vendor_ce
app   app-private    cache        gsi           media       misc_de   preloads     rollback-observer  system_ce                  user         vendor_de
  ```

  /data/直下はFBEによる暗号化は行われていないようです。

そして、/data/data/に相当する箇所を確認すると、暗号化されていることがわかります。

```
$ ls tmp/data/
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFv+,EJZbQdnYQu,rXx6wTNt6JEB5BEUVfoPVzahC
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFv+65J8t6x7+3YXOzBqRWJk0nZdM39n2JkYkNodB
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFv+oKatth8Nj8ikU7+6YN1+2nZdM39n2JkYkNodB
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFvAYNJohexPAXaBEWJ75Flh8,LLjzQ,UyFg9JQr0o7KvB
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFvJPVfmuqrowGnKw9GraApO0nZdM39n2JkYkNodB
AAAAAAAAAAA,2zZ+TpRQYRLlS6VnXUFvUNayNrjrDFXzDkoOrxMUF+,LLjzQ,UyFg9JQr0o7KvB
（以下省略）
```

## 手順2. Android10のエミュレータから、復号用の鍵をダンプ
次に、復号用の鍵をAndroid10のエミュレータから取得します。ここでは、Androidのソースコードをダウンロードし、その中のvoldのコードの一部を改変した上でビルドして実行する方法について記述します。Fridaを使う方法については後述します。

[ファイルベースの暗号化: 鍵の派生](https://source.android.com/security/encryption/file-based?hl=ja#key-derivation)の中に、以下のような記述があります。

> 512 ビットの鍵であるファイルベースの暗号鍵は、TEE に保持された別の鍵（256 ビット AES-GCM 鍵）で暗号化されてから保存されます。

TEEとは[Trusted Execution Environment](https://source.android.com/security/trusty?hl=ja)のことです。TEEについての説明は割愛しますが、要するに復号用の鍵が単体のファイルとして存在するわけではないということです。

復号用の鍵を手軽に取得する方法はないかと探していたところ、[ファイルベースの暗号化: 例とソース](https://source.android.com/security/encryption/file-based?hl=ja#examples-and-source)に、以下のような記述がありました。

> Android にはファイルベースの暗号化のリファレンス実装が用意されており、vold（system/vold）によって Android のストレージ デバイスとボリュームを管理する機能が提供されます。FBE を追加すると、複数のユーザーの CE 鍵と DE 鍵の管理に対応したいくつかの新しいコマンドが vold に提供されます。


CE鍵とDE鍵については、[ファイルベースの暗号化: ダイレクト ブート](https://source.android.com/security/encryption/file-based?hl=ja#direct-boot)を参照してください。/data/data/領域の復号に必要なのはCE鍵です。上記の記述によると、voldのコードを参照すると復号用の鍵の取得方法が分かりそうです。

そこでvoldのコードを調べたところ、[鍵を取得する関数](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyStorage.h#64)が見つかりました。

```
// Retrieve the key from the named directory.
bool retrieveKey(const std::string& dir, const KeyAuthentication& auth, KeyBuffer* key,
                 bool keepOld = false);
```
第１引数には、鍵に関連するファイルが置いてあるディレクトリを指定します。今回の目的では、/data/misc/vold/user_keys/ce/0/currentを指定することになります。
第２引数には、[鍵の派生](https://source.android.com/security/encryption/file-based?hl=ja#key-derivation)で説明されている情報に関連するデータを渡します。しかし、スクリーンロック設定が無い場合は[kEmptyAuthentication](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyStorage.h#42)を指定するだけで良いのでお手軽です。
第３引数は、この関数の実行結果を格納するバッファです。このデータが今回求めている復号用の鍵そのものです。

関数retrieveKey()を呼び出せば、/data/data/の復号用の鍵を取得できることが分かりました。そこで、Androidのソースコードをダウンロードして、voldのコードをretrieveKey()を呼び出してその結果を保存するように書き換えた上でビルドすることにしました。それをAndroid10のエミュレータ上で実行すれば、復号用の鍵を取得できます。

  ### ソースコードのダウンロード

  詳細は[ソースのダウンロード](https://source.android.com/setup/build/downloading?hl=ja)を参照してください。

  ```
  $ mkdir tmp
$ cd tmp
$ repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r45
$ repo sync
  ```

  ### voldのソースコードの改変
  ダウンロードしたソースコードの system/vold/main.cpp を以下の様に改変します。

  ```
  #include <android-base/file.h>
#include "KeyStorage.h"

int main(int argc, char** argv) {
    if(argc != 3) return -1;

    android::vold::KeyBuffer key;
    android::vold::retrieveKey(argv[1], android::vold::kEmptyAuthentication, &key);
    android::base::WriteStringToFile(std::string(key.data(), key.size()), argv[2], false);
}

  ```

retrieveKey()を実行して、結果（つまり復号用の鍵）をargv[2]で渡されたファイル名で保存して終了するプログラムです。retrieveKey()は実行の成否をbool値で返してくれるようですが、今回は気にしないことにしています。

  ### ビルド
改変したvoldをビルドします。[公式ページ](https://source.android.com/setup/build/initializing?hl=ja)によると、ビルド環境としてUbuntuのバージョンがいくつか挙げられています。今回はDockerのUbuntu 18.04で作業することにしました。本稿ではDockerの導入や使用方法については割愛します。また、ダウンロードしたソースコードもDocker内から参照できるようにmountしておきます。本稿では/mnt/で参照できるようにしています。

まずはDockerを起動します。

```
$ sudo docker run -it -v ダウンロードしたソースコードの絶対パス:/mnt/ ubuntu:18.04
```
次に、必要なパッケージをインストールします。

```
root@263eeed4469e:~# apt update
root@263eeed4469e:~# apt-get install -y git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig python
```

そして、ビルドします。
```
root@263eeed4469e:~# cd /mnt/
root@263eeed4469e:/mnt# export LC_ALL=C
root@263eeed4469e:/mnt# source build/envsetup.sh
root@263eeed4469e:/mnt# lunch aosp_x86_64-eng
root@263eeed4469e:/mnt# cd system/vold/
root@263eeed4469e:/mnt/system/vold# mma
```

手元の環境では、15分程度で完了しました。/mnt/out/target/product/generic_x86_64/system/bin/vold が求めるファイルです。このファイルをAndroid10のエミュレータにコピーします。本物のvoldと混同しない様にファイル名も変更しておきます。

```
$ adb push out/target/product/generic_x86_64/system/bin/vold /data/local/tmp/dump

```

  ### 実行
  ```
  generic_x86_64:/ # cd /data/local/tmp
generic_x86_64:/data/local/tmp # chmod +x dump
generic_x86_64:/data/local/tmp # ./dump /data/misc/vold/user_keys/ce/0/current ce_key                                                                                            
generic_x86_64:/data/local/tmp # xxd -g 1 ce_key                               
00000000: 63 47 c7 01 04 00 27 c4 d5 8f 2b 1b a8 94 9c ed  cG....'...+.....
00000010: a9 65 92 44 fe 25 ea 91 93 25 97 05 e1 41 48 1e  .e.D.%...%...AH.
00000020: ed 52 3c fe 67 83 d5 ca c1 40 fe f3 f3 f1 98 10  .R<.g....@......
00000030: 88 64 46 16 95 69 59 d5 b2 19 ce 72 fc 48 a1 6a  .dF..iY....r.H.j
  ```
復号用の鍵が取得できました。ランダムに生成されているようなので、値は環境ごとに異なります。

## 手順3. ダンプしたデータと鍵を使用して、PCで復号
手順1でマウントはしているので、鍵を登録すれば読めるようになります。

[ファイルベースの暗号化: fscrypt暗号化](https://source.android.com/security/encryption/file-based?hl=ja#fscrypt-encryption)の項目に記載されている通り、fscryptによって暗号化されています。そこで、[fscryptctl](https://github.com/google/fscryptctl)を使って鍵の登録を行います。

本稿ではAndroid10のエミュレータを対象としていますが、[ファイルベースの暗号化: 内部ストレージ](https://source.android.com/security/encryption/file-based?hl=ja#enabling-fbe-on-internal-storage)の項目に記載されている通り、Android10では暗号化ポリシーとしてv1が使用されています。

> v1 フラグを指定するとバージョン 1 の暗号化ポリシーが、v2 フラグを指定するとバージョン 2 の暗号化ポリシーが選択されます。バージョン 2 の暗号化ポリシーでは、安全性と柔軟性に優れた鍵導出関数が使用されます。デフォルトは、Android 11 以降を搭載して出荷されたデバイス（ro.product.first_api_level によって特定される）の場合は v2、Android 10 以前を搭載して出荷されたデバイスの場合は v1 です。

そして、[fscryptctlでは2021/2/3のコミットでv1のサポートが落とされた](https://github.com/google/fscryptctl/commit/d2066cded914860164ffacebc24ea0afc0b57963)ため、その１つ前のコミットを使用することにします。

```
$ git clone https://github.com/google/fscryptctl/
$ cd fscryptctl/
$ git checkout 60812534fc3ecf3a43b67bfeee5c5a1bc7daadc8
$ make
```
そして、鍵を読み込ませます。

```
$ adb pull /data/local/tmp/ce_key
$ ./fscryptctl insert_key --ext4 < ce_key
```

ここで/data/data/に相当する箇所を確認すると、復号できていることが確認できます。成功しました。
```
$ ls tmp/data/
android                                                        com.android.theme.color.orchid
com.android.apps.tag                                           com.android.theme.color.purple
com.android.backupconfirm                                      com.android.theme.color.space
com.android.bips                                               com.android.theme.font.notoserifsource
com.android.bluetooth                                          com.android.theme.icon.roundedrect
（以下省略）
```

## 手順2-2. Fridaを使用して、復号用の鍵をダンプ
以上の手順で、voldのretrieveKey()を呼べば復号用の鍵が取得できることが分かりました。本物のvoldはAndroidの中で動いているため、[Frida](https://frida.re/docs/android/)を使用して本物のvoldの中でretrieveKey()を呼んでも復号用の鍵が取得できるのではないかと考えました。試してみたところ、取得に成功したのでその手順を記載します。Fridaの説明や使い方については[Fridaのサイト](https://frida.re/docs/android/)を参照してください。

本物のvold内のretrieveKey()を呼び出すために、retrieveKey()のアドレスを特定する必要があります。特定するために、Android10のエミュレータから本物のvoldを取得します。

```
$ adb root
$ adb pull /system/bin/vold
$ sha1sum vold
d638e304a741ed6fbb1fa36a989d953208a73bfa  vold
```

取得したvoldを解析すれば、retrieveKey()のアドレスを取得することができます。解析方法については割愛します。関数内で参照している[Version mismatch, expected](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyStorage.cpp#517)という文字列で探せばすぐに見つかります。本稿で取得したvoldでは0x0004c230でした。

retrieveKey()は引数にC++の型が出てくるため、Fridaで実行する場合はここも工夫が必要です。

```
// Retrieve the key from the named directory.
bool retrieveKey(const std::string& dir, const KeyAuthentication& auth, KeyBuffer* key,
                 bool keepOld = false);

```

以下の方針で試したところ、うまくいきました。

第１引数のstd::stringは、[std::string StringPrintf(const char* fmt, ...)](https://android.googlesource.com/platform/system/core/+/android-10.0.0_r45/base/include/android-base/stringprintf.h#29)を使用して生成する。
第２引数は[kEmptyAuthentication](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyStorage.h#42)を指定するので、本物のvoldからkEmptyAuthenticationのアドレスを探して使用する。[voldの元々のコード上にもkEmptyAuthenticationを指定してretrieveKey()を呼び出している箇所がある](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyUtil.cpp#175)ので、retrieveKey()のアドレスが分かっていれば探すのは難しくありません。本稿で取得したvoldでは0x000e8178でした。
第３引数のKeyBufferは[charのvector](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-10.0.0_r45/KeyBuffer.h#51)ですが、ある程度のサイズの0初期化されたメモリを渡してやったところ、うまく動いた（メモリの先頭８バイトの部分に、復号用の鍵の先頭アドレスが入っていた）のでヨシ！としました。

最終的に以下のようなプログラムになりました。

```
let offset_kEmptyAuthentication = 0x000e8178;
let offset_retrieveKey = 0x0004c230;

let find = Module.findExportByName('libbase.so', '_ZN7android4base12StringPrintfEPKcz');
let android_base_StringPrintf = new NativeFunction(find, 'pointer', ['pointer', 'pointer']);
let format = Memory.allocUtf8String('%s');
let dir = android_base_StringPrintf(format, Memory.allocUtf8String('/data/misc/vold/user_keys/ce/0/current'));

let vold = Process.findModuleByName('vold');
let kEmptyAuthentication = (vold.base.add(offset_kEmptyAuthentication));
let keyBuffer = Memory.alloc(1024);
let retrieveKey = new NativeFunction(vold.base.add(offset_retrieveKey), 'bool', ['pointer', 'pointer', 'pointer', 'bool']);
retrieveKey(dir, kEmptyAuthentication, keyBuffer, 0);

let key_addr = keyBuffer.readU64();
let key = (ptr(key_addr)).readByteArray(64);
console.log('key:');
console.log(key);
```
このプログラムをmain.jsというファイル名で保存して、Fridaで実行します。

まずvoldのPIDを取得します

```
$ frida-ps -U | grep vold
1747  vold
```
取得したPIDに対してmain.jsを実行します。

```
$ frida -U -l main.js -p 1747
（中略）
key:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  63 47 c7 01 04 00 27 c4 d5 8f 2b 1b a8 94 9c ed  cG....'...+.....
00000010  a9 65 92 44 fe 25 ea 91 93 25 97 05 e1 41 48 1e  .e.D.%...%...AH.
00000020  ed 52 3c fe 67 83 d5 ca c1 40 fe f3 f3 f1 98 10  .R<.g....@......
00000030  88 64 46 16 95 69 59 d5 b2 19 ce 72 fc 48 a1 6a  .dF..iY....r.H.j

```
手順2で取得した復号用の鍵と一致していることが分かります。

## 今後の課題

- 実機での実行について
        本稿ではAndroid10のエミュレータで実行しましたが、復号用の鍵が取得できれば実機でも同様に復号できるのではないかと考えています。しかし、実機でも復号用の鍵の取得が本稿の方法で取得できるかどうかは、実際に検証する必要があるだろうと考えています。
- スクリーンロック設定がある場合について
        スクリーンロック設定がある場合はretrieveKey()の第２引数に[ファイルベースの暗号化: 鍵の派生](https://source.android.google.cn/security/encryption/file-based?hl=ja#key-derivation)で記載されている情報を詰めてやる必要があるようです。特に１つ目の要件の認証トークンは、voldのコードを軽く読んだ感じでは他のプログラムからvoldに対して渡される情報のようなので、どのプログラムが渡しているのかを調査する必要がありそうです。この検証も、実機を使って検証する必要があるだろうと考えています。
- Android11以降での実行について
        [ファイルベースの暗号化: 内部ストレージ](https://source.android.com/security/encryption/file-based?hl=ja#enabling-fbe-on-internal-storage)に説明がある通り、FBEに関してAndroid11から導入された仕組みが存在し、本稿の実行環境であるAndroid10と違いがあるように見えます。Android11以降での復号についても、今後の課題としたいと思います。


