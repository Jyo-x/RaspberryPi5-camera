# Raspberry Pi 5 + CSIカメラ + Picamera2 遠隔監視システム 完全セットアップマニュアル

**対象日：2026年11月25日**  
**使用環境：Windows 11 + Raspberry Pi 5 + Raspberry Pi OS Desktop 64-bit**  
**リモートデスクトップ：Raspberry Pi Connect**  
**カメラ制御：Picamera2（libcamera）**  
**Web監視：Picamera2 Web Monitor**  
**外部ネットワークからのアクセス：Tailscale Funnel**

## このマニュアルで作るシステム

このマニュアルでは、Raspberry Pi 5、CSIカメラモジュール、Windows 11パソコンを使って、ブラウザから映像を確認できる監視システムを作ります。Raspberry Piには、画面を操作できるデスクトップ版のRaspberry Pi OSをインストールします。普段はRaspberry Pi Connectを使って、WindowsからRaspberry Piのデスクトップを遠隔操作します。

このシステムでできることは、次のとおりです。

1. Windows 11からRaspberry Pi Connectを使って、Raspberry Piのデスクトップを遠隔操作する。
2. Picamera2を使って、CSIカメラの映像を取り込む。
3. ブラウザでカメラのライブ映像を見る。
4. ボタンを押して録画を開始・停止する。
5. 映像の変化を検出して、自動で録画する。
6. 写真を撮影し、Webページから録画ファイルや写真を確認する。
7. 7日より古い録画ファイルと写真を自動で削除する。
8. Tailscale Funnelを使って、外出先からHTTPSで監視ページにアクセスする。

動体検知には、OpenCVによる映像の変化検出を使います。AIによる人物認識ではありません。明るさ、影、自動露出の変化でも録画が始まることがあります。まずは同じネットワーク内で十分にテストしてから、インターネットからのアクセスを有効にしてください。

## カメラとソフトウェア

このマニュアルは、**Raspberry Pi Camera Module 3** を使う前提で説明します。カメラはCSIケーブルでRaspberry Pi 5の `CAM/DISP0` または `CAM/DISP1` コネクタに接続します。Raspberry Pi 5では、**Standard-Miniカメラケーブル**が必要です。カメラ側は標準15ピン、Raspberry Pi 5側はmini 22ピンです。

Picamera2は、カメラを操作するためのPythonライブラリです。Web監視機能は、このマニュアルに付属する `picam_monitor.py` プログラムが担当します。Picamera2は、システムのカメラ関連ソフトウェアとバージョンを合わせるため、Raspberry Pi OSの `apt` パッケージからインストールします。

このマニュアルに付属するファイル：

- `picam_monitor.py`：Web監視プログラム
- `picam-monitor.service`：起動時に自動実行するための設定ファイル
- `picam-monitor.env.example`：Webログイン用パスワード設定のサンプル

---

## 1. 作業を始める前に：必要なもの

作業を始める前に、次の機器を準備してください。

| 部品 | 必要なもの・条件 | 用途 |
|---|---|---|
| Raspberry Pi 5 | 手元にあるもの | 本体 |
| CSIカメラ | Raspberry Pi Camera Module 3を推奨 | 映像の撮影 |
| Standard-Miniカメラケーブル | カメラ側15ピン → Raspberry Pi 5側22ピン | カメラとRaspberry Piの接続 |
| microSDカード | 64GBを推奨、最低32GB | OSと最初の録画データの保存 |
| USB microSDカードリーダー | パソコンにmicroSDスロットがない場合に必要 | OSを書き込むために使用 |
| USB-C電源 | Raspberry Pi 5公式27W（5V/5A）電源を推奨 | Raspberry Piへの給電 |
| ヒートシンク／ファン | 冷却ファン付きのものを推奨 | 長時間使用時の冷却 |
| Windows 11パソコン | 手元にあるもの | OSの書き込み、ホットスポット、遠隔操作 |

**重要：** Raspberry Pi 5のカメラコネクタはmini 22ピンです。一般的な15ピン同士のケーブルは、そのままではRaspberry Pi 5に接続できません。購入時に、Raspberry Pi 5で使えるStandard-Mini（15ピン to 22ピン）ケーブルであることを確認してください。

Raspberry Pi公式カメラ接続ガイド：  
https://www.raspberrypi.com/documentation/accessories/camera-software.html

## 2. Windows 11でモバイルホットスポットを設定する

このマニュアルでは、Windows 11パソコンのモバイルホットスポットを使い、Raspberry Piを大学のネットワークに接続します。

### 2.1 先に大学のネットワークへ接続する

1. Windows 11を起動します。
2. 大学のWi-Fi、または大学の有線ネットワークに接続します。
3. ブラウザを開き、インターネットに接続できることを確認します。

大学のネットワークでWeb認証が必要な場合は、先にWindowsパソコンで認証を完了してください。

### 2.2 Windowsモバイルホットスポットを作る

1. Windowsの **設定** を開きます。
2. **ネットワークとインターネット → モバイルホットスポット** を開きます。
3. 「共有するインターネット接続」で、現在インターネットに接続しているネットワークアダプターを選択します。
4. 「プロパティ」または「編集」をクリックします。
5. 次の内容を設定します。

| 項目 | 設定例 |
|---|---|
| ネットワーク名 | `RPi5_Net` |
| ネットワークパスワード | 自分で設定する。8文字以上 |
| ネットワーク帯域 | 2.4 GHz（最初の接続ではこちらを推奨） |

6. 保存をクリックします。
7. 「モバイルホットスポット」のスイッチをオンにします。
8. スマートフォンなど、別の機器からこのWi-Fiが見えることを確認します。

### 2.3 ホットスポットが自動でオフにならないようにする

1. 「モバイルホットスポット」のページで、省電力に関する設定を探します。
2. 「接続している機器がないときにモバイルホットスポットを自動的にオフにする」などの設定をオフにします（Windowsのバージョンによって表示が少し違います）。
3. **設定 → システム → 電源とバッテリー → 画面とスリープ** を開きます。
4. 「電源に接続しているとき、デバイスをスリープ状態にする」を「なし」にします。

**注意：** パソコンのホットスポットを使う場合、パソコンは電源を入れたままにし、大学のネットワークに接続し、スリープさせないでください。パソコンの電源が切れる、ネットワークが切れる、またはホットスポットがオフになると、Raspberry Piもインターネットに接続できなくなります。

---

## 3. Windows 11にRaspberry Pi Imagerをインストールする

### 3.1 ダウンロードする

公式サイトを開きます。

https://www.raspberrypi.com/software/

Windows版のRaspberry Pi Imagerをダウンロードし、インストールして起動します。

### 3.2 microSDカードをパソコンに挿入する

1. この時点では、Raspberry Piに電源を接続しないでください。
2. microSDカードをUSBカードリーダーに挿入します。
3. カードリーダーをWindowsパソコンに接続します。
4. Windowsに「ディスクをフォーマットする必要があります」と表示された場合は、**キャンセル**をクリックしてください。自分でフォーマットしないでください。

### 3.3 デバイス、OS、保存先を選択する

Raspberry Pi Imagerで、次の順番に選択します。

1. **Choose Device：** `Raspberry Pi 5`
2. **Choose OS：** `Raspberry Pi OS (64-bit)`。デスクトップがあるバージョンを選びます。`Lite`は選ばないでください。
3. **Choose Storage：** 今パソコンに接続したmicroSDカードを選びます。

ストレージの容量をよく確認してください。OSを書き込むと、選択したカードのデータはすべて消去されます。Windowsのシステムドライブや、別のUSBハードディスクを絶対に選ばないでください。

### 3.4 初期設定を入力する

カスタマイズ設定（OS Customisation / Edit Settings）を開きます。Imagerのバージョンによって、ボタン名や画面の順番が少し違う場合があります。

次の内容を入力してください。

| 設定項目 | 入力内容 |
|---|---|
| Hostname | `pi-cam` |
| Username | `camadmin` |
| Password | 自分で強いパスワードを設定 |
| Wi-Fi SSID | `RPi5_Net`（さきほど設定したホットスポット名） |
| Wi-Fi Password | Windowsホットスポットのパスワード |
| WLAN Country | `JP` |
| Time Zone | `Asia/Tokyo` |
| Keyboard | `US`（US配列キーボードを使う場合） |

ユーザー名とパスワードは必ずメモしてください。後でRaspberry Piにログインするときに使います。

### 3.5 SSHを有効にする

Services / Remote Accessの画面で、次の設定をします。

1. `Enable SSH` にチェックを入れます。
2. `Use password authentication` を選びます。
3. 設定を保存します。

SSHは、もう一つの遠隔管理方法です。普段はRaspberry Pi Connectのグラフィカルなデスクトップを使います。

### 3.6 OSを書き込む

1. デバイス、OS、microSDカード、カスタマイズ設定を確認します。
2. `Write` をクリックします。
3. microSDカードのデータを消去することを確認します。
4. 書き込みと検証が両方終わるまで待ちます。
5. 完了メッセージが表示されたら `Finish` をクリックします。

書き込み中はカードリーダーを抜いたり、パソコンをシャットダウンしたりしないでください。

### 3.7 WindowsからmicroSDカードを安全に取り外す

1. Imagerの書き込みが完了するまで待ちます。
2. Windowsのタスクバーから「ハードウェアを安全に取り外してメディアを取り出す」を選びます。
3. USBカードリーダーを取り出します。
4. カードリーダーをパソコンから抜きます。
5. microSDカードをカードリーダーから取り出します。

Windowsがもう一度パーティションのフォーマットを求めてきた場合も、キャンセルしてください。

---

## 4. CSIカメラをRaspberry Pi 5に接続する

**この作業は、Raspberry Piの電源が完全に切れている状態で行ってください。CSIカメラケーブルを電源が入ったまま抜き差ししないでください。**

### 4.1 電源を切る

1. Raspberry Piの電源が入っている場合は、先に安全な方法でシャットダウンします（後の章で説明します）。
2. シャットダウンが完了するまで待ちます。
3. USB-C電源ケーブルを抜きます。
4. Raspberry Piを乾燥した平らな机の上に置きます。

### 4.2 Raspberry Pi 5のカメラコネクタを確認する

Raspberry Pi 5には、カメラ／ディスプレイ用のコネクタが2つあります。次の表示を探してください。

- `CAM/DISP0`
- `CAM/DISP1`

コネクタはmicro-HDMIとEthernetポートに近い基板の端にあります。どちらか一方を使えば大丈夫です。

### 4.3 ケーブルをRaspberry Pi側に接続する

1. ケーブルのmini 22ピン側を用意します。
2. カメラコネクタにある黒色または濃い色の固定クリップを探します。
3. クリップを少しだけ外側に引きます。完全に取り外さないでください。
4. ケーブルをまっすぐ、コネクタの奥まで差し込みます。
5. 公式の接続方法に従い、ケーブルの金属接点を固定クリップと反対側に向けます。
6. ケーブルが斜めになっていないことを確認してから、クリップを元の位置に戻します。

ケーブルを強く引っ張らないでください。また、急な角度で折り曲げないでください。

### 4.4 ケーブルをカメラ側に接続する

1. ケーブルの標準15ピン側を用意します。
2. カメラ基板のケーブルコネクタを探します。
3. コネクタの固定クリップをゆっくり開きます。
4. ケーブルをまっすぐ差し込み、金属接点をカメラ基板側に向けます。
5. クリップを元に戻してケーブルを固定します。
6. ケーブルが奥まで入り、斜めになっていないことを確認します。

カメラのレンズに保護フィルムが付いている場合は、はがしてください。

### 4.5 microSDカードを挿入する

Raspberry Piにまだ電源が接続されていないことを確認してください。

1. Raspberry Piを裏返します。
2. microSDカードスロットを探します。
3. microSDカードを正しい向きで、まっすぐ差し込みます。
4. 無理に押し込まないでください。

### 4.6 電源を入れる

1. CSIケーブルが固定されていることを確認します。
2. microSDカードが挿入されていることを確認します。
3. Raspberry Pi 5のUSB-C電源を接続します。
4. OSが最初に起動するまで待ちます。2～3分ほど待つことをおすすめします。

このマニュアルでは、Raspberry Piに実際のモニター、キーボード、マウスを接続する必要はありません。

---

## 5. Raspberry PiのIPアドレスを確認し、初めてSSHログインする

### 5.1 Windowsのホットスポット画面でRaspberry Piを探す

1. Windows 11に戻ります。
2. **設定 → ネットワークとインターネット → モバイルホットスポット** を開きます。
3. 「接続されているデバイス」を探します。
4. 新しいデバイスが表示されるまで待ちます。
5. `pi-cam` という名前のデバイスを探します。
6. その横に表示されているIPアドレスをメモします。例：`192.168.137.25`

このIPアドレスは例です。実際のアドレスはWindowsの画面に表示されたものを使ってください。

### 5.2 Windows Terminalを開く

1. `Win + X` を押します。
2. メニューから「ターミナル」または「Windows Terminal」をクリックします。
3. PowerShellのタブを開きます。

次のコマンドを入力します。IPアドレスは自分の実際のアドレスに置き換えてください。

```powershell
ssh camadmin@192.168.137.25
```

初めて接続するときに、接続を続けるかどうか聞かれた場合は、次のように入力します。

```text
yes
```

次に、Imagerで設定したRaspberry Piのパスワードを入力します。入力中、画面には文字が表示されません。これは正常です。パスワードを入力してEnterを押してください。

接続に成功すると、次のような表示が出ます。

```text
camadmin@pi-cam:~ $
```

これはRaspberry Piのコマンド画面にログインできたという意味です。

### 5.3 接続できない場合

- デバイスが見つからない：Raspberry Piが起動しているか、Windowsホットスポットがオンになっているか確認します。
- `.local` の名前で接続できない：Windowsホットスポット画面に表示されたIPアドレスを使います。
- `Connection refused`：ImagerでSSHを有効にしていない可能性があります。モニターとキーボードを一時的に接続して確認するか、SSH設定を確認してください。
- `Permission denied`：ユーザー名とパスワードを確認してください。

---

## 6. Raspberry Pi Connectでグラフィカルデスクトップを開く

このマニュアルでは、普段の操作にグラフィカルデスクトップを使います。WindowsのブラウザにRaspberry Piのデスクトップを表示し、マウスでアイコンをクリックできます。

### 6.1 Connectサービスを有効にする

先ほどのSSH画面で、次のコマンドをコピーして実行します。

```bash
rpi-connect on
```

続けて、次を実行します。

```bash
rpi-connect signin
```

ターミナルに、`https://connect.raspberrypi.com/verify/` で始まる認証用URLが表示されます。

### 6.2 Windowsで認証する

1. SSH画面に表示されたURLをコピーします。
2. WindowsのEdgeまたはChromeで開きます。
3. Raspberry Pi IDアカウントでログインします。
4. Webページの案内に従って、デバイスの登録を許可します。
5. 完了したらターミナルに戻ります。

### 6.3 WindowsからRaspberry Piのデスクトップを開く

1. Windowsのブラウザを開きます。
2. 次のページにアクセスします。

   https://connect.raspberrypi.com/

3. 同じRaspberry Pi IDでログインします。
4. Devicesページを開きます。
5. `pi-cam` を探します。
6. `Connect via` をクリックします。
7. `Screen sharing` を選びます。
8. 画面が表示されるまで待ちます。

これでRaspberry Pi OSのデスクトップが見えるはずです。

HDMIモニターを接続していない場合、Raspberry Pi OSはデスクトップ用の仮想ディスプレイを作成します。Connectは、その画面をブラウザに表示できます。

### 6.4 重要な設定

- `Lite`ではなく、デスクトップ版のRaspberry Pi OSを選んでください。Liteにはグラフィカルデスクトップがありません。
- デスクトップの自動ログインを無効にしないでください。再起動後にScreen sharingで共有するデスクトップセッションがない場合があります。
- Connectが使えない場合は、SSH画面で次のコマンドを実行します。

```bash
rpi-connect doctor
```

表示された内容を確認し、Wayland、Screen sharingサービス、ネットワークを確認してください。

公式説明：  
https://www.raspberrypi.com/documentation/services/connect.html

---

## 7. OSを更新し、Picamera2と監視プログラムに必要なソフトをインストールする

ここからの多くの操作は、Raspberry Pi Connectで開いたデスクトップから行えます。

### 7.1 デスクトップからTerminalを開く

1. リモートデスクトップでRaspberry Piのメニューをクリックします。
2. Accessories / アクセサリを探します。
3. Terminalを開きます。
4. ターミナルにコマンドをコピーし、Enterを押します。

### 7.2 OSを更新する

```bash
sudo apt update
```

完了したら、次を実行します。

```bash
sudo apt full-upgrade -y
```

システムの更新が終わるまで待ちます。

### 7.3 カメラとWeb監視に必要なソフトをインストールする

```bash
sudo apt install -y python3-picamera2 python3-flask python3-opencv python3-numpy rpicam-apps
```

それぞれのソフトの役割は次のとおりです。

| ソフト | 役割 |
|---|---|
| `python3-picamera2` | CSIカメラを操作する |
| `python3-flask` | Web監視画面を提供する |
| `python3-opencv` | 画像処理、動体検知、録画ファイルの作成 |
| `python3-numpy` | 画像データの処理に使うライブラリ |
| `rpicam-apps` | 公式のコマンドラインカメラテストツール |

Picamera2は、Raspberry Pi OSの `apt` パッケージからインストール・更新してください。`pip` を使って別にインストールしたり、アップグレードしたりしないでください。システム内のlibcamera関連ソフトとのバージョンを合わせるためです。

### 7.4 Picamera2が正しくインストールされたか確認する

```bash
python3 -c "from picamera2 import Picamera2; print('Picamera2 インストール成功')"
```

次のように表示されたら成功です。

```text
Picamera2 インストール成功
```

PythonからPicamera2を読み込めるという意味です。

---

## 8. CSIカメラを確認する

### 8.1 カメラ一覧を表示する

Raspberry PiのデスクトップにあるTerminalで、次を入力します。

```bash
rpicam-hello --list-cameras
```

正常な場合、カメラ名、型番、またはセンサー情報が表示されます。

カメラが見つからない場合は、先に進まないでください。次の点を確認します。

1. Raspberry Piの電源を完全に切ってから、ケーブルを挿し直したか。
2. ケーブルが正しい向きで、奥まで入っているか。
3. 22ピンから15ピンへの正しいケーブルを使っているか。
4. CAM/DISP0またはCAM/DISP1のコネクタの固定クリップが正しく閉じているか。
5. カメラの型番が、現在のlibcameraドライバーに対応しているか。

### 8.2 カメラのプレビューを表示する

Raspberry Pi ConnectのデスクトップにあるTerminalで、次を実行します。

```bash
rpicam-hello -t 5000
```

正常なら、約5秒間カメラのプレビューが表示され、その後自動で閉じます。

プレビュー画面が正常に表示されたら、カメラ本体とシステムドライバーは基本的に正常です。

### 8.3 テスト写真を撮る

次を実行します。

```bash
rpicam-still -o /home/camadmin/test.jpg
```

次に、デスクトップのFile Manager（ファイルマネージャー）を開きます。Home / `camadmin` フォルダーに移動し、`test.jpg` をダブルクリックします。

写真が表示されることを確認してから、次の作業に進みます。

公式カメラソフトウェア説明：  
https://www.raspberrypi.com/documentation/accessories/camera-software.html

---

## 9. このマニュアルに付属するWeb監視プログラムを転送する

このマニュアルには、次の2つのファイルが付属しています。

- `picam_monitor.py`：Web監視プログラム
- `picam-monitor.service`：起動時に自動実行するためのサービス設定

最初にWindows 11で、このマニュアルに付属するZIPファイルを展開します。次の2つのファイルを探してください。

- `picam_monitor.py`
- `picam-monitor.service`

`picam-monitor.env.example` はパスワード設定のサンプルファイルです。これは後で別に説明します。ここでは、SFTPソフトのWinSCPを使って、上の2つのファイルをRaspberry Piに転送します。

### 9.1 WinSCPをインストールする

Windowsのダウンロードページ：

https://winscp.net/eng/download.php

WinSCPをダウンロードしてインストールします。

### 9.2 WinSCPでRaspberry Piに接続する

1. WinSCPを開きます。
2. ファイルプロトコルで `SFTP` を選びます。
3. ホスト名に、Windowsのモバイルホットスポット画面で確認したRaspberry PiのIPアドレスを入力します。例：`192.168.137.25`
4. ポート番号に `22` を入力します。
5. ユーザー名に `camadmin` を入力します。
6. パスワードにRaspberry Piのシステムパスワードを入力します。
7. `Login` をクリックします。
8. 初めて接続するときは、ホストキーを確認して受け入れます。

WinSCPの左側はWindowsのファイル、右側はRaspberry Piのファイルです。

### 9.3 2つのファイルをアップロードする

1. WinSCPの左側で、WindowsのDownloads / ダウンロードフォルダーを開きます。
2. 右側で、次のフォルダーを開きます。

```text
/home/camadmin
```

3. `picam_monitor.py` と `picam-monitor.service` を右側のウィンドウへドラッグします。
4. 2つのファイルの転送が終わるまで待ちます。

転送が終わったら、Raspberry Piの `/home/camadmin` フォルダーに2つのファイルが見えることを確認します。

---

## 10. Web監視ページのログインパスワードを設定する

Web監視ページには、ユーザー名とパスワードを設定します。簡単なパスワードを使わないでください。また、パスワードを他の人に教えないでください。

### 10.1 パスワード設定ファイルを作る

Raspberry PiのデスクトップでTerminalを開き、次を入力します。

```bash
nano /home/camadmin/picam-monitor.env
```

エディターが開いたら、次の2行を入力します。

```text
CAMERA_USER=viewer
CAMERA_PASSWORD=ここに自分で設定した強いパスワードを入力
```

2行目は、自分で設定した長いパスワードに置き換えてください。16文字以上をおすすめします。英字、数字、記号を組み合わせてください。サンプルの文字をそのまま使わないでください。

保存方法：

1. `Ctrl + O` を押します。
2. Enterを押して保存します。
3. `Ctrl + X` を押して終了します。

### 10.2 パスワードファイルの読み取り権限を制限する

次を実行します。

```bash
chmod 600 /home/camadmin/picam-monitor.env
```

これで、`camadmin` ユーザーだけがこのファイルを読み取れるようになります。

---

## 11. Web監視サービスをインストールして起動する

### 11.1 systemdサービスファイルをコピーする

次を実行します。

```bash
sudo cp /home/camadmin/picam-monitor.service /etc/systemd/system/picam-monitor.service
```

このマニュアルに付属するサービスファイルは、ユーザー名 `camadmin` を使います。Imagerで別のユーザー名を設定した場合は、サービスファイル内のユーザー名とフォルダーの場所も変更する必要があります。

### 11.2 カメラを使うためのユーザー権限を確認する

次を実行します。

```bash
groups camadmin
```

結果に `video` または `render` がない場合は、次を実行します。

```bash
sudo usermod -aG video,render camadmin
```

その後、Raspberry Piを再起動します。

```bash
sudo reboot
```

Raspberry Pi Connectでデスクトップにもう一度接続してから、次の作業に進みます。

### 11.3 起動時の自動実行を設定する

Terminalを開き、次を実行します。

```bash
sudo systemctl daemon-reload
```

続けて、次を実行します。

```bash
sudo systemctl enable --now picam-monitor
```

### 11.4 動作状態を確認する

次を実行します。

```bash
sudo systemctl status picam-monitor --no-pager
```

正常な場合、次の表示が出ます。

```text
Active: active (running)
```

`failed` と表示された場合は、次のコマンドで原因を確認します。

```bash
sudo journalctl -u picam-monitor -n 80 --no-pager
```

よくある原因は、カメラが認識されていない、Pythonソフトが正しくインストールされていない、パスワードファイルがない、ユーザー権限が足りない、などです。

---

## 12. Windowsのブラウザで監視ページを開く

### 12.1 Raspberry PiのIPアドレスを確認する

Windows 11で、次のページを開きます。

**設定 → ネットワークとインターネット → モバイルホットスポット → 接続されているデバイス**

`pi-cam` を探し、IPアドレスを確認します。

### 12.2 Webページを開く

WindowsのEdgeまたはChromeのアドレス欄に、次のように入力します。

```text
http://192.168.137.25:8080
```

例のIPアドレスを、自分のRaspberry PiのIPアドレスに置き換えてください。

ブラウザにユーザー名とパスワードを聞かれます。

- ユーザー名：`viewer`（パスワードファイルで自分が設定した名前）
- パスワード：自分で設定したWeb監視用パスワード

ログインに成功すると、Picamera2の監視ページが表示されます。

### 12.3 ページの機能

| 機能 | 操作 |
|---|---|
| ライブ映像 | ページに自動で表示される |
| 手動録画を開始 | `Start manual recording` をクリック |
| 手動録画を停止 | `Stop manual recording` をクリック |
| 写真を撮る | `Take snapshot` をクリック |
| 動体検知をオン／オフ | `Turn motion detection on/off` をクリック |
| 録画を見る | Recordingsの下にあるファイル名をクリック |
| 写真を見る | Snapshotsの下にあるファイル名をクリック |

動体検知は初期設定でオンです。プログラムが一定時間続く画面の変化を検出すると録画を開始します。最後に動きを検出してから約10秒間録画を続け、その後終了します。

### 12.4 動画の解像度とフレームレート

付属プログラムの初期設定は次のとおりです。

| 項目 | 初期設定 |
|---|---|
| 動画の解像度 | 1280 × 720 |
| フレームレート | 10 FPS |
| Web JPEG画質 | 75 |
| 動体検知用の画像 | 320 × 180 |
| 動きを検出した後の録画時間 | 10秒 |
| ファイルの保存期間 | 7日 |

Webページの表示が遅い場合は、次のように設定を下げます。

1. Raspberry PiのデスクトップでFile Managerを開きます。
2. `/home/camadmin` に移動します。
3. `picam_monitor.py` を右クリックし、Text Editorで開きます。
4. ファイルの最初にある設定部分を探します。
5. `WIDTH = 1280` を `640` に変更します。
6. `HEIGHT = 720` を `480` に変更します。
7. `FPS = 10` を `5` に変更します。
8. `JPEG_QUALITY = 75` を `60` に変更します。
9. ファイルを保存します。
10. Terminalで次を実行します。

```bash
sudo systemctl restart picam-monitor
```

その後、ブラウザのページを再読み込みします。

### 12.5 録画と写真の保存場所

録画ファイル：

```text
/home/camadmin/PiCameraMonitor/recordings
```

写真ファイル：

```text
/home/camadmin/PiCameraMonitor/snapshots
```

プログラムは7日より古い録画ファイルと写真を削除します。また、空き容量が約1.5GBより少なくなると、新しい録画を停止します。microSDカードの容量がいっぱいになるのを防ぐためです。

録画形式は、最初にMP4（mp4v）を試します。システムのOpenCVがこのコーデックに対応していない場合は、AVI（MJPG）に切り替えます。Windowsの標準プレーヤーでAVIファイルを再生できない場合は、VLCの利用をおすすめします。

https://www.videolan.org/vlc/

---

## 13. Tailscaleをインストールし、外出先のスマートフォンから監視ページを開く

この章では、元のTailscale Funnel方式を使います。スマートフォンにTailscaleをインストールしなくても、スマートフォンのブラウザとモバイルデータ通信で公開HTTPSアドレスにアクセスできます。

**重要なセキュリティ注意：** Funnelを有効にすると、Webサービスがインターネット上に公開されます。URLを知っている人は誰でもアクセスを試すことができます。そのため、Webページのユーザー名と強いパスワードを必ず設定してください。URLとパスワードを同時に公開しないでください。パスワードのないカメラページをFunnelで公開しないでください。

### 13.1 Tailscaleに登録する

Windowsのブラウザで次のページを開きます。

https://tailscale.com/

個人アカウントを使って登録し、ログインします。

### 13.2 Raspberry PiにTailscaleをインストールする

Raspberry PiのTerminalで、次を入力します。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

インストールが終わったら、次を実行します。

```bash
sudo tailscale up
```

ターミナルにログイン用URLが表示されます。そのURLをWindowsのブラウザにコピーし、先ほど登録したTailscaleアカウントでログインして、Raspberry Piを許可します。

### 13.3 Funnelを有効にする

Web監視サービスが動いていることを確認してから、次を実行します。

```bash
sudo tailscale funnel --bg 8080
```

初回の設定では、Web認証用のリンクが表示される場合があります。そのリンクをブラウザで開き、画面の案内に従ってFunnel、MagicDNS、HTTPS証明書を有効にします。

認証が完了したら、もう一度次を実行します。

```bash
sudo tailscale funnel --bg 8080
```

状態を確認します。

```bash
sudo tailscale funnel status
```

ターミナルに、次のような公開URLが表示されます。

```text
https://pi-cam.example-tailnet.ts.net
```

実際に使うURLはターミナルに表示されたものです。完全なURLを保存してください。

### 13.4 スマートフォンでテストする

1. スマートフォンのWi-Fiをオフにします。
2. 4G / 5Gのモバイルデータ通信を使います。
3. SafariまたはChromeを開きます。
4. Tailscale Funnelに表示されたHTTPSのURLを入力します。
5. Web監視用のユーザー名とパスワードを入力します。
6. ライブ映像、録画、写真の機能を確認します。

公開URLはHTTPSの標準ポートを使います。URLの最後に `:8080` を自分で追加しないでください。

Tailscale Funnel公式説明：  
https://tailscale.com/docs/features/tailscale-funnel

### 13.5 インターネットからの公開を停止する

外部からアクセスする必要がなくなった場合は、Raspberry Piで次を実行します。

```bash
sudo tailscale funnel reset
```

これでFunnelによる外部公開は停止します。ただし、同じローカルネットワーク内から `http://Raspberry PiのIPアドレス:8080` で監視ページを開くことはできます。

Tailscale Funnelには、自分で変更できない通信速度の制限があります。スマートフォンからの映像が遅い場合は、Webページの解像度、フレームレート、JPEG画質を下げてください。

---

## 14. 自動起動と再起動後の動作を確認する

### 14.1 監視プログラムが自動起動に設定されているか確認する

次を実行します。

```bash
sudo systemctl is-enabled picam-monitor
```

正常な結果：

```text
enabled
```

続けて、次を実行します。

```bash
sudo systemctl is-active picam-monitor
```

正常な結果：

```text
active
```

### 14.2 再起動してテストする

次を実行します。

```bash
sudo reboot
```

Raspberry Piが再起動するまで待ち、次の項目を確認します。

1. WindowsでRaspberry Pi Connectを開きます。
2. デスクトップに再接続できることを確認します。
3. ローカルの監視ページ `http://Raspberry PiのIPアドレス:8080` を開きます。
4. 映像が正常に表示されることを確認します。
5. Tailscale FunnelのURLを開き、外部ネットワークからアクセスできることを確認します。

Windowsパソコンのホットスポットがオフになると、Raspberry Piがインターネットに接続できなくなる場合があります。そのため遠隔アクセスもできなくなります。Raspberry Pi自体は起動を続けている場合がありますが、ネットワークに再接続する必要があります。

---

## 15. 普段の使い方と正しいシャットダウン

### 15.1 録画だけを停止する

Webページで `Stop manual recording` をクリックします。ただし、映像に動きがあり、動体検知がオンの場合は、自動録画が続くことがあります。

### 15.2 監視サービスを停止する

Terminalで次を入力します。

```bash
sudo systemctl stop picam-monitor
```

もう一度起動する場合は、次を実行します。

```bash
sudo systemctl start picam-monitor
```

### 15.3 安全にシャットダウンする

Terminalで次を入力します。

```bash
sudo poweroff
```

Raspberry Piのシャットダウンが完了するまで待ってから、USB-C電源ケーブルを抜いてください。

### 15.4 microSDカードやカメラケーブルを抜くとき

microSDカードを交換する場合やCSIカメラを取り付け直す場合だけ、カードやケーブルを抜いてください。

必ず次の順番で行います。

1. `sudo poweroff` を実行します。
2. Raspberry Piが完全にシャットダウンするまで待ちます。
3. USB-C電源を抜きます。
4. その後でmicroSDカードまたはCSIケーブルを抜きます。

**Raspberry Piの電源が入っているときにCSIカメラケーブルを抜き差ししないでください。**

---

## 16. よくある問題と確認方法

| 問題 | 確認すること |
|---|---|
| Raspberry Piがネットワークに接続できない | Windowsホットスポット、大学ネットワーク、ホットスポットのパスワードを確認する |
| Connectでデバイスが見えない | `rpi-connect status` を確認し、デバイスが認証済みか確認する |
| ConnectにScreen sharingが表示されない | LiteではなくDesktop版か確認し、`rpi-connect doctor` を実行する |
| `rpicam-hello --list-cameras` でカメラが見つからない | 電源を切って、CSIケーブル、コネクタの向き、カメラの型番を確認する |
| プレビュー画面が黒い | カメラケーブル、レンズ保護フィルム、ドライバー対応を確認する |
| Pythonで `No module named picamera2` と表示される | `sudo apt install -y python3-picamera2` を実行し、システムの `python3` で実行する |
| Webページを開けない | `sudo systemctl status picam-monitor` を実行する |
| WebページにCamera errorと表示される | CSIカメラが認識されているか確認し、サービスログを見る |
| 録画ファイルができない | Webページの状態、空き容量、動体検知の設定を確認する |
| スマートフォンから外部アクセスできない | Windowsホットスポット、Tailscaleの接続状態、`sudo tailscale funnel status` を確認する |
| リモート映像が遅い | `WIDTH`、`HEIGHT`、`FPS`、`JPEG_QUALITY` の値を下げる |

監視プログラムのログを確認する：

```bash
sudo journalctl -u picam-monitor -n 80 --no-pager
```

カメラを確認する：

```bash
rpicam-hello --list-cameras
```

Tailscaleを確認する：

```bash
tailscale status
```

Funnelを確認する：

```bash
sudo tailscale funnel status
```

ストレージの空き容量を確認する：

```bash
df -h
```

---

## 17. 最終チェックリスト

- [ ] CSIカメラモジュールを準備した。
- [ ] 22ピン to 15ピン Standard-Miniケーブルを準備した。
- [ ] Raspberry Pi ImagerでRaspberry Pi OS 64-bit Desktopを書き込んだ。
- [ ] Wi-Fi、ユーザー名、パスワード、SSHを設定した。
- [ ] 電源を切った状態でカメラケーブルを接続した。
- [ ] Raspberry PiがWindows 11のホットスポットに接続できる。
- [ ] WindowsからSSHでRaspberry Piにログインできる。
- [ ] Raspberry Pi ConnectでRaspberry Piのデスクトップが表示される。
- [ ] `rpicam-hello --list-cameras` でカメラが見つかる。
- [ ] `rpicam-hello -t 5000` でプレビューが表示される。
- [ ] Picamera2を正常に読み込める。
- [ ] Web監視プログラムのサービスが `active (running)` になっている。
- [ ] Windowsのブラウザでライブ映像が見える。
- [ ] 手動録画と写真の保存が正常にできる。
- [ ] 動体検知で自動録画が始まる。
- [ ] Tailscale Funnelを使い、スマートフォンのモバイルデータ通信からアクセスできる。
- [ ] 再起動後に監視サービスが自動で起動する。
- [ ] 強いパスワードを設定し、Funnelで公開する危険性を理解した。

---

## 18. システム構成と仕組み

```text
CSIカメラモジュール
      │ MIPI CSIケーブル
      ▼
Raspberry Pi 5
      │
      ├── Raspberry Pi OS Desktop 64-bit
      │       └── Raspberry Pi Connect：リモートデスクトップ
      │
      ├── libcamera / Picamera2
      │       └── 画像と動画フレームを取り込む
      │
      ├── Picamera2 Web Monitor（このマニュアルの付属プログラム）
      │       ├── MJPEG形式のライブ映像
      │       ├── 手動録画
      │       ├── 動体検知による録画
      │       └── 写真と録画ファイルの一覧
      │
      └── Tailscale Funnel
              └── HTTPSで外部からアクセス

Windows 11
      ├── 大学ネットワークへの接続
      ├── モバイルホットスポット
      ├── ブラウザでRaspberry Pi Connectを利用
      └── WinSCPでファイルを転送
```

### 公式参考資料

1. Raspberry Piカメラソフトウェアの説明：https://www.raspberrypi.com/documentation/accessories/camera-software.html
2. Picamera2公式マニュアル：https://pip-assets.raspberrypi.com/categories/652-raspberry-pi-camera-module-2/documents/RP-008156-DS/picamera2-manual.pdf
3. Raspberry Pi Connect：https://www.raspberrypi.com/documentation/services/connect.html
4. Tailscale Funnel：https://tailscale.com/docs/features/tailscale-funnel
5. WinSCP：https://winscp.net/eng/download.php

**このマニュアルの初期設定は1280×720、10 FPS、動体検知による録画です。実際の性能は、カメラの型番、明るさ、Raspberry Piの冷却状態、Windowsホットスポット、大学ネットワークの接続状態によって変わります。まずローカルネットワーク内でテストしてから、インターネットからのアクセスを有効にしてください。**
