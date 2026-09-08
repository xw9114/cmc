# 从 USB 摄像头到 OpenCV：树莓派实时视觉数据流的完整理解

在树莓派上用 USB 摄像头做视觉识别时，我们真正关心的并不只是“怎么把摄像头打开”，而是要理解一张图像从摄像头产生以后，究竟经过了哪些环节，最后为什么会变成 OpenCV 里的 `frame`。只有把这条数据链路真正串起来，后面遇到帧率上不去、延迟很大、MJPEG 不生效、缓存堆积这些问题时，才知道应该去查哪一层。

从整体上看，这条数据链路可以先写成：

```text
摄像头传感器
    ↓
采集原始像素数据（RAW / Bayer）
    ↓
摄像头内部 ISP 处理
    ↓
形成可以输出的视频数据
    ↓
选择输出格式
    ├─ YUYV：直接通过 USB 发送
    └─ MJPEG：摄像头内部 JPEG 编码后再通过 USB 发送
    ↓
树莓派 USB 控制器
    ↓
Linux USB / UVC 摄像头驱动
    ↓
V4L2
    ↓
GStreamer（可选的视频流处理层）
    ↓
OpenCV VideoCapture
    ↓
cap.read()
    ↓
frame
    ↓
HSV / 二值化 / 轮廓 / moments / 数字识别
    ↓
得到目标位置、颜色或其他视觉结果
```

这条流程里，每一层做的事情都不一样。摄像头负责“产生图像”，USB 负责“把数据传进树莓派”，V4L2 负责“让 Linux 程序以统一方式访问摄像头”，GStreamer 负责“管理视频数据流”，而 OpenCV 才真正负责“分析图像里面有什么”。

---

## 图像并不是一开始就是 MJPEG

摄像头内部首先是传感器。传感器感受到光以后得到的并不是我们平时看到的 JPEG 图片，也不是 OpenCV 的 BGR 图像，而是更接近 RAW / Bayer 这样的原始像素数据。

这些原始数据通常还需要经过摄像头内部 ISP（Image Signal Processor，图像信号处理器）的处理，例如去马赛克、曝光、白平衡以及颜色处理等。经过这些处理以后，摄像头才得到适合继续输出的视频图像。

到了这里，摄像头还需要决定“用什么格式把图像送给树莓派”。这正是我们实际编程时经常会控制的地方。

如果选择 YUYV，可以把它粗略理解成一种接近未压缩的视频像素格式。摄像头得到图像以后，就可以直接把 YUYV 数据通过 USB 发出去。

```text
传感器
   ↓
ISP
   ↓
YUYV
   ↓
USB
```

如果选择 MJPEG，摄像头会先在内部对每一帧做 JPEG 编码，然后再把压缩后的数据通过 USB 发送。

```text
传感器
   ↓
ISP
   ↓
JPEG 编码
   ↓
MJPEG
   ↓
USB
```

因此，“摄像头采完图就直接进行 MJPEG 压缩”并不是固定流程。更准确的说法应该是：

> 摄像头在形成可输出的视频图像以后，根据主机请求的输出模式，选择输出 YUYV，或者先编码成 MJPEG 再输出。

很多 USB 摄像头同时支持 YUYV 和 MJPEG，所以究竟走哪条路线，取决于摄像头支持什么，以及我们向它请求什么。

在 Linux 上可以使用：

```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

查看摄像头支持的格式、分辨率和帧率。例如一个摄像头可能出现：

```text
MJPG
    640x480 @ 60 FPS
    1280x720 @ 30 FPS

YUYV
    640x480 @ 30 FPS
```

这种情况下，同样是 `640×480`，MJPEG 可能可以达到更高帧率，其中一个重要原因就是 MJPEG 减少了 USB 上需要传输的数据量。

---

## MJPEG 为什么能够减轻 USB 压力

假设摄像头使用 `320×240 @ 60 FPS` 的 YUYV。YUYV 可以粗略按每个像素 2 字节估算，那么每秒的数据量大约为：

```text
320 × 240 × 2 × 60
≈ 9.2 MB/s
```

分辨率继续升高以后，这个数据量还会快速增加。

MJPEG 的思路则是让摄像头在数据进入 USB 之前就先完成 JPEG 压缩：

```text
原始视频帧
    ↓
摄像头内部 JPEG 编码
    ↓
体积更小的 MJPEG 数据
    ↓
USB
```

因此，MJPEG 并不是“让 USB 变快”，而是让同样数量的图像使用更少的 USB 数据量。

不过这种做法也有代价。YUYV 到了树莓派以后，本身已经是像素数据，不需要 JPEG 解压；MJPEG 到了树莓派以后仍然是压缩数据，OpenCV 如果要进行 HSV 转换、二值化、轮廓检测，就必须先把它解码成普通图像。

所以可以把两种方式理解成一种取舍：

```text
YUYV：
USB 带宽压力较大
JPEG 解码压力低

MJPEG：
USB 带宽压力较小
需要进行 JPEG 解码
```

这也是为什么实际性能不能只看摄像头标称帧率，还要同时看 USB 带宽、JPEG 解码速度以及后面的 OpenCV 算法速度。

---

## USB 把摄像头数据传到树莓派以后发生了什么

摄像头通过 USB 发出的数据不会直接进入 OpenCV。数据首先进入树莓派的 USB 主机控制器，然后由 Linux 的 USB 子系统和摄像头驱动接收。

普通 USB 摄像头通常遵循 UVC（USB Video Class）标准，因此 Linux 中常见的驱动是 `uvcvideo`。可以通过：

```bash
lsmod | grep uvc
```

查看系统是否加载了相关驱动。

摄像头在 USB 上传输时，一帧图像往往会被拆成多个 USB 数据包。Linux 驱动需要接收这些数据包并重新组织成视频帧。假设摄像头输出的是 MJPEG，那么这一层整理完成以后，得到的仍然是 MJPEG 压缩帧，并不会突然变成 OpenCV 的 BGR 图像。

可以把这一段理解成：

```text
摄像头产生一帧 MJPEG
        ↓
拆分成多个 USB 数据包
        ↓
通过 USB 传输
        ↓
树莓派 USB 控制器
        ↓
Linux USB / UVC 驱动
        ↓
重新组织成视频帧
```

到了这里，Linux 还需要给上层应用程序提供一种统一的方法来访问这个摄像头，于是 V4L2 就出现了。

---

## V4L2 是驱动和应用程序之间的接口

V4L2 全称是 **Video4Linux2**。它不是一种视频压缩格式，也不是 USB 传输协议，更不是一个专门“存放图像”的地方。

V4L2 更适合被理解为：

> Linux 给摄像头、采集卡等视频设备规定的一套统一访问接口。

USB 摄像头插到树莓派以后，通常会出现：

```text
/dev/video0
```

这个 `/dev/video0` 可以理解成 Linux 向用户空间暴露出来的一个视频设备节点。OpenCV、GStreamer 等程序可以通过 V4L2 去打开它、设置参数并取得视频帧。

因此整条关系更接近：

```text
USB 摄像头
    ↓
USB
    ↓
Linux UVC 驱动
    ↓
V4L2
    ↓
/dev/video0
    ↓
OpenCV / GStreamer
```

V4L2 可以用来控制或查询很多摄像头参数，例如分辨率、帧率、MJPEG / YUYV 格式、曝光、增益、白平衡、亮度以及对比度等。

这也解释了为什么我们在 OpenCV 里会看到：

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
```

这行代码并不是“读取了一张图片”，而是在告诉 OpenCV：

> 使用 V4L2 后端，打开编号为 0 的视频设备。

通常编号 `0` 对应 `/dev/video0`，编号 `1` 对应 `/dev/video1`。

---

## `VideoCapture()` 建立的是视频采集通道，而 `read()` 才是在取图像

理解 V4L2 以后，`cv2.VideoCapture()` 和 `cap.read()` 的区别就自然了。

当我们写：

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
```

OpenCV 会创建一个 `VideoCapture` 对象，并通过 V4L2 打开摄像头。这里的 `cap` 不是一张图片，它更像是程序和这路视频之间已经建立好的“采集通道”或“控制对象”。

接下来真正需要一帧图像时，才调用：

```python
ret, frame = cap.read()
```

这里的 `ret` 表示是否成功拿到图像，而 `frame` 才是真正交给 OpenCV 算法处理的图像。

所以程序通常是：

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)

while True:
    ret, frame = cap.read()

    if not ret:
        break

    cv2.imshow("camera", frame)
```

对应的数据关系就是：

```text
VideoCapture()
    ↓
打开视频设备并建立采集通道
    ↓
得到 cap
    ↓
cap.read()
    ↓
读取一帧
    ↓
frame
    ↓
下一次 cap.read()
    ↓
再读取一帧
    ↓
……
```

从 OpenCV 的接口上看，`read()` 还可以粗略理解成：

```python
cap.read()
≈
cap.grab()
+
cap.retrieve()
```

`grab()` 负责抓取或推进到一帧，`retrieve()` 再把这一帧取出来并形成程序可以使用的图像。平时直接使用 `read()`，只是 OpenCV 把这两步组合起来了。

---

## 为什么直接使用 V4L2 以后，还会需要 GStreamer

如果我们的要求只是“摄像头能打开，能够不断取得图像”，那么直接：

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
```

已经可以完成很多任务。

例如可以进一步请求 MJPEG、分辨率以及帧率：（这里V4L2是负责向摄像头请求，当他在树莓派上的时候就是提供数据的接口）

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)

cap.set(
    cv2.CAP_PROP_FOURCC,
    cv2.VideoWriter_fourcc(*"MJPG")
)

cap.set(cv2.CAP_PROP_FRAME_WIDTH, 320)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 240)
cap.set(cv2.CAP_PROP_FPS, 60)
```

问题在于，机器人视觉不仅关心“能不能读到图像”，还非常关心缓存了多少帧、如果算法跟不上怎么办、旧帧要不要丢、MJPEG 在哪里解码，以及最后以什么像素格式交给 OpenCV。

当我们希望更明确地控制这条视频数据流时，GStreamer 就有意义了。

GStreamer 可以理解成一个多媒体流水线框架。它本身并不是用来识别颜色、数字或者圆形目标的，而是负责把视频从输入端一步一步处理到输出端。

例如：

```text
读取摄像头
    ↓
确定输入格式
    ↓
控制缓存
    ↓
必要时丢帧
    ↓
JPEG 解码
    ↓
像素格式转换
    ↓
交给 OpenCV
```

因此，GStreamer 负责的是“视频数据怎么流动”，OpenCV 负责的是“图像里面有什么”。

---

## GStreamer 并没有取代 V4L2

这也是初学时非常容易混淆的一点。

直接使用 V4L2 时：

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
```

数据流可以理解成：

```text
摄像头
   ↓
Linux 驱动
   ↓
V4L2
   ↓
OpenCV
```

使用 GStreamer 时：

```python
cap = cv2.VideoCapture(pipeline, cv2.CAP_GSTREAMER)
```

如果 `pipeline` 里面使用：

```text
v4l2src device=/dev/video0
```

那么实际关系是：

```text
摄像头
   ↓
Linux 驱动
   ↓
V4L2
   ↓
GStreamer
   ↓
OpenCV
```

也就是说，`CAP_GSTREAMER` 不是在说“不用 V4L2 了”，而是在说：

> OpenCV 不再直接管理摄像头的数据流，而是把这件事交给 GStreamer；GStreamer 自己仍然可以通过 V4L2 去读取 `/dev/video0`。

因此三者可以这样理解：

```text
V4L2：
Linux 怎么访问摄像头

GStreamer：
视频数据怎么读取、缓存、解码、转换和传递

OpenCV：
拿到图像以后怎么做视觉识别
```

---

## 把一条 GStreamer Pipeline 当成一句完整的话来读

例如：

```python
pipeline = (
    "v4l2src device=/dev/video0 io-mode=2 ! "
    "image/jpeg,width=320,height=240,framerate=60/1 ! "
    "queue leaky=downstream max-size-buffers=1 ! "
    "jpegdec ! videoconvert ! "
    "video/x-raw,format=BGR ! "
    "appsink drop=true max-buffers=1 sync=false"
)

cap = cv2.VideoCapture(pipeline, cv2.CAP_GSTREAMER)
```

不要把这些参数看成十几个彼此无关的知识点。整条 Pipeline 其实是在描述一句完整的话：

> 通过 V4L2 从 `/dev/video0` 读取摄像头，请求它输出 `320×240 @ 60FPS` 的 MJPEG；视频进入树莓派以后只保留很少的缓存，如果下游来不及处理就允许丢掉旧数据；随后把 MJPEG 解码成原始图像，转换成 BGR，最后通过 `appsink` 交给 OpenCV，而且 OpenCV 来不及取时也不要让旧帧一直堆积。

顺着数据方向看就非常清楚：

```text
v4l2src
    ↓
image/jpeg, 320×240, 60FPS
    ↓
queue
    ↓
jpegdec
    ↓
videoconvert
    ↓
BGR
    ↓
appsink
    ↓
OpenCV
```

`v4l2src` 表示 GStreamer 通过 V4L2 打开 `/dev/video0`。`io-mode=2` 对应 V4L2 的内存映射方式（mmap），可以理解为 GStreamer 通过 V4L2 的缓冲区机制取得视频数据。

紧接着：

```text
image/jpeg,width=320,height=240,framerate=60/1
```

是在请求：

```text
MJPEG
320×240
60 FPS
```

这里要特别注意：写了 `60/1` 只代表“请求 60 FPS”，并不能保证摄像头一定会以 60 FPS 工作。摄像头本身必须支持这个格式、分辨率和帧率组合。

当 MJPEG 数据进入 GStreamer 后：

```text
queue leaky=downstream max-size-buffers=1
```

开始负责中间缓存。这里的目的并不是尽可能保存所有帧，而是尽量避免旧帧不断堆积。

随后：

```text
jpegdec
```

把 MJPEG 的 JPEG 压缩帧解码成普通视频图像。

解码完成以后，图像的像素格式不一定正好是 OpenCV 最方便使用的 BGR，所以继续经过：

```text
videoconvert
```

再通过：

```text
video/x-raw,format=BGR
```

明确要求输出 BGR 图像。

最后：

```text
appsink
```

是 GStreamer 流水线交给应用程序的出口。

于是完整关系变成：

```text
GStreamer
    ↓
appsink
    ↓
cv2.VideoCapture
    ↓
cap.read()
    ↓
frame
```

此时 OpenCV 得到 `frame` 以后，才开始真正的视觉算法：

```python
hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
mask = cv2.inRange(hsv, lower, upper)
```

再继续进行轮廓、霍夫圆、`moments`、中心坐标或者数字识别。

---

## 缓存为什么会直接影响机器人的实时性

现在再回来看 Pipeline 里的：

```text
queue leaky=downstream max-size-buffers=1
```

以及：

```text
appsink drop=true max-buffers=1 sync=false
```

就能理解这些参数为什么重要了。

假设摄像头是 60 FPS，也就是大约每 `16.67 ms` 产生一帧：

```text
F1  F2  F3  F4  F5  F6  F7 ...
```

而你的 OpenCV 算法处理一帧需要 30 ms，那么算法实际上只能达到大约 33 FPS。

这时候出现了典型的生产者和消费者速度不一致：

```text
摄像头：
60 FPS

算法：
约 33 FPS
```

如果中间允许大量缓存，真实世界可能已经到了 `F10`，程序却还在处理 `F6`：

```text
真实世界：
F10

缓存：
[F6][F7][F8][F9][F10]
 ↑
OpenCV 正在处理
```

这时候程序可能完全没有报错，画面也一直在更新，但机器人实际看到的是过去的画面。这就是视觉系统里非常危险的一种延迟。

对于录像来说，“每一帧都保留下来”可能很重要；但对于机器人来说，通常更重要的是：

> 我现在拿到的是不是尽可能新的画面？

因此实时机器人视觉经常愿意接受：

```text
摄像头：
F1 F2 F3 F4 F5 F6 F7

OpenCV：
F1 → F4 → F7
```

虽然中间的 `F2、F3、F5、F6` 被丢掉了，但每次处理的画面更接近现实世界的当前状态。

这就是 `leaky`、`drop=true`、`max-buffers=1` 这类参数存在的核心原因。

`sync=false` 的思路也类似。普通播放器需要按照时间戳正确播放视频，但机器人不是在“看电影”。机器人更关心的是尽快取得最新数据，所以通常不希望为了视频播放时钟而额外等待。

---

## OpenCV 自己也有缓存控制，但不一定像 GStreamer 那么明确

OpenCV 可以尝试：

```python
cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)
```

希望减少视频缓冲。

它的目标同样是避免：

```text
[F6][F7][F8][F9][F10]
```

大量排队，而尽可能接近：

```text
[F10]
```

但是 `CAP_PROP_BUFFERSIZE` 是否真正按照预期生效，会受到 OpenCV 后端以及驱动实现的影响。因此，如果实时性要求比较高，希望明确控制“缓存几帧、什么时候丢帧、在哪个位置丢帧”，GStreamer 往往更加容易把这些逻辑写清楚。

这也是 `CAP_V4L2` 和 `CAP_GSTREAMER` 最核心的实际区别。

```python
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
```

相当于：

```text
OpenCV
   ↓
直接通过 V4L2 读取摄像头
```

而：

```python
cap = cv2.VideoCapture(pipeline, cv2.CAP_GSTREAMER)
```

相当于：

```text
OpenCV
   ↓
从 GStreamer 的输出端取图
   ↑
GStreamer 负责读取、缓存、解码和转换
   ↑
V4L2
   ↑
摄像头
```

所以 GStreamer 并不是一定“更快”，它真正的优势是让视频数据流更加可控。

---

## 调试 60 FPS 时，不能只测程序循环速度

理解了整条数据流以后，还需要避免一个常见错误：把所有 FPS 都当成一个东西。

实际上，一套视觉系统至少可以区分：

```text
摄像头传感器实际采集 FPS
        ↓
USB / V4L2 实际获得 FPS
        ↓
GStreamer / OpenCV 取帧 FPS
        ↓
完整视觉算法 FPS
```

假设最后测到视觉算法只有 35 FPS，并不能立即说明摄像头没有达到 60 FPS。

完全可能是：

```text
摄像头           60 FPS
V4L2             60 FPS
OpenCV 读取      60 FPS
完整视觉算法     35 FPS
```

这时候瓶颈明显在算法层，例如数字识别、轮廓检测或者其他计算太耗时。

也可能是：

```text
摄像头请求       60 FPS
V4L2 实际        30 FPS
OpenCV 读取      30 FPS
完整视觉算法     30 FPS
```

这种情况下再怎么优化 `cvtColor()` 都没有意义，因为视频在进入 OpenCV 以前就已经只有 30 FPS 了。

因此实际调试时应该顺着数据链路逐层确认，而不是只在程序最外层放一个 FPS 计数器。

---

## 回到 Robocon 视觉系统，真正应该追求的是“新鲜的数据”

对于 Robocon 这种实时机器人任务，摄像头、GStreamer 和 OpenCV 最终并不是为了得到一段好看的视频，而是为了给机械臂、底盘或者控制系统提供尽可能及时的环境信息。

整个系统可以最终理解成：

```text
真实场景
    ↓
摄像头传感器
    ↓
ISP
    ↓
选择 YUYV / MJPEG 输出
    ↓
如果是 MJPEG，则摄像头内部 JPEG 编码
    ↓
USB
    ↓
树莓派 USB / UVC 驱动
    ↓
V4L2
    ↓
GStreamer
    ↓
控制缓存与丢帧
    ↓
MJPEG 解码
    ↓
转换成 BGR
    ↓
appsink
    ↓
OpenCV VideoCapture
    ↓
cap.read()
    ↓
frame
    ↓
BGR → HSV
    ↓
inRange
    ↓
轮廓 / Hough / moments / 数字识别
    ↓
目标中心坐标、颜色、数字
    ↓
ROS2 / 单片机 / 机械臂
```

沿着这条链路再回头看前面的几个概念，它们的关系就非常清楚了。

**MJPEG** 解决的是“摄像头以什么形式通过 USB 发送图像”，它通过压缩降低 USB 数据量，但树莓派后面需要解码。

**V4L2** 解决的是“Linux 程序怎样统一访问摄像头”，它位于摄像头驱动和 OpenCV / GStreamer 之间。

**GStreamer** 解决的是“视频数据进入树莓派以后，怎样读取、缓存、丢帧、解码、转换并交给应用程序”。

**`cv2.VideoCapture()`** 负责建立 OpenCV 的视频采集通道。

**`cap.read()`** 则是在已经建立好的采集通道上真正取得一帧。

**OpenCV** 最后才开始分析这一帧里面到底有什么。

如果以后遇到“摄像头明明设置了 60 FPS，程序为什么只有 30 FPS”“为什么机械臂看到的位置总是慢半拍”“为什么 MJPEG 比 YUYV 帧率高”“为什么使用 GStreamer 还要出现 `v4l2src`”这些问题，都可以重新回到这条数据链路上定位：问题究竟发生在摄像头输出、USB、V4L2、GStreamer、取帧，还是视觉算法这一层。
