# Raspberry Pi 5 Camera

[中文](#中文说明) | [日本語](#日本語)

---

## 中文说明

### 项目简介

本仓库整理了在 **Raspberry Pi 5** 上搭建远程摄像头监控系统所需的操作手册、程序和安装文件。目前提供两种摄像头方案：

- **CSI 摄像头方案**：使用 Raspberry Pi Camera Module 3 等 CSI 摄像头，通过 Picamera2 采集画面。
- **USB 摄像头方案**：使用海康威视 DS-E12a USB 摄像头，通过 Motion 和 motionEye 实现监控。

两种方案均以 Raspberry Pi OS Desktop 64-bit 为基础，并可通过 Raspberry Pi Connect 远程操作树莓派桌面。手机从外部网络直接访问监控网页的功能使用 Tailscale Funnel，属于可选配置。

### 选择摄像头方案

| 项目 | CSI 摄像头 | USB 摄像头 |
|---|---|---|
| 摄像头示例 | Raspberry Pi Camera Module 3 | 海康威视 DS-E12a |
| 连接方式 | CSI 排线（Raspberry Pi 5 需要 Standard-Mini 15-pin to 22-pin 排线） | USB-A |
| 摄像头控制 | Picamera2 / libcamera | V4L2 / Motion |
| 监控网页 | Picamera2 Web Monitor（Flask） | motionEye |
| 本地网页端口 | `8080` | `8765` |
| 初始视频设置 | 1280×720，10 FPS | 1280×720，10 FPS |
| 主要功能 | 实时画面、手动录像、移动侦测录像、拍照、媒体文件列表 | 实时画面、移动侦测、事件录像 |
| 远程访问 | Tailscale Funnel（可选） | Tailscale Funnel（可选） |

> **如何选择：** 使用树莓派专用 CSI 摄像头时，请阅读 `csi-camera/` 下的手册；使用 DS-E12a 等 USB 摄像头时，请阅读 `usb-camera/` 下的手册。两套方案的软件和配置不同，请按对应手册操作，不要混用安装命令。

### 目录结构

```text
RaspberryPi5-camera/
├── README.md
│
├── csi-camera/
│   ├── 树莓派5_Picamera2_CSI摄像头监控系统完整操作手册_2026-11-25.md
│   ├── Raspberry_Pi_5_Picamera2_CSIカメラ遠隔監視システム_完全操作マニュアル_2026-11-25.md
│   ├── 树莓派5_Picamera2_监控系统完整安装包_Web監視プログラム_2026-11-25.zip
│   └── other/
│       └── rpi_camera_surveillance_system.py
│
└── usb-camera/
    ├── 树莓派5_DS-E12a监控系统_远程图形桌面版_2026-11-25.md
    └── Raspberry_Pi_5_DS-E12a_監視システム_リモートデスクトップ版_2026-11-25.md
```

### 操作手册

#### 1. CSI 摄像头 + Picamera2

- [中文操作手册](csi-camera/树莓派5_Picamera2_CSI摄像头监控系统完整操作手册_2026-11-25.md)
- [日本語操作マニュアル](csi-camera/Raspberry_Pi_5_Picamera2_CSIカメラ遠隔監視システム_完全操作マニュアル_2026-11-25.md)
- [Picamera2 网页监控程序安装包（ZIP）](csi-camera/树莓派5_Picamera2_监控系统完整安装包_Web監視プログラム_2026-11-25.zip)

CSI 方案的主要功能：

- 使用 Picamera2 采集 CSI 摄像头画面。
- 在浏览器中查看 MJPEG 实时视频。
- 手动开始和停止录像。
- 开启或关闭基于画面变化的移动侦测录像。
- 拍摄照片，并通过网页查看照片和录像。
- 自动清理超过 7 天的录像和照片。
- 使用 HTTP Basic Authentication 保护监控网页。

网页默认端口为 `8080`，初始视频设置为 1280×720、10 FPS。具体设置、密码配置、程序上传和开机自启动方法请参阅对应操作手册。

安装包内包含：

- `picam_monitor.py`：Picamera2 网页监控程序。
- `picam-monitor.service`：systemd 开机启动配置。
- `picam-monitor.env.example`：网页登录信息配置示例。
- 中文操作手册。

`csi-camera/other/rpi_camera_surveillance_system.py` 是基于旧版 `picamera` 库的参考示例，不属于当前 Picamera2 网页监控方案的安装流程。当前方案请使用上方 ZIP 安装包中的 `picam_monitor.py`。

#### 2. USB 摄像头 + Motion / motionEye

- [中文操作手册](usb-camera/树莓派5_DS-E12a监控系统_远程图形桌面版_2026-11-25.md)
- [日本語セットアップガイド](usb-camera/Raspberry_Pi_5_DS-E12a_監視システム_リモートデスクトップ版_2026-11-25.md)

USB 方案以海康威视 DS-E12a 为示例，主要内容包括：

- 通过 USB 连接摄像头，并使用 V4L2 检查设备和支持的视频格式。
- 安装 Motion、FFmpeg 和 motionEye。
- 在 motionEye 中添加本地 USB 摄像头。
- 设置实时画面、移动侦测和移动触发录像。
- 通过浏览器管理监控画面和录像。
- 可选配置 Tailscale Funnel，从外部网络访问监控网页。

motionEye 默认网页端口为 `8765`，推荐先以 1280×720、10 FPS 测试，再根据设备和网络状况调整。**本手册中的 Motion/motionEye 方案不提供同步音频录制功能**，请将其视为视频监控系统。

### 通用环境

- Raspberry Pi 5
- Raspberry Pi OS Desktop 64-bit
- Windows 11 电脑（用于烧录系统和远程操作）
- microSD 卡（建议 64 GB 或更大）
- 稳定的 USB-C 电源和散热设备
- 可用的网络连接

手册以 Windows 11 的移动热点共享校园网为例；如果使用家用路由器，也可以在系统设置时直接连接路由器 Wi-Fi。若树莓派通过 Windows 电脑的热点联网，Windows 电脑需要保持开机、联网且不进入睡眠状态，否则树莓派的外部网络连接可能中断。

### 远程访问与安全注意事项

- **Raspberry Pi Connect** 用于远程查看和操作树莓派桌面。
- **Tailscale Funnel** 用于从外部网络通过 HTTPS 直接访问监控网页，是可选功能。
- 启用 Funnel 前，必须先设置强度足够且未在其他网站重复使用的监控网页密码。
- Funnel 会将网页入口发布到公网。不要公开分享访问地址和密码，也不要在未设置密码时启用公网访问。
- 不需要外网访问时，请按照对应手册关闭 Funnel。
- 移动侦测是基于画面变化的检测，不等同于 AI 人体识别；光线、阴影或自动曝光变化也可能造成误触发。

### 使用建议

1. 先根据摄像头类型选择 CSI 或 USB 方案。
2. 按照对应语言的完整操作手册，从系统安装开始逐步配置。
3. 先在本地网络中确认实时画面、录像和登录验证正常。
4. 确认密码和安全设置后，再考虑启用 Tailscale Funnel。
5. 完成后执行一次重启测试，确认监控服务可以正常恢复。

---

## 日本語

### プロジェクト概要

このリポジトリには、**Raspberry Pi 5** を使った遠隔カメラ監視システムの構築に必要なマニュアル、プログラム、インストールファイルをまとめています。現在、次の2種類のカメラ構成に対応しています。

- **CSIカメラ構成：** Raspberry Pi Camera Module 3などのCSIカメラをPicamera2で制御します。
- **USBカメラ構成：** ハイクビジョン DS-E12a USBカメラをMotionとmotionEyeで監視します。

どちらの構成もRaspberry Pi OS Desktop 64-bitを使用します。Raspberry Pi Connectでデスクトップを遠隔操作できます。スマートフォンなどから外部ネットワーク経由で監視ページに直接アクセスする場合は、オプションとしてTailscale Funnelを利用できます。

### カメラ構成の選択

| 項目 | CSIカメラ | USBカメラ |
|---|---|---|
| カメラの例 | Raspberry Pi Camera Module 3 | ハイクビジョン DS-E12a |
| 接続方法 | CSIケーブル（Raspberry Pi 5ではStandard-Mini 15-pin to 22-pinが必要） | USB-A |
| カメラ制御 | Picamera2 / libcamera | V4L2 / Motion |
| 監視ページ | Picamera2 Web Monitor（Flask） | motionEye |
| ローカル接続ポート | `8080` | `8765` |
| 初期映像設定 | 1280×720、10 FPS | 1280×720、10 FPS |
| 主な機能 | ライブ映像、手動録画、動体検知録画、写真撮影、メディア一覧 | ライブ映像、動体検知、イベント録画 |
| 外部からのアクセス | Tailscale Funnel（任意） | Tailscale Funnel（任意） |

> **選び方：** Raspberry Pi専用のCSIカメラを使う場合は `csi-camera/` のマニュアルを、DS-E12aなどのUSBカメラを使う場合は `usb-camera/` のマニュアルを参照してください。2つの構成では使用するソフトウェアや設定が異なるため、別の構成向けのコマンドを混在させないでください。

### ディレクトリ構成

```text
RaspberryPi5-camera/
├── README.md
│
├── csi-camera/
│   ├── 中国語版 Picamera2 CSIカメラ操作マニュアル
│   ├── 日本語版 Picamera2 CSIカメラ操作マニュアル
│   ├── Picamera2 Web Monitor インストールパッケージ（ZIP）
│   └── other/
│       └── rpi_camera_surveillance_system.py
│
└── usb-camera/
    ├── 中国語版 DS-E12a 監視システムマニュアル
    └── 日本語版 DS-E12a 監視システムマニュアル
```

### セットアップマニュアル

#### 1. CSIカメラ + Picamera2

- [中文操作手册](csi-camera/树莓派5_Picamera2_CSI摄像头监控系统完整操作手册_2026-11-25.md)
- [日本語操作マニュアル](csi-camera/Raspberry_Pi_5_Picamera2_CSIカメラ遠隔監視システム_完全操作マニュアル_2026-11-25.md)
- [Picamera2 Web Monitor インストールパッケージ（ZIP）](csi-camera/树莓派5_Picamera2_监控系统完整安装包_Web監視プログラム_2026-11-25.zip)

CSIカメラ構成では、次の機能を利用できます。

- Picamera2によるCSIカメラ映像の取得。
- ブラウザーでのMJPEGライブ映像表示。
- 手動録画の開始・停止。
- 画面変化を利用した動体検知録画の有効化・無効化。
- 写真撮影と、Webページからの写真・録画ファイルの確認。
- 7日より古い録画ファイルと写真の自動削除。
- HTTP Basic AuthenticationによるWebページの保護。

Webページの標準ポートは `8080`、初期映像設定は1280×720、10 FPSです。詳細な設定、パスワードの設定、プログラムの転送、自動起動の方法は対応するマニュアルを参照してください。

インストールパッケージには、次のファイルが含まれています。

- `picam_monitor.py`：Picamera2 Web監視プログラム。
- `picam-monitor.service`：systemd自動起動設定。
- `picam-monitor.env.example`：Webログイン情報の設定例。
- 中国語版の操作マニュアル。

`csi-camera/other/rpi_camera_surveillance_system.py` は旧版の `picamera` ライブラリを使った参考サンプルであり、現在のPicamera2 Web Monitorのインストール手順では使用しません。現在の構成では、上記ZIPに含まれる `picam_monitor.py` を使用してください。

#### 2. USBカメラ + Motion / motionEye

- [中文操作手册](usb-camera/树莓派5_DS-E12a监控系统_远程图形桌面版_2026-11-25.md)
- [日本語セットアップガイド](usb-camera/Raspberry_Pi_5_DS-E12a_監視システム_リモートデスクトップ版_2026-11-25.md)

USBカメラ構成では、ハイクビジョン DS-E12aを例に、次の内容を扱います。

- USBカメラの接続と、V4L2によるデバイス・対応映像形式の確認。
- Motion、FFmpeg、motionEyeのインストール。
- motionEyeへのローカルUSBカメラの追加。
- ライブ映像、動体検知、動体検知時の録画設定。
- ブラウザーからの監視画面と録画ファイルの管理。
- オプションとしてTailscale Funnelを設定し、外部ネットワークから監視ページにアクセス。

motionEyeの標準Webポートは `8765` です。まず1280×720、10 FPSで動作を確認し、カメラやネットワークの状態に合わせて調整してください。**このマニュアルのMotion/motionEye構成では、映像と音声を同期して録音する機能は扱いません。** 音声なしの映像監視システムとして使用してください。

### 共通の動作環境

- Raspberry Pi 5
- Raspberry Pi OS Desktop 64-bit
- Windows 11パソコン（OSの書き込みとリモート操作に使用）
- microSDカード（64 GB以上を推奨）
- 安定したUSB-C電源と冷却装置
- インターネット接続

マニュアルでは、Windows 11のモバイルホットスポットを使って大学のネットワークを共有する方法を例として説明しています。自宅のルーターを使う場合は、OSの初期設定時にルーターのWi-Fiへ接続してもかまいません。Windowsパソコンのホットスポット経由でRaspberry Piを接続する場合、Windowsパソコンを起動したまま、ネットワークに接続し、スリープさせないでください。Windows側の接続が切れると、Raspberry Piから外部ネットワークへ接続できなくなる場合があります。

### リモートアクセスとセキュリティ

- **Raspberry Pi Connect：** Raspberry Piのデスクトップを遠隔から表示・操作するために使用します。
- **Tailscale Funnel：** 外部ネットワークからHTTPSで監視ページへ直接アクセスするための任意機能です。
- Funnelを有効にする前に、推測されにくく、他のサービスで使い回していないパスワードを設定してください。
- Funnelを使うと監視ページの入口がインターネット上に公開されます。URLとパスワードを一緒に公開せず、パスワード未設定の状態で公開しないでください。
- 外部からアクセスする必要がなくなった場合は、各マニュアルの手順に従ってFunnelを停止してください。
- 動体検知は画面の変化を検出する方式であり、AIによる人物認識ではありません。照明や影、自動露出の変化でも誤検知する場合があります。

### 推奨する導入手順

1. カメラの種類に合わせてCSIまたはUSBの構成を選びます。
2. 対応する言語のセットアップマニュアルに従い、OSのインストールから設定します。
3. まずローカルネットワーク内でライブ映像、録画、ログイン認証を確認します。
4. パスワードとセキュリティ設定を確認してから、必要に応じてTailscale Funnelを有効にします。
5. 最後にRaspberry Piを再起動し、監視サービスが正常に再起動することを確認します。

---

## 注意 / Notes

このREADMEはリポジトリ内のマニュアルとプログラムの概要を示すものです。実際の配線、インストールコマンド、ユーザー名、パスワード設定、トラブル対応については、使用するカメラ構成に対応した詳細マニュアルを優先してください。
