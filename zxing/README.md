# ZXing 源码解析

> 排行榜 #9 · 类别：条码扫描

## 一句话定位

ZXing（"Zebra Crossing"）是 Java 实现的条码/二维码编解码库：解码侧把图片转成**亮度源 → 二值化位矩阵 → 定位/采样 → 纠错解码 → Result**；编码侧用 `Writer` 把内容渲染成 `BitMatrix`。Android 上常见的扫码 App 都是它的封装。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/zxing/zxing |
| 快照 commit | `16ef4322b619b02d716c4ada40b0fefd719168d9` |
| 克隆方式 | `--depth 1`（仓库较大，只看 core 模块） |

模块：`core/`（纯 Java 编解码核心）、`javase/`（桌面端封装）、`android/`（已拆分出去的历史模块）、`android-core/`（相机辅助）。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `MultiFormatReader` | 多格式解码入口：按提示依次尝试各 Reader | `MultiFormatReader.java` L38、L82、L169 |
| `QRCodeReader` | 二维码专用 Reader：提取位矩阵、调 Detector、Decoder | `QRCodeReader.java` L44、L68、L95、L118 |
| `Detector` | 定位二维码三个 finder pattern，做透视变换与采样 | `Detector.java` L39、L75 |
| `Decoder` | Reed-Solomon 纠错解码，还原字节与文本 | `Decoder.java` L36、L76 |
| `BitMatrix` | 位矩阵：黑白点阵的紧凑表示 | `BitMatrix.java` L36、L58 |
| `BinaryBitmap` | 位图源 + Binarizer 的组合，按需生成 BitMatrix | `BinaryBitmap.java` L28、L78 |
| `Binarizer` / `HybridBinarizer` | 二值化抽象 / 局部自适应阈值实现 | `Binarizer.java` L30、L56；`HybridBinarizer.java` L40、L62 |
| `LuminanceSource` | 灰度亮度源抽象（相机 YUV / BufferedImage / RGB） | `LuminanceSource.java` L28、L50 |
| `Result` | 解码结果：文本、原始字节、格式、位置点 | `Result.java` L27、L53 |
| `MultiFormatWriter` | 多格式编码入口 | `MultiFormatWriter.java` L42、L53 |

## 关键流程

### 1. 发起解码

`MultiFormatReader.decodeWithState(binaryBitmap)` 按 `hints` 里声明的可能格式（或全部）依次尝试。以二维码为例：`QRCodeReader.decode` 先尝试 `extractPureBits`（图像本身就是干净的码图时直接取位矩阵），失败再走 `Detector`。

### 2. 亮度源 → 二值化

`LuminanceSource` 把图片/相机帧转成灰度数组；`HybridBinarizer` 用分块局部阈值（每块估算黑点比例）生成 `BitMatrix`，对光照不均、阴影更鲁棒。

### 3. 定位与采样

`Detector.detect()` 找三个 finder pattern，确定版本与方向，构造透视变换网格，逐点采样出干净的模块矩阵（`BitMatrix`）。

### 4. 纠错解码

`Decoder.decode()` 读取格式信息与版本，按 Reed-Solomon 纠错恢复数据码字，解码成字节，再按模式（数字/字母/字节/汉字）还原文本，返回 `Result`。

### 5. 回退与多格式

单格式失败时 `MultiFormatReader` 切换下一个 Reader；全部失败抛 `NotFoundException`。`MultiFormatReader.java` L169 处就是遍历候选 reader 的回退逻辑。

## 设计要点

1. **解码流水线清晰**：`LuminanceSource → Binarizer → BinaryBitmap → Reader → Detector → Decoder → Result`，每层都可替换（相机、文件、不同二值化算法）。
2. **纯 Java 核心**：`core` 不依赖 Android，可直接在服务端/JVM 跑，是它被广泛移植的原因。
3. **读者注册表**：格式与 Reader 解耦，新增格式只需实现 `Reader` 并注册。
4. **纠错能力**：Reed-Solomon 让污损、遮挡的码仍可识别；解码器对版本/掩码/纠错等级的处理是学习二维码原理的最佳材料。
5. **编码对称**：`Writer` 与 `Reader` 一一对应，`MultiFormatWriter` 提供统一入口。

## 阅读路径

1. `core/src/main/java/com/google/zxing/MultiFormatReader.java`
2. `core/.../qrcode/QRCodeReader.java`
3. `core/.../common/HybridBinarizer.java`、`GlobalHistogramBinarizer.java`
4. `core/.../BinaryBitmap.java`、`Binarizer.java`、`LuminanceSource.java`
5. `core/.../qrcode/detector/Detector.java`
6. `core/.../qrcode/decoder/Decoder.java`
7. `core/.../common/BitMatrix.java`、`MultiFormatWriter.java`

## 版本注记

- 该 commit 为当前 master（3.x 线），`android/` 客户端已迁出主仓，核心仍是纯 Java。
- 排行榜成文时（2017）ZXing 3.3.x 是主流；类名与流水线结构自 2.x 起基本稳定，本目录图表以该 commit 为准。
