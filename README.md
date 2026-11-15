# Flutter 适配 iPhone Duo 手册

更新于 2026-11-16

## 概览

一个 Flutter app 要在 iPhone Duo 上表现正确，需要处理四件事：用 iOS 27.1 SDK 编译、实时检测折叠状态、把导航和顶栏放进状态栏所在的位置、修正展开后的界面方向。本手册来自 [PiliPlus](https://github.com/ShionjiART/PiliPlus) 的适配，结论主要在 iOS 27.1 的 iPhone Duo 模拟器上验证；合上状态冷启动、展开、画中画也在 iPhone Duo 真机（iOS 27.1）上验证过，展开竖屏的顶栏还没在真机上确认。

对应的代码（PiliPlus 的 `release` 分支）：

- 折叠检测：[`lib/utils/ios/foldable_helper.dart`](https://github.com/ShionjiART/PiliPlus/blob/release/lib/utils/ios/foldable_helper.dart)
- 原生部分（系统标签栏、竖栏按钮、铰链处理、顶栏位置）：[`ios/Runner/SceneDelegate.swift`](https://github.com/ShionjiART/PiliPlus/blob/release/ios/Runner/SceneDelegate.swift)
- Dart 端的标签栏和竖栏按钮：[`lib/utils/ios/native_*.dart`](https://github.com/ShionjiART/PiliPlus/tree/release/lib/utils/ios)
- 左右安全区：[`lib/common/widgets/view_safe_area.dart`](https://github.com/ShionjiART/PiliPlus/blob/release/lib/common/widgets/view_safe_area.dart)

| 项目 | 外屏（合上） | 内屏（展开） |
| --- | --- | --- |
| 窗口尺寸（pt） | 466 × 678 | 951 × 669 横拿；669 × 951 竖拿 |
| 最短边 | 466，小于 600 | 669，不小于 600 |
| 竖状态栏（安全区） | 右侧 84 | 右侧约 82–84；旋转后会变成顶部 82 |
| 底部安全区 | 34 | 34 |
| 系统方向模式 | `dynamic`：遵守 app 的方向锁 | `alwaysAll`：始终允许所有方向，app 用代码改方向会被拒绝（`UISceneErrorDomain` 101） |

要处理的四类问题：

1. **编译**：iOS 27.1 以下的 SDK 编译的 app，在内屏上以兼容模式小窗运行。
2. **折叠检测**：展开或合上时窗口尺寸会变，Flutter 里缓存过的布局值必须改成实时读取。
3. **竖状态栏**：时间、Wi-Fi 在机身一侧的竖条里。原生 app 的标签栏和返回键会自动放进去，Flutter 只会把它当安全区空着。展开竖屏时状态栏只占右上角，页面顶栏要放进那一行（见「布局规范：栏放在哪里」）。
4. **方向**：合上时锁竖屏的 app，展开后界面会转错 90° 或 180°，需要原生代码修正。

## 构建与运行环境

必须用 Xcode 27.1（iOS 27.1 SDK）编译；模拟器上要绕开 Flutter 工具链，直接用 xcodebuild。

- **SDK**：编译后用 `PlistBuddy -c "Print :DTSDKName" Runner.app/Info.plist` 确认是 `iphoneos27.1`。
- **CI**：截至 2026-10，GitHub 托管的 `macos-26` 最高只有 Xcode 26.6，预览版 `xcode-27` 只有 27.0 beta，都不够。需要等镜像更新，或用自建 runner。
- **模拟器**：Xcode 27 里没有 Simulator.app，换成了 `Xcode.app/Contents/Applications/DeviceHub.app`。窗口底部的 Closed / Partially Open / Open 按钮用来模拟合上、半开、展开。
- **Flutter SDK**：PiliPlus 只能用打过补丁的 SDK 编译，见 PiliPlus 仓库 [`CLAUDE.md`](https://github.com/ShionjiART/PiliPlus/blob/release/CLAUDE.md)「构建」一节。
- **flutter run 不可用**：Duo 模拟器报告的架构是 `arm64e.x1`，写这份手册时的 Flutter 3.47.8 不认识，报 `No Xcode build settings have been found`。
- **不指定设备的 `flutter build ios --simulator` 会编成 x86\_64**，而 iOS 27 模拟器只跑 arm64，安装时报「需要更新」。如果某个依赖只提供 x86\_64 的模拟器版本（PiliPlus 里是极验的 `GT3Captcha-iOS`），arm64 编译会失败，模拟器调试时要暂时去掉它。

模拟器编译与安装（先让 Flutter 只生成配置，再用 xcodebuild 按 arm64 编译）：

```bash
flutter build ios --simulator --debug --config-only
xcodebuild -workspace ios/Runner.xcworkspace -scheme Runner \
  -configuration Debug -sdk iphonesimulator \
  ARCHS=arm64 ONLY_ACTIVE_ARCH=NO \
  -derivedDataPath build/ios_sim_arm64 build
xcrun simctl install <UDID> build/ios_sim_arm64/Build/Products/Debug-iphonesimulator/Runner.app
xcrun simctl launch <UDID> <bundle id>
```

在干净的构建目录里，`--config-only` 可能要跑两次：第一次可能会在 `ios/Flutter/Generated.xcconfig` 里写入 `EXCLUDED_ARCHS[sdk=iphonesimulator*]=i386 arm64`，导致 arm64 编译失败。运行 xcodebuild 前先检查这一行。

改了 Dart 代码后只需重跑 xcodebuild（它的 Run Script 阶段会重新编译 Dart）。debug 包不接 Flutter 工具也能独立运行。

## 检测折叠状态

在 iPhone 上，窗口最短边 ≥ 600pt 就是展开。所有直板 iPhone 的最短边都小于 600，所以不需要识别机型。

```dart
abstract final class IOSFoldableHelper {
  static bool _isIPhone = false;
  static bool? _lastUnfolded;

  static Future<void> init() async {
    final info = await DeviceInfoPlugin().iosInfo;
    _isIPhone = info.model.toLowerCase().contains('iphone');
    if (_isIPhone) WidgetsBinding.instance.addObserver(_FoldObserver());
  }

  static bool get isUnfolded {
    final v = WidgetsBinding.instance.platformDispatcher.views.first;
    return (v.physicalSize / v.devicePixelRatio).shortestSide >= 600;
  }

  static void _onMetricsChanged() {
    final unfolded = isUnfolded;
    if (unfolded == _lastUnfolded) return; // 只在状态变化时处理
    _lastUnfolded = unfolded;
    // 合上 → 锁竖屏；展开 → 放开所有方向（见「全屏播放与方向锁」）
  }
}

class _FoldObserver with WidgetsBindingObserver {
  @override
  void didChangeMetrics() => IOSFoldableHelper._onMetricsChanged();
}
```

要点：

- 在 `main()` 里、第一次设置方向之前 `await IOSFoldableHelper.init()`。
- 首帧前窗口尺寸可能是 0，要忽略。
- **不要缓存布局相关的值**。代码里的 `late final x = Pref.horizontalScreen` 这类写法要改成 getter（`bool get x => ...`），否则展开后布局不会变。先全局搜一遍这类字段。
- 若 app 有「横屏/平板模式」开关，在 iPhone 上提供「自动 / 开 / 关」三选一，而不是再加一个独立开关。两个开关同时存在时，手动开关会显示一个已经不生效的旧值。

## 竖状态栏布局

系统把竖状态栏作为一侧的安全区（约 84pt）告诉 app。有两种做法：用系统标签栏，让系统自己把它放进竖栏（PiliPlus 现在的做法）；或者由 Flutter 把导航画进这条安全区（PiliPlus 只作为后备）。

### 系统标签栏（iOS 26+ 的 iPhone）

把 `FlutterViewController` 放进 `UITabBarController`，主导航改用系统的液态玻璃标签栏。系统在 Duo 上会把它放进竖栏：合上、展开横拿时在竖栏里，展开竖拿时悬浮在底部。

结构（都在 `SceneDelegate.scene(_:willConnectTo:options:)` 里替换窗口的根控制器）：

```text
NativePageNavigationController   仅 iOS 27.1 SDK：放返回键和页面按钮，导航栏本身隐藏
└─ NativeTabBarController        系统标签栏
   └─ FlutterHostViewController  每个标签一个，只装同一个 FlutterViewController
      └─ DuoFlutterViewController
```

- 只有一个 Flutter 视图。切换标签时，把它移到选中标签的容器里（`tabBarController(_:shouldSelect:)`）。
- 标签内容由 Dart 通过 `piliplus/native_tab_bar` 通道下发，未读数显示为系统角标。进入二级页面时隐藏标签栏。
- iOS 26 以下或不是 iPhone 时，不替换根控制器，继续用 Flutter 自己画的导航。

这样做有四个坑：

1. **启动停在启动屏**：容器不自动转发出现/消失事件（`shouldAutomaticallyForwardAppearanceMethods` 为 false，这样在标签间移动 Flutter 视图时引擎不会暂停）。于是 `FlutterViewController` 可能一直不创建渲染表面，真机上合上状态冷启动会停在启动屏。要由标签栏控制器手动调 `beginAppearanceTransition` / `endAppearanceTransition`，第一次要等视图进了窗口、布局完再转发。
2. **安全区为 0**：Flutter 视图不是窗口根控制器时，拿不到系统安全区。由容器在 `viewSafeAreaInsetsDidChange` 里把安全区发给 Dart，在 `main.dart` 的 builder 里覆盖 `MediaQuery` 的内边距。
3. **方向锁失效**：系统从根控制器读取允许的方向。每一层容器都要把 `supportedInterfaceOrientations` 转给子控制器；Flutter 调 `setNeedsUpdateOfSupportedInterfaceOrientations` 时，也要转给窗口根控制器。
4. **状态栏样式**：容器要通过 `childForStatusBarStyle`、`childForStatusBarHidden` 等把状态栏交给 Flutter 控制。

**竖栏里的页面按钮**（需要 iOS 27.1 SDK）：在标签栏外面再包一层 `UINavigationController`，导航栏平时隐藏。二级页面时显示导航栏，返回键会出现在竖栏里；一级页面在同一位置放头像、搜索、私信。`toolbarItems` 也会进竖栏，动态详情页的点赞、评论等就放在这里。是否有竖栏用 `traitCollection.verticalBarEdge != .unspecified` 判断，并用 `registerForTraitChanges(UITraitCollection.systemTraitsAffectingVerticalBarEdge)` 监听开合。竖栏有返回键时，Flutter 页面左上角的返回键要去掉，否则会有两个（PiliPlus 用 `lib/scripts/material/app_bar_native_back.patch` 给 `AppBar` 加了钩子）。

**视频页隐藏竖栏**：视频页通知原生层后，标签栏控制器在有竖栏时让 `prefersStatusBarHidden` 返回 true，内容铺满竖栏的位置，和官方客户端一样。注意：

- 状态栏隐藏后，竖栏的安全区仍然存在（摄像头在里面）。原生层要把「竖栏已隐藏」告诉 Dart，由 Dart 决定哪些内容可以铺过去：合上时播放器控件继续避开它，下方内容铺满。
- 不要用 `preferredVerticalBarBehavior = .disabled`：它会把状态栏改回顶部横排，整个布局都会变。只想在某个页面隐藏时，用状态栏的显示/隐藏接口。
- 底部弹窗在这个页面也要横向铺满，不能再按左右安全区缩进。

### Flutter 自己画（后备）

没有启用系统标签栏时，PiliPlus 把主导航竖着画进这条安全区，排在 Wi-Fi 下面。iOS 27.1 的 Duo 上系统标签栏总会启用，所以这部分只是后备。下面的第 1、3 个坑与导航用哪种做法无关，都要处理。

**识别**：iPhone 且 `MediaQuery.viewPaddingOf(context).right >= 72`。普通 iPhone 横屏时灵动岛一侧约 59–62pt，不会误判。设备旋转后状态栏会变成顶部安全区，这时退回原来的布局即可。原生层可以用 iOS 27.1 的 `UITraitCollection.verticalBarEdge`（leading / trailing / unspecified）更精确地判断，上面的系统标签栏就是这样做的。

**布局**：内容区去掉右侧安全区，导航宽度等于安全区宽度、垂直居中。

```dart
Row(children: [
  Expanded(
    child: MediaQuery.removePadding(
      context: context,
      removeRight: true,
      child: Padding(
        padding: EdgeInsets.only(top: padding.top, left: padding.left),
        child: KeyedSubtree(key: _bodyKey, child: pages), // 见下
      ),
    ),
  ),
  MediaQuery.removePadding(
    context: context,
    removeLeft: true, removeTop: true, removeRight: true,
    child: NavigationRail(
      minWidth: padding.right,
      groupAlignment: 0.0, // 垂直居中，落在 Wi-Fi 下方
      labelType: NavigationRailLabelType.all,
      backgroundColor: Colors.transparent,
      destinations: ...,
    ),
  ),
])
```

三个坑：

1. **页面状态丢失**：合上或展开的过渡中安全区会短暂变化，布局在两种结构间来回切。PageView 被重建后会回到 `initialPage`，导航高亮和内容就对不上。给内容区套 `KeyedSubtree(key: GlobalKey())`，让它在结构间移动时保留状态。
2. **顶部栏依赖导航模式**：如果首页是否显示搜索栏取决于「是否用底部导航」或「是否竖屏」，状态栏导航模式下要按手机模式处理，否则横拿展开时搜索入口会消失。
3. **宽度公式要扣安全区**：按 `MediaQuery.size.width` 算分栏宽度、最后才减安全区的写法，在 Duo 上会把侧栏压窄 84pt。PiliPlus 视频页右栏就因此从 280 变成 200，标签重叠、列表溢出。改成先算 `availableWidth = width - padding.horizontal`，再套公式。iPad 没有这条栏，所以上游一般没碰到过。

### 二级页面

没有读左右安全区的页面，内容会铺到竖栏下面。PiliPlus 的做法：

- `Routes.getPages` 里的命名路由统一套上 `SideSafeArea`：按左右安全区留出内边距，同时从 `MediaQuery` 里移除，已经自己处理的页面不会重复留白。主页、视频页、直播间自己处理，不套。
- 用 `Get.to(...)` 直接打开的页面不走命名路由，改用 `Get.toSafe(...)`，效果相同。
- 播放器的顶栏、底栏和隐藏控件时的细进度条，始终避开左右安全区；否则合上时最右边的按钮会被竖栏盖住，点不到。

## 布局规范：栏放在哪里

每种姿态下，标题栏、工具栏、标签栏都有系统指定的位置：合上和展开横屏时在侧边竖栏里，展开竖屏时在顶部状态栏那一行里。依据是 Apple 的设计指南和开发文档、9 个已适配 app 的真机截图，以及 iPhone Duo 模拟器（iOS 27.1 运行时，Xcode 27.2 SDK）上的实测；真机数值略有不同（例如展开竖屏的顶部安全区，真机约 79、模拟器 82），所以代码一律向系统读取。

1. **栏放在系统指定的位置**。指南：一般不要改默认的栏位置。
2. **不留空**。状态栏只占一角时，那一行其余的地方给页面顶栏用；只有没适配的 app 才空着一行。
3. **只避开系统占用的区域**。摄像头、状态栏、折痕都由系统以「占用区域」给出，按它让开，其他地方照常用。
4. **背景铺满，内容让开**。头部背景、头图、播放画面铺到屏幕边缘，文字和按钮让开安全区和占用区域。
5. **跟着姿态变**。开合、旋转、半开都会改变这些位置，布局要在系统数值变化时重新计算，不能只在页面创建时决定一次。

| 姿态 | 窗口（pt） | 状态栏在哪 | 标题栏、工具栏、标签栏放哪 | 内容区 |
| --- | --- | --- | --- | --- |
| 外屏竖屏（合上） | 466 × 678 | 右侧竖栏顶端，和摄像头在一起 | 右侧竖栏里，从上到下：摄像头 → 状态栏 → 工具栏 → 标签栏 | 宽 382，从屏幕顶边开始 |
| 展开竖屏 | 669 × 951 | 顶部右上角一块（约 144 × 82） | 顶栏放进状态栏那一行；标签栏在底部 | 从约 82 开始，往上滚时滚进那一行底下 |
| 展开横屏 | 951 × 669 | 右侧竖栏顶端 | 右侧竖栏里，和外屏竖屏一致 | 宽 867，从屏幕顶边开始 |
| 半开 | 同展开 | 同展开 | 同展开 | 避开中间 40 宽的折痕 |
| 外屏横屏 | — | 未实测 | 指南：竖栏和硬件对齐，与外屏摄像头同侧 | 未实测 |

## 展开竖屏：顶栏放进状态栏那一行

展开后竖着拿，状态栏只占右上角，页面的顶栏就放在这一行里，而不是另起一行放在下面。系统「设置」推进「关于本机」时，返回键和标题就在这一行；指南说这个姿态「保留标准的横向栏」，意思是栏不移到侧边，而不是放在状态栏下面。

- **位置**：用系统给的顶栏区域。左边约 16（避开圆角），右边让开状态栏（约 144）；顶栏在这一行里垂直居中，中线约 y = 48，上边距不小于 24。高 44 的栏上边距 26，高 48 或 56 的栏上边距 24。
- **排列**：返回键在左，标题居中或大标题靠左，操作按钮靠右但停在状态栏左边。
- **内容**：从这一行下面开始（约 82），往上滚时滚进这一行底下。原生的状态栏胶囊会自己带一块底色；Flutter 里自己画的顶栏如果会随滚动收起，要把它裁在页面顶边或者同时隐藏状态栏，否则会画到状态栏上。
- **大标题页**：大标题就放在这一行，往上滚时滚走；带搜索框的页面，搜索框往上滚时升进这一行，停在状态栏左边。
- **头部背景**铺到最顶，按钮仍按上面的位置放。
- **底部标签栏**保持在底部。
- **沉浸式页面**（视频播放）：隐藏状态栏，画面铺到最顶。隐藏后顶部安全区是 0，也没有占用区域：内屏前置摄像头藏在屏幕下面，只有开启时才变为占用，那时界面要让开。

PiliPlus 的做法：给 Material 的 `AppBar` / `SliverAppBar` / `Scaffold` 加一个钩子（补丁 `app_bar_status_bar_row.patch`），主标题栏在这个姿态下按系统位置放进这一行；自己按「安全区 + 标题栏高度」排版的可折叠头部改用同一个取值函数。只对贴着屏幕顶边、顶部安全区没被外层用掉的标题栏生效，底部弹窗里的不受影响。

## 外屏竖屏与展开横屏：栏在竖栏里

合上时状态栏和摄像头都在右侧竖栏顶端，屏幕顶边是空的，页面的第一行（标题、分类标签）直接从顶部开始。展开横屏和它一致，保证开合前后按钮位置不变。

- **竖栏从上到下**：摄像头（灵动岛）→ 状态栏 → 工具栏（返回、关闭等主要导航放最上面，其次是「完成」这类主要操作，再其次是其余按钮）→ 标签栏（最下面）。页面的顶栏按钮和底部标签栏都移进竖栏。
- **顶部安全区**：系统报告的 24 来自圆角；真机在通话等情况下可能报约 20，而顶边实际没有东西。PiliPlus 在侧边安全区达到竖栏宽度时把顶部安全区当作 0，页面自己留出圆角边距。
- **内容区宽度**：外屏 466 − 84 = 382，展开横屏 951 − 84 = 867。背景和头图可以铺到竖栏下面（原生用 `UIBackgroundExtensionView`）。
- **隐藏竖栏的沉浸式页面**：可以铺满整个宽度，但外屏摄像头始终占用，按钮要避开 (382, 0, 84, 174)；不怕被挡的东西（进度条）可以伸进去。竖栏隐藏后，弹窗在整个屏幕里居中。
- **按钮要有图标**：竖栏里只显示图标；只有文字的按钮和自定义视图不会放进竖栏，会收进「更多」菜单。
- **空间不够时**：以导航为主的页面保留标签栏、把工具栏按钮收进「更多」；以任务为主的页面保留工具栏、把标签栏缩成一个按钮。
- **分栏**（展开横屏）：标题栏按钮放在它控制的那一栏上方，只有最右边那一栏的按钮进竖栏。
- **半开**：折痕变为「分割」区域并生效，横屏时是中间 40 宽的竖条，竖屏时是中间 40 高的横条。重要内容不放在里面；分栏改成等宽、网格用偶数列，只做小调整。

外屏横屏没有实测。指南说竖栏和硬件对齐，始终和外屏摄像头在同一侧；PiliPlus 合上时锁竖屏，允许外屏横屏的 app 要自己实测。

## 用系统给的位置，不写死

上面的位置都由 UIKit 给出：Flutter 拿不到这些值，由原生层（宿主 view controller）读取后通过 MethodChannel 发给 Dart。

| 要什么 | 接口 | 说明 |
| --- | --- | --- |
| 状态栏、摄像头、折痕的位置 | `view.reservedRegions(kind: .occlusion / .division, options: .includeInactive)` | iOS 27.1。每项有 frame 和 isActive；状态栏被 app 隐藏后，状态栏那块不再出现 |
| 顶栏或竖栏该放哪 | `view.edgeInsets(for: .bar(onEdge: UIRectEdge.top, extent: 44))` | iOS 27.1。返回四边距离；extent 是栏的高度（竖栏时是宽度）。Swift 里只写 `.top` 会报「ambiguous」 |
| 竖栏在哪一边 | `traitCollection.verticalBarEdge` | iOS 27.1。`.unspecified` 表示没有竖栏（展开竖屏） |
| 圆角避让 | `view.edgeInsets(for: .safeArea(cornerAdaptation: .horizontal))` | iOS 26。只给出左右约 16 的圆角距离，对顶部那一行没有帮助 |

- 在 `viewDidLayoutSubviews` 里读取并去重后发送：安全区变化的回调来得早，那时占用区域可能还没算出来。
- app 自己隐藏状态栏时（视频页、顶栏收起），保留状态栏显示时的值，否则布局会跳。
- Dart 侧用可监听的值（`ValueNotifier` + `InheritedNotifier`）保存，相关组件依赖它重建，开合、旋转时才会跟着变。
- 原生值是原生 pt，app 有界面缩放设置时要和安全区一样换算。
- 外层如果用 `SafeArea` 吃掉了左右安全区，子页面里 `MediaQuery` 的 viewPadding 也会变成 0；要读原始值用 `View.of(context).viewPadding`。

```swift
// 发给 Dart 的「状态栏那一行里的顶栏位置」（只有展开竖屏才有值）
let cornerOnly = view.reservedRegions(kind: .occlusion).contains {
  $0.frame.minY <= 0 && $0.frame.width < view.bounds.width / 2
}
if traitCollection.verticalBarEdge == .unspecified && cornerOnly {
  let bar = view.edgeInsets(for: .bar(onEdge: UIRectEdge.top, extent: 44))
  let tall = view.edgeInsets(for: .bar(onEdge: UIRectEdge.top, extent: 64))
  // height = 顶部安全区；top/left/right = bar；
  // center = bar.top + 22，minTop = tall.top；高 h 的栏上边距 = max(minTop, center - h/2)
}
```

实测数值（模拟器，区域为 (x, y) 宽×高）：

| 姿态 | 安全区 上/左/下/右 | 顶栏 extent 44 | 竖栏（右，extent 48） | 占用区域 |
| --- | --- | --- | --- | --- |
| 外屏竖屏 466×678 | 24 / 0 / 34 / 84 | 上 24 左 2.3 右 84 | x 394，上 174，下 34 | 摄像头 (399.7, 29.3) 37×37；摄像头 + 状态栏 (382, 0) 84×174 |
| 展开竖屏 669×951 | 82 / 0 / 83 / 0 | 上 26 左 16 右 144 | 无竖栏 | 状态栏 (525, 0) 144×82；内屏摄像头 (611, 677.3) 37×58 未生效；折痕 (0, 455.5) 669×40 未生效 |
| 展开竖屏，状态栏隐藏 | 0 / 0 / 34 / 0 | 上 26 左 16 右 16 | 无竖栏 | 顶部无占用 |
| 展开横屏 951×669 | 24 / 0 / 34 / 84 | 上 24 左 16 右 84 | x 879，上 124，下 34 | 状态栏 (867, 0) 84×124；折痕 (455.5, 0) 40×669 未生效 |
| 半开（横） | 24 / 0 / 34 / 84 | 上 24 左 16 右 84 | x 879，上 124，下 34 | 折痕 (455.5, 0) 40×669 生效 |

## 常见错误与真机对照

最常见的错误是把状态栏那一行空着，标题栏另起一行放在下面；这是没适配的表现，不是指南的要求。

- **写死宽度或高度**（状态栏宽 144、顶部 82、竖栏 84）。真机和模拟器不同，状态栏图标多了也会变宽。
- **用「侧边安全区 ≥ 72」判断竖栏**。能用，但系统已经直接给出 `verticalBarEdge`。
- **用 `statusBarManager.statusBarFrame`**。在 Duo 上它只返回 2pt 高的一条，没有用。
- **只在页面创建时决定布局**。合上状态启动 app、展开后再转竖屏，页面没有重建，会停在旧布局。
- **状态栏隐藏后重新读占用区域**，得到「顶部没有占用」，按钮又伸到状态栏的位置，状态栏再出现时重叠。
- **把页面的顶部安全区挪进页面里面后忘了裁剪**。外层容器从屏幕顶边开始时，往上收起的顶栏会画进状态栏和灵动岛（PiliPlus 在直板 iPhone 上碰到过）；给页面加只裁上边的裁剪。

展开竖屏时，已适配 app 的状态栏那一行（真机截图，2026-11）：

| App | 状态栏那一行放了什么 |
| --- | --- |
| App Store「Today」 | 大标题靠左，头像停在状态栏左边；往下滚时内容滚进这一行底下，状态栏胶囊带底色 |
| App Store「Search」 | 大标题在这一行；往下滚时搜索框升进这一行，停在状态栏左边 |
| 系统设置 | 右栏标题「General」居中；推进「关于本机」后返回键和标题都在这一行（模拟器验证） |
| Upcoming | 大标题和头像、「+」按钮；绿色背景铺到最顶 |
| KikoFlu | 筛选标签 |
| Telegram | 二维码、搜索、编辑按钮；紫色头部背景铺到最顶 |
| Tomaco | 空着，内容从下面开始（未适配） |

来源：[Designing for iPhone Duo（Apple 设计指南）](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo)、[Preparing your app for iPhone Duo（Apple 开发文档）](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo)、[Design for iPhone Duo（Apple Tech Talk）](https://developer.apple.com/videos/play/tech-talks/111466/)、UIKit 头文件 `UIViewLayoutRegion.h` 和 `UIViewReservedRegion.h`（iOS 27.2 SDK）。

## 展开后界面方向错误

合上时锁竖屏的 Flutter app，展开后界面会停在内屏的原生竖屏方向，相对机身转歪。修法是用 iOS 27.1 的铰链 API，在铰链刚离开「合上」、屏幕还没切换时解除锁定。

### 原因

- 内屏的原生方向是竖的（669 × 951）。按书本方式展开时，系统会把界面转到 `landscapeLeft`，「设置」等原生 app 都是这样。
- 屏幕切换那一刻，系统用的是**合上时缓存的方向限制**，不会重新询问 app。只允许竖屏，就停在竖屏。
- 切到内屏后方向模式变成 `alwaysAll`，app 再调 `SystemChrome.setPreferredOrientations` 会被拒绝（`UISceneErrorDomain` 101），也不会自动纠正。
- Flutter 的锁定存在两个地方，**两个都要解开**：ViewController 的 `supportedInterfaceOrientations`，以及通过 `requestGeometryUpdate` 存下的场景方向偏好（系统日志里的 `Scene updated orientation preferences: ( Pu )`）。只解开前者仍然会转错。

### 修复

1. 新建 `FlutterViewController` 子类（可以直接写在已有的 `SceneDelegate.swift` 里，不用改 Xcode 工程文件）：

```swift
class DuoFlutterViewController: FlutterViewController {
  private var isHingeOpen = false

  override func viewDidLoad() {
    super.viewDidLoad()
    if #available(iOS 27.1, *) {
      let interaction = UIHingeInteraction { [weak self] _, update in
        self?.hingeDidUpdate(update.hinge)
      }
      view.addInteraction(interaction)
    }
  }

  @available(iOS 27.1, *)
  private func hingeDidUpdate(_ hinge: UIHinge?) {
    let isOpen = hinge.map { $0.status != .closed && $0.status != .unknown } ?? false
    guard isOpen != isHingeOpen else { return }
    isHingeOpen = isOpen
    setNeedsUpdateOfSupportedInterfaceOrientations()
    // 场景还在外屏、允许程序修改时，清掉 Flutter 存下的竖屏偏好
    if isOpen, let windowScene = view.window?.windowScene {
      windowScene.requestGeometryUpdate(.iOS(interfaceOrientations: .all))
    }
  }

  override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
    isHingeOpen ? .all : super.supportedInterfaceOrientations
  }
}
```

2. 在 `ios/Runner/Base.lproj/Main.storyboard` 里把根控制器换成子类：

```xml
<viewController id="BYZ-38-t0r" customClass="DuoFlutterViewController"
  customModule="Runner" customModuleProvider="target" sceneMemberID="viewController">
```

3. 如果用了上面的系统标签栏，根控制器不再是这个子类。`supportedInterfaceOrientations` 要一层层转下来，`setNeedsUpdateOfSupportedInterfaceOrientations` 也要转给窗口根控制器，否则修复不生效。
4. Dart 侧不用改。合上后窗口变小，折叠检测会重新调 `portraitUpMode()`，外屏的竖屏锁就恢复了。

UIHinge 只在 iOS 27.1 SDK（Xcode 27、Swift 6.4）里有。用 `#if compiler(>=6.4)` 包住，旧版 Xcode 也能编译，只是没有铰链处理。

时序已在模拟器日志里确认：铰链变成半开（角度约 1.7 rad）比屏幕切换早约 0.1–0.2 秒，足够清掉锁定。

### 试过但无效的做法

| 做法 | 结果 |
| --- | --- |
| Info.plist 给 iPhone 加 `PortraitUpsideDown` | 仍然转错 |
| 重写 `SceneDelegate.supportedInterfaceOrientations(for:)`，内屏返回 `.all` | 屏幕切换时系统不询问它，仍然转错 |
| 调用 `super.supportedInterfaceOrientations(for:)` | Swift 要求写 `override`，但 `FlutterSceneDelegate` 在运行时没有实现，启动即崩溃（`doesNotRecognizeSelector`） |
| ViewController 按当前屏幕尺寸在内屏返回 `.all` | 屏幕切换完才生效，太晚 |
| 铰链打开时只改 ViewController 掩码 | 场景偏好还是竖屏，仍然转错 |
| 合上时完全不锁竖屏 | 展开正确，但外屏会跟着手机转成横屏 |

## 全屏播放与方向锁

折叠状态一变就重设方向：合上锁竖屏，展开放开所有方向。但全屏播放中要延后处理，等退出全屏时再按新状态恢复。

```dart
static bool _pendingOrientation = false;

static void _onMetricsChanged() {
  // ...只在折叠状态变化时继续
  if (isPlayerFullScreen) {        // 全屏中不动播放器设的横屏锁
    _pendingOrientation = true;
    return;
  }
  _pendingOrientation = false;
  isUnfolded ? fullMode() : portraitUpMode();
}

/// 退出全屏时调用：全屏期间展开/合上过就返回 true，并清除标记
static bool consumePendingOrientation() {
  final pending = _pendingOrientation;
  _pendingOrientation = false;
  return pending;
}
```

播放器退出全屏时，如果方向模式是「不改变当前方向」，原来会直接 return。要把 `consumePendingOrientation()` 也算进去，否则全屏中合上后，外屏会一直允许横屏。

其他要点：

- 外屏的方向模式是 `dynamic`，和普通 iPhone 一样：合上时把手机横过来自动进入横屏全屏、转回来自动退出，上述原生修复不影响这一点（已在模拟器验证）。
- 内屏上方向由系统决定，播放器的横屏锁不生效；内屏的全屏应该只是「铺满当前窗口」。如果 app 有「平板上全屏不改变方向」之类的默认设置，iPhone 上不要因为第一次在内屏启动就套用它，否则合上后全屏不会自动横过来。

## 测试清单与调试方法

每次测方向前都重启模拟器。Device Hub 的「旋转」会留下设备朝向，状态一乱就分不清是 app 的问题还是模拟器被转过。

```bash
xcrun simctl shutdown <UDID> && xcrun simctl boot <UDID>
xcrun simctl bootstatus <UDID> -b
```

**看日志**：Flutter 的 `debugPrint` 和 Swift 的 `NSLog` 都进系统日志，不进 stdout。下面的过滤能看到方向模式、场景方向、方向偏好和自己的日志：

```bash
xcrun simctl spawn <UDID> log stream --style compact \
  --predicate 'process == "Runner"' | \
  grep --line-buffered -E "flutter|interface orientation|orientation mode|orientation preferences|Failed to change"
```

正常展开时应该看到 `orientation mode: dynamic -> alwaysAll` 紧接 `Scene will change interface orientation: landscapeLeft (4)`。没有第二行，说明界面卡在了竖屏。拿「设置」做对照：把 predicate 里的 `Runner` 换成 `Preferences`。

打印安全区的临时代码：

```dart
final v = WidgetsBinding.instance.platformDispatcher.views.first;
final r = v.devicePixelRatio, p = v.viewPadding;
debugPrint('size ${v.physicalSize / r} pad L${p.left / r} T${p.top / r} R${p.right / r} B${p.bottom / r}');
```

**截图**：不加参数的 `xcrun simctl io <UDID> screenshot` 只截内屏，合上时是黑屏。用 `--display` 指定屏幕：

```bash
xcrun simctl io <UDID> screenshot --display=1 folded.png    # 外屏（合上）
xcrun simctl io <UDID> screenshot --display=3 unfolded.png  # 内屏（展开）
```

也可以用 Device Hub 自带的截图按钮。

必测场景：

- [ ] 合上：手机布局，标签栏在竖状态栏里
- [ ] 进入二级页面：标签栏隐藏，竖栏里有返回键，页面左上角没有第二个返回键
- [ ] 视频页：竖栏隐藏，内容铺满；返回后竖栏恢复
- [ ] 合上时把设备横过来：普通页面保持竖屏
- [ ] 合上 → 展开：界面正向、平板布局（至少来回两次）
- [ ] 合上 → 半开 → 全开：界面正向
- [ ] 在第二个标签页时展开或合上：页面和导航高亮保持一致
- [ ] 合上时播放视频、把设备横过来：自动横屏全屏；转回来：自动退出
- [ ] 内屏上的分栏页面：侧栏不被状态栏挤压、没有 `OVERFLOWED` 条纹
- [ ] 展开竖屏：首页和二级页面的顶栏都在状态栏那一行，按钮不压到时间和 Wi-Fi
- [ ] 展开竖屏时上滑收起顶栏、再下滑恢复：位置不变；直板 iPhone 上顶栏不画进灵动岛
- [ ] 「横屏适配」的自动 / 开 / 关各切换一次
- [ ] release 编译：`flutter build ios --release --no-codesign` 通过

## 已知限制与未验证事项

以下几点还没有确认，或者没有处理。

- **真机上的时序**：铰链事件早于屏幕切换约 0.1–0.2 秒，是在模拟器日志里测的。真机上正常展开没有问题，快速甩开时是否一样还没专门测过。
- **状态栏在顶部时**：展开竖屏时状态栏只占右上角，页面顶栏已按「展开竖屏：顶栏放进状态栏那一行」放进这一行（PiliPlus [#52](https://github.com/ShionjiART/PiliPlus/pull/52)、[#58](https://github.com/ShionjiART/PiliPlus/pull/58)、[#59](https://github.com/ShionjiART/PiliPlus/pull/59)），还没在真机上确认。
- **旧 Xcode**：用 Xcode 26 编译（跳过铰链代码）没有实测。
- **窄侧栏里的列表项**：侧栏在公式的最小宽度（PiliPlus 是 280pt）时，横排视频卡片仍会溢出 2–25 像素。这是上游在最小宽度下本来就有的问题，release 包里只是裁掉，不显示条纹。
- **原生部分依赖当前的 Flutter 实现**：修复假设 Flutter 用 `supportedInterfaceOrientations` 和 `requestGeometryUpdate` 实现方向锁（写手册时是 Flutter 3.47.8，现在是 3.47.9）。升级 Flutter 后要重新跑一遍上面的必测场景。以后 Flutter 如果自己实现了 `UIWindowSceneDelegate.supportedInterfaceOrientations(for:)`，也需要重新评估。系统标签栏手动转发出现/消失事件，也依赖 `FlutterViewController` 当前创建渲染表面的时机。
- **竖状态栏的官方 API 只接了一部分**：`reservedRegions`、`.bar(onEdge:extent:)` 和 `verticalBarEdge` 已用于展开竖屏的顶栏位置（见「用系统给的位置，不写死」）；`UIViewController.preferredVerticalBarBehavior`（可以关掉竖状态栏，适合全屏播放器）还没接，Flutter 自己画的后备导航判断竖栏也还在用「侧边安全区 ≥ 72」（系统标签栏一侧已用 `verticalBarEdge`）。
