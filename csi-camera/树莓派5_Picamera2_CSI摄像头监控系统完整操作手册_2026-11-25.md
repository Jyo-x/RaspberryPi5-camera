# 树莓派 5 + CSI 摄像头 + Picamera2 远程监控系统完整安装手册

**适用日期：2026 年 11 月 25 日**  
**操作环境：Windows 11 + Raspberry Pi 5 + Raspberry Pi OS Desktop 64-bit**  
**远程桌面：Raspberry Pi Connect**  
**摄像头控制：Picamera2（libcamera）**  
**网页监控：Picamera2 Web Monitor**  
**外网访问：Tailscale Funnel**

## 本手册的最终系统

本手册使用 Raspberry Pi 5、CSI 摄像头模块和一台 Windows 11 电脑，搭建可通过浏览器查看的远程监控系统。树莓派安装带图形桌面的 Raspberry Pi OS，日常通过 Raspberry Pi Connect 远程操作桌面。

系统包含以下功能：

1. 在 Windows 11 上通过 Raspberry Pi Connect 远程查看和操作树莓派桌面；
2. 使用 Picamera2 采集 CSI 摄像头画面；
3. 在浏览器中实时查看监控画面；
4. 手动开始和停止录像；
5. 根据画面变化进行移动侦测和自动录像；
6. 手动拍照，并在网页中查看录像和照片；
7. 自动清理超过 7 天的录像和照片；
8. 使用 Tailscale Funnel 从外网通过 HTTPS 访问监控网页。

移动侦测采用 OpenCV 的画面变化检测，不是 AI 人形识别。光线、阴影或自动曝光变化也可能触发录像。请先在局域网内完成测试，再开启公网访问。

## 摄像头与软件

本手册按 **Raspberry Pi Camera Module 3** 编写。摄像头通过 CSI 排线连接树莓派 5 的 `CAM/DISP0` 或 `CAM/DISP1` 接口。树莓派 5 需要使用一条 **Standard-Mini 摄像头排线**：摄像头端为标准 15-pin，树莓派 5 端为 mini 22-pin。

Picamera2 是控制摄像头的 Python 库。网页监控功能由本手册配套的 `picam_monitor.py` 程序提供。安装 Picamera2 时使用 Raspberry Pi OS 提供的 `apt` 软件包，以保持它与系统摄像头组件的版本匹配。

本手册配套文件：

- `picam_monitor.py`：网页监控程序；
- `picam-monitor.service`：开机自动运行配置；
- `picam-monitor.env.example`：网页登录密码配置示例。

---

## 一、开始之前：需要准备的配件

请在开始操作前准备以下硬件。

| 配件 | 要求 | 用途 |
|---|---|---|
| Raspberry Pi 5 | 已有 | 主机 |
| CSI 摄像头 | 建议 Raspberry Pi Camera Module 3 | 采集视频 |
| Standard-Mini 摄像头排线 | 15-pin 摄像头端 → 22-pin 树莓派 5 端 | 连接摄像头与树莓派 |
| microSD 卡 | 建议 64GB，最低 32GB | 操作系统和初期录像 |
| USB microSD 读卡器 | 电脑没有 microSD 卡槽时需要 | 将系统写入卡内 |
| USB-C 电源 | 推荐树莓派 5 官方 27W（5V/5A）电源 | 给树莓派供电 |
| 散热器 / 风扇 | 建议主动散热 | 长时间运行时帮助散热 |
| Windows 11 电脑 | 已有 | 烧录系统、开热点、远程操作 |

**特别注意：** Raspberry Pi 5 的相机接口是 mini 22-pin。普通 15-pin 对 15-pin 排线不能直接插入树莓派 5。购买时确认是树莓派 5 可用的 Standard-Mini（15-pin to 22-pin）排线。

树莓派官方摄像头连接说明：
https://www.raspberrypi.com/documentation/accessories/camera-software.html

## 二、在 Windows 11 开启移动热点

本手册继续使用 Windows 11 电脑的移动热点，让树莓派通过电脑连接校园网。

### 2.1 先连接校园网

1. 打开 Windows 11。
2. 先连接学校 Wi-Fi 或校园网有线网络。
3. 打开浏览器，确认电脑可以正常访问互联网。

如果校园网需要网页认证，先在 Windows 电脑上完成认证。

### 2.2 创建 Windows 移动热点

1. 打开 Windows **设置**。
2. 进入 **网络和 Internet → 移动热点**。
3. 在“共享我的 Internet 连接来自”中，选择当前可以上网的网卡。
4. 点击“属性”或“编辑”。
5. 设置以下内容：

| 项目 | 示例 |
|---|---|
| 网络名称 | `RPi5_Net` |
| 网络密码 | 自行设置，至少 8 位 |
| 网络频段 | 2.4 GHz（优先使用，便于初次连接） |

6. 点击保存。
7. 打开“移动热点”总开关。
8. 用手机或其他设备确认能够搜到该 Wi-Fi。

### 2.3 防止热点自动关闭

1. 在“移动热点”页面找到节能选项。
2. 关闭“没有设备连接时自动关闭移动热点”之类的节能功能（不同 Windows 版本文字可能不同）。
3. 进入 **设置 → 系统 → 电源和电池 → 屏幕和睡眠**。
4. 将“接通电源时，使设备进入睡眠状态”设置为“从不”。

**注意：** 使用电脑热点时，电脑必须保持开机、连接校园网并且不能休眠。电脑关机、断网或热点关闭后，树莓派也会失去互联网连接。

---

## 三、在 Windows 11 安装 Raspberry Pi Imager

### 3.1 下载

打开官方网页：

https://www.raspberrypi.com/software/

下载 Windows 版 Raspberry Pi Imager，安装并打开。

### 3.2 将 microSD 卡插入电脑

1. 树莓派此时不要通电。
2. 将 microSD 卡插入 USB 读卡器。
3. 将读卡器插入 Windows 电脑。
4. 如果 Windows 弹出“需要格式化磁盘”提示，点击**取消**，不要自行格式化。

### 3.3 选择设备、系统和存储卡

在 Raspberry Pi Imager 中依次选择：

1. **Choose Device：** `Raspberry Pi 5`。
2. **Choose OS：** `Raspberry Pi OS (64-bit)`，选择带桌面的版本，不要选择 `Lite`。
3. **Choose Storage：** 选择刚插入的 microSD 卡。

请仔细核对存储卡容量。写入系统会清除所选卡上的全部数据，绝对不要选择 Windows 系统盘或其他移动硬盘。

### 3.4 设置系统初始信息

打开自定义设置（OS Customisation / Edit Settings）。不同版本 Imager 的按钮名称和页面排列可能略有不同。

建议填写：

| 设置 | 内容 |
|---|---|
| Hostname | `pi-cam` |
| Username | `camadmin` |
| Password | 自行设置一个强密码 |
| Wi-Fi SSID | `RPi5_Net`（你刚才设置的热点名称） |
| Wi-Fi Password | Windows 热点密码 |
| WLAN Country | `JP` |
| Time Zone | `Asia/Tokyo` |
| Keyboard | `US`（若使用美式键盘） |

用户名和密码请自己记录。后续登录树莓派时会使用。

### 3.5 开启 SSH

在 Services / Remote Access 页面：

1. 勾选 `Enable SSH`。
2. 选择 `Use password authentication`。
3. 保存设置。

SSH 是备用远程管理入口。日常主要使用 Raspberry Pi Connect 图形桌面。

### 3.6 写入系统

1. 检查设备型号、系统、microSD 卡和自定义设置。
2. 点击 Write。
3. 确认清除存储卡。
4. 等待写入和验证全部完成。
5. 看到成功提示后点击 Finish。

写入期间不要拔出读卡器，也不要关闭电脑。

### 3.7 从 Windows 安全取出 microSD 卡

1. 等待 Imager 完成写入。
2. 在 Windows 任务栏中选择“安全删除硬件并弹出媒体”。
3. 弹出 USB 读卡器。
4. 拔下读卡器。
5. 从读卡器中取出 microSD 卡。

如果 Windows 再次询问是否格式化某个分区，请点击取消。

---

## 四、把 CSI 摄像头连接到树莓派 5

**此步骤必须在树莓派完全断电时进行。不要带电插拔 CSI 摄像头排线。**

### 4.1 断电

1. 如果树莓派已经通电，先执行安全关机（后文会说明）。
2. 等待关机完成。
3. 拔掉 USB-C 电源线。
4. 将树莓派放在干燥、平整的桌面上。

### 4.2 认识树莓派 5 的相机接口

树莓派 5 有两个相机 / 显示接口，标记为：

- `CAM/DISP0`
- `CAM/DISP1`

接口在靠近 micro-HDMI 和 Ethernet 的板边。任选一个连接摄像头即可。

### 4.3 连接排线到树莓派

1. 找到排线的 22-pin mini 一端，将它连接到树莓派 5。
2. 找到相机接口上的黑色或深色固定卡扣。
3. 轻轻将卡扣向外拉开一点，不要整个拔掉。
4. 将排线平直插入接口底部。
5. 按官方连接方式，使排线金属触点朝向远离卡扣的一侧。
6. 确认排线没有歪斜，然后把卡扣压回原位。

不要用力拉扯排线，也不要让排线急剧弯折。

### 4.4 连接排线到摄像头

1. 找到排线 15-pin 的标准端。
2. 找到摄像头板上的排线接口。
3. 轻轻打开接口卡扣。
4. 将排线平直插入，金属触点朝向摄像头电路板一侧。
5. 压回卡扣固定排线。
6. 检查排线是否插到底、是否歪斜。

如果摄像头镜头上有保护膜，请撕掉镜头保护膜。

### 4.5 插入 microSD 卡

确认树莓派仍然没有接电源。

1. 将树莓派翻到背面。
2. 找到 microSD 卡槽。
3. 按正确方向平直插入 microSD 卡。
4. 不要强行按压。

### 4.6 接通电源

1. 确认 CSI 排线已经固定。
2. 确认 microSD 卡已经插入。
3. 将树莓派 5 USB-C 电源接好。
4. 等待系统首次启动，建议等待 2～3 分钟。

本手册不要求连接实体显示器、键盘或鼠标。

---

## 五、获取树莓派 IP 地址并首次 SSH 登录

### 5.1 在 Windows 热点页面查找树莓派

1. 回到 Windows 11。
2. 打开 **设置 → 网络和 Internet → 移动热点**。
3. 找到“已连接的设备”。
4. 等待新设备出现。
5. 找到名称为 `pi-cam` 的设备。
6. 记录旁边的 IP 地址，例如 `192.168.137.25`。

这里的 IP 只是示例，实际地址以你的 Windows 页面为准。

### 5.2 打开 Windows Terminal

1. 按 `Win + X`。
2. 点击“终端”或“Windows Terminal”。
3. 打开 PowerShell 标签页。

输入以下命令，将 IP 替换为实际地址：

```powershell
ssh camadmin@192.168.137.25
```

第一次连接时，如果出现是否继续连接的提示，输入：

```text
yes
```

接着输入 Imager 中设置的树莓派密码。输入密码时屏幕不会显示字符，这是正常现象。输入完按 Enter。

成功后会看到类似：

```text
camadmin@pi-cam:~ $
```

这表示已经进入树莓派命令行。

### 5.3 如果无法连接

- 找不到设备：确认树莓派已经启动、Windows 热点已开启。
- `.local` 名称无法解析：使用 Windows 热点页面显示的 IP 地址。
- `Connection refused`：可能是 Imager 中没有打开 SSH。需要用显示器和键盘临时进入系统，或检查 SSH 设置。
- `Permission denied`：检查用户名和密码。

---

## 六、开启 Raspberry Pi Connect 图形桌面

本手册的日常操作界面是图形桌面。你会在 Windows 浏览器中看到树莓派桌面，并使用鼠标点击图标。

### 6.1 打开 Connect 服务

在刚才的 SSH 窗口中复制执行：

```bash
rpi-connect on
```

再执行：

```bash
rpi-connect signin
```

终端会显示一个以 `https://connect.raspberrypi.com/verify/` 开头的授权网址。

### 6.2 在 Windows 中授权

1. 复制 SSH 窗口显示的网址。
2. 在 Windows Edge 或 Chrome 中打开。
3. 登录 Raspberry Pi ID 账户。
4. 按网页提示确认设备绑定。
5. 完成后回到终端。

### 6.3 从 Windows 进入树莓派图形桌面

1. 打开 Windows 浏览器。
2. 访问：

   https://connect.raspberrypi.com/

3. 登录同一个 Raspberry Pi ID。
4. 进入 Devices 页面。
5. 找到 `pi-cam`。
6. 点击 `Connect via`。
7. 选择 `Screen sharing`。
8. 等待画面加载。

现在应该可以看到 Raspberry Pi OS 桌面。

如果没有连接 HDMI 显示器，Raspberry Pi OS 会为桌面创建虚拟显示器，Connect 可以将其显示在浏览器中。

### 6.4 重要设置

- 不要选择 Raspberry Pi OS Lite，因为 Lite 没有图形桌面。
- 不要关闭桌面自动登录，否则重启后可能没有已经运行的桌面会话供 Screen sharing 分享。
- 如果 Connect 显示不可用，在 SSH 窗口执行：

```bash
rpi-connect doctor
```

并根据输出检查 Wayland、Screen sharing 服务和网络。

官方说明：https://www.raspberrypi.com/documentation/services/connect.html

---

## 七、更新系统并安装 Picamera2 和监控程序依赖

之后大部分操作都可以在 Raspberry Pi Connect 的桌面中进行。

### 7.1 从桌面打开 Terminal

1. 在远程桌面中点击 Raspberry Pi 菜单。
2. 找到 Accessories / 附件。
3. 打开 Terminal。
4. 在终端中复制命令并按 Enter。

### 7.2 更新系统

```bash
sudo apt update
```

完成后执行：

```bash
sudo apt full-upgrade -y
```

等待系统更新结束。

### 7.3 安装相机与网页监控所需软件

```bash
sudo apt install -y python3-picamera2 python3-flask python3-opencv python3-numpy rpicam-apps
```

这些软件分别用于：

| 软件 | 作用 |
|---|---|
| `python3-picamera2` | 控制 CSI 摄像头 |
| `python3-flask` | 提供网页监控界面 |
| `python3-opencv` | 画面处理、移动侦测和录像文件写入 |
| `python3-numpy` | 图像数据处理依赖 |
| `rpicam-apps` | 官方命令行摄像头测试工具 |

Picamera2 请通过 Raspberry Pi OS 的 `apt` 软件包安装和更新，不要使用 `pip` 单独安装或升级，以保持它与系统中的 libcamera 组件版本一致。

### 7.4 检查 Picamera2 是否安装成功

```bash
python3 -c "from picamera2 import Picamera2; print('Picamera2 安装成功')"
```

如果显示：

```text
Picamera2 安装成功
```

说明 Python 已经能够导入 Picamera2。

---

## 八、检测 CSI 摄像头

### 8.1 列出摄像头

在树莓派桌面 Terminal 输入：

```bash
rpicam-hello --list-cameras
```

正常情况下会显示摄像头名称、型号或传感器信息。

如果显示没有摄像头，先不要继续。请检查：

1. 树莓派是否完全断电后再插过排线；
2. 排线是否插正、插到底；
3. 是否使用了 22-pin 到 15-pin 的正确排线；
4. CAM/DISP0 或 CAM/DISP1 接口卡扣是否压好；
5. 摄像头型号是否支持当前 libcamera 驱动。

### 8.2 打开摄像头预览

在 Raspberry Pi Connect 图形桌面的 Terminal 中执行：

```bash
rpicam-hello -t 5000
```

预期效果：显示约 5 秒的摄像头预览，然后自动关闭。

如果预览窗口正常出现，说明摄像头硬件和系统驱动已经基本正常。

### 8.3 拍摄一张测试照片

执行：

```bash
rpicam-still -o /home/camadmin/test.jpg
```

随后打开桌面上的 File Manager（文件管理器），进入 Home / `camadmin` 文件夹，双击 `test.jpg`。

确认能够看到照片后，继续下一步。

官方摄像头软件说明：https://www.raspberrypi.com/documentation/accessories/camera-software.html

---

## 九、下载本手册附带的网页监控程序

本手册配套提供两个文件：

- `picam_monitor.py`：网页监控程序；
- `picam-monitor.service`：让监控程序开机自动运行的服务配置。

先在 Windows 11 将本手册配套的 ZIP 压缩包解压。找到以下两个文件：

- `picam_monitor.py`
- `picam-monitor.service`

`picam-monitor.env.example` 是密码配置示例文件，后面会单独说明。接下来使用图形化 SFTP 软件 WinSCP，把前面两个文件传到树莓派。

### 9.1 安装 WinSCP

Windows 下载地址：

https://winscp.net/eng/download.php

下载并安装 WinSCP。

### 9.2 用 WinSCP 连接树莓派

1. 打开 WinSCP。
2. 文件协议选择 `SFTP`。
3. 主机名填写 Windows 移动热点页面中的树莓派 IP 地址，例如 `192.168.137.25`。
4. 端口填写 `22`。
5. 用户名填写 `camadmin`。
6. 密码填写树莓派系统密码。
7. 点击 Login。
8. 第一次连接时，确认并接受主机密钥。

WinSCP 左边是 Windows 文件，右边是树莓派文件。

### 9.3 上传两个文件

1. 在 WinSCP 左侧打开 Windows 的 Downloads / 下载 文件夹。
2. 在右侧进入：

```text
/home/camadmin
```

3. 将 `picam_monitor.py` 和 `picam-monitor.service` 拖到右侧窗口。
4. 等待两个文件传输完成。

完成后，树莓派的 `/home/camadmin` 文件夹中应该能看到这两个文件。

---

## 十、设置网页监控登录密码

网页监控会使用用户名和密码保护。不要使用简单密码，也不要把密码发给别人。

### 10.1 创建密码配置文件

在树莓派桌面打开 Terminal，输入：

```bash
nano /home/camadmin/picam-monitor.env
```

打开编辑器后输入两行：

```text
CAMERA_USER=viewer
CAMERA_PASSWORD=请在这里填写你自己的强密码
```

将第二行替换为你自己设置的长密码。建议至少 16 位，混合字母、数字和符号，不要照抄示例文字。

保存方法：

1. 按 `Ctrl + O`。
2. 按 Enter 保存。
3. 按 `Ctrl + X` 退出。

### 10.2 限制密码文件的读取权限

执行：

```bash
chmod 600 /home/camadmin/picam-monitor.env
```

只有 `camadmin` 用户可以读取该文件。

---

## 十一、安装并启动网页监控服务

### 11.1 复制 systemd 服务文件

执行：

```bash
sudo cp /home/camadmin/picam-monitor.service /etc/systemd/system/picam-monitor.service
```

本手册配套的服务文件使用账户 `camadmin`。如果你在 Imager 中使用了其他用户名，需要相应修改服务文件中的用户名和目录。

### 11.2 确认摄像头用户权限

执行：

```bash
groups camadmin
```

如果结果中没有 `video` 或 `render`，执行：

```bash
sudo usermod -aG video,render camadmin
```

然后重启树莓派：

```bash
sudo reboot
```

重新进入 Raspberry Pi Connect 桌面后继续。

### 11.3 设置开机自动启动

打开 Terminal，执行：

```bash
sudo systemctl daemon-reload
```

接着执行：

```bash
sudo systemctl enable --now picam-monitor
```

### 11.4 检查运行状态

执行：

```bash
sudo systemctl status picam-monitor --no-pager
```

正常时应看到：

```text
Active: active (running)
```

如果显示 `failed`，执行下面的命令查看原因：

```bash
sudo journalctl -u picam-monitor -n 80 --no-pager
```

常见原因包括：摄像头未识别、Python 软件包没装好、密码文件不存在、用户权限不足。

---

## 十二、在 Windows 浏览器打开监控网页

### 12.1 获取树莓派 IP

仍然在 Windows 11 的：

**设置 → 网络和 Internet → 移动热点 → 已连接的设备**

找到 `pi-cam`，查看它的 IP 地址。

### 12.2 打开网页

在 Windows Edge 或 Chrome 地址栏输入：

```text
http://192.168.137.25:8080
```

将示例 IP 替换成你的实际 IP。

浏览器会要求输入用户名和密码：

- 用户名：`viewer`（或你在密码文件里设置的名字）
- 密码：你设置的网页监控密码

成功登录后会看到 Picamera2 监控网页。

### 12.3 页面功能

网页包括：

| 功能 | 操作 |
|---|---|
| 实时视频 | 页面自动显示 |
| 开始手动录像 | 点击 `Start manual recording` |
| 停止手动录像 | 点击 `Stop manual recording` |
| 拍照 | 点击 `Take snapshot` |
| 开关移动侦测 | 点击 `Turn motion detection on/off` |
| 查看录像 | 点击 Recordings 下方文件名 |
| 查看照片 | 点击 Snapshots 下方文件名 |

自动移动侦测默认开启。程序检测到持续画面变化后开始录像；最后一次检测到移动后，继续录像约 10 秒再结束。

### 12.4 视频分辨率和帧率

配套程序的默认参数为：

| 参数 | 默认值 |
|---|---|
| 视频分辨率 | 1280 × 720 |
| 帧率 | 10 FPS |
| 网页 JPEG 质量 | 75 |
| 移动检测画面 | 320 × 180 |
| 移动后继续录像 | 10 秒 |
| 文件保留时间 | 7 天 |

如果网页卡顿，可以降低参数：

1. 在树莓派桌面打开 File Manager。
2. 进入 `/home/camadmin`。
3. 右键 `picam_monitor.py`，用 Text Editor 打开。
4. 找到文件开头的设置区。
5. 将 `WIDTH = 1280` 改成 `640`。
6. 将 `HEIGHT = 720` 改成 `480`。
7. 将 `FPS = 10` 改成 `5`。
8. 将 `JPEG_QUALITY = 75` 改成 `60`。
9. 保存文件。
10. 在 Terminal 执行：

```bash
sudo systemctl restart picam-monitor
```

之后刷新网页。

### 12.5 录像和照片保存位置

录像：

```text
/home/camadmin/PiCameraMonitor/recordings
```

照片：

```text
/home/camadmin/PiCameraMonitor/snapshots
```

程序会删除超过 7 天的录像和照片，并在剩余空间低于约 1.5GB 时停止新录像，避免存储卡被写满。

录像格式优先尝试 MP4（mp4v）；若系统 OpenCV 不支持该编码，会改用 AVI（MJPG）。如果 Windows 默认播放器无法打开 AVI，建议使用 VLC：

https://www.videolan.org/vlc/

---

## 十三、安装 Tailscale，让手机在外网直接打开监控网页

本阶段保留原来的 Tailscale Funnel 方案。手机不需要安装 Tailscale，使用手机浏览器和移动数据即可访问公开 HTTPS 地址。

**重要安全说明：** Funnel 会将网页服务暴露到公共互联网。任何知道网址的人都能尝试访问，因此必须保留网页用户名和强密码。不要把网址和密码同时公开，也不要把没有密码的摄像头页面放到 Funnel 上。

### 13.1 注册 Tailscale

在 Windows 浏览器打开：

https://tailscale.com/

使用个人账户注册并登录。

### 13.2 在树莓派安装 Tailscale

在树莓派 Terminal 输入：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

安装完成后执行：

```bash
sudo tailscale up
```

终端会显示一个登录网址。复制到 Windows 浏览器，使用刚才注册的 Tailscale 账户登录并授权这台树莓派。

### 13.3 开启 Funnel

确认网页监控服务正在运行后，执行：

```bash
sudo tailscale funnel --bg 8080
```

第一次启用时，可能会显示一个网页授权链接。复制链接到浏览器，按照页面提示启用 Funnel、MagicDNS 和 HTTPS 证书。

授权完成后，再执行一次：

```bash
sudo tailscale funnel --bg 8080
```

执行状态检查：

```bash
sudo tailscale funnel status
```

终端会显示类似下面的公网网址：

```text
https://pi-cam.example-tailnet.ts.net
```

实际网址以终端显示为准。请保存完整网址。

### 13.4 用手机测试

1. 手机关闭 Wi-Fi。
2. 使用 4G / 5G 移动数据。
3. 打开 Safari 或 Chrome。
4. 输入刚才 Tailscale Funnel 显示的 HTTPS 网址。
5. 输入网页监控用户名和密码。
6. 检查实时画面、录像和截图功能。

公网网址使用 HTTPS 标准端口，不要手动加 `:8080`。

Tailscale Funnel 官方说明：
https://tailscale.com/docs/features/tailscale-funnel

### 13.5 关闭公网访问

不需要公网访问时，在树莓派执行：

```bash
sudo tailscale funnel reset
```

这会关闭 Funnel 公网转发，但不会影响同一局域网内通过 `http://树莓派IP:8080` 访问监控网页。

Tailscale Funnel 有不可自行配置的带宽限制。若手机外网观看卡顿，先降低网页分辨率、帧率和 JPEG 质量。

---

## 十四、开机自动运行与断电恢复测试

### 14.1 检查监控程序是否开机启动

执行：

```bash
sudo systemctl is-enabled picam-monitor
```

正常结果：

```text
enabled
```

再执行：

```bash
sudo systemctl is-active picam-monitor
```

正常结果：

```text
active
```

### 14.2 重启测试

执行：

```bash
sudo reboot
```

等待树莓派重新启动，然后：

1. 在 Windows 打开 Raspberry Pi Connect。
2. 确认还能进入桌面。
3. 打开本地监控网页 `http://树莓派IP:8080`。
4. 确认画面正常。
5. 打开 Tailscale Funnel 网址，检查外网访问。

如果 Windows 电脑热点关闭，树莓派可能无法连接互联网，因此远程访问也会中断。树莓派本身仍可能继续启动，但需要重新连接网络。

---

## 十五、日常使用与正确关机

### 15.1 仅停止录像

网页中点击 `Stop manual recording`。如果仍然检测到移动，自动录像仍可能继续。

### 15.2 停止监控服务

在 Terminal 输入：

```bash
sudo systemctl stop picam-monitor
```

需要重新启动时：

```bash
sudo systemctl start picam-monitor
```

### 15.3 安全关机

在 Terminal 输入：

```bash
sudo poweroff
```

等待树莓派完成关机，再拔掉 USB-C 电源线。

### 15.4 什么时候拔 microSD 卡或摄像头排线？

只有需要更换系统卡或重新安装 CSI 摄像头时才拔。

顺序必须是：

1. 执行 `sudo poweroff`。
2. 等待树莓派完全关机。
3. 拔掉 USB-C 电源。
4. 再拔 microSD 卡或 CSI 排线。

**绝对不要在树莓派通电时插拔 CSI 摄像头排线。**

---

## 十六、常见问题排查

| 问题 | 检查方法 |
|---|---|
| 树莓派无法联网 | 检查 Windows 热点、校园网连接和热点密码 |
| Connect 看不到设备 | 检查 `rpi-connect status`，确认设备已授权 |
| Connect 没有 Screen sharing | 确认安装的是 Desktop 版而不是 Lite，执行 `rpi-connect doctor` |
| `rpicam-hello --list-cameras` 找不到摄像头 | 断电，重新检查 CSI 排线、插头方向和摄像头型号 |
| 预览窗口黑屏 | 检查摄像头排线、镜头保护膜和驱动支持 |
| Python 报 `No module named picamera2` | 执行 `sudo apt install -y python3-picamera2`，然后用系统 `python3` 运行 |
| 网页打不开 | 执行 `sudo systemctl status picam-monitor` |
| 网页显示 Camera error | 检查 CSI 摄像头是否识别，查看服务日志 |
| 录像未生成 | 查看网页状态、剩余空间和移动侦测开关 |
| 手机外网打不开 | 检查 Windows 热点、Tailscale 在线状态和 `sudo tailscale funnel status` |
| 远程画面卡顿 | 降低 `WIDTH`、`HEIGHT`、`FPS`、`JPEG_QUALITY` |

查看监控程序日志：

```bash
sudo journalctl -u picam-monitor -n 80 --no-pager
```

查看摄像头：

```bash
rpicam-hello --list-cameras
```

查看 Tailscale：

```bash
tailscale status
```

查看 Funnel：

```bash
sudo tailscale funnel status
```

查看存储空间：

```bash
df -h
```

---

## 十七、最终检查清单

- [ ] 已准备 CSI 摄像头模块。
- [ ] 已准备 22-pin to 15-pin Standard-Mini 排线。
- [ ] 已使用 Raspberry Pi Imager 烧录 Raspberry Pi OS 64-bit Desktop。
- [ ] 已设置 Wi-Fi、用户、密码和 SSH。
- [ ] 已将摄像头排线在断电状态下接好。
- [ ] 树莓派能连接 Windows 11 热点。
- [ ] Windows SSH 能登录树莓派。
- [ ] Raspberry Pi Connect 能显示树莓派图形桌面。
- [ ] `rpicam-hello --list-cameras` 能看到摄像头。
- [ ] `rpicam-hello -t 5000` 能显示预览画面。
- [ ] Picamera2 能正常导入。
- [ ] 网页监控程序服务状态为 `active (running)`。
- [ ] Windows 浏览器能看到实时画面。
- [ ] 手动录像和截图能正常保存。
- [ ] 移动侦测能触发自动录像。
- [ ] Tailscale Funnel 可以从手机移动网络访问。
- [ ] 重启后监控服务可以自动启动。
- [ ] 已设置强密码并了解 Funnel 的公网风险。

---

## 十八、系统结构与原理

```text
CSI 摄像头模块
      │ MIPI CSI 排线
      ▼
Raspberry Pi 5
      │
      ├── Raspberry Pi OS Desktop 64-bit
      │       └── Raspberry Pi Connect：远程图形桌面
      │
      ├── libcamera / Picamera2
      │       └── 采集图像和视频帧
      │
      ├── Picamera2 Web Monitor（本手册附带）
      │       ├── 实时 MJPEG 网页画面
      │       ├── 手动录像
      │       ├── 移动侦测录像
      │       └── 截图与录像文件列表
      │
      └── Tailscale Funnel
              └── HTTPS 公网访问

Windows 11
      ├── 校园网连接
      ├── 移动热点
      ├── Raspberry Pi Connect 浏览器
      └── WinSCP 文件传输
```

### 官方参考资料

1. Raspberry Pi 摄像头软件文档：https://www.raspberrypi.com/documentation/accessories/camera-software.html
2. Picamera2 官方手册：https://pip-assets.raspberrypi.com/categories/652-raspberry-pi-camera-module-2/documents/RP-008156-DS/picamera2-manual.pdf
3. Raspberry Pi Connect：https://www.raspberrypi.com/documentation/services/connect.html
4. Tailscale Funnel：https://tailscale.com/docs/features/tailscale-funnel
5. WinSCP：https://winscp.net/eng/download.php

**本手册的默认配置为 1280×720、10 FPS、移动触发录像。具体性能仍取决于摄像头型号、光照、树莓派散热、Windows 热点和校园网连接质量。请先完成局域网测试，再启用公网访问。**
