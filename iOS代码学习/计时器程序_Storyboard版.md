# 计时器程序（Storyboard 版）

> 用 Storyboard 拖控件 + 连线实现，代码只保留逻辑，与纯代码版对比

## 功能

- 黑色背景，上方白色大字计时（`MM:SS` 分:秒）
- 下半区域两个长方形按钮平分：左边「暂停」（蓝色 + 三角），右边「开始」（绿色 + 两个竖线）

## 布局

```
┌──────────────────────────────┐
│                              │
│           00:00              │  ← 上半：时间显示
│                              │
│──────────────────────────────│ ← 屏幕中线
│              │               │
│   [▶] 暂停   │   [❚❚] 开始    │  ← 下半：两个长方形，各占一半宽
│   （蓝色）   │   （绿色）      │
└──────────────────────────────┘
```

---

## 操作步骤

### 第一步：拖控件

从右下角控件库拖到画布：
1. 拖 1 个 **Label**
2. 拖 2 个 **Button**

### 第二步：设外观

| 控件 | 操作 |
|---|---|
| View（背景） | 属性面板 → Background → **Black** |
| Label | 文字 `00:00`、颜色 White、字号 72 |
| 暂停按钮 | Background → **Blue**、Title 填 `▶` |
| 开始按钮 | Background → **Green**、Title 填 `❚❚` |

### 第三步：设约束（右下角图标）

**Label（上方居中）：**
1. 点 ① **Align** → 勾 "Horizontally in Container"
2. 点 ② **Pin** → 顶部那条线填 80

**两个按钮（平分下半区域）：**
1. 同时选中两个按钮 → ② **Pin** → 勾 **Equal Widths**（等宽）
2. 暂停按钮 → Pin：左边线填 0、底边线填 0、顶部填 0（让高度从屏幕中线到底）
3. 开始按钮 → Pin：右边线填 0、底边线填 0、顶部填 0

> 核心：两个按钮 `top` 都对齐到屏幕中线，`bottom` 都贴底，`Equal Widths` 让它们各占一半。

### 第四步：连线（Ctrl + 拖拽）

按 `⌘⌥↩` 打开辅助编辑器，让代码和画布并排，然后按住 **Ctrl** 拖：

| 控件 | 拖到哪 | 类型 | 命名 |
|---|---|---|---|
| Label | `.h` | **Outlet** | `timeLabel` |
| 暂停按钮 | `.m` | **Action** | `pauseClicked` |
| 开始按钮 | `.m` | **Action** | `startClicked` |

---

## 完整代码

### ViewController.h

```objc
#import <UIKit/UIKit.h>

@interface ViewController : UIViewController

@property (weak, nonatomic) IBOutlet UILabel *timeLabel;   // Label 的出口（连线自动生成）

- (IBAction)startClicked:(id)sender;   // 开始按钮
- (IBAction)pauseClicked:(id)sender;   // 暂停按钮

@end
```

### ViewController.m

```objc
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;          // 计时器
@property (nonatomic, assign) NSInteger totalSeconds;  // 总秒数
@end

@implementation ViewController

// ---- 开始按钮 ----
- (IBAction)startClicked:(id)sender {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self
                                                selector:@selector(tick)
                                                userInfo:nil
                                                 repeats:YES];
}

// ---- 暂停按钮 ----
- (IBAction)pauseClicked:(id)sender {
    [self.timer invalidate];
    self.timer = nil;
}

// ---- 每秒触发 ----
- (void)tick {
    self.totalSeconds++;
    NSInteger minutes = self.totalSeconds / 60;   // 整除 → 分钟
    NSInteger seconds = self.totalSeconds % 60;   // 取余 → 秒
    self.timeLabel.text = [NSString stringWithFormat:@"%02ld:%02ld", minutes, seconds];
}

@end
```

> Storyboard 版**不需要** SceneDelegate 设根控制器（Storyboard 自动加载），也**不需要**写 `alloc init` 和约束代码（都在画布上拖好了）。

---

## 关键知识点：NSTimer 循环引用

### 什么是循环引用

```objc
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self ...];
```

这行产生两条强引用：

```
① self.timer 是 strong 属性 → VC 强引用 timer
② timer 的 target:self     → timer 强引用 VC（NSTimer 对 target 是强引用！）
```

```
   VC ──self.timer──→ timer
    ↑                   │
    └────target:self────┘
```

形成闭环，**谁也释放不了谁**。

### 后果

1. VC 被 pop 掉后，timer 还强引用它 → **VC 永远无法释放（内存泄漏）**
2. VC 无法释放 → **`dealloc` 根本不会被调用**
3. timer 还在 RunLoop 里继续跑

### 解决

```objc
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [self.timer invalidate];   // invalidate 让 timer 释放对 target 的强引用 → 断开循环
    self.timer = nil;          // 同时释放 VC 对 timer 的引用
}
```

### ⚠️ 最关键的易错点

**不能只在 `dealloc` 里 invalidate！** 因为循环引用导致 `dealloc` 根本不会被调用，你在 dealloc 里写的 invalidate 永远执行不到。必须在 `viewWillDisappear` 这种能主动触发的地方 invalidate。

> 题目本身只要求"能计时、能暂停"，所以上面的题目代码没加 `viewWillDisappear`（保持最简）。但**以后做多页面的 App，涉及 NSTimer 就一定要记得加**，否则会内存泄漏。

---

## 与纯代码版对比

| | 纯代码版 | Storyboard 版 |
|---|---|---|
| 文件数 | 3 个（多 SceneDelegate） | 2 个（不用 SceneDelegate） |
| 控件创建 | 手写 `alloc init` | 拖拽 |
| 布局 | 手写 Masonry | 拖拽/点图标 |
| 代码量 | 多（viewDidLoad 一大段） | 少（就 3 个方法） |
| 能看清原理 | ✅ | ❌（拖拽背后看不到） |

**结论：** 只求快速做出来 → Storyboard 省代码；想学懂 iOS 原理 → 纯代码更透明。
