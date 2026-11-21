# Android--source--study

Android 百大框架源码研读仓库：每个框架一个目录（目录名为框架名），包含 **源码解析（README.md）**、**类图**（Archify class diagram）与 **流程图**（Archify workflow diagram）。

框架顺序沿用 [Android_100_TOP-Projects](https://github.com/ShaunSheep/Android_100_TOP-Projects) 的 2017 年百大排行榜；每个目录的图表都基于该框架上游仓库的固定 commit（见各自 README 的“源码快照”一节），所有 `sources` 引用都经 Archify `--repo-root` 校验。

当前进度：**55 / 97**（其余框架整理中）。

## 目录约定

```
<FrameworkName>/
├── README.md                     # 源码解析：定位/快照/核心类/关键流程/设计要点/阅读路径/版本注记
├── class-diagram.html            # 类图（浏览器直接打开，支持动效与导出）
├── class-diagram.archify.json    # 类图源描述
├── flow-diagram.html             # 流程图（关键运行时序）
└── flow-diagram.archify.json     # 流程图源描述
```

## 已完成框架

| 框架 | 类别 | 源码解析 | 类图 | 流程图 | 上游仓库 |
|---|---|---|---|---|---|
| Retrofit | 网络 | [README](Retrofit/README.md) | [类图](Retrofit/class-diagram.html) | [流程图](Retrofit/flow-diagram.html) | [Retrofit](https://github.com/square/retrofit) |
| okhttp | 网络 | [README](okhttp/README.md) | [类图](okhttp/class-diagram.html) | [流程图](okhttp/flow-diagram.html) | [okhttp](https://github.com/square/okhttp) |
| Butter Knife | 代码模板 | [README](Butter-Knife/README.md) | [类图](Butter-Knife/class-diagram.html) | [流程图](Butter-Knife/flow-diagram.html) | [Butter Knife](https://github.com/JakeWharton/butterknife) |
| MPAndroidChart | 图表 | [README](MPAndroidChart/README.md) | [类图](MPAndroidChart/class-diagram.html) | [流程图](MPAndroidChart/flow-diagram.html) | [MPAndroidChart](https://github.com/PhilJay/MPAndroidChart) |
| glide | 图片 | [README](glide/README.md) | [类图](glide/class-diagram.html) | [流程图](glide/flow-diagram.html) | [glide](https://github.com/bumptech/glide) |
| leakcanary | 内存 | [README](leakcanary/README.md) | [类图](leakcanary/class-diagram.html) | [流程图](leakcanary/flow-diagram.html) | [leakcanary](https://github.com/square/leakcanary) |
| Android-Universal-Image-Loader | 图片 | [README](Android-Universal-Image-Loader/README.md) | [类图](Android-Universal-Image-Loader/class-diagram.html) | [流程图](Android-Universal-Image-Loader/flow-diagram.html) | [Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader) |
| EventBus | 事件消息 | [README](EventBus/README.md) | [类图](EventBus/class-diagram.html) | [流程图](EventBus/flow-diagram.html) | [EventBus](https://github.com/greenrobot/EventBus) |
| zxing | 条码扫描 | [README](zxing/README.md) | [类图](zxing/class-diagram.html) | [流程图](zxing/flow-diagram.html) | [zxing](https://github.com/zxing/zxing) |
| picasso | 图片 | [README](picasso/README.md) | [类图](picasso/class-diagram.html) | [流程图](picasso/flow-diagram.html) | [picasso](https://github.com/square/picasso) |
| lottie-android | 动画 | [README](lottie-android/README.md) | [类图](lottie-android/class-diagram.html) | [流程图](lottie-android/flow-diagram.html) | [lottie-android](https://github.com/airbnb/lottie-android) |
| fresco | 图片 | [README](fresco/README.md) | [类图](fresco/class-diagram.html) | [流程图](fresco/flow-diagram.html) | [fresco](https://github.com/facebook/fresco) |
| RxAndroid | 异步 | [README](RxAndroid/README.md) | [类图](RxAndroid/class-diagram.html) | [流程图](RxAndroid/flow-diagram.html) | [RxAndroid](https://github.com/ReactiveX/RxAndroid) |
| SlidingMenu | 菜单 | [README](SlidingMenu/README.md) | [类图](SlidingMenu/class-diagram.html) | [流程图](SlidingMenu/flow-diagram.html) | [SlidingMenu](https://github.com/jfeinstein10/SlidingMenu) |
| PhotoView | 图片 | [README](PhotoView/README.md) | [类图](PhotoView/class-diagram.html) | [流程图](PhotoView/flow-diagram.html) | [PhotoView](https://github.com/chrisbanes/PhotoView) |
| material-dialogs | UI | [README](material-dialogs/README.md) | [类图](material-dialogs/class-diagram.html) | [流程图](material-dialogs/flow-diagram.html) | [material-dialogs](https://github.com/afollestad/material-dialogs) |
| Android-async-http | 网络 | [README](Android-async-http/README.md) | [类图](Android-async-http/class-diagram.html) | [流程图](Android-async-http/flow-diagram.html) | [Android-async-http](https://github.com/loopj/android-async-http) |
| androidannotations | 注解 | [README](androidannotations/README.md) | [类图](androidannotations/class-diagram.html) | [流程图](androidannotations/flow-diagram.html) | [androidannotations](https://github.com/androidannotations/androidannotations) |
| fastjson | json | [README](fastjson/README.md) | [类图](fastjson/class-diagram.html) | [流程图](fastjson/flow-diagram.html) | [fastjson](https://github.com/alibaba/fastjson) |
| Material-Animations | 动画 | [README](Material-Animations/README.md) | [类图](Material-Animations/class-diagram.html) | [流程图](Material-Animations/flow-diagram.html) | [Material-Animations](https://github.com/lgvalle/Material-Animations) |
| tinker | 热修复 | [README](tinker/README.md) | [类图](tinker/class-diagram.html) | [流程图](tinker/flow-diagram.html) | [tinker](https://github.com/Tencent/tinker) |
| ViewPagerIndicator | UI | [README](ViewPagerIndicator/README.md) | [类图](ViewPagerIndicator/class-diagram.html) | [流程图](ViewPagerIndicator/flow-diagram.html) | [ViewPagerIndicator](https://github.com/JakeWharton/ViewPagerIndicator) |
| Android-CleanArchitecture | 架构 | [README](Android-CleanArchitecture/README.md) | [类图](Android-CleanArchitecture/class-diagram.html) | [流程图](Android-CleanArchitecture/flow-diagram.html) | [Android-CleanArchitecture](https://github.com/android10/Android-CleanArchitecture) |
| Android-PullToRefresh | 刷新 | [README](Android-PullToRefresh/README.md) | [类图](Android-PullToRefresh/class-diagram.html) | [流程图](Android-PullToRefresh/flow-diagram.html) | [Android-PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh) |
| flexbox-layout | UI | [README](flexbox-layout/README.md) | [类图](flexbox-layout/class-diagram.html) | [流程图](flexbox-layout/flow-diagram.html) | [flexbox-layout](https://github.com/google/flexbox-layout) |
| AndroidSwipeLayout | UI | [README](AndroidSwipeLayout/README.md) | [类图](AndroidSwipeLayout/class-diagram.html) | [流程图](AndroidSwipeLayout/flow-diagram.html) | [AndroidSwipeLayout](https://github.com/daimajia/AndroidSwipeLayout) |
| realm-java | 数据库 | [README](realm-java/README.md) | [类图](realm-java/class-diagram.html) | [流程图](realm-java/flow-diagram.html) | [realm-java](https://github.com/realm/realm-java) |
| greenDAO | 数据 | [README](greenDAO/README.md) | [类图](greenDAO/class-diagram.html) | [流程图](greenDAO/flow-diagram.html) | [greenDAO](https://github.com/greenrobot/greenDAO) |
| stetho | 调试 | [README](stetho/README.md) | [类图](stetho/class-diagram.html) | [流程图](stetho/flow-diagram.html) | [stetho](https://github.com/facebook/stetho) |
| BaseRecyclerViewAdapterHelper | UI | [README](BaseRecyclerViewAdapterHelper/README.md) | [类图](BaseRecyclerViewAdapterHelper/class-diagram.html) | [流程图](BaseRecyclerViewAdapterHelper/flow-diagram.html) | [BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper) |
| AndroidViewAnimations | 动画 | [README](AndroidViewAnimations/README.md) | [类图](AndroidViewAnimations/class-diagram.html) | [流程图](AndroidViewAnimations/flow-diagram.html) | [AndroidViewAnimations](https://github.com/daimajia/AndroidViewAnimations) |
| MaterialDrawer | 菜单 | [README](MaterialDrawer/README.md) | [类图](MaterialDrawer/class-diagram.html) | [流程图](MaterialDrawer/flow-diagram.html) | [MaterialDrawer](https://github.com/mikepenz/MaterialDrawer) |
| Android-ObservableScrollView | UI | [README](Android-ObservableScrollView/README.md) | [类图](Android-ObservableScrollView/class-diagram.html) | [流程图](Android-ObservableScrollView/flow-diagram.html) | [Android-ObservableScrollView](https://github.com/ksoichiro/Android-ObservableScrollView) |
| CircleImageView | 图片 | [README](CircleImageView/README.md) | [类图](CircleImageView/class-diagram.html) | [流程图](CircleImageView/flow-diagram.html) | [CircleImageView](https://github.com/hdodenhof/CircleImageView) |
| logger | 调试 | [README](logger/README.md) | [类图](logger/class-diagram.html) | [流程图](logger/flow-diagram.html) | [logger](https://github.com/orhanobut/logger) |
| agera | 异步 | [README](agera/README.md) | [类图](agera/class-diagram.html) | [流程图](agera/flow-diagram.html) | [agera](https://github.com/google/agera) |
| BottomBar | 菜单 | [README](BottomBar/README.md) | [类图](BottomBar/class-diagram.html) | [流程图](BottomBar/flow-diagram.html) | [BottomBar](https://github.com/roughike/BottomBar) |
| Calligraphy | 字体 | [README](Calligraphy/README.md) | [类图](Calligraphy/class-diagram.html) | [流程图](Calligraphy/flow-diagram.html) | [Calligraphy](https://github.com/chrisjenx/Calligraphy) |
| AndroidSlidingUpPanel | UI | [README](AndroidSlidingUpPanel/README.md) | [类图](AndroidSlidingUpPanel/class-diagram.html) | [流程图](AndroidSlidingUpPanel/flow-diagram.html) | [AndroidSlidingUpPanel](https://github.com/umano/AndroidSlidingUpPanel) |
| AppIntro | UI | [README](AppIntro/README.md) | [类图](AppIntro/class-diagram.html) | [流程图](AppIntro/flow-diagram.html) | [AppIntro](https://github.com/apl-devs/AppIntro) |
| recyclerview-animators | 动画 | [README](recyclerview-animators/README.md) | [类图](recyclerview-animators/class-diagram.html) | [流程图](recyclerview-animators/flow-diagram.html) | [recyclerview-animators](https://github.com/wasabeef/recyclerview-animators) |
| dagger | 依赖注入 | [README](dagger/README.md) | [类图](dagger/class-diagram.html) | [流程图](dagger/flow-diagram.html) | [dagger](https://github.com/google/dagger) |
| Android-Bootstarp | - | [README](Android-Bootstrap/README.md) | [类图](Android-Bootstrap/class-diagram.html) | [流程图](Android-Bootstrap/flow-diagram.html) | [Android-Bootstarp](https://github.com/Bearded-Hen/Android-Bootstrap) |
| RxBinding | 响应式 | [README](RxBinding/README.md) | [类图](RxBinding/class-diagram.html) | [流程图](RxBinding/flow-diagram.html) | [RxBinding](https://github.com/JakeWharton/RxBinding) |
| ListViewAnimations | 动画 | [README](ListViewAnimations/README.md) | [类图](ListViewAnimations/class-diagram.html) | [流程图](ListViewAnimations/flow-diagram.html) | [ListViewAnimations](https://github.com/nhaarman/ListViewAnimations) |
| UItimateRecyclerView | 图片 | [README](UItimateRecyclerView/README.md) | [类图](UItimateRecyclerView/class-diagram.html) | [流程图](UItimateRecyclerView/flow-diagram.html) | [UItimateRecyclerView](https://github.com/cymcsg/UltimateRecyclerView) |
| uCrop | UI | [README](uCrop/README.md) | [类图](uCrop/class-diagram.html) | [流程图](uCrop/flow-diagram.html) | [uCrop](https://github.com/Yalantis/uCrop) |
| RxJava-Android-Samples | 用例 | [README](RxJava-Android-Samples/README.md) | [类图](RxJava-Android-Samples/class-diagram.html) | [流程图](RxJava-Android-Samples/flow-diagram.html) | [RxJava-Android-Samples](https://github.com/kaushikgopal/RxJava-Android-Samples) |
| AndroidAutoLayout | 适配 | [README](AndroidAutoLayout/README.md) | [类图](AndroidAutoLayout/class-diagram.html) | [流程图](AndroidAutoLayout/flow-diagram.html) | [AndroidAutoLayout](https://github.com/hongyangAndroid/AndroidAutoLayout) |
| EffectiveAndroidUI | 性能 | [README](EffectiveAndroidUI/README.md) | [类图](EffectiveAndroidUI/class-diagram.html) | [流程图](EffectiveAndroidUI/flow-diagram.html) | [EffectiveAndroidUI](https://github.com/pedrovgs/EffectiveAndroidUI) |
| Luban | 图片 | [README](Luban/README.md) | [类图](Luban/class-diagram.html) | [流程图](Luban/flow-diagram.html) | [Luban](https://github.com/Curzibn/Luban) |
| DroidPlugin | 插件化 | [README](DroidPlugin/README.md) | [类图](DroidPlugin/class-diagram.html) | [流程图](DroidPlugin/flow-diagram.html) | [DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin) |
| otto | 响应式 | [README](otto/README.md) | [类图](otto/class-diagram.html) | [流程图](otto/flow-diagram.html) | [otto](https://github.com/square/otto) |
| FlycoTabLayout | UI | [README](FlycoTabLayout/README.md) | [类图](FlycoTabLayout/class-diagram.html) | [流程图](FlycoTabLayout/flow-diagram.html) | [FlycoTabLayout](https://github.com/H07000223/FlycoTabLayout) |
| android-testing | 测试 | [README](android-testing/README.md) | [类图](android-testing/class-diagram.html) | [流程图](android-testing/flow-diagram.html) | [android-testing](https://github.com/googlesamples/android-testing) |

## 待补充（42 个）

u2020、buck、PermissionsDispatcher、android-gif-drawable、Apktool、dynamic-load-apk、atlas、volley、androidmvp、SwipeBackLayout、FileDownloader、JieCaoVideoPlayer、glide-transformations、android-gpuimage、RxPermissions、freeline、RxLifecycle、classyshark、acra、DiskLruCache、dexposed、Litho、mosby、AndResGuard、StatusBarUtil、robolectric、Fragmentation、Small、JsBridge、richeditor-android、Transitions-Everywhere、android-viewbadger、AndroidWiFiADB、emojicon、packer-ng-plugin、android-priority-jobqueue、Android-Debug-Database、conceal、ARouter、MagicaSakura、CustomActivityOnCrash、XhsEmoticonsKeyboard

## 阅读建议

- 先看类图建立类型关系，再看流程图理解运行时序，最后读 README 的“阅读路径”定位源码文件。
- 排行榜基于 2017 年数据，部分框架已归档或被官方替代（Butter Knife、otto、androidannotations 等），保留它们是为了理解 Android 架构演进。
