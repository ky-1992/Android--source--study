# uCrop 源码解析

## 一句话定位

uCrop 是一个「调用方只拼参数、编辑页负责手势与裁剪」的图片裁剪库：`UCrop` 把源图、目标图和各种开关写进 `Intent`，`UCropActivity` 用 `UCropView` 展示图片与裁剪框，最后交给 `BitmapCropTask` 在后台完成原生裁剪。

## 源码快照

- 上游：<https://github.com/Yalantis/uCrop>
- commit：`f788b534b48c144edf786c8cddbf0e029e637804`（2025-08-07，提交信息 *Update mavenpush.gradle*）
- 仓库：`ucrop`（Android Library，包名 `com.yalantis.ucrop`）+ `sample` 示例
- 模块内分组：根目录是门面与页面（`UCrop`、`UCropActivity`、`UCropFragment`），`view/` 放图片与裁剪框视图，`task/` 放两个 `AsyncTask`，`util/` 放解码、文件与 EXIF 工具

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `UCrop` | 参数构建与 Intent 门面 | `of` L65、`withOptions` L119、`getIntent` L197、`getOutput` L223 |
| `UCrop.Options` | 可选参数集合，统一塞进 Bundle | `getOptionBundle` L324、`setCompressionFormat` L331、`setAllowedGestures` L345 |
| `UCropActivity` | 编辑页编排与结果回传 | `onCreate` L125、`setImageData` L198、`processOptions` L221、`cropAndSaveImage` L679、`setResultUri` L700 |
| `UCropView` | 图片层 + 覆盖层的组合容器 | `setListenersToViews` L41、`getCropImageView` L62、`resetCropImageView` L75 |
| `TransformImageView` | 用 `Matrix` 维护平移/缩放/旋转 | `setImageUri` L139、`getCurrentScale` L169、`setImageMatrix` L197 |
| `CropImageView` | 裁剪框与缩放边界 | `cropAndSaveImage` L73、`setCropRect` L118、`postScale` L240、`setImageToWrapCropBounds` L277 |
| `GestureCropImageView` | 缩放手势、旋转手势与双击 | `onTouchEvent` L78、`getDoubleTapTargetScale` L117 |
| `OverlayView` | 遮罩、裁剪框与九宫格绘制/拖拽 | `onDraw` L300、`onTouchEvent` L307、`updateCropViewRect` L364 |
| `BitmapLoadTask` | 后台解码 + EXIF 校正 | `doInBackground` L82、`onPostExecute` L254 |
| `BitmapCropTask` | 原生裁剪与落盘 | `doInBackground` L80、`resize` L101、`crop` L131 |

## 关键流程

1. **启动与参数**：调用方 `UCrop.of(source, destination).withAspectRatio(...).withOptions(options).start(activity)`。构造函数把两个 `Uri` 放进 `Bundle`（`EXTRA_INPUT_URI` / `EXTRA_OUTPUT_URI`），`getIntent` 里再 `setClass(context, UCropActivity.class)` 并 `putExtras`；`withMaxResultSize` 会把不足 `MIN_SIZE`（10）的宽高抬回下限，防止产出空图。
2. **页面装配**：`UCropActivity.onCreate` 依次执行 `setupSystemBars`、`setContentView(ucrop_activity_photobox)`、`setupViews`、`setImageData`、`setInitialState`、`addBlockingView`；`processOptions` 从 `Intent` 读取压缩格式/质量、允许手势、`maxBitmapSize` 以及 Overlay 的颜色与网格配置。底部有比例、旋转、缩放三个面板（`state_aspect_ratio` / `state_rotate` / `state_scale`），`setWidgetState` 切换面板时会重新调用 `setAllowedGestures`，通过 `setScaleEnabled` 与 `setRotateEnabled` 决定当前页面能用手势还是拖轮盘。
3. **解码**：`setImageData` 取到输入/输出 `Uri` 后调用 `mGestureCropImageView.setImageUri`，内部走 `BitmapLoadUtils.decodeBitmapInBackground`，按 `maxBitmapSize` 计算 `inSampleSize` 后解码，`onBitmapLoaded` 里回填输入路径、`ExifInfo` 并设置 Bitmap。`BitmapLoadTask` 会先用 `inJustDecodeBounds` 量出原始尺寸再决定采样率，`processInputUri` 区分 download、content、file 三种 Uri（content 先拷贝到目标路径，download 先落盘），任何异常都包成 `BitmapWorkerResult` 交回主线程。
4. **手势与裁剪框**：`GestureCropImageView.onTouchEvent` 把事件同时喂给 `ScaleGestureDetector`、`RotationGestureDetector` 与 `GestureDetector`；`ACTION_UP` 时 `setImageToWrapCropBounds()` 保证图片盖满裁剪框；`OverlayView` 在 freestyle 模式下按角点索引（0~3 拉角、4 移动整体）改写 `mCropViewRect`，松手回调 `onCropRectUpdated` 同步给 `CropImageView.setCropRect`。旋转面板的 `onScroll` 调用 `postRotate(delta / ROTATE_WIDGET_SENSITIVITY_COEFFICIENT)`，`onScrollEnd` 再 `setImageToWrapCropBounds`，旁边还有 90° 按钮走 `rotateByAngle(90)`、复位按钮走 `resetRotation`。
5. **裁剪与回传**：`cropAndSaveImage` 先 `cancelAllAnimations()`，再用当前裁剪框、图片矩形、`scale`、`angle` 组 `ImageState`，连同 `CropParameters` 交给 `BitmapCropTask`；任务里 `resize` 后调 native `cropCImg`，裁剪偏移与输出宽高都按 `mCurrentScale` 换算，`shouldCrop` 判定不需要裁剪时只把原图 `copyFile` 到输出路径，JPEG 时用 `ImageHeaderParser.copyExif` 回写 EXIF，最终 `onBitmapCropped` → `setResultUri` → `finish()`。

## 设计要点

- **参数与页面解耦**：所有可配置项都落在 `Bundle` 的 key 上，`UCropFragment` 复用同一份 `mCropOptionsBundle`，Activity 与 Fragment 两种入口不会出现配置漂移。
- **视图两层分工**：`UCropView` 里下层画图片、上层画遮罩与裁剪框，双方不互相持有，只通过 `CropBoundsChangeListener` 与 `OverlayViewChangeListener` 交叉绑定。
- **状态只在矩阵里**：缩放值与角度都由 `TransformImageView` 从 `Matrix` 反算（`getMatrixScale` / `getMatrixAngle`），没有第二份可变状态。
- **后台任务化**：解码（`BitmapLoadTask`）与裁剪（`BitmapCropTask`）都是 `AsyncTask`，主线程只做可见性切换和结果回传，加载期间用 `mBlockingView` 挡住输入。
- **原生兜底**：`BitmapCropTask` 静态块 `System.loadLibrary("ucrop")`，实际像素操作在 native 层完成，避免 Java 层重复采样与大图 OOM；`resize` 还会按 EXIF 的 90/270 度交换宽高，并受 `withMaxResultSize` 二次限制。
- **两套入口共享参数**：`UCrop.getFragment()` 返回 `UCropFragment`，与 Activity 入口共用同一个 `Bundle`，所以颜色、比例、自由裁剪等开关只需维护一份 key 列表。
- **缩放边界集中在一处**：`GestureCropImageView` 只把手势换算成 `postScale` / `postRotate`，上下限判断在 `CropImageView.postScale`：只有 `currentScale * delta` 落在 min/max 之间才真正执行，双击的目标倍数也用 `Math.pow(max/min, 1/steps)` 反推，避免每一层各自夹取。
- **结果字段固定**：成功时 `setResultUri` 一次写入 `EXTRA_OUTPUT_URI`、裁剪比例、图片宽高与偏移，失败时 `setResultError` 用 `RESULT_ERROR` 带出 `Throwable`；调用方用 `UCrop.getOutput`、`getOutputImageWidth`、`getError` 读取，不必约定自定义 resultCode。

## 阅读路径

1. `ucrop/src/main/java/com/yalantis/ucrop/UCrop.java`
2. `ucrop/src/main/java/com/yalantis/ucrop/UCropActivity.java`
3. `ucrop/src/main/java/com/yalantis/ucrop/view/UCropView.java`
4. `ucrop/src/main/java/com/yalantis/ucrop/view/TransformImageView.java`
5. `ucrop/src/main/java/com/yalantis/ucrop/view/CropImageView.java`
6. `ucrop/src/main/java/com/yalantis/ucrop/view/GestureCropImageView.java`
7. `ucrop/src/main/java/com/yalantis/ucrop/view/OverlayView.java`
8. `ucrop/src/main/java/com/yalantis/ucrop/task/BitmapLoadTask.java`、`task/BitmapCropTask.java`

## 版本注记

该 commit 已使用 AndroidX 与 `ActivityResultLauncher` 入口，`setupSystemBars` 通过 `EdgeToEdge` 处理系统栏；`UCrop.start` 同时保留 `startActivityForResult` 与 `ActivityResultLauncher` 两种发起方式。裁剪依赖 native 库 `ucrop`：`ucrop/src/main/jni` 下是 `uCrop.cpp`、`CImg.h` 与 JNI 桥接头文件，由 `Android.mk` 构建，`BitmapCropTask.cropCImg` 即对应这层实现。行号与结构以该 commit 为准。
