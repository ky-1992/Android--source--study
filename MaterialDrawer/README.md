# MaterialDrawer 源码解析

## 一句话定位

MaterialDrawer 是 mikepenz 出品的 Android 侧滑抽屉库：把抽屉拆成「Builder 配置 → Drawer 运行时门面 → IDrawerItem 条目」三段，底层直接复用 `DrawerLayout` 与 FastAdapter/RecyclerView，不接管 Activity 的布局与窗口标志。调用方只需链式写配置，剩下的充气、适配器、点击与状态恢复都交给库内部完成。

## 源码快照

- 上游：`https://github.com/mikepenz/MaterialDrawer`
- 研读 commit：`fdbe67378e5efb931776fddf22072b9fc45baf4a`（tag `v7.0.0`）
- 仓库默认分支 `develop` 已演进到 `10.0.0-b01` 的 Kotlin 重写（`MaterialDrawerSliderView` 时代），`Drawer`/`DrawerBuilder` 在 `v8.0.0` 被移除；本篇以最后一代保留这两个类的 `v7.0.0` 为主线，`MIGRATION.md` 记录了这次断代。
- 模块：`library` 是核心库，`library-nav` 提供 `NavigationView` 兼容，`app` 是可跑示例；核心包 `com.mikepenz.materialdrawer` 下分 `model/`（菜单项）、`holder/`（颜色/图标/文案）、`util/`（装配工具）、`view/`。

## 核心类

| 类 | 职责 | 关键方法与行号 |
|---|---|---|
| `DrawerBuilder` | 链式配置并构建抽屉 | `build()` 1300、`buildView()` 1495、`withDrawerItems()` 991 |
| `Drawer` | 构建产物与运行时门面 | `setSelection()` 488、`addItem()` 654、`onBackPressed()` 970 |
| `AccountHeaderBuilder` | 账户头像区配置与构建 | `build()` 866、`switchProfiles()` 1097 |
| `AccountHeader` | 头像区对外 API | `setDrawer()` 80、`setActiveProfile()` 168 |
| `DrawerUtils` | 内部装配工具 object | `handleHeaderView()` 191、`onFooterDrawerItemClick()` 33 |
| `IDrawerItem` | 菜单项统一协议（接口） | 定义于 13 行，含 `identifier`/`bindView()` |
| `AbstractDrawerItem` | 选中态与子项骨架 | `bindView()` 273、`getViewHolder()` 324 |
| `BaseDrawerItem` | 图标与文案通用实现 | `withName()` 78、`withLevel()` 152 |
| `PrimaryDrawerItem` | 默认一级菜单项 | 6 行，继承 `AbstractBadgeableDrawerItem` |
| `MiniDrawer` | 抽屉迷你态接管 | `build()` 266、`generateMiniDrawerItem()` 236 |

## 关键流程

1. **配置期**：`withActivity()`、`withDrawerItems()`、`withAccountHeader()` 等只写字段并返回 `this`，示例见 `DrawerBuilder.kt` 991–1004。
2. **build 入口**：`build()` 先用 `mUsed` 拦截二次复用并抛异常（1301），缺省时创建 `DrawerLayout`，再交给 `MaterializeBuilder` 包装窗口，`handleDrawerNavigation()` 接管 `ActionBarDrawerToggle`，最后调用 `buildView()`。
3. **视图装配**：`buildView()` 充气 `material_drawer_slider`，调用 `createContent()` 创建 `RecyclerView`、装配头/尾与吸附区，再用 `Drawer(this)` 包装结果，末尾把 `mActivity` 置空（1541）。
4. **内容与适配器**：`createContent()` 中处理 `header`/`footer`/`stickyFooter`，最终 `mRecyclerView.adapter = adapter`（1701）并预选 `mSelectedItemPosition`。
5. **点击回调**：`adapter.onClickListener`（1718）先问条目自身的监听，再问 `Drawer` 级监听，MiniDrawer 仅在未消费时接管；未消费且非展开型条目会调用 `closeDrawerDelayed()`（1792）收起抽屉。

## 设计要点

- **配置与运行分离**：Builder 只管收集参数，产物 `Drawer` 只操作已建好的 View 与适配器，避免长生命周期持有 Activity。
- **菜单项即 RecyclerView item**：`IDrawerItem` 继承 FastAdapter 的 `IItem`/`IExpandable`，展开、多选、稳定 id 全部由 FastAdapter 扩展完成。
- **泛型自类型链式 API**：`AbstractDrawerItem<T, VH>` 的 `withXxx()` 返回 `T`，子类无需强转即可继续链式配置。
- **三段适配器**：`headerAdapter`/`itemAdapter`/`footerAdapter` 分开，头部与吸附 footer 不搅乱主列表的选中位次。
- **默认值兜底**：未传 `DrawerLayout` 时自动创建，`withCustomView()` 可整段替换内容区，降低接入成本。
- **状态可恢复**：`withSavedInstance()` 把选中项、吸附 footer 选中位与内容切换标记写进 `Bundle`，旋转后由 `buildView()` 还原，选中位次不会丢。
- **迷你态是旁路**：`MiniDrawer` 不侵入主列表，只在 `buildView()` 末尾按 `mGenerateMiniDrawer` 生成，并用 `ICrossFader` 做宽窄切换。

## 阅读路径

1. `library/src/main/java/com/mikepenz/materialdrawer/DrawerBuilder.kt`（60 行类定义 → 302 行 with 系列 → 1300 行 build）
2. `library/src/main/java/com/mikepenz/materialdrawer/Drawer.kt`（34 行类定义 → 488 行 setSelection）
3. `library/src/main/java/com/mikepenz/materialdrawer/DrawerUtils.kt`（191 行 handleHeaderView）
4. `library/src/main/java/com/mikepenz/materialdrawer/AccountHeaderBuilder.kt`（866 行 build → 1097 行 switchProfiles）
5. `library/src/main/java/com/mikepenz/materialdrawer/AccountHeader.kt`（16 行类定义）
6. `library/src/main/java/com/mikepenz/materialdrawer/model/interfaces/IDrawerItem.kt`
7. `library/src/main/java/com/mikepenz/materialdrawer/model/AbstractDrawerItem.kt`（273 行 bindView）
8. `library/src/main/java/com/mikepenz/materialdrawer/MiniDrawer.kt`（266 行 build）

## 版本注记

以 commit `fdbe673`（v7.0.0）为准。`v8.0.0` 起库不再自动注入 `DrawerLayout`，也不再提供 `Drawer`/`DrawerBuilder`，改用 `MaterialDrawerSliderView` 与 `AccountHeaderView` 组合；`v9`/`v10` 继续 Kotlin 化并精简依赖。若阅读默认分支源码，请以 `MIGRATION.md` 的 v8 升级说明对照本篇概念。本篇的类名、方法与行号都按该 commit 逐一核对，可以直接按行跳转。
