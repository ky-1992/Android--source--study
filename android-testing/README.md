# android-testing 源码解析

## 一句话定位

这不是一个测试框架，而是 Google 官方的"测试示例仓库"：把 Espresso、UI Automator、AndroidJUnitRunner、本地 JVM 单元测试四类做法拆成 23 个可独立编译的示例工程（`projects.conf` 精选 19 个纳入批量脚本），用同一个"输入文字→点按钮→看结果"的业务场景反复演示不同 API 的写法。

## 源码快照

- 上游：https://github.com/googlesamples/android-testing
- commit：`8c9df3a534ef99e44d481d96c00a5fc1970f7c70`
- 顶层按测试类型分 4 个目录：`ui/espresso`（14 个示例）、`ui/uiautomator`（1 个）、`runner`（3 个）、`unit`（4 个）、`integration`（1 个）
- 每个子目录是独立 Gradle 工程（各带 `settings.gradle` 与 `gradle-8.7` wrapper）；根目录 `projects.conf` 列出 19 行待测工程，`test_all.sh` 顺序执行 `./gradlew testDebug nexusOneApi30DebugAndroidTest`
- 版本参数见各示例 `build.gradle` 的 `ext`：AGP 8.5.0、Kotlin 1.9.22、`espressoVersion = "3.6.1"`、`runnerVersion = "1.6.1"`、`extJUnitVersion = "1.2.1"`、`robolectricVersion = "4.13"`

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `ChangeTextBehaviorTest`（BasicSample） | Espresso 最小用例：输入、点击、断言 | `changeText_sameActivity` :60、`changeText_newActivity` :71 |
| `MainActivity`（BasicSample） | 被测界面，两个按钮分走本页改字与跳转 | `onCreate` :40、`onClick` :53 |
| `ChangeTextBehaviorTest`（uiautomator） | 从桌面启动应用并用 `UiDevice` 操作 | `startMainActivityFromHomeScreen` :59、`testChangeText_sameActivity` :88 |
| `MainActivity`（uiautomator） | 同一套布局的跨进程被测界面 | `onCreate` :41、`onClick` :53 |
| `LongListActivityTest` | `onData` 入口演示列表滚动与行内控件 | `row_Click` :96、`onRow` :143 |
| `LongListActivity` | 100 行 `ListView`，暴露 `ROW_TEXT`/`ROW_ENABLED` | `onCreate` :59、`makeItem` :77 |
| `HintMatchersTest` | 用自定义 matcher 断言 `EditText` 的 hint | `hint_isDisplayedInEditText` :85、`hint_endsWith` :96 |
| `HintMatcher` | 继承 `BoundedMatcher<View, EditText>` 扩展 Espresso | `withHint` :34、:38 |
| `SimpleIdlingResource` | 用 `AtomicBoolean` 表示空闲状态的替身 | `isIdleNow` :44、`setIdleState` :57 |
| `ChangeTextBehaviorTest`（IdlingResource） | 注册/反注册空闲资源 | `registerIdlingResource` :57、`unregisterIdlingResource` :81 |
| `DialerActivityTest` | Espresso Intents 的 `intending`/`intended` 用法 | `typeNumber_ValidInput_InitiatesCall` :89 |
| `EmailValidatorTest` | 纯 JVM 单测，无设备依赖 | `isValidEmail` 用例 :32-63 |

## 关键流程

1. **运行器接管**：示例在 `build.gradle` 声明 `testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"`，用例类用 `@RunWith(AndroidJUnit4.class)` + `@LargeTest` 标记，由 instrumentation 安装后启动。
2. **拉起界面**：`@Rule public ActivityScenarioRule<MainActivity>` 在每条用例前创建并启动 Activity，替代旧 `ActivityTestRule`；需要访问 Activity 实例时改用 `ActivityScenario.launch` + `onActivity`。
3. **定位与操作**：Espresso 侧 `onView(withId(...)).perform(typeText(...), closeSoftKeyboard())`；列表侧 `onData(hasEntry(equalTo(LongListActivity.ROW_TEXT), is(str)))` 取到 `DataInteraction` 后 `onChildView` 定位行内控件，滚动由 `onData` 负责。
4. **异步同步**：IdlingResource 示例里 `MessageDelayer` 用 `Handler.postDelayed` 延迟 3 秒；用例在 `@Before` 从 Activity 取 `getIdlingResource()` 并 `IdlingRegistry.getInstance().register(...)`，`@After` 反注册，否则断言会在延迟结束前就失败。
5. **跨进程操作**：UI Automator 用例先 `UiDevice.getInstance(getInstrumentation())`、`pressHome()`，再等 launcher 包名出现，用 `By.res(packageName, id)` 找到 `UiObject2` 并 `setText`/`click`，最后 `Until.findObject` 轮询等待结果。

## 设计要点

- **一套业务、多种打法**：BasicSample（Espresso）与 uiautomator/BasicSample 的包名、界面 id（`editTextUserInput`、`changeTextBt`、`textToBeChanged`）完全对齐，方便读者横向对比"进程内匹配"与"无障碍节点树"两种定位方式。
- **示例即脚手架**：每个示例自带 `gradlew` wrapper、`settings.gradle` 与 `build.gradle`，可单独 `./gradlew` 运行，不依赖仓库根构建。
- **测试代码分层**：需要共享给 Robolectric 与设备测试的用例放 `sharedTest`（如 CustomMatcherSample、DataAdapterSample），纯设备用例放 `androidTest`，纯 JVM 用例放 `test`。
- **为测试改生产代码**：`LongListActivity` 的 `ROW_TEXT`、`NUMBER_OF_ITEMS` 与 `MainActivity.getIdlingResource()` 都标了 `@VisibleForTesting`，说明示例接受"为可测性暴露少量内部状态"。
- **同时演示新旧 API**：`IntentsBasicSample` 仍用 `IntentsTestRule` 与 `ActivityTestRule`，而 `BasicSample` 已切到 `ActivityScenarioRule`，对比阅读能看清迁移方向。
- **套件与参数化**：`runner/AndroidJunitRunnerSample` 用 `@RunWith(Suite.class)` 把 `UnitTestSuite` 与 `InstrumentationTestSuite` 组成 `AndroidTestSuite`，另有 `CalculatorAddParameterizedTest` 以 `@RunWith(Parameterized.class)` 批量喂参数，展示了同一 Runner 下的三种组织粒度。
- **跨进程能力**：`MultiProcessSample` 的用例先用 `ActivityScenario.launch` 起主进程界面，再点按钮拉起 `:PID2` 进程，仍在同一用例里用 `onView`/`onData` 断言 `textPrivateProcessName` 与列表项，说明新版 Espresso 已支持多进程；UI Automator 则从无障碍节点树出发，可操作被测 App 之外的系统界面，两者取舍在于测试范围的边界。

## 阅读路径

1. `README.md`（示例索引，先选场景）
2. `ui/espresso/BasicSample/app/src/androidTest/java/com/example/android/testing/espresso/BasicSample/ChangeTextBehaviorTest.java`
3. `ui/espresso/BasicSample/app/build.gradle`（依赖与 `testInstrumentationRunner`）
4. `ui/espresso/DataAdapterSample/app/src/sharedTest/java/.../LongListActivityTest.java`
5. `ui/espresso/CustomMatcherSample/app/src/sharedTest/java/.../HintMatcher.java`
6. `ui/espresso/IdlingResourceSample/app/src/main/java/.../IdlingResource/SimpleIdlingResource.java` + `MessageDelayer.java`
7. `ui/uiautomator/BasicSample/app/src/androidTest/java/.../ChangeTextBehaviorTest.java`
8. `unit/BasicSample/app/src/test/java/.../EmailValidatorTest.java` 与 `runner/AndroidJunitRunnerSample/.../suite/*.java`

## 版本注记

仓库没有单一版本号，示例各自维护 `ext` 版本块：BasicSample 声明 `compileSdk 34` / `minSdkVersion 21`、AGP 8.5.0、Espresso 3.6.1，`unit/BasicSample` 仍停留在 JUnit 4.12 + Mockito 1.10.19。以该 commit 为准，`ActivityScenarioRule` 已是主流写法，`ActivityTestRule` 仅残留在 Intents 示例中。
