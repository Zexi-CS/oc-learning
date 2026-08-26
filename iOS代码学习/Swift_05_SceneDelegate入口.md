# Swift_05 — 入口设置（SceneDelegate 纯代码）

> Swift 原生版最后一步：让 App 启动后显示 UserListViewController。
> 用**纯代码**设置入口（不碰 Storyboard），对应你 OC 删 storyboard 后用 SceneDelegate 那套。
> 方法：改 SceneDelegate.swift，代码指定 rootViewController + 包导航栏。

---

## 一、为什么要它

Swift_01~04 写好了 4 个类，但**没人告诉 App"启动后显示哪个页面"**。默认工程显示的是 `Main.storyboard` 里的初始 VC。我们用代码指定首页 = `UserListViewController`。

```
App 启动
  → SceneDelegate 的 scene:willConnectToSession:
  → 创建 UIWindow
  → 设置 rootViewController = UINavigationController(UserListViewController)
  → 显示
```

---

## 二、完整代码（改 SceneDelegate.swift 这一段）

如果你的工程是 iOS 13+（自动生成 SceneDelegate.swift），改 `willConnectToSession:` 方法：

```swift
import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?

    func scene(_ scene: UIScene,
               willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        // 1. 把 scene 转成 UIWindowScene（iOS 13+ 的窗口场景）
        guard let windowScene = scene as? UIWindowScene else { return }

        // 2. 创建窗口
        window = UIWindow(windowScene: windowScene)

        // 3. 首页 = UserListViewController，包进导航栏（才有 title 和右上角"+"）
        let nav = UINavigationController(rootViewController: UserListViewController())
        window?.rootViewController = nav

        // 4. 显示窗口
        window?.makeKeyAndVisible()
    }

    // 其他默认方法（sceneDidBecomeActive 等）可以留着不动
}
```

**注意**：`scene(_:willConnectTo:)` 里原来可能有 `guard` 和读取 storyboard 的代码，整段替换成上面即可。`var window` 属性保留。

---

## 三、逐段拆解

### 3.1 `var window: UIWindow?`

对应 OC 的 `@property UIWindow *window;`。App 的窗口，根视图都挂在它上面。

### 3.2 `scene(_:willConnectTo:options:)`

对应 OC 你在删 storyboard 后写的那个 `SceneDelegate` 方法（iOS 13+ 生命周期）。App 连接窗口场景时系统自动调。

### 3.3 `guard let windowScene = scene as? UIWindowScene else { return }`

- `scene` 参数是 `UIScene`（父类），实际是 `UIWindowScene`（子类）
- `as? UIWindowScene` 转成子类，转成功用 `windowScene`，失败 return
- 对应你 OC 里学的：`UIWindowScene *windowScene = (UIWindowScene *)scene;`（父类转子类强转）

### 3.4 `UIWindow(windowScene: windowScene)`

创建窗口，需要传入 windowScene。对应 OC `[[UIWindow alloc] initWithWindowScene:windowScene]`。

### 3.5 `UINavigationController(rootViewController:)`

创建导航控制器，并指定它的根控制器是 `UserListViewController`。

对应 OC：
```objc
UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:[[UserListViewController alloc] init]];
```

包了导航栏，`UserListViewController` 里才有 `title` 显示、右上角 `UIBarButtonItem` 才生效。

### 3.6 `window?.rootViewController = nav`

把导航控制器设成窗口的根控制器。对应 OC `self.window.rootViewController = nav`。

### 3.7 `window?.makeKeyAndVisible()`

让窗口成为关键窗口并显示。对应 OC `[self.window makeKeyAndVisible]`。

---

## 四、可选：彻底不用 Storyboard（双保险）

如果你想让工程**完全不依赖** Main.storyboard（彻底代码），要再做两步：

**① Info.plist 里删掉 storyboard 入口**
打开 Info.plist，找到 `UIApplicationSceneManifest` 那项，如果你是纯代码场景可以保留（SceneDelegate 依然走）。**通常不用删**，只要 SceneDelegate 代码设了 rootViewController，storyboard 就会被忽略。

**② 确保 AppDelegate 不再加载 storyboard**
新模板 AppDelegate 没写 storyboard 加载（都在 SceneDelegate），所以不用动 AppDelegate。

> **多数情况**：只改 SceneDelegate.swift 一段就行，storyboard 留在工程里不动也不会冲突（我们的代码设了 rootViewController 就覆盖了它）。

---

## 五、完整文件清单（Swift 原生版全齐 + 入口）

```
User.swift                     ← Swift_01
NetworkManager.swift           ← Swift_02
UserCell.swift                 ← Swift_03
UserListViewController.swift   ← Swift_04
SceneDelegate.swift            ← Swift_05（这步）
```

**五个文件 + 工程，运行后应该能看到用户列表。**

---

## 六、如果跑起来有问题

| 现象 | 原因 → 处理 |
|------|-----------|
| 编译报 `UserListViewController` 未定义 | 文件不在同一 target / 类名拼写 |
| 黑屏 | `window` 没设 / windowScene 强转失败 |
| 有页面但没标题、没"+" | 没包 UINavigationController / 没用 nav 当 root |
| 列表空 | 服务器地址不通 / ATS 拦 http（真机） |

---

**到这里 Swift 原生版整套（含入口）全部完成。** 跑通后我们再做 `Swift_06_Alamofire对照版`。
