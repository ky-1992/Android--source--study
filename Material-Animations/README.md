# Material-Animations 源码解析

> 排行榜 #20 · 类别：动画

## 一句话定位

Material-Animations 是 Google 官方 Material Design 动画示例工程（教学型项目），用 3 个页面演示 **Activity 转场（Transition）**、**共享元素（Shared Element）** 与 **Fragment 转场** 的正确用法：`ActivityOptions.makeSceneTransitionAnimation` + `Window.setEnterTransition/setExitTransition` + `SharedElementCallback`。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/lgvalle/Material-Animations |
| 快照 commit | `be755fe9ebe7f9093360e4263c4293de368da1c7` |
| 克隆方式 | `--depth 1` |

工程结构：一个 `app` 模块，`MainActivity` 列表 + 若干 `TransitionActivity` / `SharedElementActivity` / `SharedElementFragment` 示例页。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `MainActivity` | 示例列表入口，点击进入各动画 Demo | `MainActivity.java` L15、L28、L52 |
| `SamplesRecyclerAdapter` | 列表适配器：收集共享元素并启动目标 Activity | `SamplesRecyclerAdapter.java` L17、L33、L76 |
| `SamplesViewHolder` | 列表项（含共享元素 View 引用） | `SamplesRecyclerAdapter.java` L89 |
| `Sample` | 示例数据模型 | `Sample.java` L13、L24 |
| `BaseDetailActivity` | 详情页基类：设置转场、共享元素回调 | `BaseDetailActivity.java` L13、L19、L32 |
| `TransitionHelper` | 工具类：`makeSceneTransitionAnimation` 的封装与共享元素收集 | `TransitionHelper.java` L32、L43、L55 |
| `TransitionActivity1/2/3` | 演示 Activity 入场/退场/共享元素动画 | `TransitionActivity1.java` L31、L99；`TransitionActivity2.java` L32；`TransitionActivity3.java` L34 |
| `SharedElementActivity` | 纯共享元素转场示例 | `SharedElementActivity.java` L29 |
| `SharedElementFragment1/2` | Fragment 间共享元素转场 | `SharedElementFragment1.java` L56；`SharedElementFragment2.java` L14 |

## 关键流程

### 1. 点击列表项，收集共享元素

`SamplesRecyclerAdapter` 的 item 点击里，用 `TransitionHelper.makeSceneTransitionAnimation(activity, pair)` 收集要共享的 View 对（`Pair<View, String>`，String 是 `transitionName`）。没有共享元素时退化为普通 Activity 启动。

### 2. 构造过渡选项并启动

```java
ActivityOptionsCompat options =
    ActivityOptionsCompat.makeSceneTransitionAnimation(activity, pairs);
ActivityCompat.startActivity(activity, intent, options.toBundle());
```

选项交给系统窗口管理器，由它负责在 Activity 间建立转场。

### 3. 目标页设置入场/退场与共享元素回调

`BaseDetailActivity` / `TransitionActivity1..3` 在 `onCreate` 里：

- `getWindow().setEnterTransition(...)` / `setExitTransition(...)` / `setReenterTransition(...)` 控制整页转场；
- `setSharedElementEnterTransition(...)` 指定共享元素的移动曲线；
- `SharedElementCallback` 在 `onSharedElementStart/End` 里做精细控制（如改变 ImageView 的 scaleType）。

### 4. 返回过渡

`onBackPressed` 里调用 `supportFinishAfterTransition()`（或 `finishAfterTransition`），系统反向播放共享元素动画回到列表页。

### 5. Fragment 共享元素

`SharedElementFragment1/2` 用 `FragmentTransaction.addSharedElement(view, transitionName)` 在 Fragment 之间共享元素，并配合 `setReorderingAllowed(true)` 与自定义 `Transition`。

## 设计要点

1. **框架能力演示而非造轮子**：全部基于 `android.transition` / `ActivityOptions` 系统 API，价值在于“正确用法 + 常见坑”。
2. **transitionName 是契约**：共享元素靠 `transitionName` 匹配，源码里每个示例都显式设置。
3. **Activity 与 Fragment 两套用法**：Activity 走 `ActivityOptions` + Window，Fragment 走 `FragmentTransaction.addSharedElement`。
4. **可组合**：整页转场与共享元素转场可以叠加，`TransitionHelper` 把样板代码收敛到一处。

## 阅读路径

1. `app/src/main/java/com/lgvalle/material_animations/MainActivity.java`
2. `.../SamplesRecyclerAdapter.java`（点击 + 启动 + 共享元素收集）
3. `.../TransitionHelper.java`（`makeSceneTransitionAnimation` 封装）
4. `.../BaseDetailActivity.java`（Window 转场设置）
5. `.../TransitionActivity1.java`、`TransitionActivity2.java`、`TransitionActivity3.java`
6. `.../SharedElementActivity.java`
7. `.../SharedElementFragment1.java`、`SharedElementFragment2.java`

## 版本注记

- 该 commit 是示例工程最后状态，依然使用 `android.support.v7` 包（排行榜成文年代），现代工程应换 AndroidX 对应 API（`ActivityOptionsCompat`、`supportFinishAfterTransition` 名称不变）。
- 它不是库，没有 release 版本概念；以该 commit 为准。
