# 05 — UserCell 自定义 TableViewCell

> 第五步：创建自定义 Cell，定义列表每一行长什么样。TableView 的"最小单元"。

---

## 一、先理解 Cell 的复用机制（本步最核心的概念）

假设用户列表有 1000 条数据，如果一次创建 1000 个 Cell，内存直接爆炸，界面卡死。iOS 的解决方案：

```
屏幕上一次只能看到约 10 个 Cell：

┌──────────────┐
│  Cell A      │  ← 用户 1     ← 正在屏幕上显示（活的）
│  Cell B      │  ← 用户 2
│  Cell C      │  ← 用户 3
│  ...         │
│  Cell J      │  ← 用户 10
├──────────────┤  ← 屏幕边界（往下滑）
│  回收池       │     Cell K~Cell 1000 根本不存在，不会提前创建
└──────────────┘

往下滑的时候：
  顶部滚出屏幕的 Cell → 扔进回收池（不销毁，只是想重复用）
  底部要出现的 Cell → 从回收池里捞一个，换上新数据
```

这就是**复用机制**：不是 1000 个 Cell，而是**十几个 Cell 换着用**。

---

## 二、UserCell.h

```objc
//
//  UserCell.h
//  自定义 TableViewCell，显示用户 ID、姓名、年龄
//
//  布局：
//  ┌─────────────────────────────────────┐
//  │  ID:1      张三              25岁   │
//  └─────────────────────────────────────┘
//  灰色小字     黑色粗体           蓝色右对齐
//

#import <UIKit/UIKit.h>

@class User;  // 前置声明，不需要 #import "User.h"，加快编译速度

/// Cell 复用标识符（全局常量，外部 .m 文件用 extern 引用）
/// 作用和钥匙一样：注册和复用 Cell 时都要用它来"对暗号"
extern NSString * const kUserCellIdentifier;


@interface UserCell : UITableViewCell

/// 用 User 对象填充 Cell 里的三个 Label
- (void)configureWithUser:(User *)user;

@end
```

### 逐行解释

**`extern NSString * const kUserCellIdentifier;`**

这就是第 7 号知识点。声明一个**全局常量**，所有用到这个 Cell 的地方都引用同一个字符串，避免手写 `@"UserCell"` 时拼错。

**`@class User;` vs `#import "User.h"`**

- `@class User;` — "我声明一下 User 是个类，但我现在不需要知道它里面有什么"（更快）
- `#import "User.h"` — "把整个 User.h 的内容贴过来"（需要知道具体方法时才用）

头文件里能用 `@class` 就用 `@class`，加快编译速度。实现文件里才用 `#import`。

---

## 三、UserCell.m

```objc
//
//  UserCell.m
//

#import "UserCell.h"
#import "User.h"
#import <Masonry/Masonry.h>

// ★ 给全局常量赋实际的值（等于告诉编译器："这个常量就是字符串 'UserCell'"）
NSString * const kUserCellIdentifier = @"UserCell";


@interface UserCell ()

// 三个 Label，在 .m 的私有扩展里声明，外部看不到
@property (nonatomic, strong) UILabel *idLabel;    // 左侧：用户 ID（灰色小字）
@property (nonatomic, strong) UILabel *nameLabel;  // 中间：用户名称（黑色粗体）
@property (nonatomic, strong) UILabel *ageLabel;   // 右侧：年龄（蓝色字）

@end


@implementation UserCell

// ============================================================
// 初始化方法 — 重写 UITableViewCell 的指定初始化器
// ============================================================
- (instancetype)initWithStyle:(UITableViewCellStyle)style
              reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self setupSubviews];  // 创建并布局三个 Label
    }
    return self;
}

// ============================================================
// 创建子视图（三个 Label）+ Masonry 布局
// ============================================================
- (void)setupSubviews {

    // ---- ID 标签（左侧，灰色小字）----
    self.idLabel = [[UILabel alloc] init];
    self.idLabel.font = [UIFont systemFontOfSize:12];       // 字号 12
    self.idLabel.textColor = [UIColor grayColor];           // 灰色
    [self.contentView addSubview:self.idLabel];

    [self.idLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.contentView).offset(15);          // 左边距 15
        make.centerY.equalTo(self.contentView);                  // 垂直居中
        make.width.mas_equalTo(60);                              // 宽 60
        make.height.mas_equalTo(40);                             // 高 40
    }];

    // ---- 名称标签（中间，黑色粗体）----
    self.nameLabel = [[UILabel alloc] init];
    self.nameLabel.font = [UIFont boldSystemFontOfSize:17]; // 粗体 17
    self.nameLabel.textColor = [UIColor blackColor];
    [self.contentView addSubview:self.nameLabel];

    [self.nameLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.idLabel.mas_right).offset(10);   // 距离 idLabel 右边 10
        make.centerY.equalTo(self.contentView);                  // 垂直居中
        make.right.equalTo(self.ageLabel.mas_left).offset(-10);  // 距离 ageLabel 左边 10
    }];

    // ---- 年龄标签（右侧，蓝色右对齐）----
    self.ageLabel = [[UILabel alloc] init];
    self.ageLabel.font = [UIFont systemFontOfSize:15];
    self.ageLabel.textColor = [UIColor systemBlueColor];
    self.ageLabel.textAlignment = NSTextAlignmentRight;     // 文字靠右
    [self.contentView addSubview:self.ageLabel];

    [self.ageLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.right.equalTo(self.contentView).offset(-15);        // 右边距 15
        make.centerY.equalTo(self.contentView);                  // 垂直居中
        make.width.mas_equalTo(50);                              // 宽 50
    }];
}

// ============================================================
// 填充数据 — 由外部列表页面调用
// ============================================================
- (void)configureWithUser:(User *)user {
    self.idLabel.text   = [NSString stringWithFormat:@"ID:%@", user.userId];
    self.nameLabel.text = user.name;
    self.ageLabel.text  = [NSString stringWithFormat:@"%@岁", user.age];
}

@end
```

---

## 四、Masonry 布局详解（本步第二大新知识点）

Masonry 是 OC 主流的第三方自动布局库，用 Block 链式调用描述视图位置关系，比 VFL 更直观。

```
视图关系图：

  左边距   ID标签   间距   姓名标签    间距   年龄标签   右边距
   15     宽60     10    自动宽度    10     宽50      15
  ┌───┐ ┌──────┐ ┌───┐ ┌────────┐ ┌───┐ ┌──────┐ ┌───┐
  │   │ │ID:1  │ │   │ │ 张三   │ │   │ │25岁  │ │   │
  └───┘ └──────┘ └───┘ └────────┘ └───┘ └──────┘ └───┘
```

三个 Label 各自的 Masonry 约束：

```objc
// idLabel：左边距 15，宽 60，垂直居中
[self.idLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    make.left.equalTo(self.contentView).offset(15);    // .left → 左边缘
    make.centerY.equalTo(self.contentView);             // .centerY → 垂直中心
    make.width.mas_equalTo(60);                        // .width → 宽度
}];

// nameLabel：夹在 idLabel 和 ageLabel 中间，自动撑满
[self.nameLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    make.left.equalTo(self.idLabel.mas_right).offset(10);      // 左贴 idLabel 右边 +10
    make.right.equalTo(self.ageLabel.mas_left).offset(-10);    // 右贴 ageLabel 左边 -10
    make.centerY.equalTo(self.contentView);                     // 垂直居中
}];

// ageLabel：右边距 15，宽 50
[self.ageLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    make.right.equalTo(self.contentView).offset(-15);   // .right → 右边缘（负值 = 内缩）
    make.centerY.equalTo(self.contentView);
    make.width.mas_equalTo(50);
}];
```

**关键技术点：**

- Masonry 自动关掉 `translatesAutoresizingMaskIntoConstraints`，不用手写
- `mas_equalTo` 接收数字，`equalTo` 接收对象或另一个视图的属性
- `.mas_right` / `.mas_left` 是 Masonry 为 UIView 扩展的属性，等价于系统的左右边缘
- `offset(-15)` 表示往左缩进 15（从右边缘往左走）
- `nameLabel` 左右都被固定，宽度自动计算

**Masonry vs VFL 对比：**

| | VFL | Masonry |
|------|-----|---------|
| 写法 | 字符串 `@"H:|-15-[v(60)]"` | 链式 `.left.offset(15).width(60)` |
| 可读性 | 需要背语法 | 一眼看懂 |
| 表达能力 | 有限（无法表达 >、< 复杂约束） | 完整 |
| 安装 | 系统自带 | 需要 CocoaPods 或 SPM 安装 |
| 暗黑模式 | 不涉及 | 不涉及 |

---

## 五、三个关键注意点

### 1. 为什么加在 `contentView` 上而不是 `self` 上？

```objc
[self.contentView addSubview:self.idLabel];  // ✅ 正确的
[self addSubview:self.idLabel];              // ❌ 位置会偏
```

UITableViewCell 自带一个 `contentView`，是专门给你放内容的容器。Cell 本身还包含了左右滑动的操作按钮区域，直接加在 `self` 上布局会乱。

### 2. 为什么 `initWithStyle:reuseIdentifier:` 而不是普通 `init`？

因为系统创建 Cell 时调用的是这个初始化器，不是 `init`。你重写 `init` 的话代码不会执行。

### 3. 怎么用到列表里？（预告，下一节写）

```objc
// 注册：告诉 TableView "我要用 UserCell 这个类"
[tableView registerClass:[UserCell class] forCellReuseIdentifier:kUserCellIdentifier];

// 复用：问回收池有没有空闲的 Cell
UserCell *cell = [tableView dequeueReusableCellWithIdentifier:kUserCellIdentifier forIndexPath:indexPath];

// 填数据
[cell configureWithUser:user];
```

---

## 六、测试方法

先不连数据，直接在列表页面里写死一个假数据验证布局：

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {

    UserCell *cell = [tableView dequeueReusableCellWithIdentifier:kUserCellIdentifier
                                                     forIndexPath:indexPath];

    // 临时假数据，纯看布局
    User *fakeUser = [[User alloc] initWithDictionary:@{
        @"id": @"99",
        @"name": @"测试用户",
        @"age": @"25"
    }];
    [cell configureWithUser:fakeUser];
    return cell;
}
```

---

## 七、本步骤涉及的知识点

| 编号 | 知识点                             | 在哪里体现                                         |
| -- | ------------------------------- | --------------------------------------------- |
| 21 | 自定义 Cell                        | 继承 UITableViewCell                            |
| 22 | 复用标识符 + registerClass / dequeue | `kUserCellIdentifier` 全局常量                    |
| 7  | extern 全局常量                     | `extern NSString * const kUserCellIdentifier` |
| -- | Masonry 布局                      | `mas_makeConstraints:` 链式调用                   |

**下一步预告：** UserListViewController — 主界面，把 TableView 和 NetworkManager 串在一起。
