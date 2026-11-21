# AndroidViewAnimations 源码解析

## 一句话定位

AndroidViewAnimations 是 daimajia 的 View 动画预设库：把 63 种常见效果（抖动、弹入、翻转、滑动、缩放、铰链等）各写成一个 `BaseViewAnimator` 子类，再用 `YoYo.with(Techniques.xxx).duration(...).playOn(view)` 的链式 API 播放。核心库 66 个 Java 文件、约 1700 行有效代码（每文件另有 20 余行许可证头），是"如何用 ObjectAnimator + AnimatorSet 封装动画预设"的典型读本。

## 源码快照

- 上游：<https://github.com/daimajia/AndroidViewAnimations>
- 分支：`master`（浅克隆，`origin` 指向该地址）
- commit：`6a35c466d23e49c5cd174289ebe8d9ccdbc69044`（2020-08-30，合并 androidx 迁移 PR #158）
- 版本：`gradle.properties` 里 `VERSION_NAME=2.4`、`VERSION_CODE=32`、`MIN_SDK_VERSION=14`、`COMPILE_SDK_VERSION=29`；库坐标 `com.daimajia.androidanimations:library:2.4`，依赖 `com.daimajia.easing:library:2.4@aar`

| 路径 | 内容 |
| --- | --- |
| `library/.../library/YoYo.java` | 链式入口：`with()`、内部类 `AnimationComposer` 与 `YoYoString`、私有的 `play()` |
| `library/.../library/BaseViewAnimator.java` | `AnimatorSet` 模板基类：`prepare()` 抽象、`setTarget()`、`reset()`、`start()` |
| `library/.../library/Techniques.java` | 枚举 63 个技法，每个携带实现类 `Class`，`getAnimator()` 反射实例化 |
| `library/.../library/attention|bouncing_entrances|fading_*|flippers|rotating_*|sliders|zooming_*|specials/` | 63 个动画子类，按效果分组 |
| `demo/src/main/java/com/daimajia/androidanimations/` | `ExampleActivity`、`EffectAdapter` 等示例用法 |

## 核心类

| 类名 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `YoYo` | 门面：`with()` 建配置器，`play()` 把参数写回动画对象并启动 | `with(Techniques)` L70、`with(BaseViewAnimator)` L74、`play()` L253 |
| `YoYo.AnimationComposer` | 链式参数收集器，全部字段都只是暂存 | `duration()` L123、`repeat()` L154、`onStart()` L173、`onEnd()` L183、`playOn()` L213 |
| `YoYo.YoYoString` | 播放句柄，可查状态、停止并可选择复位 | `isRunning()` L237、`stop(boolean)` L245 |
| `Techniques` | 技法枚举与实现类的注册表 | `getAnimator()` L177 |
| `BaseViewAnimator` | 模板：`setTarget` → `prepare` → `start`，维护 `AnimatorSet` | `prepare()` L50、`setTarget()` L52、`reset()` L72、`start()` L86 |
| `BounceAnimator` | `translationY` 衰减序列模拟弹跳 | `prepare()` L34 |
| `BounceInAnimator` | `alpha` 与 `scaleX/scaleY` 并行弹入 | `prepare()` L34 |
| `FlipInYAnimator` | `rotationY` 从 90° 翻到 0°，同时淡入 | `prepare()` L34 |
| `SlideInLeftAnimator` | 用 `parent.getWidth() - target.getLeft()` 算平移距离 | `prepare()` L35 |
| `HingeAnimator` | 借 `Glider.glide(Skill.SineEaseInOut, 1300, ...)` 做铰链缓动 | `prepare()` L36、`setDuration(1300)` L47 |
| `TakingOffAnimator` | `Skill.QuintEaseOut` 放大到 1.5 倍并淡出 | `prepare()` L12 |

完整结构见 [class-diagram.html](./class-diagram.html)；图中只保留 6 个有代表性的动画子类，其余 57 个结构相同。

## 关键流程

构建阶段：`YoYo.with(Techniques.Bounce)` 只是 `new AnimationComposer(techniques)`，构造函数里调用 `techniques.getAnimator()` 反射创建动画对象（`Techniques` L177–183）。之后 `duration()`、`delay()`、`repeat()`、`interpolate()`、`pivot()`、`onStart/onEnd` 等全部只改 `AnimationComposer` 的字段，不触碰 View，也不启动任何动画——所以配置阶段可以任意复用到多个 View。

播放阶段：`playOn(target)` 记录 target，构造 `YoYo` 实例并调用私有 `play()`（L253–280）。这里先 `animator.setTarget(target)`，`setTarget()` 内部会先 `reset()`（把 alpha/scale/translation/rotationX/Y 全部还原，L72–81）再执行子类 `prepare()`，保证同一 View 反复播放不会叠加脏状态。接着处理 pivot：`pivotX/pivotY` 若仍是哨兵值 `CENTER_PIVOT`（`Float.MAX_VALUE`），就用 `getMeasuredWidth()/2` 与 `getMeasuredHeight()/2` 设为中心轴（L256–265）。最后链式下发 `setDuration/setRepeatTimes/setRepeatMode/setInterpolator/setStartDelay`，挂上收集到的 listener，调用 `animate()`。

启动阶段：`BaseViewAnimator.start()`（L86–95）遍历 `getChildAnimations()`，把重复次数与重复模式写给每个 `ValueAnimator`，再 `mAnimatorSet.setDuration(mDuration)` 并 `start()`；`restart()` 则先 `clone()` 一个 `AnimatorSet` 再启动，避免复用已结束的动画对象。

技法实现：各子类只写 `prepare()`，用 `getAnimatorAgent().playTogether(ObjectAnimator.ofFloat(target, "属性", 关键帧...))` 添加一个或多个属性动画；关键帧数组表达中间过程（如 `BounceAnimator` 的 `0,0,-30,0,-15,0,0`）。`Hinge`、`TakingOff` 等特殊效果改用 easing 库的 `Glider.glide(Skill, duration, animator)` 把缓动曲线包到属性动画上。

停止与复位：`playOn()` 返回 `YoYoString`，`stop()` 等价 `stop(true)`，先 `animator.cancel()` 再 `reset(target)`，因此可以实现"点一下播放、再点一下归位"的交互。

流程细节见 [flow-diagram.html](./flow-diagram.html)。

## 设计要点

- **延迟绑定**：配置与执行彻底分离，`AnimationComposer` 只是参数袋，`playOn` 才创建 `YoYo` 并作用到 View，天然支持链式与复用。
- **枚举 + 反射做注册表**：新增技法只需在 `Techniques` 加一个枚举值并写一个子类，不用改 `YoYo`；`getAnimator()` 用 `newInstance()` 创建，失败时抛 `Error`。
- **统一重置**：`reset()` 用 `ViewCompat` 归一所有可变属性，这是"同一 View 连续播放不同技法"能正常工作的前提。
- **AnimatorSet 聚合**：一个技法对应一个 `AnimatorSet`，多属性动画用 `playTogether` 并行，时长、插值器、repeat 都只在集合层面配置一次，子类几乎不需要关心调度。
- **哨兵值表达"居中"**：用 `Float.MAX_VALUE` 而不是 `-1`，避免与合法的 pivot 负值混淆。
- **反射只在入口**：反射只发生在 `with()` 创建动画对象那一刻，后续 `setTarget/start` 全是强类型调用，播放路径没有反射开销。
- **缓动外包**：曲线来自 `com.daimajia.easing`，本库只负责把 `Skill` 包进 `Glider`，职责边界清晰。

## 阅读路径

1. `library/src/main/java/com/daimajia/androidanimations/library/BaseViewAnimator.java`：先看模板方法骨架。
2. `library/src/main/java/com/daimajia/androidanimations/library/YoYo.java`：`AnimationComposer` L100 起、`play()` L253 起。
3. `library/src/main/java/com/daimajia/androidanimations/library/Techniques.java`：枚举表与 `getAnimator()`。
4. `attention/BounceAnimator.java`：最简单的单属性关键帧。
5. `bouncing_entrances/BounceInAnimator.java`：多属性 `playTogether`。
6. `sliders/SlideInLeftAnimator.java`：依赖父容器宽度的动态距离。
7. `specials/HingeAnimator.java` 与 `specials/out/TakingOffAnimator.java`：`Glider`/`Skill` 缓动写法。
8. `demo/src/main/java/com/daimajia/androidanimations/ExampleActivity.java`：真实调用方式。

## 版本注记

该 commit 是 2020 年合并 androidx 迁移后的 master 头，库仍停留在 `2.4` 版本线：`compileSdkVersion 29`、`minSdkVersion 14`，`implementation 'androidx.core:core:1.3.1'`，并依赖 `com.daimajia.easing:library:2.4`。README 中的下载说明与 demo 均按 2.4 编写；浅克隆无 tag，具体 API 与行号以本 commit 为准。
