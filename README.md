# Android6.0ビルド手順

Android10が出ているなか、4世代前のOSバージョンのビルド手順を連携します。  
とはいえで、ただビルドするだけだとつまらないので、今回はNexus5に焼くところまでを  
連携できればと思っています。  

これが可能になれば、自作カスタムOS開発の第一歩を踏むことができますので、  
一度目を通していただければと思います。  

なお、今回Nexus5へのOS導入を目指すので、  
端末のブートローダーのUnlockはあらかじめ済ませてください。  

## 実機（仮想）環境構築
仮想環境でもなんでもいいので、以下の環境を構築してください。  
まあ、実機が良いことに越したことはございません。  
とにかく、ビルドにはメモリとストレージの容量は多めにしておきましょう。  

|  |  |  
|:-:|:-:|
| OS | Ubuntu14.04(64bit) |
| CPU | Corei7（そのうち4コア利用） |
| メモリ | 16GB |
| ストレージ容量 | 300GB |

自分の場合、HDDを300GB確保しておりましたが、最終的に半分無くなりました・・・。  
CPUはなるべく性能の良いのがおすすめです。  
また、ストレージはSSDのほうがビルドも早くなります。  

OSは以下より入手可能です。  
https://www.ubuntulinux.jp/News/ubuntu1404-ja-remix

## ビルド環境構築
1. 「apt-get」を最新にする
    ```
    $ sudo apt-get update
    ```

2. JDKを入れる  
本件に必要なJDKのバージョンは７になります。  
    ```
    $ sudo apt-get install openjdk-7-jdk
    ```

3. ビルドに必要なライブラリのインストール
Ubuntuのバージョンごとに入れるライブラリが変わりますが、  
14.04は以下の２つのコマンドでOKです。  
    ```
    $ sudo apt-get install g++-multilib
    $ sudo apt-get install bison g++-multilib git gperf libxml2-utils make python-networkx zlib1g-dev:i386 zip
    ```
    ※2つ目のみでOK・・・と思っていたのですが、自分の環境だと依存関係でエラーになってしまった

4. 「repo」のインストール
ソースの取得に必要になります。  
curlコマンドを使うので、初めにインストールしてください。  
※最初から入っていれば以下のコマンドは不要です。
    ```
    $ sudo apt-get install curl
    ```
    続けて、以下を実施。
    ```
    $ mkdir ~/bin
    $ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    $ chmod a+x ~/bin/repo
    $ PATH=~/bin:$PATH
    ```

5. ソースの取得準備
「repo」でのダウンロードとはいえ、中身自体はGitらしく、以下の設定が必要。  
    結局は適当でも問題なし。  
    ただ、ここで設定された名前は、最終的に端末情報のビルド番号に反映されたりする。  
    自身がビルドした証明にもなるので、個人的にはきっちり決めていれたいところ。  
    ```
    $ git config --global user.name "hirosawa"
    $ git config --global user.email "tomoaki.hirosawa@up-frontier.jp"
    ```
    ソースの格納ディレクトリを作り、その中にCDします。  
    ディレクトリ名は適当だが、とりあえず自分で把握できる名前にしました。  
    ```
    $ mkdir Android6_0_1
    $ cd Android6_0_1/
    ```
    続けて`repo`の初期化をします。
    ここでどのブランチのソースコードを落とすかを設定します。  
    今回は、NEXUS5用の最新ブランチを入れたいので、以下のコマンドになります。  
    ```
    $ repo init -u https://android.googlesource.com/platform/manifest -b android-6.0.1_r77
    ```
    なお、以下のブランチリストを確認したところ、`android-6.0.1_r81`が最終ブランチになっていました。 
    https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds   
    実は自分がNexus5の実機に入れたのは、上記最終ブランチのものになります。  
    これでも問題なく動作できたのですが、今回は`r77`にしました。  
    興味ある方は、最終ブランチを入れてみてもいいかと思います。  

6. ソースの取得
取得準備ができたので、ついにソースのダウンロードをします。  
その前に、以下のコマンドでCPUのコア数を確認しましょう。  
    ```
    $ nproc
    4
    ```
    上記で4コアであることがわかりました。  
    そのため、4コアでソース同期すれば、ダウンロードが早くなるかと思います。  
    `-j`のあとにコア数を指定しましょう。  
    ```
    $ repo sync -j4
    ```
    ここから結構時間がかかります。  
    自分は1時間半くらいかかったかな・・・？  
    コードの同期が終了したら、いよいよビルドを実行します。  
    参考値ですが、ソースコードの同期前後でのストレージ容量を連携します。  
    同期前  
    ```
    $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1       280G  3.8G  262G   2% /
    ```
    同期後  
    ```
    $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1       280G  109G  158G  41% /
    ```
    ソースコードのダウンロードで100GB以上取られていることが分かりました。  
    そういう意味でも、ストレージの容量は余裕をもって300GBあったほうがいいっす！  

7. Nexus5用のドライバを入れる  
以下より、Nexus5用の３つのドライバファイルをダウンロードしてください。  
https://developers.google.com/android/drivers?hl=ja#hammerheadm4b30z  
その後、解凍すると以下のshellファイルが入っているので、ソースディレクトリ直下（ここでは`Android6_0_1`）に置いてください。  
    ```
    -rwxr-x--x   1 poropi poropi    76931 Dec  8  2016 extract-broadcom-hammerhead.sh*
    -rwxr-x--x   1 poropi poropi   199191 Dec  8  2016 extract-lge-hammerhead.sh*
    -rwxr-x--x   1 poropi poropi 24346773 Dec  8  2016 extract-qcom-hammerhead.sh*
    ```
    上記を実行すると、ソースディレクトリ直下に`vendor`ディレクトリが生成され、  
    各自ドライバがインストールされます。  
    ```
    drwxr-x---   5 poropi poropi     4096 Dec  8  2016 vendor/
    ```
    最終的に、以下の感じになっていればOKです。  
    ```
    drwxrwxr-x  29 poropi poropi     4096 Sep 20 02:38 ./
    drwxr-xr-x  20 poropi poropi     4096 Sep 20 02:30 ../
    drwxrwxr-x   3 poropi poropi     4096 Sep 19 22:48 abi/
    drwxrwxr-x  16 poropi poropi     4096 Sep 19 22:48 art/
    drwxrwxr-x  12 poropi poropi     4096 Sep 19 22:48 bionic/
    drwxrwxr-x   3 poropi poropi     4096 Sep 19 22:48 bootable/
    drwxrwxr-x   7 poropi poropi     4096 Sep 19 22:48 build/
    drwxrwxr-x  12 poropi poropi     4096 Sep 19 22:48 cts/
    drwxrwxr-x  11 poropi poropi     4096 Sep 19 22:49 dalvik/
    drwxrwxr-x   6 poropi poropi     4096 Sep 19 22:49 developers/
    drwxrwxr-x  20 poropi poropi     4096 Sep 19 22:49 development/
    drwxrwxr-x  11 poropi poropi     4096 Sep 19 22:49 device/
    drwxrwxr-x   3 poropi poropi     4096 Sep 19 22:49 docs/
    drwxrwxr-x 205 poropi poropi     4096 Sep 19 22:53 external/
    -rwxr-x--x   1 poropi poropi    76931 Dec  8  2016 extract-broadcom-hammerhead.sh*
    -rwxr-x--x   1 poropi poropi   199191 Dec  8  2016 extract-lge-hammerhead.sh*
    -rwxr-x--x   1 poropi poropi 24346773 Dec  8  2016 extract-qcom-hammerhead.sh*
    drwxrwxr-x  18 poropi poropi     4096 Sep 19 22:54 frameworks/
    drwxrwxr-x  14 poropi poropi     4096 Sep 19 22:55 hardware/
    drwxrwxr-x  17 poropi poropi     4096 Sep 19 22:55 libcore/
    drwxrwxr-x   5 poropi poropi     4096 Sep 19 22:55 libnativehelper/
    -r--r--r--   1 poropi poropi       87 Sep 19 22:48 Makefile
    drwxrwxr-x   8 poropi poropi     4096 Sep 19 22:55 ndk/
    drwxrwxr-x   9 poropi poropi     4096 Sep 19 22:56 packages/
    drwxrwxr-x   6 poropi poropi     4096 Sep 19 22:56 pdk/
    drwxrwxr-x   6 poropi poropi     4096 Sep 19 22:56 platform_testing/
    drwxrwxr-x  16 poropi poropi     4096 Sep 19 22:59 prebuilts/
    drwxrwxr-x   7 poropi poropi     4096 Sep 19 22:48 .repo/
    drwxrwxr-x  25 poropi poropi     4096 Sep 19 22:59 sdk/
    drwxrwxr-x  11 poropi poropi     4096 Sep 19 22:59 system/
    drwxrwxr-x   3 poropi poropi     4096 Sep 19 22:59 tools/
    drwxr-x---   5 poropi poropi     4096 Dec  8  2016 vendor/
    ```

## ソースコードを自分用にカスタム  
今回は省略します。  
カスタムOS作りたい人は、ここのタイミングでコードをいじることになります。

## ソースコードのビルド  
ビルド環境が構築できたので、いよいよビルドを実施しましょう。  

1. ビルド前の初期設定  
ビルド前に以下のshellを実施しておきます。  
    ```
    $ source build/envsetup.sh
    ```

2. ビルド対象デバイスの指定  
lunchでビルド対象デバイスを指定します。  
一覧はここで確認できます。  
https://source.android.com/source/running.html  
以下のコマンドを実施しましょう。  
    ```
    $ lunch

    You're building on Linux

    Lunch menu... pick a combo:
        1. aosp_arm-eng
        2. aosp_arm64-eng
        3. aosp_mips-eng
        4. aosp_mips64-eng
        5. aosp_x86-eng
        6. aosp_x86_64-eng
        7. aosp_deb-userdebug
        8. aosp_flo-userdebug
        9. full_fugu-userdebug
        10. aosp_fugu-userdebug
        11. mini_emulator_arm64-userdebug
        12. m_e_arm-userdebug
        13. mini_emulator_mips-userdebug
        14. mini_emulator_x86-userdebug
        15. mini_emulator_x86_64-userdebug
        16. aosp_flounder-userdebug
        17. aosp_angler-userdebug
        18. aosp_bullhead-userdebug
        19. aosp_hammerhead-userdebug
        20. aosp_hammerhead_fp-userdebug
        21. hikey-userdebug
        22. aosp_shamu-userdebug

    Which would you like? [aosp_arm-eng]
    ```
    `Which would you like?`と聞かれるので、`19`を入力します。  
    （Nexus5は`hammerhead`なので）

3. ビルド実行  
以下の`make`コマンドで実行します。  
こちらもコア数の指定が可能なので、少しでも早くするためにも多く指定しておきましょう。  
    ```
    $ make -j4
    ```
    なお、メモリが少なかったり、ストレージの容量が少ないと、ビルド途中でエラーになります。  
    自分は、よくそれでビルドが失敗することが多かったですね。  
    参考値になりますが、自身の環境ではビルドに1時間半かかりました。  
    コア数の増加、メモリ多め、SSD環境でやることで、ビルド時間短縮できるかと思います。  
    みごとビルド成功したら`Success`と表示されるかと。  
    `ls`コマンドで`out`ディレクトリが生成されていることを確認してください。  
    ```
    drwxrwxr-x   4 poropi poropi     4096 Sep 19 23:02 out/
    ```
    お疲れ様でした！

## Nexus5へのOSインストール  
ついにOSインストールのときがやってきました。  
そのまえに、ビルドによって生成されている以下のディレクトリの存在を確認しておきましょう。  
```
$ ll /home/poropi/Android6_0_1/out/target/product/hammerhead
total 595412
drwxrwxr-x 11 poropi poropi      4096 Sep 20 03:52 ./
drwxrwxr-x  4 poropi poropi      4096 Sep 20 02:42 ../
-rw-rw-r--  1 poropi poropi        25 Sep 20 02:44 android-info.txt
-rw-r--r--  1 poropi poropi   9381888 Sep 20 03:36 boot.img
drwxrwxr-x  2 poropi poropi      4096 Sep 20 03:24 cache/
-rw-r--r--  1 poropi poropi  13668740 Sep 20 03:24 cache.img
-rw-rw-r--  1 poropi poropi     73009 Sep 20 02:42 clean_steps.mk
drwxrwxr-x  3 poropi poropi      4096 Sep 20 03:36 data/
drwxrwxr-x  2 poropi poropi      4096 Sep 20 03:16 fake_packages/
drwxrwxr-x  5 poropi poropi      4096 Sep 20 02:50 gen/
-rw-rw-r--  1 poropi poropi     76331 Sep 20 03:52 installed-files.txt
-rw-rw-r--  1 poropi poropi   8454264 Sep 20 02:45 kernel
drwxrwxr-x 13 poropi poropi      4096 Sep 20 03:52 obj/
-rw-rw-r--  1 poropi poropi       781 Sep 20 02:42 previous_build_config.mk
-rw-rw-r--  1 poropi poropi    922647 Sep 20 03:36 ramdisk.img
-rw-rw-r--  1 poropi poropi   1800485 Sep 20 03:36 ramdisk-recovery.img
drwxrwxr-x  3 poropi poropi      4096 Sep 20 03:36 recovery/
-rw-rw-r--  1 poropi poropi        67 Sep 20 03:36 recovery.id
-rw-r--r--  1 poropi poropi  10260480 Sep 20 03:36 recovery.img
drwxrwxr-x 10 poropi poropi      4096 Sep 20 03:36 root/
drwxrwxr-x  6 poropi poropi      4096 Sep 20 03:36 symbols/
drwxrwxr-x 14 poropi poropi      4096 Sep 20 03:37 system/
-rw-r--r--  1 poropi poropi 423938424 Sep 20 03:52 system.img
-rw-r--r--  1 poropi poropi 141044692 Sep 20 03:40 userdata.img
```
OSのファクトリーイメージを見たことある方であれば、  
なんとなくイメージできるファイルが確認できるかと思います。  
上記がビルドで生成されたOSのイメージファイルになります。  
以降の手順にて、OSインストールを実施します。  

ここでも念のため言っておきますが、  
端末のブートローダーのアンロックは事前にやっておいてください。  

1. SUに切り替え  
ソースコードの格納ディレクトリ（ここでは「Android6_0_1」）直下にて以下のコマンドを実施。  
    ```
    $ sudo -s
    # 
    ```

2. ANDROID_PRODUCT_OUT変数に生成OSイメージディレクトリを設定  
    ```
    # export ANDROID_PRODUCT_OUT=/home/poropi/Android6_0_1/out/target/product/hammerhead
    ```

3. 開発ツールのPATHを定義
ビルド時に生成された`out`ディレクトリ内に`adb`等のお馴染みの開発ツールが入っているので、  
そこまでのpathを設定します。  
    ```
    # PATH=/home/poropi/AndroidSrc/out/host/linux-x86/bin:$PATH
    ```

4. Nexus5へのOSインストール
Nexus5端末を電源ONの状態でUSBに接続します。  
その後、以下のコマンドを実施することで、OSインストールが開始されます。  
    ```
    # adb reboot bootloader
    # fastboot flashall -w
    ```
    正確には、最初のコマンドで端末がリブートされ、ブートローダーが起動。  
    2つ目のコマンドで実際にOSイメージを焼く作業になります。  
    成功すると、自動的にリブートされ、OSが起動するかと思います。

## 最後に  
無事起動した方、おめでとうございます！  
いつものAndroid6.0の画面が表示されたかと思います。  
ん？  
いつものとは違うって？  
GooglePlayが無いじゃないか！？って？  
いいんです・・・。  
それが仕様です。  
本来の純粋なAndroidにはGooglePlay等の便利なものは存在していません。  
Android自体はフリーなOSですが、Googleのアプリはフリーではないので、  
これこそが本来のOSなのです。  
一応、設定画面の端末情報のビルド番号を確認してみましょう。  
ビルドユーザーの名前が出ているかと思います。  
以上で終了です。

## 番外編  
とはいえで、GooglePlay等のツールが無いと厳しいですよね？  
実のところ、以下のサイトでイメージの入手が可能なので、  
https://opengapps.org/  
次回、GooglePlayも含めたOSイメージの生成手段を紹介します。
