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
