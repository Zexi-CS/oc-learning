# SDWebImage 四大任务 — 完整学习笔记

> 2026-08-10 起 · Category + Associated Objects + 多线程锁 + 图片下载列表

---

## 一、任务总览

| 任务 | 主题 | 核心知识点 | 依赖关系 |
|------|------|-----------|---------|
| 1 | NSString 拼接分类 | Category、方法链式复用、`componentsJoinedByString:` | 独立 |
| 2 | UIView 显示分类 | Associated Objects、懒加载 getter、readonly 保护 | 独立 |
| 3 | 300 子线程 + 三种锁 | GCD 多线程、NSLock/@synchronized/semaphore | 独立工程 |
| 4 | 图片下载进度列表 | SDWebImage、进度回调、Cell 复用取消 | 依赖 1+2 |

---

## 二、任务 1 — NSString 拼接分类

### 题目要求

给 NSString 设计一个分类，扩展 6 个方法，分别有 1~6 个 NSString 参数。返回值均为 NSString，用下划线 `_` 拼接所有参数。按 SDWebImage 复用方法的方式实现。

### 核心概念：SDWebImage 方法复用模式

只写一个真正干活的方法（6 参），其余 1~5 参全部用 `@""` 补空缺，然后链式调用下一级：

```
concat1 → concat2 → concat3 → concat4 → concat5 → concat6（真干活）
```

### NSString+Concatenation.h

```objc
#import <Foundation/Foundation.h>

@interface NSString (Concatenation)

+ (NSString *)concat1:(NSString *)a;
+ (NSString *)concat2:(NSString *)a _:(NSString *)b;
+ (NSString *)concat3:(NSString *)a _:(NSString *)b __:(NSString *)c;
+ (NSString *)concat4:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d;
+ (NSString *)concat5:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e;
+ (NSString *)concat6:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e _____:(NSString *)f;

@end
```

### NSString+Concatenation.m

```objc
#import "NSString+Concatenation.h"

@implementation NSString (Concatenation)

// 地基：只有这一个真干活
+ (NSString *)concat6:(NSString *)a _:(NSString *)b __:(NSString *)c
                   ___:(NSString *)d ____:(NSString *)e _____:(NSString *)f {
    NSMutableArray *parts = [NSMutableArray array];
    if (a.length > 0) [parts addObject:a];
    if (b.length > 0) [parts addObject:b];
    if (c.length > 0) [parts addObject:c];
    if (d.length > 0) [parts addObject:d];
    if (e.length > 0) [parts addObject:e];
    if (f.length > 0) [parts addObject:f];
    return [parts componentsJoinedByString:@"_"];
}

// 以下 5 个：补空 → 调下一级
+ (NSString *)concat1:(NSString *)a { return [self concat2:a _:@""]; }
+ (NSString *)concat2:(NSString *)a _:(NSString *)b { return [self concat3:a _:b __:@""]; }
+ (NSString *)concat3:(NSString *)a _:(NSString *)b __:(NSString *)c { return [self concat4:a _:b __:c ___:@""]; }
+ (NSString *)concat4:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d { return [self concat5:a _:b __:c ___:d ____:@""]; }
+ (NSString *)concat5:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e { return [self concat6:a _:b __:c ___:d ____:e _____:@""]; }

@end
```

### 为什么写成类方法（`+`）？

拼接不需要预先存在一个 NSString 对象——你还没拼出来。类方法让调用者直接 `[NSString concat1:@"A"]`，不依赖任何已有实例。写成实例方法的话，2~6 参没有合适的宿主，语法不自然。

### `componentsJoinedByString:` — NSArray 系统方法

等价于 C 语言循环遍历数组，每个元素之间插分隔符，最后一个元素不加。空字符串自动跳过，不会产生连续下划线。

---

## 三、任务 2 — UIView 显示分类（Associated Objects）

### 题目要求

给 UIView 扩展一个分类，实现 `image` 属性（像 UIImageView 一样显示图片）和 `text` 属性（像 UILabel 一样显示文字）。需考虑性能（子视图只创建一次）。

### 核心问题：Category 不能加成员变量

```objc
// 普通类 → 编译器自动生成 _image
@property (nonatomic, strong) UIImage *image;  // ✅

// Category 里 → 只声明 getter/setter，无 _image
@property (nonatomic, strong) UIImage *image;  // ❌ 编译器不生成变量
```

**原因：** 对象的内存布局编译时就确定了（isa 指针 + 所有 ivar 按顺序排列）。Category 在运行时才加载，对象的尺寸已固定，无法追加变量。

### 解决方案：Associated Objects

Runtime 用全局哈希表给对象绑定外部值：

```objc
objc_setAssociatedObject(self, &myKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);  // 存
objc_getAssociatedObject(self, &myKey);                                            // 取
```

### 存储策略与 @property 修饰符对应

| Associated Object 策略 | 对应 @property | 适用场景 |
|-----------------------|---------------|---------|
| `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | strong, nonatomic | UIImage、UIView 等对象 |
| `OBJC_ASSOCIATION_COPY_NONATOMIC` | copy, nonatomic | NSString |
| `OBJC_ASSOCIATION_ASSIGN` | assign | 基本类型/枚举 |
| `OBJC_ASSOCIATION_RETAIN` | strong, atomic | 极少用 |

### 完整数据流（以 image 为例）

```
box.image = [UIImage imageNamed:@"test.jpg"]
    ↓ 编译器翻译
[box setImage:图片指针]
    ↓ 进 setImage:
① objc_setAssociatedObject → 存图片数据
② self.displayImageView → 调懒加载 getter
    ↓ 进 getter
    ③ objc_getAssociatedObject → 查有没有现成的 UIImageView
    ④ nil（第一次）→ alloc init 创建 → 设属性 → addSubview → Masonry → objc_set 存起
    ⑤ 非 nil（之后）→ 直接返回
    ↓ 回到 setImage:
⑥ iv.image = 图片 → hidden = NO → 屏幕显示
```

### UIView+Display.h

```objc
#import <UIKit/UIKit.h>

@interface UIView (Display)

@property (nonatomic, strong, nullable) UIImage *image;
@property (nonatomic, copy, nullable) NSString *text;
@property (nonatomic, strong, readonly) UIImageView *displayImageView;
@property (nonatomic, strong, readonly) UILabel *displayTextLabel;

@end
```

### UIView+Display.m

```objc
#import "UIView+Display.h"
#import <objc/runtime.h>
#import <Masonry/Masonry.h>

static const void *kImageKey      = &kImageKey;
static const void *kTextKey       = &kTextKey;
static const void *kImageViewKey  = &kImageViewKey;
static const void *kTextLabelKey  = &kTextLabelKey;

@implementation UIView (Display)

// ─── image setter ───
- (void)setImage:(UIImage *)image {
    objc_setAssociatedObject(self, kImageKey, image, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    if (image) {
        self.displayImageView.image = image;
        self.displayImageView.hidden = NO;
    } else {
        self.displayImageView.hidden = YES;
    }
}

// ─── image getter ───
- (UIImage *)image {
    return objc_getAssociatedObject(self, kImageKey);
}

// ─── text setter ───
- (void)setText:(NSString *)text {
    objc_setAssociatedObject(self, kTextKey, text, OBJC_ASSOCIATION_COPY_NONATOMIC);
    if (text.length > 0) {
        self.displayTextLabel.text = text;
        self.displayTextLabel.hidden = NO;
    } else {
        self.displayTextLabel.hidden = YES;
    }
}

// ─── text getter ───
- (NSString *)text {
    return objc_getAssociatedObject(self, kTextKey);
}

// ─── 懒加载：UIImageView ───
- (UIImageView *)displayImageView {
    UIImageView *iv = objc_getAssociatedObject(self, kImageViewKey);
    if (!iv) {
        iv = [[UIImageView alloc] init];
        iv.contentMode = UIViewContentModeScaleAspectFill;
        iv.clipsToBounds = YES;
        iv.hidden = YES;
        [self addSubview:iv];
        [iv mas_makeConstraints:^(MASConstraintMaker *make) {
            make.edges.equalTo(self);
        }];
        objc_setAssociatedObject(self, kImageViewKey, iv, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return iv;
}

// ─── 懒加载：UILabel ───
- (UILabel *)displayTextLabel {
    UILabel *lb = objc_getAssociatedObject(self, kTextLabelKey);
    if (!lb) {
        lb = [[UILabel alloc] init];
        lb.font = [UIFont systemFontOfSize:14];
        lb.textColor = [UIColor whiteColor];
        lb.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5];
        lb.textAlignment = NSTextAlignmentCenter;
        lb.hidden = YES;
        [self addSubview:lb];
        [lb mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.right.bottom.equalTo(self);
            make.height.mas_equalTo(30);
        }];
        objc_setAssociatedObject(self, kTextLabelKey, lb, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return lb;
}

@end
```

### 调用方式

```objc
UIView *box = [[UIView alloc] init];
box.image = [UIImage imageNamed:@"test.jpg"];  // 显示图片
box.text  = @"进度：45%";                        // 底部显示文字
```

### 为什么 `hidden = NO` 是显示？

`hidden` 字面意思 = "隐藏"。`hidden = YES` = 藏起来，`hidden = NO` = 不藏（显示）。和直觉反着来。

### 为什么默认 `hidden = YES`？

displayImageView 创建时不保证一定有图。如果默认不隐藏，屏幕上会显示一个空白方块。先藏起来，有图再显示。

### 懒加载 getter 的意义

第一次调 → 关联表空 → 创建 → 存入关联表。之后每次调 → 关联表有 → 直接返回。子视图只创建一次，后续只更新内容（设图、设文字），不重建。这也是题目要求的"考虑性能问题"。

### 为什么 `readonly`？

防止外部替换内部子视图。外部可以拿 `displayImageView` 设图，但不能把整个 UIImageView 替换掉——替换会导致旧视图还在视图树上但新视图没加上，屏幕乱掉。

---

## 四、继承方式对照

### 为什么需要对照

分类 + Associated Objects 的心智负担远大于继承。用继承写一遍（和 UserCell 一样的套路），再对照分类版，才能看清每行 Runtime 代码在模拟什么。

### DisplayView.h（继承版）

```objc
@interface DisplayView : UIView
@property (nonatomic, strong) UIImage *image;
@property (nonatomic, copy) NSString *text;
@end
```

三行收工——`@property` 自动生成 `_image`、setter、getter。

### DisplayView.m（继承版）

```objc
@interface DisplayView ()
@property (nonatomic, strong) UIImageView *imageView;
@property (nonatomic, strong) UILabel *textLabel;
@end

@implementation DisplayView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) { [self setupSubviews]; }
    return self;
}

- (void)setupSubviews {
    _imageView = [[UIImageView alloc] init];
    _imageView.contentMode = UIViewContentModeScaleAspectFill;
    _imageView.clipsToBounds = YES;
    _imageView.hidden = YES;
    [self addSubview:_imageView];
    [_imageView mas_makeConstraints:^(MASConstraintMaker *make) { make.edges.equalTo(self); }];

    _textLabel = [[UILabel alloc] init];
    _textLabel.font = [UIFont systemFontOfSize:14];
    _textLabel.textColor = [UIColor whiteColor];
    _textLabel.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5];
    _textLabel.textAlignment = NSTextAlignmentCenter;
    _textLabel.hidden = YES;
    [self addSubview:_textLabel];
    [_textLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.right.bottom.equalTo(self);
        make.height.mas_equalTo(30);
    }];
}

- (void)setImage:(UIImage *)image {
    _image = image;
    if (image) {
        _imageView.image = image;
        _imageView.hidden = NO;
    } else {
        _imageView.hidden = YES;
    }
}

- (void)setText:(NSString *)text {
    _text = text;
    if (text.length > 0) {
        _textLabel.text = text;
        _textLabel.hidden = NO;
    } else {
        _textLabel.hidden = YES;
    }
}

@end
```

### 继承 vs 分类 逐行对照

| 干了什么事 | 继承版（简单） | 分类版（绕路） |
|-----------|-------------|-------------|
| 存图片 | `_image = image`（直接写变量） | `objc_setAssociatedObject(...)` |
| 取图片 | `return _image`（编译器生成） | `objc_getAssociatedObject(...)` |
| 创建子视图 | `init` 里一次建完 | 懒加载 getter 里 `if (!iv)` 创建 |
| 复用子视图 | 不管（init 只跑一次） | `if (!iv)` 保证只创建一次 |
| 声明属性 | `@property` 三行全自动 | `@property` 只给声明，getter/setter 手写 |

### 什么时候用哪种

| 场景 | 用什么 |
|------|--------|
| 自己新建的类 | 继承（简单直接） |
| 给 UIView、NSString 等系统类加能力 | 分类 + Associated Objects（唯一选择） |

---

## 五、知识模块详解

### 5.1 Category（分类）的本质与限制

- Category 在不修改原类代码的情况下给已有类添加新方法
- **核心限制：不能加成员变量** → 对象内存布局编译时锁定，运行时加载的分类挤不进去
- 分类中写 `@property` 只声明 getter/setter，不生成 `_成员变量`，不自动实现

### 5.2 Associated Objects 工作机制

- Runtime 维护全局哈希表：key = 对象指针 + 关联 key，value = 关联的值
- 一个宿主对象上每种不同的 key 只能存一个值
- 存储策略与 `@property` 修饰符一一对应
- key 推荐用静态变量地址（`&kVar`）——全局唯一、零额外内存、编译期确定

### 5.3 懒加载模式

```objc
// 标准懒加载模板
- (某类型 *)某个属性 {
    某类型 *obj = objc_getAssociatedObject(self, &key);
    if (!obj) {
        obj = [[某类型 alloc] init];
        // ... 初始化配置 ...
        objc_setAssociatedObject(self, &key, obj, 策略);
    }
    return obj;
}
```

### 5.4 UIView 家族继承链

```
UIView（祖宗：位置、尺寸、层级）
  ├── UILabel
  ├── UIImageView
  ├── UIButton
  ├── UITextField
  ├── UIScrollView → UITableView / UICollectionView / UITextView
  ├── UIProgressView / UISlider / UISwitch
  └── UIStackView
```

UIImage 不在此列——继承 NSObject，是数据模型，不是视图。

### 5.5 Masonry 常用写法速查

| 写法 | 含义 |
|------|------|
| `make.left.equalTo(父).offset(15)` | 左边 = 父左边 + 15 |
| `make.right.equalTo(父).offset(-15)` | 右边 = 父右边 - 15（负值往里缩） |
| `make.centerY.equalTo(父)` | 垂直居中 |
| `make.edges.equalTo(父)` | 上下左右全撑满 |
| `make.center.equalTo(父)` | 水平+垂直双居中 |
| `make.size.mas_equalTo(CGSizeMake(w, h))` | 同时设宽高 |
| `make.width.mas_equalTo(60)` | 固定宽（数字用 mas_equalTo） |
| `.insets(UIEdgeInsetsMake(t, l, b, r))` | 四边距 |
| `mas_equalTo(数字)` | 接数字用这个 |
| `equalTo(视图属性)` | 接视图用这个 |

---

## 六、问答索引

### 6.1 任务 1 问答

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | 为什么写成类方法（`+`）而不是实例方法？ | 不需要已有 NSString 实例 |
| 2 | `componentsJoinedByString:` 是系统方法吗？ | NSArray 系统方法 |
| 3 | `[NSMutableArray array]` 是什么方法？ | 工厂方法（类方法） |
| 4 | `(Concatenation)` 就是分类吗？ | Category 命名 |
| 5 | 别人怎么用我写的分类？ | `#import` 头文件即可 |

### 6.2 任务 2 问答

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | Category 为什么不能直接 `@property`？ | 内存布局编译时锁定 |
| 2 | `objc_set` 和 `objc_get` 各自做什么？ | Runtime 关联表存取 |
| 3 | 每一步谁调的谁？数据流怎么走？ | setImage → getter → 懒加载 → 创建 → 显示 |
| 4 | UIImage、UIImageView、UIView 三者区别？ | 数据模型 vs 显示控件 vs 容器 |
| 5 | `hidden = NO` 为什么是显示？ | hidden 字面意思 = 隐藏 |
| 6 | 为什么默认 `hidden = YES`？ | 防空白方块 |
| 7 | `self.displayImageView.image` 一个点语法走了几步？ | getter → UIImageView → 设图 |
| 8 | 懒加载 getter 的 `if (!iv)` 做什么？ | 首次创建，之后复用 |
| 9 | `readonly` 有什么保护作用？ | 防外部替换子视图 |
| 10 | `addSubview` 里的 `self` 是谁？ | UIView 本身（不是 ViewController） |
| 11 | OC 视图体系家族有哪些？ | UIView 继承链 |
| 12 | `colorWithAlphaComponent:0.5` 什么意思？ | 半透明度 |

### 6.3 继承方式问答

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | 继承版和分类版哪个更简单？ | 继承一行 `@property` |
| 2 | 任务 4 为什么不能用继承？ | 题目要求使用分类 |
| 3 | `initWithFrame:` 谁调的？和 `init` 的关系？ | UIView 指定初始化器 |
| 4 | 系统怎么知道我们重写了 `initWithFrame:`？ | Runtime 方法表覆写 |

### 6.4 Masonry 问答

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | `mas_equalTo` vs `equalTo` 区别？ | 数字用 mas_，视图用 equalTo |
| 2 | `.edges` 快捷写法？ | top+left+bottom+right |
| 3 | `.center` 快捷写法？ | centerX+centerY |
| 4 | `.insets` 什么时候用？ | 四边距 |
| 5 | Masonry 约束是立即执行还是等回调？ | 立即执行（不等系统） |

---

## 七、任务 3 — 300 子线程 + 三种锁

### 题目要求

在 300 个子线程中，将 UIImage 转成 NSData，打印线程编号 + 图片体积（KB，两位小数），并将字符串加到可变数组。最后在主线程打印数组。分别用 `@synchronized`、`NSLock`、`dispatch_semaphore_t` 实现。

### 7.1 为什么需要锁

300 个线程同时往一个 `NSMutableArray` 里 `addObject:` 会导致数据覆盖或崩溃。`addObject:` 不是原子操作——内部"读长度→扩内存→写元素→更新长度"可能被多个线程同时打断。锁保证同一时刻只有一个人在操作数组。

### 7.2 方案一：`@synchronized`

```objc
- (void)runSynchronizedDemo {
    UIImage *image = [UIImage imageNamed:@"test"];
    NSMutableArray *results = [NSMutableArray array];
    dispatch_group_t group = dispatch_group_create();

    for (int i = 0; i < 300; i++) {
        dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
            NSData *data = UIImageJPEGRepresentation(image, 1.0);
            double sizeKB = data.length / 1024.0;
            NSString *str = [NSString stringWithFormat:@"线程%d：%.2fKB", i + 1, sizeKB];

            @synchronized(results) {
                [results addObject:str];
            }
        });
    }

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        for (NSString *s in results) { NSLog(@"%@", s); }
        NSLog(@"总计 %lu 条", (unsigned long)results.count);
    });
}
```

### 7.3 方案二：`NSLock`

和 `@synchronized` 一样的效果，但需要手动 `lock`/`unlock`。忘记 `unlock` 会导致死锁。

```objc
NSLock *lock = [[NSLock alloc] init];
// Block 里：
[lock lock];
[results addObject:str];
[lock unlock];
```

### 7.4 方案三：`dispatch_semaphore_t`（信号量）

信号量内部维护一个计数器。`create(1)` 设初始值为 1，`wait` 减 1（为 0 时后来者等待），`signal` 加 1（唤醒等待者）。

```objc
dispatch_semaphore_t sem = dispatch_semaphore_create(1);  // 只允许 1 个线程进入
// Block 里：
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
[results addObject:str];
dispatch_semaphore_signal(sem);
```

信号量不止能做锁——`create(3)` 就可以让 3 个线程同时进入，控制并发数。

### 7.5 三种锁对比

| | @synchronized | NSLock | dispatch_semaphore |
|------|-------------|--------|-------------------|
| 写法 | `@synchronized(obj) {}` | `[lock lock]` / `[lock unlock]` | `wait` / `signal` |
| 解锁 | 自动（出 `}` 即解） | 手动 | 手动 |
| 风险 | 无 | 忘 unlock 会死锁 | 计数值设错会错乱 |
| 额外能力 | 无 | 有 tryLock（锁不上就跳过） | 控制并发数（不止 1） |

---

## 八、GCD 体系速查

### 8.1 GCD 是什么

Grand Central Dispatch — 苹果的多线程框架。你只关心"丢什么任务"，不关心"哪个线程跑它"。和 NSThread（手动管理线程生命周期）的对比：GCD 自动分配线程，你只管 Block。

### 8.2 GCD 核心函数族

| 函数 | 作用 | 今天用了吗 |
|------|------|-----------|
| `dispatch_async(队列, Block)` | 往队列丢任务，不等结果直接走（异步） | ✅ 网络回调切主线程 |
| `dispatch_sync(队列, Block)` | 往队列丢任务，干完才走（同步） | ❌ |
| `dispatch_get_main_queue()` | 获取主线程队列 | ✅ |
| `dispatch_get_global_queue(0, 0)` | 获取子线程队列（第一个0=优先级，第二个0=保留字段） | ✅ |
| `dispatch_group_create()` | 建一个任务组 | ✅ |
| `dispatch_group_async(组, 队列, Block)` | 往组里扔任务（进场+1，出场-1） | ✅ |
| `dispatch_group_notify(组, 主线程, Block)` | 组里全完成时回调 | ✅ |
| `dispatch_semaphore_create(n)` | 建信号量（计数器 = n） | ✅ |
| `dispatch_semaphore_wait(sem, FOREVER)` | 计数器 -1（为 0 则等待） | ✅ |
| `dispatch_semaphore_signal(sem)` | 计数器 +1 | ✅ |
| `dispatch_once(&token, Block)` | 只执行一次（单例用） | 上周 NetworkManager |

### 8.3 `dispatch_group` 工作机制

不是 for 循环结束就认为完成——系统在 Block 执行到最后一个 `}` 时自动增减组内计数器。`dispatch_group_notify` 等待计数器归零才执行回调。

### 8.4 异步 vs 同步

| | 异步（async） | 同步（sync） |
|------|-------------|-----------|
| 丢完任务后 | 继续往下走，不等结果 | 等着，干完才往下走 |
| GCD 函数 | `dispatch_async` | `dispatch_sync` |
| 本周使用 | 全部场景 | 未使用 |

### 8.5 `dispatch_group_t` 为什么没有 `*`

`*` 被藏在 typedef 里——`typedef dispatch_group_s *dispatch_group_t`。类型名自带指针。`dispatch_queue_t`、`dispatch_semaphore_t` 同理。

---

## 九、C 函数 vs OC 方法

### 9.1 区分标准

| 信号 | C 函数 | OC 方法 |
|------|--------|---------|
| 调用语法 | `func(a, b)` 小括号 | `[obj method:a]` 方括号 |
| 有没有对象在前 | 无 | 有（`self`、`dict`、`arr`） |
| 命名风格 | 驼峰大写开头 | 驼峰小写开头 |

### 9.2 常见 C 函数一览

| C 函数 | 作用 |
|--------|------|
| `UIImageJPEGRepresentation(image, 质量)` | UIImage→NSData（JPEG） |
| `UIImagePNGRepresentation(image)` | UIImage→NSData（PNG） |
| `CGRectMake(x, y, w, h)` | 创建坐标+尺寸结构体 |
| `NSLog(@"...")` | 打印日志 |
| `objc_setAssociatedObject(...)` | 关联对象存值 |
| `objc_getAssociatedObject(...)` | 关联对象取值 |

### 9.3 常见 NSData 转换方法

| 方法 | 写法 |
|------|------|
| 图片→JPEG | `UIImageJPEGRepresentation(image, 1.0)` |
| 图片→PNG | `UIImagePNGRepresentation(image)` |
| 本地文件→NSData | `[NSData dataWithContentsOfFile:path]` |
| 字符串→NSData | `[str dataUsingEncoding:NSUTF8StringEncoding]` |
| NSData→文件 | `[data writeToFile:path atomically:YES]` |
| NSData→UIImage | `[UIImage imageWithData:data]` |

---

## 十、问答索引（任务 3）

### 10.1 锁相关

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | 为什么要加锁？300 个线程怎么冲突的？ | `addObject:` 非原子操作，多线程覆盖 |
| 2 | `@synchronized` 和 `NSLock` 效果一样吗？ | 完全一样，只是写法不同 |
| 3 | 数组顺序是 1→2→3 吗？ | 不是——谁先跑到锁谁先进，顺序随机 |
| 4 | `[lock lock]` / `[lock unlock]` 是怎么把中间锁住的？ | 门闩逻辑——lock 闩门，unlock 开门 |
| 5 | 信号量的 `wait` 是 "先放行再减" 还是 "先减再放行"？ | 先看计数器，有证就减并放行，没证就等 |

### 10.2 GCD 相关

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | GCD 是什么？ | 苹果多线程框架，你丢任务它分配线程 |
| 2 | `dispatch_group_create()` 是固定写法吗？ | 固定模板：建组→丢任务→等通知 |
| 3 | `dispatch_get_global_queue(0, 0)` 两个 0 什么意思？ | 第一个=优先级，第二个=保留字段 |
| 4 | 系统怎么知道组里任务跑完了？ | Block 进出自动增减内部计数器 |
| 5 | 为什么子线程用 `group_async`，主线程用 `group_notify`？ | async=丢任务干活，notify=等通知 |
| 6 | 异步（async）和同步（sync）的区别？ | async 不等结果，sync 等 |
| 7 | `dispatch_group_t` 为什么没有 `*`？ | typedef 把 `*` 藏在类型名里 |
| 8 | `DISPATCH_TIME_FOREVER` 是什么？ | 常量 = 永远等，等不到信号就不走 |
| 9 | GCD 还能建什么？ | group、semaphore、自定义 queue、timer 等 |

### 10.3 C 函数 / 数据转换

| 序号 | 问题 | 知识点 |
|------|------|--------|
| 1 | `UIImageJPEGRepresentation` 为什么不用方括号？ | C 函数，不属于任何类 |
| 2 | `1.0` 参数什么意思？ | JPEG 压缩质量 0.0~1.0 |
| 3 | 有哪些文件格式转 NSData 的方法？ | JPEG/PNG/文件/字符串/JSON 全部可转 |
| 4 | 怎么判断用 C 函数还是 OC 方法？ | 看小括号还是方括号 |
