# iOS 基础语法问题笔记

> 记录日常学习中遇到的基础概念和语法问题，持续更新。



---

## 一、数组排序

### 1.1 `sortUsingSelector:` 和 `sortUsingComparator:`

两者都是 `NSMutableArray` 的系统方法，本质：**你只定义"怎么比大小"，系统负责"反复调用来排好序"。**

| 方法                                      | 比较规则在哪              | 适用场景                           |
| --------------------------------------- | ------------------- | ------------------------------ |
| `sortUsingSelector:@selector(compare:)` | 元素自带的 `compare:` 方法 | NSString、NSNumber、NSDate 等系统类型 |
| `sortUsingComparator:(Block)`           | 你现场写的 Block 里       | 自定义对象、多字段排序、复杂逻辑               |

**核心理解：代码只写一行，没有 for/while。系统内部会自动多轮扫描，反复调用你的比较规则，直到全部排好。**

### 1.2 排序算法执行过程

以冒泡排序为例：不是相邻比一遍就完事，而是**反复多轮扫描**，每轮把最大的数"浮"到最后。比如 `[1,3,5,1,5,2,7,4]` 需要 4 轮共 22 次比较。

### 1.3 Block 里也能调用 `compare:`

```objc
// 按 name（NSString）排序 → name 自带 compare:
return [a.name compare:b.name];

// 按 age（NSInteger）排序 → 基本类型不能接收消息，直接 if/else 比较
if (a.age < b.age) return -1;
if (a.age > b.age) return  1;
return 0;
```

**Block 只是一个"容器"，里面可以写 if/else，也可以调用 `compare:`，只要最终返回 `NSComparisonResult`。**

### 1.4 返回值约定

| 返回值                       | 含义                  |
| ------------------------- | ------------------- |
| `NSOrderedAscending` (-1) | obj1 排在 obj2 **前面** |
| `NSOrderedSame` (0)       | 相等，顺序不变             |
| `NSOrderedDescending` (1) | obj1 排在 obj2 **后面** |

---

## 二、类型转换

### 2.1 `integerValue` / `intValue` / `floatValue` / `doubleValue`

**作用：把对象（NSString / NSNumber）转成基本类型，用于数学计算。**

```objc
NSString *str = @"25";
NSInteger age = [str integerValue];  // "25" → 25，可以算了

str = @"3.5";
[str integerValue];  // 3    ← 只取整数部分！
[str floatValue];    // 3.5  ← 取小数用这个
[str doubleValue];   // 3.5  ← 精度更高
```

**关键：对象不能直接加减乘除，必须转成基本类型。**

---

## 三、字典操作

### 3.1 `allValues` / `allKeys`

```objc
NSDictionary *dict = @{@"name": @"张三", @"age": @25};
NSArray *values = [dict allValues];  // @[@"张三", @25]  无序！
NSArray *keys   = [dict allKeys];    // @[@"name", @"age"] 无序！
```

### 3.2 字典用 `%@` 打印

```objc
NSDictionary *a = @{@"x": @90};
NSLog(@"%@", a);
// 输出：{x = 90;}  或  {x = 90;}
```

---

## 四、分类（Category）

### 4.1 命名规则

```objc
// NSString+fz.h
@interface NSString (fz)    // ← 括号里的 fz 是分类名，随意取
- (BOOL)containsDigit;
@end

// NSString+fz.m
@implementation NSString (fz)
- (BOOL)containsDigit { ... }
@end
```

文件名惯例：`原类名+分类名.h/.m`

---

## 五、NSRange 和 `rangeOfCharacterFromSet:`

### 5.1 NSRange

```objc
typedef struct {
    NSUInteger location;  // 起始位置（从0开始）
    NSUInteger length;    // 长度
} NSRange;

NSRange r = NSMakeRange(3, 2);  // {3, 2} → 从第3位开始，取2个
```

### 5.2 `rangeOfCharacterFromSet:`

从字符串头部扫描，找**第一个**匹配字符集中任意字符的位置。

```objc
NSString *str = @"abc123xy";
NSRange range = [str rangeOfCharacterFromSet:[NSCharacterSet decimalDigitCharacterSet]];
// range = {3, 1} → '1' 在第 3 位

// 没找到时：range.location == NSNotFound
if (range.location != NSNotFound) {
    // 找到了
}
```

---

## 六、通知（NSNotificationCenter）

### 6.1 发送通知

```objc
// 不带数据
[[NSNotificationCenter defaultCenter] postNotificationName:@"NotifyName" object:nil];

// 带数据
[[NSNotificationCenter defaultCenter] postNotificationName:@"NotifyName"
                                                    object:nil
                                                  userInfo:@{@"key": @"value"}];
```

### 6.2 接收通知

```objc
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(handleNotification:)
                                             name:@"NotifyName"
                                           object:nil];

- (void)handleNotification:(NSNotification *)notification {
    NSString *value = notification.userInfo[@"key"];  // 取附带数据
}
```

### 6.3 命名规范

```objc
// .h 文件
extern NSString * const kMyNotification;

// .m 文件
NSString * const kMyNotification = @"MyNotification";
```

发和收都用这个常量，避免字符串拼错。

### 6.4 解除监听（必须做）

```objc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```

---

## 七、App 生命周期

> 生命周期 = 系统在特定时间点自动调用的方法。你不需要手动调用它们，只需要重写（override），在合适的时机做合适的事。

### 7.1 AppDelegate（应用级别）

AppDelegate 遵守 `UIApplicationDelegate` 协议。系统在应用的各个阶段自动调用这些方法。

```objc
// ============================================================
// AppDelegate.m — 完整的生命周期方法
// ============================================================

#import "AppDelegate.h"

@implementation AppDelegate

// -------------------- 启动阶段 --------------------

// 【调几次】应用冷启动后只调一次
// 【做什么】全局初始化：设置 window、配置第三方 SDK、注册推送
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    // iOS 12 及以前：在这里创建 window
    // self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    // self.window.rootViewController = [[ViewController alloc] init];
    // [self.window makeKeyAndVisible];

    NSLog(@"① didFinishLaunching → 应用启动完成");
    return YES;
}

// -------------------- 前后台切换 --------------------

// 【调几次】每次从后台回来 / 刚启动完成时各调一次
// 【做什么】恢复动画、继续播放视频、重新开始定位
- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Became = 变成了 Active 状态
    // Active 状态 = 用户能看到界面 + 可以触摸操作
    NSLog(@"② didBecomeActive → 应用可以交互了");
}

// 【调几次】每次即将失去 Active 状态时调
// 【做什么】暂停游戏、暂停动画、停止定位（省电）
- (void)applicationWillResignActive:(UIApplication *)application {
    // Resign = 放弃 Active 状态
    // 触发场景：来电话、锁屏、拉出通知中心、双击 Home
    // 注意：此时还能看到界面，但无法交互
    NSLog(@"③ willResignActive → 即将失去交互能力");
}

// 【调几次】从 WillResignActive 之后，应用完全不可见时调
// 【做什么】保存关键数据、关闭数据库连接、释放共享资源
- (void)applicationDidEnterBackground:(UIApplication *)application {
    // 用户按了 Home 键 / 上滑回桌面
    // ★ 这里是保存数据的最佳时机！
    // 因为系统可能随时杀死后台应用
    NSLog(@"④ didEnterBackground → 已进入后台");
}

// 【调几次】从后台切回来，界面还没显示时调
// 【做什么】刷新数据、恢复 UI 状态
- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSLog(@"⑤ willEnterForeground → 即将回到前台");
}

// -------------------- 终止 --------------------

// 【调几次】极少触发
// 【注意】用户从多任务界面划掉 App 不会触发这个！
// 【注意】系统在低内存时直接杀进程也不会触发！
// 只有在应用主动退出（极少见）或特定调试场景才会调
- (void)applicationWillTerminate:(UIApplication *)application {
    NSLog(@"⑥ willTerminate → 应用即将被终止");
}

@end
```

**完整的前后台切换链路：**

```
前台 → 来电话 → willResignActive → didEnterBackground → （在后台等待）
                                                                      ↓
                                                            （用户切回来）
                                                                      ↓
                                                          willEnterForeground → didBecomeActive → 回到前台
```

**为什么会有 Active 和 Inactive 之分？**

`Active` = 可以触摸交互，`Inactive` = 能看到但摸不了。比如来了电话，你还能看到 App 的界面，但不能点按钮了——这就是 Inactive 状态。

---

### 7.2 SceneDelegate（iOS 13+，单窗口级别）

iOS 13 开始，一个 App 可以有多个"窗口"（iPad 分屏时左右各是一个 Scene，每个 Scene 有独立的生命周期）。SceneDelegate 遵守 `UIWindowSceneDelegate` 协议。

**为什么需要 SceneDelegate？**

```
iOS 12 及以前：                     iOS 13 及以后：
┌─ App ──────────┐                 ┌─ App ──────────────────┐
│  一个 Window    │                 │  ┌ Scene A ┐ ┌ Scene B ┐│
│  一套前后台回调  │                 │  │ Window  │ │ Window  ││
└────────────────┘                 │  │ 前后台   │ │ 前后台   ││
                                   │  └─────────┘ └─────────┘│
                                   └─────────────────────────┘

一个 App 一个窗口                   一个 App 多个窗口（iPad 分屏）
AppDelegate 管全局                  AppDelegate 管全局 + 各 Scene 管自己
```

```objc
// ============================================================
// SceneDelegate.m — 完整代码
// ============================================================

#import "SceneDelegate.h"
#import "ViewController.h"

@implementation SceneDelegate

// 【调几次】Scene 刚创建、还没有连接时调
// 【做什么】创建 window，设置 rootViewController
// 【关键】window 从此不再由 AppDelegate 创建！
- (void)scene:(UIScene *)scene
    willConnectToSession:(UISceneSession *)session
                 options:(UISceneConnectionOptions *)connectionOptions {

    // scene 是一个抽象概念，需要强转成 UIWindowScene
    UIWindowScene *windowScene = (UIWindowScene *)scene;

    // 用 windowScene 初始化 window（参数是 init 出来的 frame 自动跟 Scene 匹配）
    self.window = [[UIWindow alloc] initWithWindowScene:windowScene];

    // 设置根控制器
    self.window.rootViewController = [[ViewController alloc] init];

    // 让 window 显示出来
    [self.window makeKeyAndVisible];

    NSLog(@"Scene ① willConnectToSession → 创建 window");
}

// 【调几次】Scene 断开连接时调（iPad 拖走一个分屏窗口）
// 【注意】系统可能随时回收，需要释放这个 Scene 的资源
- (void)sceneDidDisconnect:(UIScene *)scene {
    NSLog(@"Scene ⑥ didDisconnect → Scene 被回收");
}

// -------------------- 单 Scene 的前后台回调 --------------------

// 【跟 AppDelegate 的一样，但是作用于单个 Scene】
- (void)sceneDidBecomeActive:(UIScene *)scene {
    NSLog(@"Scene ② didBecomeActive → 这个窗口可以交互了");
}

- (void)sceneWillResignActive:(UIScene *)scene {
    NSLog(@"Scene ③ willResignActive → 这个窗口即将失去交互");
}

- (void)sceneDidEnterBackground:(UIScene *)scene {
    NSLog(@"Scene ④ didEnterBackground → 这个窗口进入后台");
}

- (void)sceneWillEnterForeground:(UIScene *)scene {
    NSLog(@"Scene ⑤ willEnterForeground → 这个窗口即将回前台");
}

@end
```

**Xcode 11+ 新建项目自动带有 SceneDelegate。老项目迁移到 iOS 13+ 只有 AppDelegate，window 和前后台回调都还在 AppDelegate 里。**

|           | AppDelegate       | SceneDelegate |
| --------- | ----------------- | ------------- |
| 存在版本      | iOS 所有版本          | iOS 13+       |
| 管什么       | 整个应用（全局配置、数据库、推送） | 单个窗口（UI、前后台）  |
| window 在哪 | iOS 12 以前在这       | iOS 13 以后在这   |
| 前后台回调     | 有（老项目）            | 有（新项目优先用这个）   |

---

### 7.3 UIViewController 生命周期

ViewController 是日常开发中用得最多的生命周期，理解透每个方法的调用次数和时机很关键。

```objc
// ============================================================
// MyViewController.m — 完整生命周期方法示例
// ============================================================

#import "MyViewController.h"

@implementation MyViewController

// ============================================================
// 阶段一：初始化（二选一，取决于创建方式）
// ============================================================

// 【调几次】一次。纯代码 [[MyVC alloc] init] 创建 VC 时走这里
// 【做什么】初始化成员变量、设置默认值
// 【不能做什么】此时 self.view 还没创建，不能设置 UI
- (instancetype)init {
    self = [super init];
    if (self) {
        // 初始化非 UI 相关的属性
        self.title = @"我的页面";
        NSLog(@"VC ① init");
    }
    return self;
}

// 【调几次】一次。从 xib/storyboard 加载 VC 时走这里
// 【做什么】xib 反序列化完成后的额外初始化
// 【不能做什么】IBOutlet 已连接，但 self.view 仍可能没完全加载好
- (void)awakeFromNib {
    [super awakeFromNib];
    NSLog(@"VC ①' awakeFromNib");
}

// ============================================================
// 阶段二：视图创建
// ============================================================

// 【调几次】一次。系统在需要用到 view 时才调
// 【做什么】创建 self.view（一般不需要重写）
// 【什么时候重写】完全不用 xib/storyboard，纯手写 view 层级时才重写
// 【重写时绝对不能调 [super loadView]！】
- (void)loadView {
    // 默认实现：从 xib/storyboard 加载，找不到就创建一个空的 UIView
    // 初学者几乎不需要重写这个方法
    [super loadView];
    NSLog(@"VC ② loadView → 创建了 self.view");
}

// 【调几次】★★★ 只调一次！全身量周期最重要！
// 【做什么】view 已加载到内存。添加子控件、设置约束、请求初始数据
// 【最佳实践】viewDidLoad 里只做「初始化」，不做「刷新」
- (void)viewDidLoad {
    [super viewDidLoad];

    // 设置背景色
    self.view.backgroundColor = [UIColor whiteColor];

    // 创建子控件
    UILabel *label = [[UILabel alloc] init];
    label.text = @"Hello World";
    [self.view addSubview:label];

    // 设置约束（Masonry）
    [label mas_makeConstraints:^(MASConstraintMaker *make) {
        make.center.equalTo(self.view);
    }];

    // 请求初始数据
    // [self fetchData];

    NSLog(@"VC ③ viewDidLoad → view 加载完毕");
}

// ============================================================
// 阶段三：即将显示（每次显示都调）
// ============================================================

// 【调几次】★★ 每次 view 即将出现都调
// 【做什么】刷新数据、显示/隐藏导航栏、开始动画
// 【经典场景】从详情页返回列表页，刷新列表数据
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];

    // animated 参数：YES=带动画出现（push 进来），NO=不带动画（tab 切换）
    // [self.tableView reloadData];  // 刷新列表

    NSLog(@"VC ④ viewWillAppear: → view 即将显示");
}

// ============================================================
// 阶段四：已经显示
// ============================================================

// 【调几次】每次 view 完全出现后调
// 【做什么】开启动画、开始定时器、开始视频播放
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];

    // [self startTimer];  // 开始定时器
    // [self startAnimation];  // 开启动画

    NSLog(@"VC ⑤ viewDidAppear: → view 已经显示");
}

// ============================================================
// 阶段五：即将消失（每次离开都调）
// ============================================================

// 【调几次】每次 view 即将消失时调
// 【做什么】隐藏键盘、保存编辑中的内容
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];

    // [self.view endEditing:YES];  // 收起键盘

    NSLog(@"VC ⑥ viewWillDisappear: → view 即将消失");
}

// ============================================================
// 阶段六：已经消失
// ============================================================

// 【调几次】每次 view 完全消失后调
// 【做什么】停止定时器、取消网络请求、释放临时资源
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];

    // [self stopTimer];  // 停止定时器
    // [self cancelRequest];  // 取消请求

    NSLog(@"VC ⑦ viewDidDisappear: → view 已经消失");
}

// ============================================================
// 阶段七：销毁
// ============================================================

// 【调几次】一次。VC 引用计数归零时调
// 【做什么】移除通知监听、释放 KVO、清理代理
- (void)dealloc {
    // ★ 移除通知监听不要忘
    [[NSNotificationCenter defaultCenter] removeObserver:self];

    NSLog(@"VC ⑧ dealloc → VC 被释放销毁");
}

@end
```

**完整调用时间线：**

```
创建：init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
                                                                   │
                                            push 到下一个页面 / present
                                                                   │
离开：viewWillDisappear → viewDidDisappear                         │
                                                                   │
                                               pop 回来 / dismiss
                                                                   │
回来：viewWillAppear → viewDidAppear                                │
                                                                   │
                                               pop 到根 / 强制释放
                                                                   │
销毁：viewWillDisappear → viewDidDisappear → dealloc
```

**关键理解：**

| 方法                   | 调几次 | 经典用途           | 此时 view 存在吗 |
| -------------------- | --- | -------------- | ----------- |
| `init`               | 1 次 | 初始化属性、标题       | ❌ 没有        |
| `loadView`           | 1 次 | 创建 view（一般不重写） | ❌ 正在创建      |
| `viewDidLoad`        | 1 次 | 添加子控件、设约束、请求数据 | ✅ 已加载       |
| `viewWillAppear:`    | N 次 | 刷新列表、显示导航栏     | ✅           |
| `viewDidAppear:`     | N 次 | 开始定时器、开启动画     | ✅           |
| `viewWillDisappear:` | N 次 | 隐藏键盘、保存草稿      | ✅           |
| `viewDidDisappear:`  | N 次 | 停止定时器、取消请求     | ✅（但仍存在）     |
| `dealloc`            | 1 次 | 移除通知、释放资源      | ✅（即将释放）     |

**易错点：**

- `viewDidLoad` 只调一次，`viewWillAppear` 每次显示都调。刷新列表的逻辑放 `viewWillAppear`。
- `viewDidLoad` 里 `self.view.frame` 可能还不是最终尺寸，布局代码要放约束里而不是直接设 frame。
- `dealloc` 不会自动调，说明 VC 没有被释放（可能有循环引用等问题）。

---

### 7.4 UIView 生命周期

UIView 的生命周期比较简单，核心只有三个方法。

```objc
// ============================================================
// MyCustomView.m — 自定义 UIView 生命周期
// ============================================================

#import "MyCustomView.h"

@implementation MyCustomView

// ============================================================
// 阶段一：初始化
// ============================================================

// 【调几次】一次。代码 [[MyView alloc] initWithFrame:...] 创建时调
// 【frame 是什么】一个 CGRect，{x, y, width, height}，决定 view 的初始位置和大小
- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        // 在这里初始化子控件
        self.backgroundColor = [UIColor lightGrayColor];

        UILabel *subLabel = [[UILabel alloc] init];
        subLabel.text = @"子控件";
        [self addSubview:subLabel];

        NSLog(@"View ① initWithFrame: → {{%.0f, %.0f}, {%.0f, %.0f}}",
              frame.origin.x, frame.origin.y, frame.size.width, frame.size.height);
    }
    return self;
}

// 【调几次】一次。从 xib/storyboard 加载时走这个方法（代替 initWithFrame:）
- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super initWithCoder:coder];
    if (self) {
        // xib 反序列化成功后做额外初始化
        NSLog(@"View ①' initWithCoder: → 从 xib 加载");
    }
    return self;
}

// ============================================================
// 阶段二：布局（会被多次调用）
// ============================================================

// 【调几次】★★ 多次！每次 frame 变化都调！
// 【触发时机】首次显示、屏幕旋转、调用 setNeedsLayout、约束变化
// 【做什么】手动布局子控件的位置和大小
// 【注意】如果用了 AutoLayout/Masonry，通常不需要重写这个方法
- (void)layoutSubviews {
    [super layoutSubviews];  // 必须调 super！

    // 示例：手动布局，把子控件居中
    // CGFloat centerX = self.bounds.size.width / 2;
    // CGFloat centerY = self.bounds.size.height / 2;
    // self.subLabel.center = CGPointMake(centerX, centerY);

    NSLog(@"View ② layoutSubviews → frame 或约束更新了");
}

// ============================================================
// 阶段三：绘制（很少需要重写）
// ============================================================

// 【调几次】每次需要重绘时调（首次显示、调用 setNeedsDisplay）
// 【做什么】用 Core Graphics 画自定义图形
// 【注意】大多数场景不需要重写，用子控件组合即可
- (void)drawRect:(CGRect)rect {
    [super drawRect:rect];

    // 示例：画一条红线
    // CGContextRef ctx = UIGraphicsGetCurrentContext();
    // CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
    // CGContextMoveToPoint(ctx, 0, 0);
    // CGContextAddLineToPoint(ctx, rect.size.width, rect.size.height);
    // CGContextStrokePath(ctx);

    // rect 参数：需要重绘的具体区域（不一定是整个 view）

    NSLog(@"View ③ drawRect: → 重绘区域 {{%.0f,%.0f},{%.0f,%.0f}}",
          rect.origin.x, rect.origin.y, rect.size.width, rect.size.height);
}

@end
```

**UIView 生命周期总结：**

| 方法               | 调几次 | 经典用途         | 使用频率 |
| ---------------- | --- | ------------ | ---- |
| `initWithFrame:` | 1 次 | 初始化子控件、设置背景色 | 每次都写 |
| `layoutSubviews` | 多次  | 手动布局（不用约束时）  | 偶尔   |
| `drawRect:`      | 按需  | 自定义绘制（画线画图形） | 极少   |

---

### 7.5 完整联动时间线：从冷启动到用户看到第一个页面

```
1. AppDelegate: application:didFinishLaunchingWithOptions:
   → 全局初始化（SDK、数据库、推送）
                     ↓
2. SceneDelegate: scene:willConnectToSession:options:
   → 创建 window，设置 rootViewController
                     ↓
3. ViewController: init
   → VC 对象创建，属性初始化
                     ↓
4. ViewController: loadView
   → 创建 self.view
                     ↓
5. ViewController: viewDidLoad
   → 添加子控件、设约束、请求数据 ★
                     ↓
6. View: initWithFrame:
   → 每个子 view 初始化
                     ↓
7. ViewController: viewWillAppear:
   → 刷新 UI
                     ↓
8. View: layoutSubviews （每个子 view）
   → 计算并设置最终位置
                     ↓
9. ViewController: viewDidAppear:
   → 开启动画/定时器
                     ↓
10. AppDelegate: applicationDidBecomeActive:  / SceneDelegate: sceneDidBecomeActive:
    → 应用可以交互了
                     ↓
                 用户看到页面
```

> **记忆口诀：App 先启动 → Scene 造窗口 → VC建视图加载控件 → View 布局位置 → 用户看到。**

---

## 八、视图层级

```
Window（整个屏幕）
  └── rootViewController.view
        ├── UILabel / UIButton / UIImageView  （直接放在 view 上）
        ├── UITableView（可滚动列表）
        │     └── UITableViewCell（一行）
        │           └── contentView（系统自带的内容区域 ★ 子控件放这！）
        │                 ├── UILabel
        │                 └── UIImageView
        └── UICollectionView
              └── UICollectionViewCell
                    └── contentView
```

### 8.1 为什么子控件必须加在 `contentView` 上

Cell 内部有分层：

- `cell` 本身不动
- `cell.contentView` 会在编辑模式（删除按钮出现）时自动向右缩进
- 如果子控件加在 `cell` 上（和 contentView 平级），就不会跟着挪

```objc
// ✅ 正确
[cell.contentView addSubview:myLabel];

// ❌ 错误
[cell addSubview:myLabel];
```

---

## 九、AutoLayout 与 Storyboard

**两者不是一回事：**

|     | AutoLayout | Storyboard |
| --- | ---------- | ---------- |
| 本质  | 布局计算规则     | 可视化编辑工具    |
| 管什么 | 控件放哪、多大    | 控件长什么样     |

两者可以组合，不冲突。

### 9.1 纯代码 vs 控件+代码

- **纯代码**：不用 Storyboard，全部 `alloc init` + 代码设约束
- **控件+代码**：Storyboard 拖控件，用 `IBOutlet` 连到代码，代码只管逻辑

### 9.2 一行代码不写能做完 App 吗？

静态展示页面（几个 Label、ImageView）**可以**，但真实 App **不行**——页面跳转、网络请求、列表加载、按钮逻辑都必须写代码。

### 9.3 Storyboard 连线（IBOutlet / IBAction）

**怎么拖：** 打开辅助编辑器（`⌘⌥↩`）让代码和界面并排 → 按住 **Ctrl** 从控件拖到代码 → 松手弹窗。

| 类型             | 作用                | 拖到哪                      |
| -------------- | ----------------- | ------------------------ |
| **Outlet**（属性） | 让代码"控制"控件（改文字/颜色） | `.h` 的 `@interface`      |
| **Action**（方法） | 让控件"触发"代码（点击响应）   | `.m` 的 `@implementation` |

**判断标准：** 想"操作它" → Outlet；想"被它通知" → Action。一个控件可以同时连两种（比如按钮既要改文字又要响应点击）。

**弹窗字段：** Connection（Outlet/Action）、Object（挂到哪个对象，默认不动）、Name（起名）、Type（类型，自动识别）、Event（事件，按钮选 Touch Up Inside）、Arguments（方法带不带参数，建议 Sender）。

### 9.4 Storyboard 右下角四个图标

| 图标                           | 作用                            |
| ---------------------------- | ----------------------------- |
| ① Align                      | 对齐：水平/垂直居中、左右对齐（管"居不居中"）      |
| ② Add New Constraints（Pin）   | 间距+尺寸：四根线是间距，Width/Height 是尺寸 |
| ③ Resolve Auto Layout Issues | 修复约束冲突（红）/缺失（黄）               |
| ④ Embed In                   | 把控件包进 Stack View 等容器          |

**关键：** ① Align 管"对齐"，② Pin 管"间距+尺寸"，两者配合才能定死一个控件。

- Align 选项后面的数字 = **偏移量**（对齐后再额外挪多少，一般填 0）
- Pin 四根线的实线/虚线 = **开关**（实线=加这个约束，虚线=不加，点一下切换）

**红 vs 黄：** 🟡 黄 = 缺约束（位置没定死）；🔴 红 = 约束冲突（两个约束打架，比如拖控件时 Xcode 悄悄加了"距边缘"约束，你又加"底部对齐"就冲突了）。

---

## 十、Scene / Window / WindowScene 与启动机制

### 10.1 Scene / Window / WindowScene 三者关系

**层层包含：**

```
UIApplication（整个应用）
  └── UIScene（场景，抽象概念，管生命周期）
        └── UIWindowScene（UIScene 的具体子类，带窗口的场景）
              └── UIWindow（窗口，真正装视图的容器）
                    └── rootViewController.view
```

|                 | 是什么           | 你 new 吗   |
| --------------- | ------------- | --------- |
| `UIScene`       | 抽象基类（概念）      | ❌ 抽象不能实例化 |
| `UIWindowScene` | `UIScene` 的子类 | ⚠️ 系统创建   |
| `UIWindow`      | 视图容器          | ✅ 你手动创建   |

**【易混淆】** 用户问"Scene、Window、WindowScene 谁是谁"——核心是**包含关系**：Scene 是抽象概念，WindowScene 是"带窗口的具体场景"，Window 是"装视图的箱子"。

### 10.2 Storyboard 与纯代码的启动机制

**关键在 Info.plist 里有没有配置"Main storyboard"。**

|            | Storyboard 项目             | 纯代码项目                          |
| ---------- | ------------------------- | ------------------------------ |
| Info.plist | ✅ 有 Main storyboard       | ❌ 没有                           |
| 启动         | 系统自动加载 Storyboard + 设根控制器 | 调 `willConnectToSession:` 你手动写 |

**纯代码为什么要在 SceneDelegate 手动设根控制器？** 因为 Info.plist 没有配置，系统不知道显示啥，就调 `willConnectToSession:` 这个"钩子"让你手动创建 window + 设根控制器。

**【难点】** 用户问"如果没删 Main storyboard 又写了代码，为什么能正常显示不冲突？"——因为 `willConnectToSession:` **总会调用**，且系统先自动加载 Storyboard（先设置），你的代码后执行（后设置），**后者覆盖前者**。所以显示的是你代码里的 VC，但系统其实"白加载"了一次 Storyboard。

**【易混淆】** "不写 SceneDelegate，直接改 Storyboard 把 ViewController 换成自定义类行不行？"——**可以**，但有两个前提：① 页面如果要 push 跳转，Storyboard 里**必须包 Navigation Controller**（否则 `self.navigationController` 是 nil，push 失效）；② SceneDelegate 里**删掉** window/根控制器代码，且确认 Info.plist 的 "Main storyboard file base name" = Main。用 Storyboard 启动，结构 = Navigation Controller（初始控制器）包着列表页。

### 10.3 Custom Class（Storyboard 和代码的桥梁）

Storyboard 里的每个场景都有一个 **Custom Class** 设置，意思是"这个界面由哪个代码类管理"。

- Xcode 新建项目时，默认场景的 Custom Class = `ViewController`
- 所以你在 `ViewController.m` 写的代码才生效
- 自己新建的类，Storyboard 没指向它、代码也没创建它，就永远不跑

### 10.4 页面跳转（push / present）

**两种方式：**

|          | push/pop                      | present/dismiss                  |
| -------- | ----------------------------- | -------------------------------- |
| 需要导航控制器吗 | ✅ 必须先包 UINavigationController | ❌ 不需要                            |
| 视觉       | 从右滑入，自带返回按钮                   | 从下弹出                             |
| 返回       | `popViewControllerAnimated:`  | `dismissViewControllerAnimated:` |
| 场景       | 详情页、下级页面                      | 登录弹窗、设置页                         |

```objc
// push
DetailVC *detail = [[DetailVC alloc] init];
[self.navigationController pushViewController:detail animated:YES];
// 返回
[self.navigationController popViewControllerAnimated:YES];

// present
LoginVC *login = [[LoginVC alloc] init];
[self presentViewController:login animated:YES completion:nil];
// 收起
[self dismissViewControllerAnimated:YES completion:nil];
```

**传数据：** 跳转前给目标 VC 的属性赋值即可（`detail.name = @"张三"`）。

### 10.5 导航控制器与标签栏控制器

|     | UINavigationController  | UITabBarController |
| --- | ----------------------- | ------------------ |
| 管什么 | 顶部 navigationBar + 中间内容 | 底部 tabBar + 中间内容   |
| 职责  | 页面"纵向"深入（push/pop）      | 栏目"横向"切换（tab）      |

**按需设根：** 单页面 → 直接设 VC；要跳转 → 设导航控制器；要分栏目 → 设标签栏控制器。

**真实 App 结构（两者叠用）：**

```
UITabBarController（根）
  ├── tab1: UINavigationController → HomeVC
  ├── tab2: UINavigationController → MessageVC
  └── tab3: UINavigationController → ProfileVC
```

### 10.6 `initWithRootViewController:`

作用：**创建导航控制器的同时，指定"第一个页面"（栈底、首页）**。

```objc
HomeVC *home = [[HomeVC alloc] init];
UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:home];
```

导航栈必须有"第一本书"，这个方法就是创建导航控制器时塞进去的首页。之后 push 的页面都叠在它上面，pop 到最深处就是它。

**【易混淆】** 两个 root 别混：`initWithRootViewController:` 是"导航栈的栈底页面"，`window.rootViewController` 是"App 最外层控制器"。

---

## 十一、焦点（Active / Inactive / Background）

App 有三个状态，不是只有"前台/后台"两种：

| 状态                 | 能看到吗 | 能交互吗 |
| ------------------ | ---- | ---- |
| **Active**（激活）     | ✅    | ✅    |
| **Inactive**（失焦）   | ✅    | ❌    |
| **Background**（后台） | ❌    | ❌    |

**"焦点" = 谁能接收用户的触摸输入。** 失焦 = 看得到但点不了。

**失焦但没进后台的场景：** 来电话、下拉通知中心/控制中心、系统弹窗、进多任务。这些情况 App 还在前台显示，但交互被系统接管。

**前后台与焦点的时序（不是同步）：**

```
进后台：willResignActive（先失焦）→ didEnterBackground（再进后台）
进前台：willEnterForeground（先回前台）→ didBecomeActive（再得焦）
```

**【难点】** 用户问"进前台就同步得焦，进后台就同步失焦吗？"——不是同步，是一前一后：失焦先于进后台，得焦后于进前台。

---

## 十二、AutoLayout 进阶

### 12.1 安全区（Safe Area）

**安全区 = 屏幕去掉刘海、状态栏、Home 条后，剩下的"可放心放内容的区域"。**

**核心三点：**

1. **没有固定大小** —— 随设备和横竖屏动态变（刘海屏顶部高，横屏刘海在左右两侧）
2. **内部都是安全的** —— 不用"离安全区远一点"，安全区内随便放
3. **`offset(30)` 只是美观间距**，不是"安全距离"

```objc
// 四个安全区
mas_safeAreaLayoutGuideTop      // 顶部（状态栏/刘海）
mas_safeAreaLayoutGuideBottom   // 底部（Home 条）
mas_safeAreaLayoutGuideLeft     // 左侧（横屏刘海）
mas_safeAreaLayoutGuideRight    // 右侧（横屏刘海）
```

**口诀：靠屏幕哪边，就用哪边的安全区；居中就不用管。** 竖屏注意顶/底，横屏注意左/右。

**为什么不能写死数字？** 写死 88 只对某个机型对，换机型就错。用 `safeAreaLayoutGuide` 系统自动算。

**【易混淆】** 用户问"离安全区底部更远是不是更安全"——这是把"安全区"理解成"危险区"了。安全区内部本来就安全，`offset` 越大只是离刘海越远（美观），不是"越远越安全"。

### 12.2 UIStackView

**作用：自动排列子视图，解决"摆放"问题。** 本身不能滚动（滚动是 UIScrollView 的事）。

```objc
UIStackView *stack = [[UIStackView alloc] init];
stack.axis = UILayoutConstraintAxisHorizontal;        // 横排（默认 Vertical 竖排）
stack.spacing = 15;                                   // 间距
stack.distribution = UIStackViewDistributionFillEqually;  // 等宽

[stack addArrangedSubview:block1];   // ← 用 addArrangedSubview，不是 addSubview
[stack addArrangedSubview:block2];
```

| 属性             | 作用                              |
| -------------- | ------------------------------- |
| `axis`         | 排列方向（Horizontal 横 / Vertical 竖） |
| `spacing`      | 子视图间距                           |
| `distribution` | 尺寸分配（`FillEqually` = 等宽均分）      |
| `alignment`    | 对齐方式                            |

**为什么不用设宽？** `distribution = FillEqually` 会自动把 Stack 的宽度平均分给每个子视图。你只定 Stack 宽度，子视图宽度自动平分。

**【难点】** `addArrangedSubview` vs `addSubview`：前者"加进来 + 交给 Stack 排列"，后者"只加进来，位置自己管"。用 Stack 必须用前者。

**什么时候用 Stack？** 一组"排整齐"的控件（等距等宽）用 StackView 省代码；控件位置各异、没有规律用手动约束。

### 12.3 equalTo vs mas_equalTo

|              | `equalTo`     | `mas_equalTo` |
| ------------ | ------------- | ------------- |
| 本质           | 方法            | 宏（自动包装基本类型）   |
| 传对象（view、属性） | ✅             | ✅             |
| 传数字（100）     | ❌ 要写 `@(100)` | ✅ 直接写 `100`   |

**口诀：数字用 mas_equalTo，对象用 equalTo。**

```objc
make.top.equalTo(self.view);       // 对象 → equalTo
make.width.mas_equalTo(150);       // 数字 → mas_equalTo
```

`mas_equalTo(100)` 内部就是 `equalTo(@(100))`——宏帮你把数字包装成 NSNumber。

### 12.4 contentMode（图片显示模式）

UIImageView 的 `contentMode` 控制图片在"框"里怎么显示：

| 模式                | 效果                    |
| ----------------- | --------------------- |
| `ScaleAspectFit`  | 等比缩放，**完整显示**，留白（最常用） |
| `ScaleAspectFill` | 等比缩放，**填满框**，裁切（头像）   |
| `ScaleToFill`     | 拉伸填满，**变形**（默认值，不推荐）  |

**默认是 ScaleToFill（会变形），所以建议显式设置。**

---

## 十三、单例模式

### 13.1 什么是单例

**整个程序运行期间，这个类只有一个实例，所有地方共享它。** 三个特征：全局唯一、全局共享、长生命周期。

### 13.2 实现原理（dispatch_once）

```objc
static DBManager *_instance = nil;   // 静态指针，初始指向空

+ (instancetype)sharedManager {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [[DBManager alloc] init];   // 只执行一次
    });
    return _instance;
}
```

**dispatch_once 保证 block 只执行一次：** 第一次调用执行（创建对象）+ 标记；之后调用跳过 block 直接返回 `_instance`。

### 13.3 类方法 vs 实例方法（单例的关键）

|       | 类方法（+） | 实例方法（-） |
| ----- | ------ | ------- |
| 调用者   | 类名     | 对象      |
| 需要对象吗 | ❌ 不需要  | ✅ 需要    |

```objc
[DBManager sharedManager];   // 类方法，类名直接调
[m insertContact:c];         // 实例方法，必须先有对象
```

**`sharedManager` 为什么是类方法？** 因为"获取单例"发生在**还没有对象**的时候，没对象就没法调实例方法，只能用类名调。

**`[[Manager sharedManager] 方法]` 是两层：**

```objc
[[DBManager sharedManager] insertContact:c];
// 第一层：[DBManager sharedManager] → 类方法返回唯一对象
// 第二层：[对象 insertContact:c]     → 对象调实例方法
```

**【难点】** 用户问"manager 不是类吗，为什么能调方法？"——因为 `sharedManager` 是**类方法**（+号），类名可以直接调；返回的是对象，再在对象上调实例方法。

### 13.4 alloc 会破坏单例吗？

**默认不报错，但会创建新对象，破坏"唯一性"。**

```objc
DBManager *a = [DBManager sharedManager];  // 唯一对象
DBManager *b = [[DBManager alloc] init];   // 新对象！和 a 不是同一个
```

**"破坏单例" = 破坏"唯一性"的意义，不是让对象消失**（单例还在，只是多了一个不共享数据的新对象）。

要防止，需重写 `allocWithZone:`（进阶），或靠约定"大家都用 sharedManager，别用 alloc"。

### 13.5 什么时候用单例

**三个信号（命中任意一个）：**

| 信号       | 例子         |
| -------- | ---------- |
| 全局共享状态   | 登录用户、配置    |
| 资源昂贵只需一份 | 数据库连接、网络会话 |
| 必须全局唯一   | 通知中心、文件管理  |

**系统自带的单例（你早用过了）：** `[NSNotificationCenter defaultCenter]`、`[NSUserDefaults standardUserDefaults]`、`[NSFileManager defaultManager]`、`[UIApplication sharedApplication]`——方法名都是"类方法返回唯一对象"，和 `sharedManager` 一样。

**别滥用：** 普通业务对象（如 Contact 联系人，每个都不同）不能用单例。

---

## 十四、NSTimer

### 14.1 scheduledTimer vs timerWith

```objc
// ✅ scheduledTimer：创建 + 自动加入 RunLoop，自动开跑
[NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(tick) userInfo:nil repeats:YES];

// ⚠️ timerWith：只创建，不自动跑，要手动 addTimer
NSTimer *t = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(tick) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:t forMode:NSDefaultRunLoopMode];
```

**【易混淆】** 就差一个 `scheduled`，行为完全不同。用 `timerWith` 忘了手动加入 RunLoop，计时器不会跑（用户踩过这个坑）。

### 14.2 invalidate

```objc
[self.timer invalidate];   // ① 让计时器作废（真正停止）
self.timer = nil;          // ② 清掉引用
```

**为什么必须 invalidate？** 计时器被 RunLoop 持有，光 `self.timer = nil` 只是松开自己的手，RunLoop 还抓着它，它还在跑。`invalidate` 让计时器从 RunLoop 移除。

**正确停止 = invalidate + 置 nil，两步一起。**

---

## 十五、FMDB 数据库（通讯录实战）

### 15.1 三个类

|      | FMDatabase | FMDatabaseQueue | FMDatabasePool |
| ---- | ---------- | --------------- | -------------- |
| 是什么  | 单连接        | 队列（串行）          | 连接池（并行）        |
| 线程安全 | ❌ 不安全      | ✅ 安全            | ✅ 安全           |
| 适用   | 简单单线程      | **大多数场景 ★**     | 高并发            |

**类比银行柜台：** FMDatabase = 一个柜台没人维持秩序；FMDatabaseQueue = 一个柜台排队；FMDatabasePool = 多个柜台同时服务。

**初学者用 FMDatabaseQueue 就够了。**

### 15.2 沙盒与路径

**沙盒：** 每个 App 有自己独立的存储空间，别的 App 进不来。

```
沙盒
├── Documents/  ← 持久数据，会备份（数据库放这）
├── Library/    ← 缓存、偏好
└── tmp/        ← 临时文件
```

```objc
// 找 Documents 目录路径（通用搜目录方法，换第一个参数可找不同目录）
NSString *docPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];

// 拼上文件名（专拼路径，自动处理斜杠）
NSString *dbPath = [docPath stringByAppendingPathComponent:@"contact.db"];

// 建连接
_queue = [FMDatabaseQueue databaseQueueWithPath:dbPath];
```

- `NSSearchPathForDirectoriesInDomains`：通用"搜目录"方法，换第一个参数（`NSDocumentDirectory`/`NSCachesDirectory` 等）可找不同目录。iOS 上实际只返回一个，`firstObject` 就是它。
- `stringByAppendingPathComponent`：**专拼路径**，自动补斜杠/去重斜杠（比 `stringWithFormat` 安全）。

### 15.3 SQL 占位符 ?

```objc
NSString *sql = @"INSERT INTO contact (name, phone, avatar) VALUES (?, ?, ?)";
[db executeUpdate:sql, name, phone, avatar];
// 参数按顺序填到 ? 的位置，FMDB 自动处理特殊字符（防 SQL 注入）
```

- `?` = 占位符，后面参数按顺序替换
- 用 `?` 而不是直接拼字符串，是为了**防 SQL 注入**

### 15.4 `__block` 与 `[NSNull null]`

**`__block`：** block 默认"拷贝副本"捕获外部变量，不能修改；加 `__block` 变成"引用同一个"，block 里改外部能看到。

```objc
__block BOOL success = NO;
[_queue inDatabase:^(FMDatabase *db) {
    success = [db executeUpdate:...];   // 加 __block 才能改
}];
// 这里 success 是真正结果
```

**`[NSNull null]`：** `nil` 在方法参数里 = "参数结束"标记，会导致后面参数错位。要用 `[NSNull null]` 这个"空值对象"表示空。

```objc
contact.avatarPath ?: [NSNull null]
// avatarPath 有值用它，是 nil 就用 NSNull 代替
```

### 15.5 装箱 `@()`

```objc
@(contactId)   // 把基本类型 NSInteger 包装成 NSNumber 对象
```

**为什么？** SQL 占位符 `?` 的参数必须是对象，`NSInteger` 是基本类型不能直接传，所以装箱。

`@()` = 装箱（基本类型 → 对象），对应拆箱 `integerValue`（对象 → 基本类型）。和 `mas_equalTo(100)` 内部机制一样。

---

## 十六、字符串拼接与时间戳

### 16.1 三种字符串拼接方法（怎么选）

| 方法                                | 用途          | 会处理斜杠吗    |
| --------------------------------- | ----------- | --------- |
| `stringByAppendingString:`        | 拼**普通字符串**  | ❌ 不管斜杠    |
| `stringWithFormat:`               | 拼**带格式/数字** | ❌ 不管斜杠    |
| `stringByAppendingPathComponent:` | 拼**文件路径**   | ✅ 自动补/去斜杠 |

```objc
// ① 拼普通文字 → stringByAppendingString:
[@"你好，" stringByAppendingString:@"张三"];   // "你好，张三"

// ② 拼带数字/格式 → stringWithFormat:
[NSString stringWithFormat:@"%02ld:%02ld", minutes, seconds];   // "01:30"

// ③ 拼文件路径 → stringByAppendingPathComponent:（首选）
[docPath stringByAppendingPathComponent:@"contact.db"];   // ".../Documents/contact.db"
```

**【难点】** 拼路径为什么不能用 `stringByAppendingString:`？

```objc
NSString *doc = @"/var/.../Documents";   // 注意：末尾没有斜杠

[doc stringByAppendingString:@"contact.db"];         // ❌ "/var/.../Documentscontact.db"（缺斜杠，非法路径）
[doc stringByAppendingPathComponent:@"contact.db"];  // ✅ "/var/.../Documents/contact.db"（自动补斜杠）
```

- `stringByAppendingString:` = **纯拼接**，两个字符串直接粘一起，**不管斜杠**，拼路径会缺斜杠（或双斜杠）
- `stringByAppendingPathComponent:` = **专为路径设计**，自动补斜杠、去重斜杠，保证永远是合法路径

**口诀：拼路径用 AppendingPathComponent，拼普通文字用 AppendingString，带格式用 stringWithFormat。**

### 16.2 时间戳（生成唯一文件名）

```objc
[NSString stringWithFormat:@"avatar_%ld.png", (long)[[NSDate date] timeIntervalSince1970]]
//                         ↑ 模板           ↑ 时间戳（每张都不同）
```

**三层拆解：**

```objc
[NSDate date]                          // ① 拿"现在"时刻（NSDate = 日期时间类）
[[NSDate date] timeIntervalSince1970]  // ② 转成秒数（1970-01-01 到现在的秒数，返回 double，即"时间戳"）
(long)[[NSDate date] timeIntervalSince1970]   // ③ (long) 把小数转成整数
```

**为什么文件名要带时间戳？** 生成**唯一文件名**，防止多张头像存成同名互相覆盖：

```
时间戳 = 每存一张都不一样（时间一直在走）
avatar_1786919763.png   ← 第一张
avatar_1786921234.png   ← 第二张（时间戳不同，文件名不同，不覆盖）
```

`%ld` 是整数占位符，把时间戳数字填进 `avatar_` 和 `.png` 之间。

---

## 十七、UITableView 列表

### 17.1 数据源方法（numberOfRows / cellForRow）

TableView 靠两个数据源方法（遵守 `UITableViewDataSource` 协议）显示数据：

```objc
// ① 告诉 TableView 一共几行
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.contacts.count;   // 3 个联系人 → 返回 3 → 显示 3 行
}

// ② 系统为每一行调用一次，问"这一行显示啥"
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    Contact *c = self.contacts[indexPath.row];   // 用行号取数组里的数据
    cell.textLabel.text = c.name;
    return cell;
}
```

**运行逻辑：** 系统先问"几行"（得到 3）→ 逐行调用 `cellForRowAtIndexPath:`，自动传 indexPath(row=0/1/2)。

### 17.2 indexPath（行位置）

```objc
indexPath.section  // 第几组（默认都第 0 组）
indexPath.row      // 第几行（0、1、2...）
```

**不是你自己获取的，是系统传进来的参数。** 作用：用 `indexPath.row` 当数组下标取数据（`self.contacts[indexPath.row]`）。

### 17.3 Cell 复用机制

```objc
static NSString *cellId = @"cell";   // 复用标签（static 只初始化一次）
UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellId];  // 从复用池捞
if (!cell) {   // 池里没有 → 第一次创建
    cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:cellId];
}
```

- **复用池：** 滚出屏幕的 cell 被回收，滚进来的新行从池里捞出来复用（只改文字），避免几百行都 alloc 导致内存爆炸。
- **Cell ID ≠ 系统 cell：** Cell ID 是"复用标签"（参与复用都要有）；"系统 cell"指的是用 `UITableViewCell` 类 + 某样式。两者各管各的。

**【难点】** 用户问"不是用系统 cell 吗，怎么还设 Cell ID？"——Cell ID 和"是不是系统 cell"是两码事。复用标签不管系统还是自定义 cell 都要有。

### 17.4 系统 cell 四种样式

| 样式             | 有哪些位置                                         |
| -------------- | --------------------------------------------- |
| `Default`      | 只有 textLabel（一行）                              |
| `Subtitle`（常用） | imageView + textLabel + detailTextLabel（上下两行） |
| `Value1`       | textLabel + detailTextLabel（左右）               |
| `Value2`       | textLabel + detailTextLabel                   |

`Subtitle` 天生"图片 + 主标题 + 副标题"，正好放头像 + 名字 + 电话。**系统样式放不下（要按钮/开关/复杂布局）才自定义 cell。**

### 17.5 初始化（initWithFrame:style:）

```objc
[[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
//                                     ↑ 先占位 0，后面约束覆盖   ↑ 样式必须创建时定
```

- style 只有两种：`Plain`（普通，默认白底）/ `Grouped`（分组，浅灰底）
- **style 创建时定死、改不了** → 放初始化方法里；`backgroundColor` 随时能改 → 属性设置

### 17.6 点击某行（didSelectRowAtIndexPath）

```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    // 用户点击了某一行，indexPath 是被点的那行
}
```

方法名拆读：`didSelect`（已选中）+ `Row`（某行）+ `AtIndexPath`（在某个位置）= "点击了某一行"。

### 17.7 删除（commitEditingStyle + 删除三步）

```objc
- (void)tableView:(UITableView *)tableView
    commitEditingStyle:(UITableViewCellEditingStyle)editingStyle
     forRowAtIndexPath:(NSIndexPath *)indexPath {

    if (editingStyle == UITableViewCellEditingStyleDelete) {   // 判断是不是删除
        Contact *c = self.contacts[indexPath.row];
        [[DBManager sharedManager] deleteContact:c.contactId];          // ① 删数据库
        [self.contacts removeObjectAtIndex:indexPath.row];              // ② 删数组
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];  // ③ 删界面
    }
}
```

**删除三步缺一不可：** 数据库、数组、界面是**三份独立数据**。删数据库不会自动刷新界面（`viewWillAppear` 只在页面重新出现时触发，删除时页面还在显示，不触发）。

**【难点】** editingStyle 判断：这个方法是"删除+插入"**共用的回调**，系统统一调这一个方法，用 `editingStyle` 参数告诉你是哪种。两个按钮不是各调各的方法，而是都调这同一个方法、靠参数区分。不判断的话，插入也会执行删除。

### 17.8 数组删除方法

| 方法                      | 按什么删               |
| ----------------------- | ------------------ |
| `removeObjectAtIndex:`  | 按**下标**（位置，快又准）    |
| `removeObject:`         | 按**对象**（内容，遍历比较，慢） |
| `removeObjectsInArray:` | 按**数组**（批量）        |

**有准确位置（行号）就用 `removeObjectAtIndex:`**；只知道"删掉某个对象"才用 `removeObject:`。

### 17.9 deleteRowsAtIndexPaths 动画

```objc
[tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
//                         ↑ 数组（可一次删多行）       ↑ 动画（Fade 淡出）
```

动画类型：`Fade`（淡出）、`Left`/`Right`（左/右滑出）、`Top`/`Bottom`（上/下滑出）、`None`（无）、`Automatic`（默认）。

### 17.10 barButtonItem 单复数

| 属性                        | 放几个 | 类型                  |
| ------------------------- | --- | ------------------- |
| `rightBarButtonItem`（单数）  | 1 个 | `UIBarButtonItem *` |
| `rightBarButtonItems`（复数） | 多个  | `NSArray`（第一个显示在最右） |

---

## 十八、FMDB 查询（结果集）

### 18.1 executeQuery vs executeUpdate

| 方法               | 对应 SQL                      | 返回                 |
| ---------------- | --------------------------- | ------------------ |
| `executeUpdate:` | INSERT/DELETE/UPDATE/CREATE | `BOOL`（成没成）        |
| `executeQuery:`  | SELECT                      | `FMResultSet`（结果集） |

增删改问"成没成"，查询问"有哪些数据"。

### 18.2 FMResultSet 游标 + [rs next]

```objc
FMResultSet *rs = [db executeQuery:@"SELECT * FROM contact"];  // 游标停在第一行之前

while ([rs next]) {   // 游标下移一行 + 返回"还有没有"（BOOL）
    c.name = [rs stringForColumn:@"name"];   // 读"当前行"的字段
}
[rs close];   // 用完关闭，释放资源（查询必须 close，增删改不用）
```

**`[rs next]` 做两件事：① 游标下移一行 ② 返回这一行有没有数据（YES/NO）。**

### 18.3 xxxForColumn 读取字段

**规律：要转成什么类型，就用对应方法。**

| 方法                 | 转成               |
| ------------------ | ---------------- |
| `intForColumn:`    | int              |
| `longForColumn:`   | long / NSInteger |
| `doubleForColumn:` | double           |
| `stringForColumn:` | NSString         |
| `boolForColumn:`   | BOOL             |

两种定位：`xxxForColumn:`（用**列名**，可读性好，平时用这个）/ `xxxForColumnIndex:`（用**列序号** 0 起，结果没列名时用）。

### 18.4 SELECT COUNT(*) 计数

```objc
FMResultSet *rs = [db executeQuery:@"SELECT COUNT(*) FROM contact WHERE name = ?", name];
if ([rs next]) {
    exist = [rs intForColumnIndex:0] > 0;   // COUNT 结果只有一列一个数字，用序号 0 取
}
```

`COUNT(*)` 是**计数**（数几行），结果不是数据集，而是"一个数字"（一行一列），**没有列名**，所以用 `intForColumnIndex:0`。数字 > 0 说明存在。

### 18.5 AUTOINCREMENT 自增主键

```sql
id INTEGER PRIMARY KEY AUTOINCREMENT
```

- `PRIMARY KEY` = 主键（唯一标识，不重复）
- `AUTOINCREMENT` = 自动递增（每插一行 id 自动 +1）

**所以 INSERT 语句里不写 id**，数据库自动分配。

---

## 十九、交互控件

### 19.1 手势识别（UITapGestureRecognizer）

```objc
self.avatarView.userInteractionEnabled = YES;   // ① UIImageView 默认不能点击，先开交互
UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(chooseAvatar)];
[self.avatarView addGestureRecognizer:tap];     // ② 挂到视图上
```

- `UITapGestureRecognizer` = 点击检测器（还有 Swipe 滑动、Pinch 捏合、LongPress 长按、Pan 拖动）
- `initWithTarget:action:` 和按钮的 `addTarget:action:` 是**同一个 Target-Action 模式**
- **UIImageView 默认不能点击**（只显示图片），要先开 `userInteractionEnabled` + 加手势，两步缺一不可

### 19.2 UITextField

```objc
self.nameField.placeholder = @"请输入姓名";                  // 灰色提示文字（可选，美化）
self.nameField.borderStyle = UITextBorderStyleRoundedRect;   // 边框（默认 None = 纯空白）
self.phoneField.keyboardType = UIKeyboardTypeNumberPad;      // 数字键盘（功能增强）
```

- `placeholder`、`borderStyle` 是**美化**，不写也能输入；`keyboardType` 是**功能增强**
- `borderStyle` 默认 `None`（纯空白无框），`RoundedRect`（圆角灰框，常用）

### 19.3 相册选择（UIImagePickerController）

> **UIImagePickerController = 系统自带的"选图弹窗"，帮你弹出相册/相机，让用户选一张照片或拍一张，再通过代理把结果交给你。** 你不用自己写相册界面。

```objc
// ① 创建 picker 对象
UIImagePickerController *picker = [[UIImagePickerController alloc] init];

// ② 设定图片来源（sourceType）
picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;  // 从相册选
//    其他选项：Camera（相机拍摄）/ SavedPhotosAlbum（仅最近照片）

// ③ 设置代理：选完图回调给谁（给 self 处理）
picker.delegate = self;

// ④ 弹出来（把 picker 这个控制器作为弹出层显示到屏幕）
[self presentViewController:picker animated:YES completion:nil];

// ⑤ 选完回调（delegate/代理方法）
- (void)imagePickerController:(UIImagePickerController *)picker
    didFinishPickingMediaWithInfo:(NSDictionary *)info {
    UIImage *image = info[UIImagePickerControllerOriginalImage];   // 取出选中的图
    [picker dismissViewControllerAnimated:YES completion:nil];      // 关掉相册
}
```

#### 三个最容易混的点（重点）

**① `picker.delegate = self` 不是"自己代理自己"，是 picker 委托 self。**
picker 只负责"弹窗 + 用户选"；选完图，picker **回调给 self**，由 self 处理图片。代码里 self 既创建了 picker、又当它代理，但"收图处理"的活是 self 干的。

**② 代理方法不是 picker"自己"的方法，是协议定义的、由 self 实现。**
```objc
@interface ContactEditViewController () <UIImagePickerControllerDelegate, ...>
// ↑ 承诺实现协议方法
picker.delegate = self;
// ↑ 兑现承诺：picker 选完图调用 self 的方法
```
这就是**代理模式**：picker 是委托方（只负责"提醒"，调用代理方法把 info 递过来），self 是代理方（真正处理图片）。

**③ present 弹 picker 跟 sourceType 没关系。**
- `sourceType` = 决定 **picker 里面装的是什么**（相册列表 还是 相机）
- `presentViewController:picker` = 决定 **把它整个弹到屏幕上**
- 写 picker 是因为"要弹出的对象就是 picker"，不是因为 sourceType。

#### 取图 key 固定写法

```objc
info[UIImagePickerControllerOriginalImage]   // 原始图片（最常用）
info[UIImagePickerControllerEditedImage]     // 裁剪后图片（若允许编辑）
```

> **一句话记忆：创建 picker → 设来源（相册/相机）→ 设代理 → present 弹出 → 代理方法里用固定的 key 取图。**

### 19.4 图片存沙盒

```objc
[UIImagePNGRepresentation(image) writeToFile:filePath atomically:YES];
// ① UIImagePNGRepresentation：图片 → NSData（二进制数据）
// ② writeToFile:atomically:：数据写入文件（真正落盘，atomically = 原子写入防损坏）
```

### 19.5 完整闭环：选图 → 存路径 → 入库 → 读图

> **核心设计：图片不直接存数据库，只存"路径字符串"，用的时候按路径从本地读文件。** 因为数据库/对象存不了大图片。

**完整链路（从头到尾）：**

```
用户点头像
  → 创建 picker（相册）→ present 弹出
  → 用户选图 → 点"完成/使用"按钮（不是单击图就触发！）
  → picker 自动调 self 的代理方法 didFinishPickingMediaWithInfo:
     → 从 info[OriginalImage] 取出 UIImage（原始图）
     → 显示到头像框
     → saveImage: 存沙盒 → 得到路径 avatarPath
  → 保存联系人：avatarPath 存进数据库
  → 下次列表页：从数据库取 avatarPath → 按路径读图 → 显示
```

**① 选图（代理方法，用户点"完成"才触发）**

```objc
- (void)imagePickerController:(UIImagePickerController *)picker
    didFinishPickingMediaWithInfo:(NSDictionary *)info {
    UIImage *image = info[UIImagePickerControllerOriginalImage];  // 从 info 拆出原始图
    self.avatarView.image = image;                                // 显示到头像框
    self.selectedAvatarPath = [self saveImage:image];             // 存沙盒拿路径
    [picker dismissViewControllerAnimated:YES completion:nil];    // 关相册
}
```

**为什么接收 picker 和 info，不直接传图片？**

| 参数 | 是什么 | 干嘛用 |
|---|---|---|
| `picker` | 触发回调的那个 picker | 识别/关掉它（`[picker dismiss...]`） |
| `info` | 装着图片等一堆信息的**字典** | 用固定 key 拆：`info[OriginalImage]` |

> 系统把原始图、编辑图、媒体类型等打包成字典，**图片在 info 里，用固定 key 取**，不是直接传图片对象。

**② 存沙盒（UIImage → PNG 数据 → 写文件 → 返回路径）**

```objc
- (NSString *)saveImage:(UIImage *)image {
    NSString *docPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *fileName = [NSString stringWithFormat:@"avatar_%ld.png", (long)[[NSDate date] timeIntervalSince1970]];  // 时间戳防重名
    NSString *filePath = [docPath stringByAppendingPathComponent:fileName];
    [UIImagePNGRepresentation(image) writeToFile:filePath atomically:YES];   // 图片写成文件
    return filePath;   // 拿到文件路径
}
```

**③ 存数据库（只存路径）**

```objc
c.avatarPath = self.selectedAvatarPath;                 // 把路径字符串存进对象
[[DBManager sharedManager] insertContact:c];            // insert/update 只把路径写进库
```

**④ 下次读图（从数据库取路径 → 按路径读本地文件）**

```objc
if (c.avatarPath) {                                    // 从数据库取出的路径
    cell.imageView.image = [UIImage imageWithContentsOfFile:c.avatarPath];   // 按路径从本地读文件显示
} else {
    cell.imageView.image = [UIImage imageNamed:@"默认头像"];  // 没路径 → 默认头像
}
```

> **一句话记忆：图片存文件（沙盒），数据库只记文件路径，用的时候 imageWithContentsOfFile 按路径读回来。**

---

## 二十、分类里方法能不能带参数

> **能。** 分类里的方法和普通方法一样，参数完全自由。要不要参数由**功能**决定，跟"在不在分类里"无关。

之前看的分类例子没参数（如 `containsDigit`），是因为那些方法只靠 `self` 就能判断（比如"自己有没有数字"），不需要外部传值。

```objc
@interface NSString (fz)
- (BOOL)containsString:(NSString *)str;   // 判断"是否包含某子串"，必须传参数
@end

@implementation NSString (fz)
- (BOOL)containsString:(NSString *)str {
    return [self rangeOfString:str].location != NSNotFound;
}
@end
```

| 方法                | 参数 | 为什么                 |
| ----------------- | -- | ------------------- |
| `containsDigit`   | 无  | 判断自己有没有数字，无需外部输入    |
| `containsString:` | 有  | 判断有没有包含 XX，需要外部传 XX |

> **判断标准：这方法需不需要外部给它数据？需要就带参，不需要就不带。与是否分类无关。**

---

## 二十一、NSTimer 销毁：invalidate 不会自动清零【深度】

**先把两个动作分清楚——"停止"和"清零"是两件事：**

```objc
[self.timer invalidate];   // ① 停止：计时器失效、不再触发
self.timer = nil;          // ② 清零：指针清空，变成 nil
```

### 关键：invalidate 不会自动清零

```objc
[self.timer invalidate];
// 执行完这行：计时器停了（失效了）
// 但 self.timer 这个指针，还指向那个已失效的 timer 对象！没有变成 nil

self.timer = nil;
// 执行完这行：指针才被清空，self.timer 变成 nil ✅
```

| 代码                        | 干什么       | 谁变 nil 了         |
| ------------------------- | --------- | ---------------- |
| `[self.timer invalidate]` | 停止计时器（作废） | ❌ 没有，指针还指向它      |
| `self.timer = nil`        | 清空指针      | ✅ timer 属性变成 nil |

**为什么 invalidate 不能清零？** 因为 `invalidate` 是 timer 对象的方法，它管不到"外面指向它的指针"。你关掉了闹钟，但遥控器（`self.timer`指针）还在手里。

### 为什么只 invalidate 不置 nil 会出错

```objc
[self.timer invalidate];   // 只停止
// self.timer 还指向那个已失效的对象 → 若判断 if (self.timer) 它是非 nil
// 就会误以为计时器还在跑 → 逻辑出错
```

> **类比：timer 对象 = 闹钟，`self.timer` 指针 = 遥控器。`invalidate` = 关闹钟（闹钟停），`self.timer = nil` = 扔遥控器。关闹钟后遥控器还在手，必须主动扔。**
>
> **口诀：先 `invalidate` 停止，再 `置 nil` 清零，两步一起，缺一不可。**

---

## 二十二、通知销毁：dealloc vs viewWillDisappear【深度】

### 核心判断标准：会不会循环引用 → dealloc 会不会被调用

|               | NSTimer                | 通知（NSNotificationCenter） |
| ------------- | ---------------------- | ------------------------ |
| 对 self 的引用    | target **强引用**         | observer **弱引用**(iOS9+)  |
| 会循环引用吗        | ✅ 会                    | ❌ 不会                     |
| dealloc 会被调用吗 | ❌ 不会（循环引用）             | ✅ 会（正常释放）                |
| 清理放哪          | 必须 `viewWillDisappear` | `dealloc` 就行             |

**通知**：`addObserver` 弱引用 observer，不会形成 `VC ↔ 通知中心` 的循环引用，VC 能正常释放，dealloc 正常调。所以通知**可以**在 dealloc 移除：

```objc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];   // ✅ 通知这么写没问题
}
```

**计时器**：`timer.target` 强引用 VC → 循环引用 → VC 无法释放 → dealloc 不被调用 → 放 dealloc 是死代码。所以计时器必须放 `viewWillDisappear`：

```objc
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [self.timer invalidate];
    self.timer = nil;
}
```

### dealloc vs viewWillDisappear 怎么选

| 写法                     | 含义             | 适用场景               |
| ---------------------- | -------------- | ------------------ |
| `dealloc` 移除           | 对象**销毁时**才停止监听 | 整个生命周期都要接收（登录状态变化） |
| `viewWillDisappear` 移除 | 页面**一消失**就停止监听 | 只在显示期间接收（键盘弹出）     |

> **口诀：会循环引用导致 dealloc 不调 → 放 viewWillDisappear；不会循环引用 → 放 dealloc。**
>
> 全局事件用 dealloc，页面相关事件用 viewWillDisappear。

---

## 二十三、NSNotification 体系：那 4 个类【骨架】

> Xcode 补全里 `NSNotification` 的 4 个角色。

| 类型                       | 是什么   | 作用              | 常用度     |
| ------------------------ | ----- | --------------- | ------- |
| **NSNotificationCenter** | 广播站   | 发/收通知           | ⭐⭐⭐ 天天用 |
| **NSNotification**       | 一条通知  | 回调参数，含名称+数据     | ⭐⭐      |
| **NSNotificationName**   | 通知名类型 | 本质是 NSString 别名 | ⭐⭐⭐     |
| **NSNotificationQueue**  | 通知队列  | 延迟/异步派发         | ⭐ 忽略    |

### defaultCenter 的三组方法（发/收/移）

```objc
// 发
[[NSNotificationCenter defaultCenter] postNotificationName:@"xxx" object:nil];

// 收（selector 版，最常用）
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handle:) name:@"xxx" object:nil];

// 收（block 版，少用）
addObserverForName:object:queue:usingBlock:

// 移
[[NSNotificationCenter defaultCenter] removeObserver:self];                       // 全移
[[NSNotificationCenter defaultCenter] removeObserver:self name:@"xxx" object:nil]; // 精确移
```

> **一句话：日常只用 `defaultCenter` + 发/收/移。回调收到的是 NSNotification（消息体），Name 是字符串，Queue 忽略。**

---

## 二十四、字符串/数组/字典：初始化、赋值、输出【骨架】

> **字面量三兄弟：字符串 `@"..."`、数组 `@[...]`、字典 `@{key:value}`。**

|         | 初始化（字面量）       | 赋值                      | 输出      |
| ------- | -------------- | ----------------------- | ------- |
| **字符串** | `@"hello"`     | `str = @"new"`（换指向）     | `%@`    |
| **数组**  | `@[@1,@2]`     | 可变用 `addObject:`        | `%@` 整串 |
| **字典**  | `@{@"k":@"v"}` | 可变用 `setObject:forKey:` | `%@` 整串 |

### 三个坑

```objc
// ① 不可变不能增删
NSArray *a = @[@1,@2];
[a addObject:@3];   // ❌ NSArray 不能增删，要用 NSMutableArray

// ② 字典取不到值是 nil
id v = dict[@"不存在的键"];   // v = nil，不崩，要用 objectForKey: 判断

// ③ 下标访问是"第几个元素"，不是字节
arr[0]   // 第 0 个元素
```

> **要增删改用可变版本（NSMutableArray / NSMutableDictionary），输出统一 `%@`。**

---

## 二十五、六大概念默写 + 多态/代理/类扩展【骨架】

> 考核默写专用，每个只留骨架 + 最容易卡的点。

### 25.1 Block（最卡：`^` 后面跟名字还是空）

**`^` 后永远不跟名字，跟的一定是 `(参数)` 或 `{`。名字只出现在"给它起变量名"时。**

```objc
// 定义变量：名字在类型里
int (^myBlock)(int a) = ^(int a) { return a * 2; };

// 作为方法参数：类型无名字
- (void)do:(void (^)(int))completion {
    completion(10);
}
[self do:^(int a) { NSLog(@"%d", a); }];   // 传值时名字不写
```

### 25.2 继承

```objc
@interface Animal : NSObject   // : 父类
- (void)eat;
@end
@interface Dog : Animal        // 子类继承父类
- (void)bark;
@end
// 重写父类方法：[super eat]; 再写自己的
```

### 25.3 分类（Category）

```objc
@interface NSString (fz)   // (分类名) 给已有类加方法
- (BOOL)containsDigit;
@end
```

### 25.4 协议（Protocol）

```objc
@protocol MyDelegate <NSObject>   // 合同
- (void)didSomething;             // 必须实现
@optional
- (void)optionalMethod;           // 可选
@end
@interface MyClass : NSObject <MyDelegate> @end   // 遵守协议
```

### 25.5 通知

```objc
// 发
[[NSNotificationCenter defaultCenter] postNotificationName:@"N" object:nil];
// 收
addObserver:selector:name:object:
// 移（dealloc 必做）
removeObserver:self;
```

### 25.6 代理（Delegate）

```objc
@protocol MyDelegate <NSObject>
- (void)didTap;
@end
@property (nonatomic, weak) id<MyDelegate> delegate;   // weak 防循环引用
if ([self.delegate respondsToSelector:@selector(didTap)]) {   // 判断已实现
    [self.delegate didTap];
}
```

### 25.7 多态【骨架】

> **口诀：看实际类型，不看指针声明类型。**

```objc
Animal *a = [[Dog alloc] init];   // 指针声明 Animal，实际是 Dog
[a eat];                          // 调 Dog 的版本（运行时动态绑定）
// 传不同对象 → 运行时找对应方法
```

### 25.8 类扩展【骨架】

> **口诀：.m 里的匿名分类（括号空），声明私有属性/方法。**

```objc
@interface MyClass ()              // () 空括号 = 类扩展
@property (nonatomic, strong) NSString *privateName;   // 私有
@end
// 与分类区别：分类 (名) 在 .h 加公开方法；类扩展 () 在 .m 声明私有
```

---

## 二十六、通讯录程序回顾·难点澄清

> 通讯录整体代码在 `通讯录程序.md`。这里只补回顾时追问出来的、课本上没有明说的关键点。

### 26.1 沙盒路径写法固定，`contact.db` 文件名自己起

**① 找沙盒路径是固定写法**（换个常量就能找不同目录）：

```objc
[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
//                                    ↑ 换第一个参数即可切换目录
// NSDocumentDirectory = Documents（持久数据）；NSCachesDirectory = 缓存；tmp 用 NSTemporaryDirectory
// 返回数组，firstObject 拿第一个；返回的是绝对路径，跟程序名无关
```

**② `.db` 扩展名是自己起的，不是固定规则：**

```objc
NSString *dbPath = [docPath stringByAppendingPathComponent:@"contact.db"];
//                                                  ↑ contact 自己起名
// 也可以叫 mymoney.sqlite / xxx.data —— SQLite 不看后缀，能打开就行
// 真正固定的是"前面的沙盒目录写法"，后面文件名 100% 自己定
```

### 26.2 FMDB 三层执行链：连接 → 队列 → block → SQL

**完整链（重点，最容易晕）：**

```
① 建连接（一次）：_queue = [FMDatabaseQueue databaseQueueWithPath:dbPath];
                                    ↑ 打开/创建数据库文件，建立连接，存进 _queue
② 后续每次操作：[_queue inDatabase:^(FMDatabase *db) { ... }];
                                    ↑ inDatabase 是"管家"：排队 + 把连接 db 递给你 + 用完归还
③ block 里：db 是管家递来的连接，真正干活
   [db executeUpdate:sql];  ← executeUpdate 才是真正执行 SQL
```

| 角色 | 是什么 | 干啥 |
|---|---|---|
| `_queue` | 数据库队列（入口） | 建立连接、排队、借出/收回 db |
| `inDatabase:` | 管家方法 | 把 db 递给你，block 执行完自动归还 |
| block | 待办清单 | 你在里面写 SQL 逻辑 |
| `db` | 实际连接 | 真正操作数据库（executeUpdate/executeQuery） |
| `executeUpdate:` | 执行增删改建表 | 返回 BOOL |

**执行顺序**：`init → createTable → inDatabase:(block) → block 里 executeUpdate:建表sql`，一层套一层。**必须先有连接（databaseQueueWithPath:），才能用它建表/操作。**

**executeUpdate vs executeQuery**：增删改/建表用 `executeUpdate:`（返回 BOOL）；查询 SELECT 用 `executeQuery:`（返回结果集）。

### 26.3 `{ }` 私有变量 vs @property 怎么选

```objc
@implementation DBManager {
    FMDatabaseQueue *_queue;   // { } = 私有成员变量，不自动生成 getter/setter
}
```

| 写法 | 可见性 | 生成 getter/setter? | 用在哪 |
|---|---|---|---|
| `@property`（.h） | 公开 | ✅ | 对外暴露的属性（Contact.name） |
| `@property`（.m 类扩展） | 私有 | ✅ | 本类私有但方便 self.xxx 访问 |
| `{ }` ivars（.m） | 私有 | ❌ | 纯内部工具，直接 `_queue` 访问 |

**选择标准**：`_queue` 是内部实现细节，不需要对外暴露、也不需要 getter/setter → 用 `{ }` 更克制干净。跟"哪个能存数据"无关，两者都能存，选"哪个更合适"。

### 26.4 本地删数据三处 vs 网络删数据一处

| | 网络删除 | 本地数据库删除 |
|---|---|---|
| 数据在哪 | **只有服务器一处**（单一真源） | 数据库 + 内存数组 + 界面 **三处** |
| 删哪里 | 删服务器就够 | 必须**三处都删** |
| 为什么 | 唯一真源，删完就没 | 三份独立拷贝，互不自动同步 |

**删三处代码（缺一不可）：**

```objc
[[DBManager sharedManager] deleteContact:c.contactId];   // ① 删数据库（永久源）
[self.contacts removeObjectAtIndex:indexPath.row];       // ② 删内存数组
[tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];  // ③ 删界面一行（带动画）
```

关键：TableView 显示的是内存数组 `self.contacts`，不是直接读数据库。只删数据库，数组和界面不会跟着变。所以必须三处同步删；删完用 `viewWillAppear` 重新读库兜底刷新。

### 26.5 数组不强制声明元素类型

```objc
@property (nonatomic, strong) NSMutableArray *contacts;          // 没写元素类型，也能跑
@property (nonatomic, strong) NSMutableArray<Contact *> *contacts;  // 推荐：写明装 Contact
```

**OC 数组内置"任何对象都能装"（id）**，所以不写类型也合法。但这是隐患：里面装错类型编译期发现不了，运行时一调方法才崩。

- 加 `<Contact *>` 是**可选**（iOS9+ 轻量泛型）：可读性 + 编译期提示
- 但**运行时不会强制拦**（跟 Java/C++ 的强泛型不同）
- **推荐写法**：`NSMutableArray<Contact *> *` 明确约定，减少出错

---

*最后更新：2026-08-18*

