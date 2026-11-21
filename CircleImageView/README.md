# CircleImageView 源码解析

## 一句话定位

CircleImageView 是只有一个类的 Android 圆形头像控件：继承 `ImageView`，把当前 `Drawable` 转成 `Bitmap`，再用 `BitmapShader` 把这张位图铺进 `drawCircle` 的圆形区域，另配一支 `STROKE` 画笔描边框。因为不用 `clipPath`、不复制位图、不叠加两次绘制，它是学习"自定义 View 绘制管线"最省事的读本。

## 源码快照

- 上游：<https://github.com/hdodenhof/CircleImageView>
- commit：`59ee0a11d7199cdeb2d454ebeaaa8396238c418f`（2023-01-10，仅改 README）
- 版本：根 `gradle.properties` 声明 `VERSION_NAME=4.0.0-SNAPSHOT`；README 里的 Gradle 示例仍写 `3.1.0`
- 构建：AGP 3.6.1、`compileSdkVersion 29`、`minSdkVersion 14`，依赖仅 `compileOnly androidx.annotation`

模块只有两个：`circleimageview/`（库，含 `CircleImageView.java` 512 行与 `res/values/attrs.xml` 9 行）和 `sample/`（14 行的 `MainActivity`）。全部逻辑集中在 `circleimageview/src/main/java/de/hdodenhof/circleimageview/CircleImageView.java`。

## 核心类

| 类 | 职责 | 关键方法与行号 |
| --- | --- | --- |
| `CircleImageView` | 圆形裁剪、边框、底色、触摸命中 | `init()` L115、`onDraw()` L156、`onSizeChanged()` L198、`setBorderWidth()` L258、`getBitmapFromDrawable()` L377、`initializeBitmap()` L405、`updateDimensions()` L425、`calculateBounds()` L438、`updateShaderMatrix()` L450、`inTouchableArea()` L488 |
| `CircleImageView.OutlineProvider` | API 21+ 上报圆形轮廓（阴影/涟漪用） | `getOutline()` L500 |
| `R.styleable.CircleImageView` | 四个自定义属性 | `civ_border_width`、`civ_border_color`、`civ_border_overlay`、`civ_circle_background_color`（`attrs.xml` L3–L8） |

类图见 [class-diagram.html](./class-diagram.html)，其中 `ImageView`、`Paint`、`RectF`、`Matrix`、`BitmapShader`、`Canvas` 是框架类型，按其在本类中的用法标注，源码行号都指向 `CircleImageView.java` 的对应使用处。

## 关键流程

1. **构造与属性**：三参构造用 `obtainStyledAttributes` 读四个 `civ_*` 属性（L103–110），然后 `init()` 把 `ScaleType` 强制成 `CENTER_CROP`（`setScaleType` 换成别的值直接抛异常，L141–145）、给三支画笔设抗锯齿/填充/描边参数，并在 API 21+ 安装 `OutlineProvider`（L135–137）。
2. **位图接入**：四个 `setImageBitmap/Drawable/Resource/URI` 重写都会调 `initializeBitmap()`（L305–331）。`getBitmapFromDrawable()` 对 `BitmapDrawable` 直接取原位图；`ColorDrawable` 造 2×2 小图；其余按 `getIntrinsicWidth/Height` 画进新 `Bitmap`（L377–403）。可变的位图会再包一层 `Canvas`，供动画 Drawable 重绘。
3. **几何换算**：`onSizeChanged`、`setPadding` 都会触发 `updateDimensions()`。`calculateBounds()` 先取去掉 padding 后的正方形边长并居中（L438–448）；`mBorderRadius` 是外矩形减边框宽度的一半，`mDrawableRect` 在非 overlay 模式下再 `inset(mBorderWidth - 1.0f)`，从而得到内容圆半径 `mDrawableRadius`（L425–436）。
4. **绘制**：`onDraw` 先按需把 Drawable 重绘进 `mBitmapCanvas`（`mDrawableDirty` 标记，L167–172），需要时 `new BitmapShader(mBitmap, CLAMP, CLAMP)` 并套上 `mShaderMatrix`（L174–181），然后 `drawCircle(contentX, contentY, mDrawableRadius, mBitmapPaint)`（L183），最后边框宽大于 0 时再画一圈 `mBorderPaint`（L186–188）。
5. **着色器矩阵**：`updateShaderMatrix()` 用"位图宽比高更宽就按高缩放、否则按宽缩放"的方式模拟 `CENTER_CROP`，再 `postTranslate` 居中，平移量刻意写成 `(int)(dx + 0.5f)` 取整以避免半像素抖动（L450–476）。
6. **交互与轮廓**：`onTouchEvent` 只在圆内才交给父类（L478–494）；`OutlineProvider.getOutline` 用 `mBorderRect` 的 `roundOut` 设圆角矩形（L496–510）。

## 设计要点

- **用 Shader 而非裁剪路径**：`BitmapShader` 加一次 `drawCircle` 就能同时获得硬件加速和抗锯齿，不需要 `clipPath`，也不用 `setXfermode` 画两遍。项目 README 把这三条列为设计约束。
- **标记位延迟重建**：`mRebuildShader` 与 `mDrawableDirty` 让着色器/内容只在真正变化时才更新，`onDraw` 每帧不做对象分配。
- **两套矩形两套半径**：`mBorderRect/mBorderRadius` 管外沿，`mDrawableRect/mDrawableRadius` 管内容，`mBorderOverlay` 决定内容是否让开边框，边框宽度改变时只需重算几何。
- **构造期回调防御**：`setImageAlpha`、`setColorFilter` 都可能被 `ImageView` 构造过程调用（API 16/19 的差异），所以用 `mInitialized` 判断后才触碰画笔。
- **圆形命中与轮廓**：触摸用 `pow(x-cx,2)+pow(y-cy,2) <= pow(radius,2)` 判断；`OutlineProvider` 让系统阴影/涟漪也呈圆形。

## 阅读路径

1. `circleimageview/src/main/res/values/attrs.xml`——四个自定义属性，先建立配置面。
2. `circleimageview/.../CircleImageView.java` L49–138——字段、三支画笔、`init()`。
3. 同文件 L377–476——`getBitmapFromDrawable`、`initializeBitmap`、`updateDimensions`、`calculateBounds`、`updateShaderMatrix`，几何与位图这一组最值得抄。
4. 同文件 L154–202——`onDraw`、`invalidateDrawable`、`onSizeChanged`。
5. 同文件 L478–510——触摸命中与 `OutlineProvider`。
6. `sample/.../MainActivity.java` 与仓库 `README.md` 的 Limitations 段，确认 `ScaleType` 与 `adjustViewBounds` 的限制是刻意的。

## 版本注记

本 commit 的 512 行版本已经**没有 `onMeasure` 重写**：早期 2.x/3.x 版本用 `onMeasure` 强制宽高相等，这一版改成 `onSizeChanged` + `calculateBounds()`，尺寸逻辑统一收敛到 `updateDimensions()`，因此阅读时不要按旧博客去找 `onMeasure`。另外根 `build.gradle` 仍用 `jcenter()` 与 AGP 3.6.1，属于仓库维护滞后的表现；`VERSION_NAME` 已是 4.0.0-SNAPSHOT，精确版本以该 commit 为准。
