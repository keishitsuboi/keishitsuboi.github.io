# Kali LinuxでBitLocker ~やってみる編 ~


Windows の BitLocker、Linux ユーザーにはとてもうらやましいですよね。
Trusted Platform Module（TPM）が搭載された PC なら手軽にディスクを暗号化できて、パスフレーズ入力無しで OS 起動できて。
そんな Windows のうらやま機能を Kali Linux でやっていきます。

<!--more-->

## 1 はじめに

技術部の松永です。

2年ほど前にネットワークペネトレーションテストチームへ拉致られて以来、Linux で Windows をやっつけるツールの開発などやっています（楽しい）。
そんな縁もあり、ペネトレーションテストに使用する Linux PC を BitLocker のようにディスク暗号化しつつ自動起動できるようにする必要性が生じ、勉強する機会を得られました。
手順や得られた知識をブログで公開していきたいと思いますが、とりあえずやってみるだけでも 1 日作業になるため今回は手順のみ紹介したいと思います。
やれる環境のある方は是非チャレンジして頂き、やれる環境の無い方は次回以降で手順を解説するのでそちらを見てフムフムして頂けると幸いです。

本稿では Kali Linux を使いますが、Debian 系の他のディストリビューションでも同様の手順で実施できると考えています。
私は Kali Linux の他に Ubuntu（Focal Fossa） も試していますが、複雑な手順が必要だったのが Kali Linux です。
Kali Linux でできるなら他の Debian 系も大丈夫だろうと思いつつ、次回以降でディストリビューションの差異の話もしたいと思います。

### できるようになること
* Kali Linux を LUKS（Linux Unified Key Setup）でディスク暗号化してインストール
* Kali Linux のセキュアブート対応
* ディスク暗号化（LUKS）のパスフレーズを Trusted Platform Module（TPM） 2.0 デバイスへ封入し、パスフレーズ入力無しで Kali Linux を起動

なぜセキュアブート対応するのか？は次回以降での解説とします。

### 必要なもの
* TPM 2.0 を搭載した PC
  * 本稿では ThinkCentre M90n-1 Nano を使いました
* Kali Linux のインストールメディア 
  * 本稿では kali-linux-2020.3-installer-netinst-amd64.iso を USB メモリへ入れて使いました

## 2 Kali Linux のインストール

インストールの前に BIOS（UEFI）セットアップメニューを開き、以下のように設定します。

{{< figure src="/bitlocker/bitlocker1.png" >}}

{{< figure src="/bitlocker/bitlocker2.png" >}}

最近の Linux は Windows よりも簡単にインストールできるので手順は詳しく解説しません。
今回は以下の設定でインストールしましたが、<font color="red">パーティショニングの方法: ガイド - ディスク全体を使い、暗号化 LVM をセットアップする</font>以外の部分はお好みで大丈夫です。
この際設定する LUKS 暗号化パスフレーズはセットアップ時に使用する他、TPM での起動ができなくなった場合にも使用するので安全に保管しておく必要があります。

* 言語: 日本語
* ロケール: 日本
* キーボード: 日本語
* ホスト名: kali
* ドメイン名: (空白)
* ユーザー名: user
* パーティショニングの方法: ガイド - ディスク全体を使い、暗号化 LVM をセットアップする
* パーティショニング機構: すべてのファイルを 1 つのパーティションに（初心者ユーザには推奨）
* インストールするソフトウェア: 全て無し

{{<figure src="/bitlocker/bitlocker3.png">}}

## パッケージのインストール
Kali Linux のインストールが完了したら、設定した暗号化パスフレーズを入力しつつ Kali Linux を起動します。
ここからしばらくコマンドラインの作業になります。
一般ユーザーで実行可能な操作はほぼ無いため、以降の作業は全て root ユーザーで実行してください。
コピペで実行しやすいよう、# や $ は付けずに表記します。

ソースパッケージを有効化しつつ、必要なパッケージをインストールします。

```
grep '^deb ' /etc/apt/sources.list | sed 's/^deb /deb-src /g' | tee /etc/apt/sources.list.d/deb-src.list
apt-get update
apt-get install -y tpm2-tools tpm2-abrmd efibootmgr efitools binutils dosfstools uuid-runtime
apt-get install -y build-essential flex bison bc rsync libelf-dev libssl-dev libncurses-dev
```
ここで一度、TPM2 デバイスへアクセスできるか確認します。

```
tpm2_pcrread sha256
```
以下のように TPM2 の PCR レジスタが表示されたら成功です（0-7の値はXでマスクしてます）。

```
sha256:
  0 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  1 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  2 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  3 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  4 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  5 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  6 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  7 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  8 : 0x0000000000000000000000000000000000000000000000000000000000000000
  9 : 0x0000000000000000000000000000000000000000000000000000000000000000
  10: 0x0000000000000000000000000000000000000000000000000000000000000000
  11: 0x0000000000000000000000000000000000000000000000000000000000000000
  12: 0x0000000000000000000000000000000000000000000000000000000000000000
  13: 0x0000000000000000000000000000000000000000000000000000000000000000
  14: 0x0000000000000000000000000000000000000000000000000000000000000000
  15: 0x0000000000000000000000000000000000000000000000000000000000000000
  16: 0x0000000000000000000000000000000000000000000000000000000000000000
  17: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  18: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  19: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  20: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  21: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  23: 0x0000000000000000000000000000000000000000000000000000000000000000

```

## パーティションの確認
lsblk コマンドで Kali Linux インストール後のパーティションの状態を確認します。
本稿で確認に使用した PC のストレージは NVMe だったため、以下のようになっています。
ハードディスクや SSD の場合は sda1 など別の結果が表示され、以降のコマンドで適宜読みかえが必要になるのでご注意ください。

`
lsblk
`
```
NAME                  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
nvme0n1               259:0    0   477G  0 disk
├─nvme0n1p1           259:2    0   512M  0 part  /boot/efi
├─nvme0n1p2           259:3    0   244M  0 part  /boot
└─nvme0n1p3           259:4    0 476.2G  0 part
  └─nvme0n1p3_crypt   254:0    0 476.2G  0 crypt
    ├─kali--vg-root   254:1    0 460.4G  0 lvm   /
    └─kali--vg-swap_1 254:2    0  15.9G  0 lvm   [SWAP]

```
以降は下記のパーティション構成を前提として記載します。

- Kali Linux をインストールしたディスク: /dev/nvme0n1
- EFIシステムパーティション（ESP）: /dev/nvme0n1p1
- ブートパーティション: /dev/nvme0n1p2
- LUKS 暗号化パーティション: /dev/nvme0n1p3

## Linuxカーネルのビルド

Kali Linux をセキュアブート対応するには、Linux カーネルのビルドが必要になります。
ビルドはとても時間がかかります。
ここは環境による差分が無いので、コマンドをそのまま貼り付けて実行して大丈夫です。
以下のコマンドを実行してビルドが始まったら、次のステップへ進みます。

```
mkdir /root/linux-image
chown -R _apt:root /root/linux-image
cd /root/linux-image
apt-get source linux-image-amd64
cd linux-*

cd certs
cat <<EOS > x509.genkey
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts

[ req_distinguished_name ]
CN = Modules

[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOS

openssl req -new -nodes -utf8 -sha256 -days 36500 -batch -x509 -config x509.genkey -outform PEM -out kernel_key.pem -keyout kernel_key.pem
cd ..
cp /boot/config-$(uname -r) .config
make olddefconfig
sed -i.bak -e 's/.*CONFIG_MODULE_SIG_ALL\W.*/CONFIG_MODULE_SIG_ALL=y/' \
  -e 's#.*CONFIG_MODULE_SIG_KEY\W.*#CONFIG_MODULE_SIG_KEY="certs/kernel_key.pem"#' \
  -e 's/.*CONFIG_SYSTEM_TRUSTED_KEYS\W.*/CONFIG_SYSTEM_TRUSTED_KEYS=""/' \
  -e 's/.*CONFIG_CRYPTO_SHA512\W.*/CONFIG_CRYPTO_SHA512=y/' \
  -e 's/.*CONFIG_DEBUG_INFO\W.*/CONFIG_DEBUG_INFO=n/' .config

: | make -j $(nproc) bindeb-pkg


```

## 起動スクリプトの作成

Kali Linux 起動時に TPM2 から LUKS 暗号化パスフレーズを開封し、LUKS 暗号化されたパーティションを自動的にオープンするスクリプトを作成します。
スクリプトや必要なモジュールを initramfs へコピーするフックスクリプト、crypttab でスクリプトが実行されるよう準備しておきます。
ここは環境による差分が無いので、コマンドをそのまま貼り付けて実行して大丈夫です。

```
cat <<EOS > /usr/local/bin/tpm2_unseal_lukskey
#!/bin/sh
tpm2_unseal -c 0x81000000 -p pcr:sha256:0,2,4,7
if [ \$? -ne 0 ]; then
  /lib/cryptsetup/askpass "Unlocking the disk fallback \$CRYPTTAB_SOURCE (\$CRYPTTAB_NAME). Enter passphrase: "
fi
EOS

cat <<EOS > /etc/initramfs-tools/hooks/tpm2-initramfs-hook
. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/lib/x86_64-linux-gnu/libtss2-tcti-device.so.0.0.0
copy_exec /usr/bin/tpm2_unseal
copy_exec /usr/local/bin/tpm2_unseal_lukskey

copy_modules_dir kernel/drivers/char/tpm
EOS

chmod +x /usr/local/bin/tpm2_unseal_lukskey
chmod +x /etc/initramfs-tools/hooks/tpm2-initramfs-hook


cp /etc/crypttab{,.bak}
echo $(cut -d' ' -f 1,2 /etc/crypttab) unseal $(cut -d' ' -f 4 /etc/crypttab ),keyscript=/usr/local/bin/tpm2_unseal_lukskey > /etc/crypttab
```
## セキュアブート鍵の作成
セキュアブートに使用するを作成しておきます。
ここは環境による差分が無いので、コマンドをそのまま貼り付けて実行して大丈夫です。

```
mkdir /root/secureboot
cd /root/secureboot

uuidgen --random > GUID.txt
openssl req -newkey rsa:4096 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=Platform Key/" -out PK.crt
openssl x509 -outform DER -in PK.crt -out PK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" PK.crt PK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt PK PK.esl PK.auth
sign-efi-sig-list -g "$(< GUID.txt)" -c PK.crt -k PK.key PK /dev/null rm_PK.auth
openssl req -newkey rsa:4096 -nodes -keyout KEK.key -new -x509 -sha256 -days 3650 -subj "/CN=Key Exchange Key/" -out KEK.crt
openssl x509 -outform DER -in KEK.crt -out KEK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" KEK.crt KEK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt KEK KEK.esl KEK.auth
openssl req -newkey rsa:4096 -nodes -keyout db.key -new -x509 -sha256 -days 3650 -subj "/CN=Signature Database key/" -out db.crt
openssl x509 -outform DER -in db.crt -out db.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" db.crt db.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k KEK.key -c KEK.crt db db.esl db.auth

```

## /boot パーティションの削除

本稿の手順で Linux を起動するようにすると、/boot 用のパーティションが不要になります（ディストリビューションによっては、最初からESPが /boot だったりしますよね）。
セキュリティ上の理由から、このパーティションは削除します。
ここは環境による差分が無いので、コマンドをそのまま貼り付けて実行して大丈夫です。

```
umount /boot/efi
mkdir /root/boot_backup
mv /boot/* /root/boot_backup
umount /boot
sed -i.bak -e '/\/boot / s/^#*/#/' -e 's#/boot/efi#/efi#' /etc/fstab
mkdir /efi
mount -a
```

## KeyToolの準備
セキュアブート鍵を UEFI ファームウェアへ登録するツールの KeyTool の使用準備をしておきます。
本稿の手順では、このパーティションを完全にお掃除する前に、一時的にセキュアブート鍵を置くストレージとして使用します。
手順としては USB メモリなどリムーバブルメディアを使ったほうがさらに安全性が高まります。

<font color="red">ここは環境による差分があるので注意してください。</font>

```
mkfs.vfat -n SBKEY /dev/nvme0n1p2
mkdir /tmp/sbkey
mount /dev/nvme0n1p2 /tmp/sbkey
cp /root/secureboot/{PK.auth,KEK.esl,db.esl} /tmp/sbkey

mkdir /efi/EFI/KeyTool
cp /usr/lib/efitools/x86_64-linux-gnu/KeyTool.efi /efi/EFI/KeyTool
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label KeyTool --loader EFI/KeyTool/KeyTool.efi

```
最後のコマンドの実行結果例です。
PC 起動時、kali 以外に KeyTool も起動できるようになりました。

```
BootCurrent: 0000
Timeout: 1 seconds
BootOrder: 0002,0000,000B,000C,0009,000A,0007,0006,0001
Boot0000* kali
Boot0001* Windows Boot Manager
Boot0006* Generic Usb Device
Boot0007* CD/DVD Device
Boot0009  UEFI: PXE IPV4 Network Card
Boot000A  UEFI: PXE IPV6 Network Card
Boot000B* UEFI: PXE IPV4 Intel(R) Ethernet Connection (6) I219-LM
Boot000C* UEFI: PXE IPV6 Intel(R) Ethernet Connection (6) I219-LM
Boot0002* KeyTool
```

## Unified Kernel Image の作成
手順の最初の方でやり始めた Linux カーネルのビルドは無事終わりましたでしょうか。
終わってないと次の手順へは進めないので、DARKMATTER の他の記事を読んで有意義な時間過ごすことをオススメします。

ビルドが終わっていたら、/root/linux-image に以下のような 3 つの deb パッケージファイルが作成されていると思います。

- linux-headers-5.8.14_5.8.14-1_amd64.deb
- linux-libc-dev_5.8.14-1_amd64.deb
- linux-image-5.8.14_5.8.14-1_amd64.deb

これらをインストールしつつ、Unified Kernel Image を作成・署名し、KeyTool と同じく EFI ブートエントリーへ登録します。

<font color="red">ここは環境による差分があるので注意してください。パーティションとカーネルリリース番号の部分です。</font>

```
dpkg -i /root/linux-image/linux-*.deb

echo "$(grep -oP 'root=\S+' /proc/cmdline) ro cryptdevice=$(cut -d\  -f2 /etc/crypttab):$(cut -d\  -f1 /etc/crypttab) quiet" > /root/secureboot/cmdline.txt

mkdir /efi/EFI/Secure-Kali
objcopy \
  --add-section .osrel=/usr/lib/os-release --change-section-vma .osrel=0x20000 \
  --add-section .cmdline=/root/secureboot/cmdline.txt --change-section-vma .cmdline=0x30000 \
  --add-section .linux=/boot/vmlinuz-5.8.14 --change-section-vma .linux=0x40000 \
  --add-section .initrd=/boot/initrd.img-5.8.14 --change-section-vma .initrd=0x3000000 \
  /usr/lib/systemd/boot/efi/linuxx64.efi.stub /efi/EFI/Secure-Kali/Secure-Kali.efi
sbsign --key /root/secureboot/db.key --cert /root/secureboot/db.crt --output /efi/EFI/Secure-Kali/Secure-Kali.efi /efi/EFI/Secure-Kali/Secure-Kali.efi

efibootmgr --disk /dev/nvme0n1 --part 1 --create --label Secure-Kali --loader EFI/Secure-Kali/Secure-Kali.efi

```
最後のコマンドの実行結果例です。
PC 起動時、作成した Unified Kernel Image も起動できるようになりました（一番下）。

```
BootCurrent: 0000
Timeout: 1 seconds
BootOrder: 0003,0002,0000,000B,000C,0009,000A,0007,0006,0001
Boot0000* kali
Boot0001* Windows Boot Manager
Boot0002* KeyTool
Boot0006* Generic Usb Device
Boot0007* CD/DVD Device
Boot0009  UEFI: PXE IPV4 Network Card
Boot000A  UEFI: PXE IPV6 Network Card
Boot000B* UEFI: PXE IPV4 Intel(R) Ethernet Connection (6) I219-LM
Boot000C* UEFI: PXE IPV6 Intel(R) Ethernet Connection (6) I219-LM
Boot0003* Secure-Kali
```

## セキュアブートのモードをセットアップモードへ

さて、ここまで来たら全体の作業の 2/3 程度は完了です。
ここからは UEFI（BIOS）セットアップメニューと KeyTool の操作になります。
UEFI（BIOS）セットアップメニューの操作は PC 環境依存となりますが、参考までに画像を使って説明します。

以下の Kali Linux を終了し、UEFI（BIOS）セットアップメニューを開きます。

```
systemctl reboot --firmware-setup
```

セキュアブートを有効化します。
{{<figure src="/bitlocker/bitlocker4.png">}}

マイクロソフトのセキュアブート鍵を削除し、セットアップモードに入ります。
後で Windows を入れ直したい方は、ファクトリーキーのリストア（図中だと Restore Factory Keys）が可能なことを確認してから進んでください。
削除したセキュアブート鍵を戻せなくなる可能性があります。

また、セットアップモードへ入る操作が無い UEFI ファームウェアの場合は、手動でセキュアブート鍵を削除する必要があります。

{{<figure src="/bitlocker/bitlocker5.png">}}

セットアップモードになったところです。

{{<figure src="/bitlocker/bitlocker6.png">}}

設定を保存して再起動します。

{{<figure src="/bitlocker/bitlocker7.png">}}

## セキュアブート鍵の登録

KeyTool を使ってセキュアブート鍵を UEFI ファームウェアへ登録します。
起動デバイスの選択画面（PC 起動時に F12 キーなど）を表示し、 KeyTool を選択して起動します。

## セキュアブートのモード確認

UEFI（BIOS）セットアップメニューを開き、モードが`Setup Mode`から`User Mode`へと切り替わったことを確認します。

## カスタムカーネルの起動
PC を再起動すると、カスタムカーネル（署名した Unified Kernel Image）の起動が始まります。
以下のようなエラーメッセージとともにパスフレーズの入力を求められるので、設定した暗号化パスフレーズを入力してカスタムカーネルを起動します。

以下のようなエラーが表示された場合、セキュアブート鍵の作成、カスタムカーネルの署名、セキュアブート鍵の登録のいずれかがうまくいっていないと考えられます。
原理的な理解が無いと、どこからやり直せばいいのか判断が難しいです。
一旦セキュアブートを無効化して起動できるようであれば試行錯誤してみて、駄目そうなら最初からやり直してください。。

```
Secure Boot Violation
Invalid signature detected.
Check Secure Boot Policy Setup.
```

上記エラーは表示されないがパスフレーズの入力を求めるところまで行かないという場合、起動時のログを多めに出さないと何が失敗してるのか判断が難しいです。
一旦セキュアブートを無効化して起動できるようであれば、cmdline（/root/secureboot/cmdline.txt）へ `loglevel=9` などと書き、Unified Kernel Image の作成で実施した `objcopy` と `sbsign` をやり直し、セキュアブートを有効化して再起動します。
多くの場合、カーネルモジュールの署名や検証の問題が起きていると思われます。
Linux カーネルのビルドあたりからやり直すと問題が解決する可能性がありますが、最初からやり直したほうがスッキリするかもしれません。。

## TPM2 へ LUKS パスフレーズを封入

無事ここまで到達されたみなさまは、大変おめでとうございます。
いよいよ本題の Kali Linux で BitLocker ぽいことができます。
設定したものとは異なるランダムな LUKS パスフレーズを追加後、TPM2 の不揮発性メモリ（NVRAM）へ同じパスフレーズを封入します。
再起動後、ディスク暗号化のパスフレーズ入力を求められることなく Kali Linux が起動したら成功です。

<font color="red">ここは環境による差分があるので注意してください。</font>

```
umask 0077
head -c 128 /dev/urandom > /root/luks_passphrase.bin
cryptsetup luksAddKey /dev/nvme0n1p3 /root/luks_passphrase.bin

mkdir /tmp/tpm2
cd /tmp/tpm2
tpm2_evictcontrol --hierarchy o --object-context 0x81000000
tpm2_pcrread --output /root/pcrs.bin sha256:0,2,4,7
tpm2_createpolicy --policy-pcr --policy pcr.policy --pcr-list sha256:0,2,4,7 --pcr /root/pcrs.bin
tpm2_createprimary --hierarchy=o --key-algorithm=ecc --key-context=prim.ctx
tpm2_create --hash-algorithm sha256 --public seal.pub --private seal.priv --sealing-input /root/luks_passphrase.bin --parent-context prim.ctx --policy pcr.policy
tpm2_load --parent-context prim.ctx --public seal.pub --private seal.priv --key-context seal.ctx
tpm2_evictcontrol --hierarchy o --object-context seal.ctx 0x81000000
tpm2_flushcontext --saved-session

systemctl reboot
```

## お掃除
不要なファイルや EFI エントリーを削除、/boot だったパーティションを削除します。
システムをセキュアに保つために重要な作業です。

<font color="red">ここは環境による差分があるので注意してください。</font>

不要な EFI バイナリ削除
```
rm -rf /efi/EFI/KeyTool
rm -rf /efi/EFI/kali
```
不要な EFI エントリーの番号を取得
```
efibootmgr | egrep 'kali|KeyTool'
```
本稿の環境では `0000` および `0002` でした。
```
Boot0000* kali
Boot0002* KeyTool
```
これらを削除。
```
efibootmgr --delete-bootnum --bootnum 0000
efibootmgr --delete-bootnum --bootnum 0002
```
/boot だったパーティションへランダムなデータを上書きして削除
```
shred /dev/nvme0n1p2
```
## セキュアブート状態の確認
```
# od --address-radix=n --format=u1 /sys/firmware/efi/efivars/SecureBoot-*
   6   0   0   0   1 <= 最後の数字が 1 だったらセキュアブート

# bootctl status
systemd-boot not installed in ESP.
No default/fallback boot loader installed in ESP.
System:
     Firmware: UEFI 2.70 (American Megatrends 5.13)
  Secure Boot: enabled <= セキュアブートが有効
   Setup Mode: user <= User Modeであることが確認できる

Current Boot Loader:
      Product: n/a
     Features: ✗ Boot counting
               ✗ Menu timeout control
               ✗ One-shot menu timeout control
               ✗ Default entry control
               ✗ One-shot entry control
               ✗ Support for XBOOTLDR partition
               ✗ Support for passing random seed to OS
               ✓ Boot loader sets ESP partition information
         Stub: systemd-stub 245.6-1
          ESP: /dev/disk/by-partuuid/93a0e4c0-9adb-42c7-81fc-5f4fbd5b63d0
         File: └─EFI/Secure-Kali/Secure-Kali.efi

Random Seed:
 Passed to OS: no
 System Token: not set
       Exists: no

Available Boot Loaders on ESP:
          ESP: /efi (/dev/disk/by-partuuid/93a0e4c0-9adb-42c7-81fc-5f4fbd5b63d0)

Boot Loaders Listed in EFI Variables:
        Title: Secure-Kali
           ID: 0x0003
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/93a0e4c0-9adb-42c7-81fc-5f4fbd5b63d0
         File: └─EFI/Secure-Kali/Secure-Kali.efi

Boot Loader Entries:
        $BOOT: /efi (/dev/disk/by-partuuid/93a0e4c0-9adb-42c7-81fc-5f4fbd5b63d0)

0 entries, no entry could be determined as default.
```
## おわりに
おつかれさまでした。
最後までうまくいった方はおめでとうございます。
うまくいかなかった方は次回以降の解説でヒントがつかめると幸いです。

これよりも難しいことをボタンぽちぽちでよろしくやってくれる BitLocker は本当にすごいですね。



