# AppIntro 源码解析

## 一句话定位

AppIntro 是一个用 Activity + Fragment + ViewPager2 拼出来的「首次启动引导页」库：宿主继承 `AppIntro`（或 `AppIntro2`）并在 `onCreate` 里 `addSlide`，翻页、指示器、跳过/完成按钮、逐屏权限申请、「这一屏必须满足条件才允许往下走」的策略校验，全部由基类 `AppIntroBase` 统一接管。

## 源码快照

- 上游：<https://github.com/AppIntro/AppIntro>
- commit：`ce7d35dafe5689a35831da4a662c06d5f899705a`（2026-11-10，`--depth 1` 浅克隆，`git log` 只有这一条）
- 版本线：`appintro/build.gradle.kts` 声明 `version = "7.0.0-beta02"`；`gradle/libs.versions.toml` 中 `min_sdk_version = 21`、`compile_sdk_version = 36`、`viewpager2 = 1.1.0`
- 模块：`appintro/` 库本体（Kotlin，约 2400 行）、`example/` 示例 App、`docs/` 迁移说明

| 包 | 内容 |
| --- | --- |
| `com.github.appintro` | `AppIntroBase`、`AppIntro`、`AppIntro2`、各类 Fragment 与接口 |
| `com.github.appintro.internal` | 分页控制器、权限包装、日志、字体、震动 |
| `com.github.appintro.indicator` | 点状 / 进度条指示器 |

## 核心类

| 类名 | 职责 | 关键成员（行号取自快照） |
| --- | --- | --- |
| `AppIntroBase` | 生命周期、按钮与页面回调、策略与权限裁决 | `addSlide()` L146、`onCreate()` L388、`onPostCreate()` L447、`updateButtonsVisibility()` L584、`onCanRequestNextPage()` L610、`requestPermissions()` L639 |
| `AppIntro` | 标题 + 文字按钮的默认版式 | `layoutId` L17、`setBarColor()` L23、`setSkipText()` L71、`showSeparator()` L219 |
| `AppIntro2` | 背景图 + 图片按钮的版式 | `layoutId` L12、`setImageSkipButton()` L59 |
| `AppIntroViewPagerController` | 接管 ViewPager2 的触摸与翻页 | `goToNextSlide()` L39、`setCurrentViewPagerItem()` L83、`addPagerTouchInterceptor()` L218 |
| `AppIntroViewPagerListener` | 翻页前的问路回调 | `onCanRequestNextPage()` L15、`onUserRequestedPermissionsDialog()` L19 |
| `SlidePolicy` | 单屏是否允许离开 | `isPolicyRespected` L10、`onUserIllegallyRequestNextPage()` L15 |
| `SlideBackgroundColorHolder` | 参与相邻屏颜色渐变 | `defaultBackgroundColorRes` L25、`setBackgroundColor()` L33 |
| `PermissionWrapper` | 某一屏要申请的权限包 | 字段 L12–L14、`equals()` L16 |
| `AppIntroBaseFragment` | Fragment 基类，用 ViewModel 承接配置变更 | `onCreate()` L44、`onCreateView()` L89、`setBackgroundColor()` L184 |
| `AppIntroFragment` | 开箱即用的图文页 | `createInstance()` L88、L140 |

## 关键流程

装配：宿主的 `onCreate` 先 `super.onCreate()`，基类在 L388 绑定 `indicator_container`、`next`、`done`、`skip`、`back` 五个控件，创建 `PagerAdapter` 与 `AppIntroViewPagerController`（L427–L444），再把按钮和 Back 回调挂上；随后宿主调用 `addSlide` 把 Fragment 塞进 `fragments`（L146），`onPostCreate` 才用 `fragments.size` 初始化指示器并补发首屏的 `onPageSelected`（L447–L474），一个 slide 都没有时直接 `finish()`。

前进判定：`NextSlideOnClickListener`（L781）先 `dispatchVibration`，再问 `onCanRequestNextPage`——当前 Fragment 若实现 `SlidePolicy` 且 `isPolicyRespected` 为 false 就拒绝；通过后若当前屏登记过权限就先请求，否则回调 `onNextPressed`/`onDonePressed` 并 `goToNextSlide()`。

权限：`askForPermissions` 把 slideNumber 映射成 `PermissionWrapper`（L167）；`requestPermissions()` 先 `setSwipeLock(true)` 再下发系统请求（L639），结果里授予则移除登记并前进，拒绝则按 `shouldShowRequestPermissionRationale` 分派「可再问」与「不再询问」，`required` 为 false 时仍然放行（L654–L708）。

翻页回调：`OnPageChangeCallback.onPageSelected`（L811–L849）依次做四件事——让指示器选中新位置、按「是否首屏/末屏」刷新按钮可见性、把当前屏是否带权限写回控制器、再触发 `onPageSelected` 与 `onSlideChanged(old, new)`。相邻屏都实现 `SlideBackgroundColorHolder` 且 `isColorTransitionsEnabled` 时，`onPageScrolled` 会用 `ArgbEvaluator` 同步插值背景色（L738–L763）。

状态恢复：`onSaveInstanceState` 把 `slidesNumber`、按钮开关、权限表、`isColorTransitionsEnabled` 存进 Bundle，其中 `currentItem` 特意换算成 RTL 无关的值；`onRestoreInstanceState` 反序列化时对 Android 13 以上的 `getSerializable` 走带类型参数的重载（L476–L527），避免直接强转报错。

## 设计要点

- **自带手势层**：`AppIntroViewPagerController` 把 `isUserInputEnabled` 置 false，改用 `GestureOverlayView` 收手势再 `fakeDrag`（L218–L254）。因为 ViewPager2 一旦开始滑动就无法中途否决，只有自己转发触摸才能在翻页前拦下。
- **策略与权限都在 Activity 侧裁决**：Fragment 只声明「我不放行」或「这屏要什么权限」，避免每个 slide 各写一套导航逻辑。
- **节流**：非法前进的提示最多 1 秒一次（`ON_ILLEGALLY_REQUESTED_NEXT_PAGE_MAX_INTERVAL`），防止连续滑动刷屏。
- **配置变更安全**：slide 参数走 `AppIntroFragmentViewModel`，Activity 侧把 `slidesNumber`、`permissionMap`、RTL 无关的 `currentItem` 写进 Bundle（L476–L527）。
- **颜色过渡有前提**：只有当相邻两屏都实现 `SlideBackgroundColorHolder` 时才用 `ArgbEvaluator` 插值（L738），否则直接报错，属于「要么全用要么别开」的约定。
- **按钮语义集中在 `updateButtonsVisibility`**：末屏把 next 换成 done、skip 在末屏隐藏、back 只在 wizard 模式且非首屏出现，首页与翻页共用一份判断（L584–L598），省得每个回调重复写。
- **指示器是极小的三方法契约**：`newInstance` 造 View、`initialize` 告知总数、`selectPosition` 选中，默认用 `DotIndicatorController`，slide 多时换成 `ProgressIndicatorController` 只改一行。
- **RTL 是全局坐标换算**：`addSlide` 在 RTL 下插到队首（L147–L153），`isFirstSlide`/`isLastSlide`/`getCurrentSlideNumber` 与 `goToNextSlide` 的方向都按 `LayoutUtil.isRtl` 反向，对外仍以「视觉上的第几屏」为准。
- **wizard 模式只是开关组合**：`isWizardMode` 的 setter 直接令 `isSkipButtonEnabled = !value`（L70–L75），于是同一套布局在「跳过」和「上一步」两种交互间切换，不需要第二份 XML。

## 阅读路径

1. `appintro/src/main/java/com/github/appintro/AppIntroBase.kt` —— 先读 L43–L136 的字段，再读 L388–L474。
2. `appintro/src/main/java/com/github/appintro/AppIntro.kt` —— 一个具体版式如何只覆写 `layoutId` 与外观。
3. `appintro/src/main/java/com/github/appintro/internal/AppIntroViewPagerController.kt` —— 触摸拦截与 `fakeDrag`（L108–L254）。
4. `appintro/src/main/java/com/github/appintro/SlidePolicy.kt`、`AppIntroViewPagerListener.kt` —— 两个很小的契约接口。
5. `appintro/src/main/java/com/github/appintro/AppIntroBaseFragment.kt` —— Fragment 侧取值与 `setBackgroundColor`。
6. `appintro/src/main/java/com/github/appintro/internal/PermissionWrapper.kt` —— 权限登记的粒度。
7. `appintro/src/main/java/com/github/appintro/indicator/IndicatorController.kt` 与 `DotIndicatorController.kt` —— 指示器的三方法契约。
8. `example/src/main/java/com/github/appintro/example/ui/default/DefaultIntro.kt` —— 最短用法样例。

配套图：[class-diagram.html](./class-diagram.html) 与 [flow-diagram.html](./flow-diagram.html)。

## 版本注记

该 commit 对应 `7.0.0-beta02`（`CHANGELOG.md` 里 7.0.0 段落日期还是占位符 `YYYY-MM-DD`），最大变化是 [#1112] 从 ViewPager 迁移到 ViewPager2，因此 `setScrollDurationFactor` 被移除、`setCustomTransformer` 改收 `ViewPager2.PageTransformer`。构建侧已用 version catalog（AGP 9.5.1、Kotlin 2.4.21、targetSdk 34），与 6.x 时代的 JitPack 配置差异较大；具体行为一律以 commit `ce7d35dafe5689a35831da4a662c06d5f899705a` 为准。
