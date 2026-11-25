# Raspberry Pi 5 + ハイクビジョン DS-E12a USBカメラ 監視システム 完全セットアップガイド

**使用する機器：** Windows 11パソコン、Raspberry Pi 5、ハイクビジョン DS-E12a USBカメラ  
**構成：** Raspberry Pi OS Desktop 64-bit + Raspberry Pi Connect（リモートデスクトップ）+ Motion + motionEye + Tailscale Funnel（スマートフォンから直接見る場合は任意）  
**ネットワーク：** Windows 11のモバイルホットスポット（大学のネットワークを共有）；自宅のWi-Fiがある場合は、ルーターのWi-Fiを使ってもかまいません。  
**このガイドの確認日：** 2026年11月25日

このガイドは、Raspberry PiやLinuxを使ったことがない人向けです。Raspberry Piには、**画面（デスクトップ）があるRaspberry Pi OS**をインストールします。Windows 11のブラウザーから **Raspberry Pi Connect** に接続すると、Raspberry Piの画面を見ながら操作できます。メニューをクリックしたり、ブラウザーやファイルマネージャーを開いたりできます。黒いコマンド画面だけで操作する必要はありません。

一部のソフトウェアをインストールするときだけ、コマンドを入力します。リモートデスクトップでRaspberry Piの **Terminal（ターミナル）** を開き、このガイドのコマンドをコピーして実行してください。Linuxのコマンドを最初から覚える必要はありません。SSHは、リモートデスクトップに接続できないときの予備の接続方法として使います。

## このシステムでできること

設定が終わると、Raspberry Pi 5が監視プログラムを動かし、DS-E12aカメラが映像を撮影します。次のことができます。

1. 同じネットワーク内で、Windowsのブラウザーからカメラの映像を見る。
2. 動きを検知したときに録画し、motionEyeの画面で録画を確認する。
3. Raspberry Piの起動時にmotionEyeとTailscaleを自動で起動する。
4. Tailscale Funnelで作ったHTTPSアドレスを使い、スマートフォンのWi-Fiを切って4G/5G回線だけで監視画面を開く。
5. Raspberry Pi Connectを使い、WindowsのブラウザーからRaspberry Piのデスクトップを見て操作する。
6. ソフトウェアのインストールに必要なコマンドを、リモートデスクトップ内のTerminalにコピーして実行する。

> **ネットワークについて：** Windows 11のモバイルホットスポットを使ってRaspberry Piを大学のネットワークにつなぐ場合、Windowsパソコンは電源を入れたままにし、インターネットに接続して、スリープしないようにしてください。Windowsパソコンの電源が切れたり、大学のネットワークが切れたりすると、Raspberry Pi本体での動作や録画は続く場合がありますが、Raspberry Pi Connectやスマートフォンからの外部アクセスは使えなくなります。

**リモートデスクトップ：** このガイドでは、公式のRaspberry Pi Connectを使います。WindowsではEdgeまたはChromeを使い、Raspberry Pi IDでログインします。WindowsにVNCソフトやTailscaleをインストールする必要はありません。

> **プライバシーに関する注意：** Tailscale Funnelを使うと、motionEyeのWebページがインターネット上に公開されます。Funnelを有効にする前に、管理者アカウントと監視用アカウントのパスワードを設定してください。公開してはいけない場所にカメラを向けないでください。また、ルーターの8765番ポートをそのままインターネットに公開しないでください。

---

# 1. 作業を始める前に：必要な機器を準備する

## 1.1 すでに持っているもの

- Raspberry Pi 5 本体
- ハイクビジョン DS-E12a USBカメラ
- Windows 11パソコン

## 1.2 追加で準備するもの

| 部品 | おすすめの仕様 | 使い道 |
|---|---|---|
| microSD（TF）カード | 64GB以上がおすすめ。信頼できるメーカーのA1/A2または高耐久タイプ | OSのインストールと、最初の録画データの保存 |
| USB microSDカードリーダー | microSDカードを読み込めるもの | microSDカードをWindowsパソコンにつなぐ |
| Raspberry Pi用電源 | Raspberry Pi公式 27W USB-C電源、5.1V/5Aを推奨 | Raspberry PiとUSB機器に安定した電力を供給する |
| 冷却装置 | アクティブクーラー、またはファン付きケースを推奨 | 長時間動かすときの温度上昇を抑える |
| ネットワーク | Windowsパソコンで使える大学のネットワーク、または自宅のWi-Fi | Raspberry Piをインターネットにつなぐ |

パソコンにmicroSDカードスロットが付いている場合、USBカードリーダーは必要ありません。Raspberry Pi 5で監視サービスを長時間動かす場合は、十分な電力を供給できる電源と冷却装置をおすすめします。

## 1.3 Raspberry Pi 5の端子を確認する

- **USB-C：** 電源を接続する端子です。
- **青いUSB-A：** USB 3.0端子です。
- **黒いUSB-A：** USB 2.0端子です。
- **Ethernet：** LANケーブルを接続する端子です。
- **micro-HDMI：** モニターを接続する端子です。このガイドではモニターを使いません。
- **microSDカードスロット：** 基板の裏側にあります。OSを書き込んだカードを入れます。

DS-E12aはUSB Type-A端子を使います。Raspberry PiのUSB-A端子に直接接続してください。CSIカメラ用のフラットケーブルは必要ありません。

**この作業では、モニター、キーボード、マウスを別に買う必要はありません。** Raspberry Pi Connectを使ってブラウザーからデスクトップを共有できます。HDMIモニターをつないでいない場合でも、Raspberry Pi OSはリモート接続用の仮想ディスプレイを作ります。リモート接続後、画面の解像度は自動的に調整されます。

## 1.4 DS-E12aカメラの仕様と、このガイドで使う設定

DS-E12aはUSB 2.0カメラです。200万画素のCMOSセンサーを使用し、最大1920×1080の映像に対応しています。マイクも内蔵しています。公開されている製品仕様には、Linux 4.14以降への対応と、MJPEG形式の複数の解像度・フレームレートが記載されています。

最初は次の設定で使います。

| 項目 | 最初の設定 |
|---|---|
| カメラ | Hikvision DS-E12a |
| 接続端子 | USB 2.0 Type-A |
| Linux上のデバイス | V4L2。通常は `/dev/video0` のように表示されます |
| 映像形式 | 選択できる場合はMJPEGを優先 |
| 最初の解像度 | 1280×720 |
| 最初のフレームレート | 10 FPS |
| 後で試せる設定 | 1920×1080。実際に使えるフレームレートは機器で確認してください |

**対応について：** DS-E12aの公開仕様にはLinux対応とUSB接続で使えることが記載されているため、標準的なUSBカメラとしてRaspberry Pi OSにつなぐ方法が適しています。ただし、最後は実際の動作確認が必要です。カメラを接続したあと、Linuxに利用可能な `/dev/video*` というデバイスが表示され、`v4l2-ctl` で対応する映像形式を確認できる必要があります。

**マイクについて：** DS-E12aにはマイクがありますが、このガイドで使うMotion/motionEyeの標準的な監視機能は、映像の取得、動体検知、録画を行います。音声を一緒に録音する機能はありません。このガイドで作るのは、**音声なしの映像監視システム**です。保存した動画に音声が入っているとは考えないでください。

---

# 2. Windows 11で大学のネットワークを共有する

自宅のルーターのWi-Fiを使う場合は、この章を飛ばしてください。OSを書き込むときに、自宅のWi-Fi名とパスワードを入力してください。

## 2.1 Windowsパソコンが大学のネットワークにつながっているか確認する

1. Windows 11パソコンを大学のネットワークにつなぎます。
2. 大学のネットワークでWeb認証が必要な場合は、ブラウザーを開いて認証を済ませます。
3. 普通のWebページを開き、Windowsパソコン自体がインターネットにつながっていることを確認します。
4. 大学のネットワーク規則で、個人パソコンのホットスポットやネットワーク共有が許可されているか確認してください。大学が共有を禁止している場合は、この方法を使わないでください。

## 2.2 モバイルホットスポットを設定する

1. Windowsのスタートメニューをクリックします。
2. **設定**を開きます。
3. **ネットワークとインターネット → モバイルホットスポット**を開きます。
4. 「インターネット接続を共有する」の項目で、現在インターネットにつながっている大学のネットワーク（Wi-FiまたはEthernet）を選びます。
5. 「共有方法」で **Wi-Fi** を選びます。
6. 「プロパティ」を開き、「編集」をクリックします。
7. ネットワーク名には、英字だけの名前を入力します。例：

   ```text
   RPi5_Net
   ```

8. 自分で管理できるWi-Fiパスワードを設定します。8文字以上にし、簡単すぎるパスワードは使わないでください。
9. 周波数帯を選べる場合は、最初は **2.4 GHz** を選びます。
10. 保存をクリックします。
11. モバイルホットスポットの画面に戻り、上にあるスイッチをオンにします。

## 2.3 ホットスポットからインターネットにつながるか確認する

1. スマートフォンを、先ほど作った `RPi5_Net` に接続します。
2. スマートフォンのモバイルデータ通信を一時的にオフにし、Wi-Fiだけを使います。
3. スマートフォンでWebページを開きます。
4. 正常にWebページが開けば、Windowsのホットスポットからインターネットを共有できています。
5. 確認が終わったら、スマートフォンをホットスポットから切断してください。次はRaspberry Piを接続します。

スマートフォンがホットスポットにつながってもインターネットを使えない場合は、Windowsの「インターネット接続を共有する」で正しいネットワークを選んでいるか確認してください。大学のネットワークによっては、ネットワーク共有、接続できる機器の数、機器同士の通信が制限される場合があります。大学の規則で共有できない場合は、自宅のルーターまたは個人のモバイルホットスポットを使ってください。

## 2.4 Windowsのホットスポットが自動で切れないようにする

Raspberry Piをインターネットにつないだままにするため、次の設定をします。

1. 「モバイルホットスポット」の設定に「省電力」または「接続機器がないときにホットスポットをオフにする」という項目があるか確認します。
2. その項目がある場合は、オフにします。
3. **設定 → システム → 電源とバッテリー → 画面とスリープ**を開きます。
4. 「電源に接続している場合、次の時間が経過した後にデバイスをスリープ状態にする」を「なし」にします。
5. ノートパソコンは、できるだけ電源アダプターにつないでください。画面を閉じるとスリープする場合があります。その場合はパソコンを開いたままにするか、Windowsの電源設定でカバーを閉じたときの動作を変更してください。

**注意：** 画面を消すことと、パソコンがスリープすることは別です。外出先から監視するには、Windowsパソコンを起動したままにし、大学のネットワークにつないでおく必要があります。

---

# 3. Windows 11にRaspberry Pi Imagerをインストールする

## 3.1 ソフトウェアをダウンロードする

1. Windowsのブラウザーで、Raspberry Pi公式ソフトウェアページを開きます。

   https://www.raspberrypi.com/software/

2. Windows版 **Raspberry Pi Imager** をダウンロードします。
3. ダウンロードした `.exe` ファイルをダブルクリックします。
4. インストーラーの案内に従ってインストールします。通常は初期設定のままでかまいません。
5. WindowsのスタートメニューからRaspberry Pi Imagerを開きます。

このガイドの確認時点（2026年11月25日）で、公式リリースページに表示されていたRaspberry Pi Imagerの正式版は **2.0.12** です。実際にダウンロードするときは、公式ページに表示される正式版を使ってください。

## 3.2 microSDカードをWindowsパソコンに接続する

**この時点ではWindowsパソコンだけを操作します。Raspberry Piには電源をつながないでください。**

1. microSDカードを取り出します。
2. USBカードリーダーのmicroSDスロットを確認します。
3. microSDカードを正しい向きで、ゆっくり差し込みます。向きが違う場合は、無理に押し込まないでください。
4. カードリーダーをWindowsパソコンのUSB端子に接続します。
5. Windowsに「ドライブを使うにはフォーマットする必要があります」などのメッセージが出た場合は、**キャンセル**をクリックしてください。フォーマットしないでください。
6. 「PC」または「このPC」を開き、新しいリムーバブルドライブが表示されることを確認します。容量も確認してください。

## 3.3 Raspberry Piの機種を選ぶ

Raspberry Pi Imagerの画面で、次の操作をします。

1. **Choose Device** をクリックします。
2. **Raspberry Pi 5** を選びます。

## 3.4 OSを選ぶ（必ずデスクトップ版を選択する）

1. **Choose OS** をクリックします。
2. 一覧から **Raspberry Pi OS (64-bit)** を選びます。これは通常のデスクトップ版です。
3. `Raspberry Pi OS Lite (64-bit)` は選ばないでください。Liteにはグラフィカルなデスクトップ画面がありません。
4. `Raspberry Pi OS Full` を選ぶ必要もありません。通常のDesktop版にデスクトップ画面と基本的なソフトウェアが入っており、Full版より保存容量を節約できます。

このガイドの確認日は2026年11月25日です。公式の現在の64-bitデスクトップ版はDebian 13 Trixieをベースにしており、2026年10月6日に公開され、Kernel 6.18を使用しています。Raspberry Pi 5に対応しています。Imagerでは、必ず **Raspberry Pi OS (64-bit)** と表示されたデスクトップ版を選んでください。名前にLiteが付いているものは選ばないでください。

> 必要なのは、壁紙、アプリケーションメニュー、ファイルマネージャー、ブラウザーがあるOSです。起動後は、普通のパソコンのようにクリックして操作できます。通常のDesktop版はLite版よりメモリと保存容量を多く使いますが、Raspberry Pi 5なら動かせます。microSDカードは64GB以上をおすすめします。

## 3.5 保存先のmicroSDカードを選ぶ

1. **Choose Storage** をクリックします。
2. 先ほど接続したmicroSDカードを一覧から選びます。
3. 容量をもう一度確認してください。Windowsのシステムドライブ、外付けハードディスク、ほかの保存装置を絶対に選ばないでください。

> **注意：** OSを書き込むと、選択した保存装置のデータはすべて消去されます。間違った装置を選ぶと、パソコンや外付けハードディスクのデータが消える可能性があります。

## 3.6 初回起動に必要な情報を設定する

続行して **OS Customisation（OSのカスタマイズ）** を開きます。次の項目を設定してください。Imager 2.xでは、項目が複数のページに分かれている場合があります。画面に表示される名前を見て設定してください。

### General / 基本情報

| 項目 | 入力する内容 |
|---|---|
| Hostname | `pi-cam` |
| Username | `camadmin` |
| Password | 自分で強いパスワードを設定し、記録する |
| Confirm Password | 同じパスワードをもう一度入力する |

ユーザー名は、小文字の英字と数字を使ってください。`12345678` のような簡単なパスワードは使わないでください。

自分のメモに、次の情報を記録してください。

```text
デバイス名：pi-cam
ユーザー名：camadmin
パスワード：自分で設定したパスワード
```

### Localisation / 地域と言語

| 項目 | 設定 |
|---|---|
| Time Zone | 日本にいる場合は `Asia/Tokyo` |
| Keyboard Layout | 実際に使うキーボードに合わせる。分からない場合は、まずUSを選ぶ |
| WLAN Country | `JP` / Japan |

### Wireless LAN / Wi-Fi

| 項目 | 入力する内容 |
|---|---|
| SSID | `RPi5_Net`（または自分で設定したホットスポット名） |
| Password | Windowsモバイルホットスポットのパスワード |

SSIDとパスワードは、大文字・小文字を区別します。自宅のルーターを使う場合は、自宅のWi-Fi名とパスワードを入力してください。

### Remote access / リモート接続

1. **Enable SSH** を探します。
2. チェックを入れてSSHを有効にします。
3. 認証方法は **Use password authentication** を選びます。

SSHは予備の接続方法です。普段は、この後に設定するRaspberry Pi Connectのデスクトップを使います。

### Raspberry Pi Connect / リモートデスクトップ（必ず設定する）

Raspberry Pi Connectを使うと、WindowsのブラウザーでRaspberry Piのデスクトップを見て操作できます。モニターがない状態で初回起動後に認証するのは難しいため、OSを書き込むときにアカウントとの連携を済ませておきます。

1. Raspberry Pi Imagerのカスタマイズ画面で、**Raspberry Pi Connect** のページを探します。
2. **Enable Raspberry Pi Connect** をオンにします。
3. **Open Raspberry Pi Connect** をクリックします。公式ConnectのWebページがブラウザーで開きます。
4. Raspberry Pi IDでログインします。アカウントがない場合は、画面の案内に従って個人アカウントを作成します。
5. Webページで個人用の **auth key（一回限りの認証キー）** を作成します。その後、**Create auth key and launch Raspberry Pi Imager** を選びます。
6. ブラウザーからRaspberry Pi Imagerを開く確認が表示されたら、許可してください。
7. Imagerに認証キーが自動で渡されます。自動入力されない場合は、Webページの「Having trouble?」の案内に従ってキーをコピーし、Imagerのtoken入力欄に貼り付けます。
8. 個人用auth keyの有効期限は6時間です。OSを書き込んだ後、必ず有効期限内にRaspberry Piの電源を入れてインターネットにつなぎ、アカウントとの連携を完了させてください。

**auth keyを他人に送ったり、公開のチャットに貼り付けたりしないでください。** このキーはRaspberry Piを自分のアカウントに登録するための認証情報です。

現在のImager画面にRaspberry Pi Connectのページがない場合は、Imagerが公式の最新バージョンか、OSとして通常のRaspberry Pi OS Desktop版を選んでいるか確認してください。Lite版に変更しないでください。

**デスクトップの自動ログイン：** Raspberry Pi OS Desktopは、通常、起動時にデスクトップへ自動ログインします。この設定は初期状態のままにし、自動ログインをオフにしないでください。Raspberry Pi Connectの画面共有には、起動中のデスクトップ画面が必要です。

## 3.7 OSを書き込む

1. 機種がRaspberry Pi 5になっていることを確認します。
2. OSがRaspberry Pi OS (64-bit)のデスクトップ版になっていることを確認します。
3. 保存先がmicroSDカードになっていることを確認します。
4. Wi-Fi名、ユーザー名、パスワード、SSHの設定を確認します。
5. **Write** をクリックします。
6. 保存装置を消去する確認が表示されたら、対象がmicroSDカードであることをもう一度確認し、消去と書き込みを許可します。
7. OSのダウンロード、書き込み、確認が終わるまで待ちます。
8. 書き込み成功またはDoneが表示されたら、Finishをクリックします。

書き込み中はカードリーダーを抜かないでください。Imagerを閉じたり、パソコンをシャットダウンまたはスリープさせたりしないでください。

## 3.8 microSDカードをWindowsから安全に取り外す

1. Raspberry Pi ImagerのFinishをクリックします。
2. Windowsで再びフォーマットの確認が出た場合は、キャンセルしてください。
3. Windowsタスクバー右側の隠れているアイコン `^` をクリックします。
4. 「ハードウェアを安全に取り外してメディアを取り出す」を探します。
5. USBカードリーダーを選んで取り出します。
6. Windowsから安全に取り外せるという表示が出たら、カードリーダーを抜きます。
7. カードリーダーからmicroSDカードを取り出します。

書き込み後のLinux用パーティションをWindowsが読み込めなくても、フォーマットしないでください。これは正常です。

---

# 4. microSDカードをRaspberry Pi 5に入れて、初めて起動する

## 4.1 Raspberry Piの電源が完全に切れていることを確認する

Raspberry Piに電源がつながっている場合は、次の手順で作業してください。

1. すでに起動している場合は、先に正常な方法でシャットダウンします。
2. USB-C電源ケーブルを抜きます。
3. 基板の電源が切れていることを確認します。

**電源が入っているときにmicroSDカードを抜き差ししないでください。**

## 4.2 microSDカードを入れる

1. Raspberry Pi 5を裏返します。
2. 基板の裏側にあるmicroSDカードスロットを探します。
3. microSDカードのラベル面を基板と反対側に向け、金色の端子を基板側に向けます。
4. スロットに合わせて、水平にゆっくり奥まで差し込みます。
5. 強く押したり、カードを曲げたりしないでください。カードの端は少し外に見えた状態になります。

カードがうまく入らないときは、向きを確認してください。無理に押し込まないでください。

## 4.3 DS-E12aカメラを接続する

1. DS-E12aのUSB Type-AプラグをRaspberry PiのUSB-A端子に差し込みます。
2. 最初のテストでは青いUSB 3.0端子を使ってください。USB 2.0カメラなので、黒いUSB 2.0端子でも使えます。
3. 最初は、別電源のないUSBハブを間に入れず、Raspberry Piに直接接続してください。
4. カメラケーブルは約1.5mです。端子に強い力がかからないように置いてください。

## 4.4 電源をつないで初回起動する

1. microSDカードが入っていることを確認します。
2. DS-E12aがUSB端子につながっていることを確認します。
3. 最後にRaspberry Pi 5のUSB-C電源を接続します。
4. 初めて電源をつなぐと、Raspberry Pi 5は自動的に起動します。電源ボタンを押す必要はありません。
5. 初回起動とネットワーク接続が終わるまで、約2～3分待ちます。
6. 待っている間は、電源やカメラを抜いたり、microSDカードに触れたりしないでください。

---

# 5. Windows 11からRaspberry Piのデスクトップにリモート接続する

このプロジェクトでは、主に **Raspberry Pi Connect** を使います。Windowsにリモートデスクトップ用ソフトをインストールしたり、最初からコマンド操作を覚えたりする必要はありません。

## 5.1 WindowsからRaspberry Pi Connectを開く

1. Raspberry Piの初回起動が終わり、インターネットにつながったら、Windows 11でEdgeまたはChromeを開きます。
2. アドレス欄に次のURLを入力します。

   ```text
   https://connect.raspberrypi.com/
   ```

3. OSを書き込むときに連携したRaspberry Pi IDでログインします。
4. **Devices** ページを開きます。
5. デバイス名 `pi-cam` を探します。
6. デバイスの横にある **Connect via** をクリックします。
7. **Screen sharing** を選びます。
8. 接続が終わるまで待ちます。

接続に成功すると、ブラウザーの画面にRaspberry Piのデスクトップが表示されます。マウスでデスクトップのアイコンをクリックしたり、メニューを開いたり、ブラウザーやファイルマネージャーを開いたりできます。普通のパソコンと同じように文字を入力できます。

## 5.2 モニターをつながなくてもデスクトップを表示できる

このプロジェクトでは、Raspberry PiにHDMIモニターをつなぐ必要はありません。モニターがつながっていない場合、Raspberry Pi OSは仮想ディスプレイを作ります。Connectで画面共有を始めると、仮想デスクトップの解像度はWindowsのブラウザー画面に合わせて調整されます。

そのため、必要なのはRaspberry Pi 5、電源、microSDカード、DS-E12a、ネットワークです。別のモニター、キーボード、マウスを買う必要はありません。

## 5.3 リモートデスクトップでTerminal（ターミナル）を開く

ほとんどの設定は、デスクトップ画面をクリックして行えます。MotionやmotionEyeをインストールするときだけ、いくつかのコマンドを使います。

1. 5.1の手順でRaspberry Piのデスクトップに接続します。
2. デスクトップ左上のRaspberry Piメニューアイコンをクリックします。
3. **Accessories（アクセサリ）** を開きます。
4. **Terminal（ターミナル）** をクリックします。
5. 黒いターミナル画面が開きます。
6. このガイドのコマンドをコピーし、ターミナル画面をクリックして貼り付け、Enterキーを押します。
7. コマンドの処理が終わり、新しい入力待ちの表示が出てから、次のコマンドを実行します。

黒いターミナル画面を使うのは、ソフトウェアのインストールや管理をするときだけです。普段は、通常のデスクトップ画面で操作できます。

## 5.4 DevicesにScreen sharingが表示されない場合

次の順番で確認してください。

1. インストールしたOSが **Raspberry Pi OS (64-bit)** のデスクトップ版であり、Lite版ではないことを確認します。
2. Raspberry Piがインターネットにつながっていて、Windowsのホットスポットがオンになっていることを確認します。
3. ImagerでRaspberry Pi Connectを有効にし、認証キーが有効な6時間以内にRaspberry Piで使われたことを確認します。
4. Raspberry Pi OSのデスクトップ自動ログインをオフにしていないことを確認します。
5. デスクトップでConnectアイコンを開き、Screen sharingが許可されているか確認します。
6. まだ接続できない場合は、デスクトップのTerminalで次のコマンドを実行します。

   ```bash
   rpi-connect status
   ```

   クライアントの状態を確認します。

   ```bash
   rpi-connect doctor
   ```

   画面共有に必要な部品やデスクトップ環境を確認します。

Raspberry Pi Connectの画面共有には、完全な `rpi-connect` クライアント、Waylandの画面環境、起動中のRaspberry Pi Desktopが必要です。通常のRaspberry Pi OS Desktopは、基本的にこれらの条件を満たしています。Lite版で使うConnect Liteでは、画面共有はできません。

## 5.5 SSHで接続する方法（予備用。普段は使わなくてよい）

Raspberry Pi Connectに一時的に接続できない場合は、SSHを予備の接続方法として使えます。

1. Windows 11でTerminalまたはPowerShellを開きます。
2. Windowsのモバイルホットスポットに接続している機器の一覧、またはルーターの画面で、`pi-cam` のIPアドレスを確認します。
3. 次のコマンドを入力します。例のIPアドレスは、自分の実際のアドレスに置き換えてください。

   ```powershell
   ssh camadmin@192.168.137.45
   ```

4. 初めて接続するときは `yes` と入力し、次にImagerで設定したパスワードを入力します。パスワードを入力しても画面に文字は表示されません。これは正常です。
5. 接続に成功すると、`camadmin@pi-cam:~ $` のような表示が出ます。

普段は5.1のグラフィカルなデスクトップを使ってください。リモートデスクトップが使えない場合や、緊急のメンテナンスが必要な場合だけSSHを使います。

---

# 6. Raspberry Piのシステムを初期設定する

この章のコマンドはすべて、**Raspberry Pi Connectで開いたリモートデスクトップ内のTerminal** で実行します。先に第5章の方法でRaspberry Piのデスクトップに接続し、Raspberry PiメニューからAccessories → Terminalを開いてください。Windows本体のPowerShellに入力しないでください。

## 6.1 ソフトウェア一覧を更新する

```bash
sudo apt update
```

コマンドが終わるまで待ちます。

## 6.2 システムを更新する

```bash
sudo apt full-upgrade -y
```

更新が終わるまで待ちます。作業中はターミナルを閉じたり、電源を切ったりしないでください。

## 6.3 Raspberry Piを再起動する

```bash
sudo reboot
```

リモート接続が切れますが、これは正常です。約1～3分待ち、WindowsブラウザーのRaspberry Pi ConnectのDevicesページを更新します。もう一度 `pi-cam → Connect via → Screen sharing` を選びます。

## 6.4 OSのバージョンとCPUの種類を確認する

OSのバージョンを確認します。

```bash
cat /etc/os-release
```

Raspberry Pi OSで、Debian 13 Trixieをベースにしていることを確認します。

CPUの種類を確認します。

```bash
uname -m
```

次の表示が出れば正常です。

```text
aarch64
```

Raspberry PiのIPアドレスを確認します。

```bash
hostname -I
```

表示されたIPアドレスをメモしてください。後でmotionEyeのWebページを開くときに使います。

---

# 7. ハイクビジョン DS-E12aがLinuxに認識されているか確認する

motionEyeをインストールする前に、カメラが認識されているか確認します。先に確認しておくと、Web画面に映像が出ない場合に、カメラの問題かソフトウェアの設定の問題かを分けて考えられます。

## 7.1 USB機器を確認する

```bash
lsusb
```

表示された一覧から、Hikvision、USB Camera、Video、または似た名前の機器を探します。ファームウェアによってはUSBのメーカー番号だけが表示され、「Hikvision」と表示されない場合もあります。

## 7.2 V4L2確認ツールをインストールする

```bash
sudo apt install -y v4l-utils
```

## 7.3 カメラのデバイスを一覧表示する

```bash
v4l2-ctl --list-devices
```

カメラが認識されると、通常は次のように表示されます。

```text
USB Camera:
    /dev/video0
```

`/dev/video0`、`/dev/video1` のように、複数のデバイスが表示される場合もあります。複数表示されても、カメラを複数台接続したという意味とは限りません。

デバイスファイルを確認します。

```bash
ls -l /dev/video*
```

## 7.4 DS-E12aが実際に対応している映像形式を確認する

前の手順で `/dev/video0` と表示された場合は、次を実行します。

```bash
v4l2-ctl -d /dev/video0 --list-formats-ext
```

実際のカメラデバイスが `/dev/video0` 以外の場合は、コマンド内のパスを実際のデバイス名に変更してください。

DS-E12aの公開仕様には、MJPEG形式と、1920×1080、1280×720、640×480などの解像度が記載されています。実際の出力に次のような表示があれば、カメラがMJPEG形式の映像を出せることを意味します。

```text
Pixel Format: 'MJPG'
Size: Discrete 1280x720
```

`/dev/video*` が複数ある場合は、それぞれに対して `--list-formats-ext` を実行します。番号だけで決めず、MJPEGや映像キャプチャ用の解像度が表示されるデバイスを選んでください。

## 7.5 `/dev/video*` が見つからない場合

次の順番で確認します。

1. カメラのUSBプラグがしっかり差し込まれているか確認します。
2. 電源が別に必要なUSBハブを使わず、カメラをRaspberry Piに直接つなぎます。
3. 次のコマンドを実行します。

   ```bash
   lsusb
   ```

4. UVCドライバーのモジュールを確認します。

   ```bash
   lsmod | grep uvcvideo
   ```

5. 何も表示されない場合は、ドライバーを読み込んでみます。

   ```bash
   sudo modprobe uvcvideo
   ```

6. 最近のカーネルログを確認します。

   ```bash
   sudo dmesg | tail -n 60
   ```

7. もう一度カメラの一覧を確認します。

   ```bash
   v4l2-ctl --list-devices
   ```

`lsusb` にはカメラが表示されるのに、使える映像デバイスが出ない場合は、`lsusb`、`dmesg`、`v4l2-ctl --list-devices` の結果を保存して、ドライバーや機器の対応状況を確認してください。映像デバイスが表示される前に、motionEyeへカメラを追加しないでください。

---

# 8. Motion、FFmpeg、motionEyeをインストールする

## 8.1 3つのソフトウェアの役割

| ソフトウェア | 役割 |
|---|---|
| Motion | カメラの映像を読み込み、動きを検知し、録画を開始する |
| FFmpeg | 動画の処理や形式の変換を行う |
| motionEye | Motionをブラウザーから設定・管理するためのWeb画面 |

MotionとFFmpegは、OSのソフトウェア管理ツールを使ってインストールします。motionEyeはPythonで動くソフトウェアです。

## 8.2 必要なソフトウェアをインストールする

Raspberry Piのリモートデスクトップ内のTerminalで、次を入力します。

```bash
sudo apt update
```

次に、ソフトウェアをインストールします。

```bash
sudo apt install --no-install-recommends -y ca-certificates curl python3 python3-pip motion ffmpeg v4l-utils
```

インストールが終わるまで待ちます。

Motionがインストールされたか確認します。

```bash
motion --version
```

FFmpegがインストールされたか確認します。

```bash
ffmpeg -version
```

両方のコマンドでバージョン情報が表示されたら、次に進みます。

## 8.3 Python pipのインストール権限を設定する

motionEye 0.44.0の公式安定版インストール説明では、システムのPythonでpipを使う方法が案内されています。Debian 13ではシステムのPythonが初期状態で保護されているため、motionEyeの説明に従ってpipを設定します。

まず、次を実行します。

```bash
grep -q '^\[global\]' /etc/pip.conf 2>/dev/null || printf '%s\n' '[global]' | sudo tee -a /etc/pip.conf >/dev/null
```

次に、次を実行します。

```bash
sudo sed -i '/^\[global\]/a break-system-packages=true' /etc/pip.conf
```

この設定は、motionEye 0.44.0の公式PyPIインストール説明に書かれている方法です。ほかのプロジェクトのPythonパッケージを、必要もないのにシステム環境へインストールしないでください。この設定は、このガイドでmotionEyeをインストールするために使います。

## 8.4 motionEye 0.44.0 安定版をインストールする

次を実行します。

```bash
sudo python3 -m pip install "motioneye==0.44.0"
```

インストールが終わるまで待ちます。その後、バージョンを確認します。

```bash
python3 -m pip show motioneye
```

表示の中に次の行があることを確認します。

```text
Version: 0.44.0
```

`0.45.0b1` のように、数字に `b` が付いているテスト版はインストールしないでください。初回のセットアップでは、PyPIで安定版として公開されている0.44.0を使います。

## 8.5 motionEyeを初期設定する

次を入力します。

```bash
sudo motioneye_init
```

初期設定プログラムが、motionEyeの設定ファイルとsystemdサービスを作成します。

次に、起動時の自動起動を有効にし、今すぐ起動します。

```bash
sudo systemctl enable --now motioneye
```

サービスの状態を確認します。

```bash
sudo systemctl status motioneye --no-pager
```

正常な場合は、次の表示が出ます。

```text
Active: active (running)
```

`failed` と表示された場合は、先にログを確認します。

```bash
sudo journalctl -u motioneye -n 80 --no-pager
```

ログの内容に合わせて問題を解決してから、次の章に進んでください。

---

# 9. WindowsのブラウザーでmotionEyeを開く

## 9.1 Raspberry PiのIPアドレスを確認する

Raspberry Piのリモートデスクトップ内のTerminalで、次を入力します。

```bash
hostname -I
```

例えば、次のように表示されます。

```text
192.168.137.45
```

これは例です。実際に表示されたIPアドレスを使ってください。

## 9.2 motionEyeのWebページを開く

Windows 11のEdgeまたはChromeのアドレス欄に、次のように入力します。

```text
http://Raspberry_Piの実際のIPアドレス:8765/
```

例えば、次のようになります。

```text
http://192.168.137.45:8765/
```

次のアドレスでも開ける場合があります。

```text
http://pi-cam.local:8765/
```

`.local` で開けない場合は、IPアドレスを使ってください。

WindowsパソコンがRaspberry Piと同じローカルネットワークにつながっていない場合、このローカルIPアドレスには直接アクセスできません。その場合は、先にRaspberry Pi ConnectでRaspberry Piのデスクトップに接続し、Raspberry PiのデスクトップでChromiumブラウザーを開いて、次のアドレスにアクセスしてください。

```text
http://localhost:8765/
```

こうすると、リモートデスクトップの中でmotionEyeの管理画面を確認できます。

## 9.3 初めてログインする

motionEyeの初期ログイン情報は次のとおりです。

- Username：`admin`
- Password：空欄のまま

初回ログイン後、すぐにパスワードを設定してください。左上のメニューから **General Settings** を開き、次の2つを設定します。

- **Admin Password：** 管理者用の強いパスワード
- **Surveillance Password：** 監視画面を見るユーザー用のパスワード

2つとも設定して保存してください。インターネットに公開する場合、パスワードを空欄のままにしないでください。

パスワードを設定したら、ページ上部の **Apply** をクリックして保存します。

---

# 10. motionEyeにハイクビジョン DS-E12aを追加する

## 10.1 ローカルUSBカメラを追加する

1. motionEyeの左上にあるカメラメニューを開きます。
2. **Add Camera...** を選びます。
3. Camera Typeで **Local V4L2 Camera** を選びます。
4. CameraまたはCamera Deviceの一覧から、第7章の `v4l2-ctl` で確認したDS-E12aの映像デバイスを選びます。
5. 一覧に `/dev/video0` と `/dev/video1` がある場合は、前にMJPEG形式を確認できた方を選びます。
6. **OK** をクリックします。
7. 数秒待ちます。

## 10.2 ライブ映像を確認する

1. DS-E12aを監視したい場所に向けます。
2. カメラの前で手を振るか、物を動かします。
3. motionEyeの画面に表示される映像も同じように変化するか確認します。
4. 映像が正常に表示されたら、解像度とフレームレートを設定します。

## 10.3 DS-E12aの映像設定をする

カメラ設定の **Video Device** を開き、最初は次の値を設定します。

| 項目 | 最初のおすすめ設定 |
|---|---|
| Camera Device | 実際のデバイス名。例：`/dev/video0` |
| Video Resolution | `1280x720` |
| Frame Rate | `10` FPS |
| Stream Quality | だいたい`70`～`75` |
| Video Format / Palette（画面に表示される場合） | `MJPG`を優先 |

DS-E12aはMJPEG形式に対応しています。motionEyeに映像形式の選択欄がある場合は、MJPGを優先してください。フレームレートの低い非圧縮形式を間違って選ばないようにします。

最初は720P、10 FPSで使います。安定して動くことを確認したら、フレームレートを上げるか、1920×1080に変更してみてください。安定して動くかどうかは、カメラの出力モード、Motion、Raspberry Piの負荷、ネットワークの上り速度などによって変わります。

設定を変更するたびに、設定欄の上にある **Apply** をクリックして保存します。

## 10.4 画面が真っ暗、または映像が表示されない場合

1. リモートデスクトップ内のTerminalに戻り、次を実行します。

   ```bash
   v4l2-ctl --list-devices
   ```

2. motionEyeのCamera Deviceが、システムで確認したデバイス名と同じか確認します。
3. そのデバイスが対応している形式を確認します。

   ```bash
   v4l2-ctl -d /dev/video0 --list-formats-ext
   ```

4. 実際のデバイスが `/dev/video0` ではない場合は、コマンドのパスを正しいものに変更します。
5. motionEyeのログを確認します。

   ```bash
   sudo journalctl -u motioneye -n 80 --no-pager
   ```

6. ほかのプログラムがカメラを使用していないか確認します。
7. motionEyeの設定を640×480、10 FPSに下げてテストし、その後少しずつ上げます。

映像の場所にカメラのアイコンが表示され、ライブ映像が出ない場合は、Video Deviceのデバイス名を確認してください。また、`Frame Rate Dimmer` が0より大きい値になっているか確認してください。

---

# 11. 動体検知と録画を設定する

設定を変更するたびに、ページ上部の **Apply** をクリックして保存してください。

## 11.1 動体検知をオンにする

1. DS-E12aのカメラ設定を開きます。
2. **Motion Detection** を探します。
3. Motion Detectionをオンにします。
4. 最初のテストでは、感度を初期設定のままにします。
5. カメラの前を歩き、動きを検知するか確認します。
6. カーテン、照明の変化、画面のノイズなどで誤検知が起きる場合は、感度、Threshold、検知しない範囲（マスク）を少しずつ調整します。

## 11.2 動きを検知したときだけ録画する

1. カメラ設定の **Movies** を開きます。
2. Moviesをオンにします。
3. **Recording Mode** を探します。
4. **Motion Triggered** を選びます。
5. 録画の画質を設定します。
6. **Preserve Movies** を探し、最初は1日または1週間の保存を選びます。
7. **Apply** をクリックして保存します。

## 11.3 録画をテストする

1. カメラを監視したい場所に向けます。
2. カメラの前で動き、動体検知を1回起こします。
3. 動くのをやめ、録画ファイルが作られるまで待ちます。
4. motionEyeのメディア／録画の画面を開きます。
5. 今作られた動画を探して再生します。
6. 動画が正常に再生できることを確認します。

注意：DS-E12aにマイクが内蔵されていても、Motion/motionEyeが標準で音声を録画するわけではありません。このガイドで作るのは映像のみの録画システムです。

## 11.4 保存容量を確認する

リモートデスクトップ内のTerminalで、次を入力します。

```bash
df -h
```

motionEyeの標準的なメディア保存先は、通常、次の場所です。

```text
/var/lib/motioneye
```

フォルダーの中を確認できます。

```bash
sudo ls -lah /var/lib/motioneye
```

microSDカードは、最初のテストや少量の動体検知録画には使えます。24時間の長時間運転や大量の動画を保存する場合は、USB SSDを追加し、録画先をSSDにすることをおすすめします。普通のmicroSDカードに書き込み続けると、カードの劣化が早くなる可能性があります。

---

# 12. （任意）Tailscaleをインストールし、Funnelでスマートフォンから監視画面を直接開く

**注意：この章はリモートデスクトップを使うために必要な作業ではありません。** WindowsからRaspberry Piのデスクトップを遠隔操作するだけなら、第5章のRaspberry Pi Connectで十分です。スマートフォンからRaspberry Pi Connectを使わず、motionEyeのログイン画面をブラウザーで直接開きたい場合に、この章を実行してください。Funnelを使うとmotionEyeのページがインターネットに公開されるため、公開することの安全上の注意を理解してから設定してください。

## 12.1 先にローカルで監視できることを確認する

インターネットに公開する前に、次を確認します。

- Windowsのブラウザーで `http://Raspberry PiのIP:8765/` を開ける。
- motionEyeの管理者パスワードを設定した。
- Surveillanceユーザーのパスワードを設定した。
- DS-E12aのライブ映像が正常に表示される。
- 動体検知と録画のテストが終わっている。

パスワードが空欄のままFunnelを有効にしないでください。

## 12.2 Tailscaleアカウントを作る

1. Windowsのブラウザーで次を開きます。

   https://tailscale.com/

2. Get Startedをクリックして、個人アカウントを登録・ログインします。
3. 画面の案内に従ってログインします。
4. Raspberry Piを認証するときも、同じアカウントを使います。

Tailscale Funnelを使うと、Tailscaleをインストールしていない外部の機器でも、普通のブラウザーから公開されたHTTPSサービスにアクセスできます。Funnelはインターネットに公開する機能です。自分だけが使えるプライベートVPNとは違います。

## 12.3 Raspberry PiにTailscaleをインストールする

Raspberry Piのリモートデスクトップ内のTerminalで、次を実行します。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

インストールが終わるまで待ちます。

次に、次を入力します。

```bash
sudo tailscale up
```

ターミナルにログイン認証用のURLが表示されます。通常は `https://login.tailscale.com/a/` から始まります。

1. ターミナルに表示されたURLをすべてコピーします。
2. WindowsのブラウザーでそのURLを開きます。
3. 先ほど作成したTailscaleアカウントでログインします。
4. 画面の案内に従って、Raspberry Piが自分のネットワークに参加することを許可します。
5. リモートデスクトップ内のTerminalに戻り、次を実行します。

   ```bash
   tailscale status
   ```

一覧にRaspberry Piの機器が表示されることを確認します。

## 12.4 Funnelを有効にしてmotionEyeを公開する

Raspberry Piのリモートデスクトップ内のTerminalで、次を実行します。

```bash
sudo tailscale funnel --bg 8765
```

初めて有効にするとき、ブラウザーでFunnelを許可するよう求められる場合があります。ターミナルに表示されたリンクを開き、案内に従ってFunnel、MagicDNS、HTTPS証明書を有効にします。許可が終わったら、リモートデスクトップ内のTerminalに戻り、もう一度次を実行します。

```bash
sudo tailscale funnel --bg 8765
```

成功すると、ターミナルに次のような表示が出ます。

```text
Available on the internet:
https://pi-cam.example-tailnet.ts.net
|-- / proxy http://127.0.0.1:8765
```

上のURLは形式の例です。ターミナルに実際に表示された `https://...ts.net` のURLをコピーし、スマートフォンのメモやブックマークに保存してください。

ここでの意味は次のとおりです。

- `8765` は、Raspberry Pi上でmotionEyeが使うポート番号です。
- Funnelはインターネット用のHTTPS入口を作ります。スマートフォンからアクセスするとき、URLの最後に `:8765` を付ける必要はありません。
- `--bg` は、Funnelをバックグラウンドで動かすための指定です。

## 12.5 スマートフォンで外部ネットワークからアクセスする

Raspberry Piのデスクトップを遠隔操作するだけなら、スマートフォンのブラウザーで `https://connect.raspberrypi.com/` にログインし、Screen sharingを選ぶ方法もあります。この方法ではFunnelは必要ありません。この節では、スマートフォンからmotionEyeの動画ページを直接開く場合を説明します。

1. Raspberry Piが起動していることを確認します。
2. Windowsパソコンが大学のネットワークにつながり、モバイルホットスポットがオンになっていることを確認します。
3. スマートフォンのWi-Fiをオフにし、4G/5Gのモバイルデータ通信だけにします。
4. スマートフォンでTailscale VPNに接続しないでください。
5. SafariまたはChromeを開きます。
6. 保存したTailscale FunnelのHTTPSアドレスを入力します。
7. motionEyeにログインし、ライブ映像が表示されることを確認します。

ログイン画面は開くのに動画が止まる場合は、motionEyeの映像設定を1280×720、10 FPSに下げてください。それでも遅い場合は640×480、10 FPSを試します。Funnelの帯域上限はユーザー側で変更できません。そのため、どのネットワークでも1080Pが滑らかに再生できるとは限りません。

## 12.6 Funnelの状態を確認する

Raspberry PiのTerminalで次を実行します。

```bash
sudo tailscale funnel status
```

現在の公開設定を確認できます。

## 12.7 インターネットからのアクセスを停止する

スマートフォンから外部アクセスする必要がなくなったら、Raspberry PiのTerminalで次を実行します。

```bash
sudo tailscale funnel reset
```

Funnelの公開設定が削除されます。motionEyeがアンインストールされたり、ローカルネットワークからのアクセスが停止したりするわけではありません。同じネットワーク内では、引き続き次のアドレスで監視画面を開けます。

```text
http://Raspberry PiのIP:8765/
```

## 12.8 Tailscaleの機器キーの有効期限について

Tailscaleの機器には、キーの有効期限が設定されている場合があります。初期設定のままにして、期限が来る前に再認証してもかまいません。または、管理画面のMachinesページで、この固定機器のkey expiryを無効にすることもできます。

key expiryを無効にする設定は任意です。必須ではありません。無効にすると、機器の認証が長期間有効なままになる場合があります。Raspberry Piを紛失したり盗まれたりした場合は、すぐにTailscale管理画面からその機器を削除してください。

## 12.9 Funnelを使うときの安全とプライバシー

- Funnelのアドレスはインターネット上のほかの人もアクセスできます。自分だけがアクセスできるわけではありません。
- motionEyeのログイン画面がインターネットに公開されるため、強いパスワードを設定してください。
- 大学のネットワーク、メール、ほかのWebサイトと同じパスワードを使わないでください。
- ルーターの8765番ポートを直接インターネットに公開しないでください。
- 機器名やtailnetのドメインに、氏名、学籍番号、部屋番号などの個人情報を入れないでください。
- TailscaleのHTTPS証明書に使われるドメイン名は、公開のCertificate Transparency（CT）ログに記録されます。機器名に個人情報を使わないでください。
- 外部から見る必要がないときは、`sudo tailscale funnel reset` を実行して公開を停止してください。

---

# 13. 起動時の自動実行を設定し、再起動テストをする

## 13.1 motionEyeの自動起動を確認する

リモートデスクトップ内のTerminalで次を入力します。

```bash
sudo systemctl is-enabled motioneye
```

次の表示が出れば正常です。

```text
enabled
```

現在の動作状態を確認します。

```bash
sudo systemctl is-active motioneye
```

次の表示が出れば正常です。

```text
active
```

起動時の自動実行が有効になっていない場合は、次を実行します。

```bash
sudo systemctl enable motioneye
```

## 13.2 Tailscaleサービスを確認する

リモートデスクトップ内のTerminalで次を入力します。

```bash
sudo systemctl is-enabled tailscaled
```

```bash
sudo systemctl is-active tailscaled
```

Tailscaleサービスが有効で、動作中であることを確認します。バックグラウンドで設定したFunnelは、サービス再起動後に復帰する設定ですが、実際にRaspberry Piを再起動して確認してください。

## 13.3 システム全体を再起動して確認する

Raspberry Piのリモートデスクトップ内のTerminalで、次を実行します。

```bash
sudo reboot
```

Raspberry Piが再起動するまで、約1～3分待ちます。

その後、次のことを確認します。

1. WindowsブラウザーのRaspberry Pi Connectを開き、`pi-cam` のScreen sharingにもう一度接続します。
2. リモートデスクトップでChromiumを開き、`http://localhost:8765/` にアクセスします。motionEyeの画面が開くことを確認します。
3. DS-E12aが引き続き認識され、映像が表示されることを確認します。
4. Raspberry PiのTerminalで次を実行します。

   ```bash
   sudo tailscale funnel status
   ```

5. スマートフォンのWi-Fiをオフにし、モバイルデータ通信でFunnelのアドレスにもう一度アクセスします。
6. カメラの前で動き、動体検知と録画が正常に動くことを確認します。

---

# 14. 普段の使い方、シャットダウン、microSDカードの取り外し

## 14.1 普段の使い方

監視システムを使っている間は、次の状態にしておきます。

- microSDカードはRaspberry Piに入れたままにします。
- USBカメラはRaspberry Piにつないだままにします。
- Raspberry Piの電源を入れたままにします。
- Windowsホットスポットを使う場合、Windowsパソコンを起動し、ネットワークにつなぎ、スリープさせないでください。
- ローカルネットワークのアドレス、またはTailscale Funnelのアドレスから映像を確認します。

## 14.2 正しい再起動方法（デスクトップのメニューを使う）

1. Raspberry Pi Connectのリモートデスクトップで、左上のRaspberry Piメニューアイコンをクリックします。
2. **Logout / ログアウト** メニューを開きます。
3. **Reboot（再起動）** を選びます。
4. Raspberry Piが再起動するまで待ちます。その後、WindowsブラウザーのRaspberry Pi Connectからもう一度接続します。

メニューに再起動が見つからない場合は、リモートデスクトップのTerminalで次を入力します。

```bash
sudo reboot
```

Raspberry Piが自動的に再起動します。再起動するために電源ケーブルを抜かないでください。

## 14.3 正しいシャットダウン方法（デスクトップのメニューを使う）

1. Raspberry Pi Connectのリモートデスクトップで、左上のRaspberry Piメニューアイコンをクリックします。
2. **Logout / ログアウト** メニューを開きます。
3. **Shutdown（シャットダウン）** を選びます。
4. リモートデスクトップの接続が切れるまで待ちます。基板の緑色の動作ランプが点滅しなくなり、システムが終了したことを確認します。
5. さらに数秒待ってからUSB-C電源を抜きます。

メニューにシャットダウンが見つからない場合は、リモートデスクトップのTerminalで次を入力します。

```bash
sudo poweroff
```

システムが完全に終了してから、電源を抜いてください。

## 14.4 microSDカードを取り外してよいとき

OSをもう一度書き込むとき、カードを交換するとき、カードのメンテナンスをするときだけ、microSDカードを取り外します。必ず次の順番で作業してください。

1. Raspberry Pi ConnectのデスクトップでTerminalを開きます。
2. `sudo poweroff` と入力します。
3. システムが完全に終了し、緑色の動作ランプが点滅しなくなるまで待ちます。
4. USB-C電源を抜きます。
5. Raspberry Piを裏返します。
6. microSDカードの端を持ち、水平にゆっくり引き抜きます。
7. Windowsパソコンで作業するときは、その後microSDカードをUSBカードリーダーに入れます。

**Raspberry Piが動いているときにmicroSDカードを抜き差ししないでください。また、通常のシャットダウンの代わりに電源を直接抜かないでください。**

---

# 15. よくあるトラブルと確認方法

| 症状 | 最初に確認すること・対処方法 |
|---|---|
| Raspberry Piの電源が入らない | 電源の仕様、microSDへの書き込み、カードの向きを確認する |
| Windowsのホットスポットからインターネットにつながらない | 共有元のネットワークアダプター、大学のWeb認証、大学のネットワーク制限を確認する |
| Raspberry Pi ConnectにScreen sharingが表示されない | Desktop版を使っているか、デスクトップの自動ログイン、Waylandセッション、完全な `rpi-connect` クライアントがあるか確認する。`rpi-connect doctor` で確認する |
| `pi-cam.local` が見つからない | Windowsホットスポットの接続機器一覧またはルーターのDHCP一覧でIPアドレスを調べ、IPアドレスで接続する |
| SSHで `Connection refused` が出る | Raspberry Piの起動が終わっているか、SSHが有効か確認する |
| SSHで `Permission denied` が出る | ユーザー名 `camadmin` とパスワードの大文字・小文字を確認する |
| `lsusb` にDS-E12aが表示されない | USBプラグ、端子、電源、カメラ本体を確認する |
| `lsusb` には表示されるが `/dev/video*` がない | `uvcvideo` ドライバー、カーネルログ、カメラの対応状況を確認する |
| MJPEG形式がない | `v4l2-ctl --list-formats-ext` の結果を確認し、実際に対応している形式を選ぶ |
| motionEyeのインストールに失敗する | インターネット、システム時刻、apt/pipのエラーメッセージを確認する |
| motionEyeサービスが起動しない | `sudo journalctl -u motioneye -n 80 --no-pager` でログを確認する |
| ブラウザーで8765にアクセスできない | motionEyeのサービス、Raspberry PiのIP、WindowsとRaspberry Piが同じネットワークかを確認する |
| motionEyeの画面が真っ暗 | Camera Deviceと `/dev/video*` が一致しているか確認し、解像度とフレームレートを下げる |
| 録画ファイルが作られない | Motion Detection、Movies、Recording Mode、空き容量を確認する |
| スマートフォンでFunnelのアドレスを開けない | Tailscaleの接続状態、Funnelの状態、Windowsホットスポット、大学のネットワークを確認する |
| スマートフォンでWebページは開くが動画が止まる | 720P/10 FPSまたは640×480/10 FPSに下げる。ホットスポットの上り速度とFunnelの帯域制限も確認する |

## 15.1 よく使うコマンド一覧

```bash
# Raspberry Pi Connectクライアントの状態を確認
rpi-connect status

# デスクトップ画面の共有に必要な環境を確認
rpi-connect doctor

# Raspberry PiのIPアドレスを確認
hostname -I

# OSのバージョンを確認
cat /etc/os-release

# CPUの種類を確認
uname -m

# USB機器を確認
lsusb

# カメラのデバイス名を確認
v4l2-ctl --list-devices

# カメラが対応する形式と解像度を確認
v4l2-ctl -d /dev/video0 --list-formats-ext

# UVCドライバーを確認
lsmod | grep uvcvideo

# Motionのバージョンを確認
motion --version

# FFmpegのバージョンを確認
ffmpeg -version

# motionEyeサービスの状態を確認
sudo systemctl status motioneye --no-pager

# motionEyeのログを確認
sudo journalctl -u motioneye -n 80 --no-pager

# ストレージの空き容量を確認
df -h

# Tailscaleの状態を確認
tailscale status

# Funnelの状態を確認
sudo tailscale funnel status

# Raspberry Piを再起動
sudo reboot

# Raspberry Piを安全にシャットダウン
sudo poweroff
```

---

# 16. 最終チェックリスト

- [ ] 64GB以上のmicroSDカードを準備した。
- [ ] USB microSDカードリーダーを準備した。
- [ ] Raspberry Pi 5に適切なUSB-C電源と冷却装置を用意した。
- [ ] Windows 11が大学のネットワークにつながっており、ネットワーク共有が許可されている。
- [ ] Windowsのモバイルホットスポットからスマートフォンがインターネットに接続できる。
- [ ] Raspberry Pi Imagerをインストールした。
- [ ] OSとして **Raspberry Pi OS (64-bit) デスクトップ版** を選び、Lite版を選んでいない。
- [ ] ImagerでRaspberry Pi Connectを有効にし、自分のRaspberry Pi IDと連携した。
- [ ] WindowsのブラウザーからRaspberry PiのScreen sharing画面を開ける。
- [ ] Hostname、ユーザー名、強いパスワード、Wi-Fi、タイムゾーン、予備用SSHを設定した。
- [ ] microSDカードへの書き込みが完了し、Windowsから安全に取り外した。
- [ ] Raspberry Piの電源を切った状態でmicroSDカードを入れた。
- [ ] DS-E12aをRaspberry PiのUSB端子に接続した。
- [ ] Raspberry Piを初めて起動できた。
- [ ] WindowsのブラウザーからRaspberry Pi Connectでデスクトップを操作できる。
- [ ] システムのCPUアーキテクチャが `aarch64` である。
- [ ] `lsusb` でカメラに関係するUSB機器を確認できた。
- [ ] `v4l2-ctl --list-devices` でカメラの映像デバイスを確認できた。
- [ ] カメラデバイスが対応する映像形式と解像度を確認できた。
- [ ] aptでMotion、FFmpeg、v4l-utilsをインストールした。
- [ ] motionEyeをインストールし、起動できた。
- [ ] motionEyeのAdminとSurveillanceの両方にパスワードを設定した。
- [ ] motionEyeにLocal V4L2 Cameraを追加した。
- [ ] ブラウザーにDS-E12aのライブ映像が表示された。
- [ ] Motion DetectionとMotion Triggered録画を有効にした。
- [ ] 録画ファイルを作成し、再生できた。
- [ ] microSDカードの空き容量を確認した。
- [ ] Tailscaleをインストールし、認証した。
- [ ] Funnelを有効にし、実際に表示されたHTTPSアドレスを保存した。
- [ ] スマートフォンのWi-Fiをオフにし、VPNを使わずにHTTPSアドレスへアクセスできた。
- [ ] Windowsのホットスポットや大学のネットワークが切れると、外部からのアクセスに影響することを理解した。
- [ ] Raspberry Piの再起動後、Raspberry Pi Connectのデスクトップ、motionEye、（有効にした場合は）Funnelが再び使えることを確認した。
- [ ] `sudo poweroff` による安全なシャットダウンとmicroSDカードの取り外し方法を確認した。

---

# 参考資料

1. Raspberry Pi公式ソフトウェア：https://www.raspberrypi.com/software/
2. Raspberry Pi OSダウンロード：https://www.raspberrypi.com/software/operating-systems/
3. Raspberry Pi Connect公式ドキュメント（画面共有、仮想ディスプレイ、ネットワーク外からの接続）：https://www.raspberrypi.com/documentation/services/connect.html
4. Raspberry Pi Imager公式カスタマイズ設定（Connect認証キー）：https://www.raspberrypi.com/documentation/computers/imager.html
5. Raspberry Pi Imager正式版リリース：https://github.com/raspberrypi/rpi-imager/releases
6. Microsoft Windowsモバイルホットスポットの説明：https://support.microsoft.com/en-us/windows/experience/connectivity-networking/use-your-windows-device-as-a-mobile-hotspot
7. Hikvision DS-E12 USBカメラ公式仕様：https://dealer-static.hikvision.com/upload/file/doc/DOC000065542-DS-E12.pdf
8. DS-E12a製品仕様（型番・仕様）：https://www.t955.com/wap/productshow.aspx?id=169717
9. motionEyeプロジェクト・インストール説明：https://github.com/motioneye-project/motioneye
10. motionEyeバージョン情報：https://pypi.org/project/motioneye/
11. motionEyeカメラのトラブルシューティング：https://github.com/motioneye-project/motioneye/wiki/Troubleshooting
12. Motion公式プロジェクト：https://motion-project.github.io/
13. Motionの音声録音に関するFAQ：https://github.com/Motion-Project/motion/wiki/FAQ
14. Tailscale Funnel公式説明：https://tailscale.com/docs/features/tailscale-funnel
15. Tailscale Funnelコマンド説明：https://tailscale.com/docs/reference/tailscale-cli/funnel
