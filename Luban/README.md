# Luban 源码解析

## 一句话定位

Luban（鲁班）是通过逆向微信朋友圈压缩效果推算出的 Android 图片压缩库。当前 master 是 Kotlin 重写版 2.0.3：协程负责并发与线程切换，`CompressionCalculator` 先算目标尺寸与预估体积，`ImageLoader` 按 `inSampleSize` 采样解码，`JpegCompressor` 调用 libturbojpeg 的 JNI 编码，长图再用质量二分逼近目标大小。库模块 Kotlin 代码约一千一百行（另有 85 行 C++ 封装），是"先预算、再解码、后编码"这类图片处理流水线的典型样本。

## 源码快照

- 上游：<https://github.com/Curzibn/Luban>
- commit：`250ab4d13c723ea037b51a2c3d7dae14881a94cd`（2026-10-08，"Move the LubanPNG callout above the table of contents"）
- 多模块：`luban/`（库，Kotlin + CMake/turbojpeg）、`app/`（Compose 示例）、`app-java/`（Java/Builder 示例）
- `luban/build.gradle.kts`：`libraryVersion = "2.0.3"`，`compileSdk = 36`、`minSdk = 21`，依赖 `kotlinx-coroutines 1.10.2` 与 `androidx.exifinterface`；`src/main/jniLibs` 内置 armeabi-v7a、arm64-v8a、x86、x86_64 四个 abi 的 `libturbojpeg.so`
- 库源码分四层：`api/`（Luban、LubanCompat、LubanDsl、CompressionTask）、`algorithm/`（CompressionCalculator、CompressionTarget）、`io/`（ImageLoader、ImageData）、`compression/`（Compressor、JpegCompressor、TurboJpegNative）

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `Luban` | 压缩门面，持有三个可替换协作者 | `compress()` L200、`compressInternal()` L266/L278、`processImage()` L288 |
| `CompressionCalculator` | 算目标宽高与预估体积 | `calculateTarget()` L23 |
| `CompressionTarget` | 目标参数数据类 | 定义 L8 |
| `ImageLoader` | 解码、采样、EXIF 旋转 | `loadFromFile()` L44、`decodeStream()` L68、`calculateInSampleSize()` L116 |
| `Compressor` / `JpegCompressor` | 编码抽象与实现（质量二分） | `compress()` L6、`bitmapToRgb()` L52 |
| `TurboJpegNative` | `libturbojpeg` 的 JNI 入口 | `compressOrThrow()` L21 |
| `LubanCompat` | Builder + 回调兼容层 | `launch()` L34 |
| `LubanDsl` | DSL 批量任务收集 | `execute()` L405 |

## 关键流程

1. **读边界**：`BitmapFactory.Options` 打开 `inJustDecodeBounds`，只取 `outWidth`/`outHeight`，不解码像素也不申请位图内存。
2. **算目标**：短边基准 1440；长边 ≥10800 且比例 >0.4 时改以长边为基准；像素数 >40960000 时短边压到原短边的 1/4；再按 10240000 像素上限等比收缩，最后对齐到偶数。
3. **采样解码**：`calculateInSampleSize` 先逐次折半，超大图再按开方比例放大 `inSampleSize`，解码后若仍超尺寸则 `createScaledBitmap`，并按 EXIF 方向旋转。
4. **编码**：普通图固定 `quality=60` 编一次；长图（比例 ≤0.4）把 `estimatedSizeKb` 传给压缩器，先试 95，若已不超目标直接返回，否则在 5~95 上二分质量。
5. **落盘**：`processImage` 把位图统一转成 `ARGB_8888`，若压缩字节数不小于原图则直接拷贝原文件，`finally` 里 `recycle()`。
6. **命名与并发**：输出文件按"原文件名 + 毫秒时间戳"生成，同目录重复压缩不会覆盖；多图用 `coroutineScope { inputs.map { async { … } }.awaitAll() }` 并发，每张内部再 `withContext(Dispatchers.IO)`，单张失败只变成 `Result.failure`，不中断其他任务。

## 设计要点

1. **预算先行**：先算目标尺寸再解码，避免把整张原图读进内存，这是省内存的关键，也是与"先解码再缩放"实现的本质差别。
2. **体积估算按像素分段**：`<50 万` 用 0.0005、`<100 万` 用 0.00015、`<300 万` 用 0.00011、其余 0.000025；比例 <0.2 且估算 <400KB 时抬到至少 250KB，保超长图清晰度。
3. **组合优于继承**：`compressor`、`imageLoader`、`calculator` 都是构造参数且各有默认实现，测试和定制都能直接替换，不必改主流程。
4. **两套 API**：协程侧 `compress`/`compressTo`/`luban { }` 返回 `Result<File>`，异常不外抛；兼容侧 `LubanCompat.Builder().load(...).launch()` 返回 `CompressionTask`，可绑 Lifecycle 的 `ON_DESTROY` 自动取消。
5. **编码下沉到 JNI**：turbojpeg 比 `Bitmap.compress` 快，也方便在循环里反复试不同 quality，二分才不至于太慢。
6. **内存双重兜底**：`calculateInSampleSize` 除了按目标尺寸折半，还额外以 1600 万像素封顶；解码选项用 `RGB_565`，先把每像素内存砍掉一半。
7. **二分有兜底**：`JpegCompressor` 在 5~95 区间二分找"不超过目标体积的最大质量"，若整个区间都没找到合格结果，就退回 `quality=5` 再编一次，保证总有输出。

## 阅读路径

建议按"先算法、后管线"的顺序读：`CompressionCalculator` 是纯函数，最容易验证；再读 `Luban` 主流程看三者如何串起来；最后看 JNI 与兼容层。

1. `luban/src/main/java/top/zibin/luban/api/Luban.kt`
2. `luban/src/main/java/top/zibin/luban/algorithm/CompressionCalculator.kt`
3. `luban/src/main/java/top/zibin/luban/io/ImageLoader.kt`
4. `luban/src/main/java/top/zibin/luban/compression/JpegCompressor.kt`
5. `luban/src/main/java/top/zibin/luban/compression/TurboJpegNative.kt` 与 `luban/src/main/cpp/turbojpeg_wrapper.cpp`
6. `luban/src/main/java/top/zibin/luban/compression/Compressor.kt`
7. `luban/src/main/java/top/zibin/luban/api/LubanCompat.kt`
8. `luban/src/test/java/top/zibin/luban/CompressionCalculatorTest.kt`

## 版本注记

结论以 commit `250ab4d` 为准，对应 2.0.3 开发线。本文快照的对外 API 全部是协程 `suspend` + `Result` 与 `LubanCompat` Builder，压缩策略由 `CompressionCalculator` 常量表驱动，并依赖内置 TurboJPEG 原生库；若参考网上基于旧版 Java 实现的尺寸对照表或 Builder 写法，注意那不是本 commit 的代码。
