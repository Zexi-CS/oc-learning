# 02 — UIView 显示分类（Associated Objects）

> 第二步：让任意 UIView 拥有 `image` 和 `text` 属性，像 UIImageView 和 UILabel 一样显示内容。

---

## 一、题目要求

给 UIView 扩展一个分类：

- `image` 属性：让 UIView 像 UIImageView 那样显示图片
- `text` 属性：让 UIView 像 UILabel 那样显示文字

要求独立完整工程，能正常运行，并考虑性能问题（子视图只创建一次）。

---

## 二、核心问题：Category 不能加成员变量

```objc
// 普通类这样写 → 编译器自动生成 _image 变量
@property (nonatomic, strong) UIImage *image;   // ✅ 在 @interface 里

// Category 里这样写 → 只声明了 getter/setter，没有 _image
@property (nonatomic, strong) UIImage *image;   // ❌ Category 里不会自动生成变量
```

**为什么？** 对象内存布局编译时就定了，Category 是运行时加载的，挤不进去新变量。

**怎么突破？** `objc_setAssociatedObject` / `objc_getAssociatedObject` → Runtime 用全局哈希表给对象绑定外部值。

---

## 三、Associated Objects 怎么用

```objc
// 存：把 image 绑定到 self 上
objc_setAssociatedObject(
    self,                    // 宿主对象（这个 UIView）
    &kImageKey,              // 钥匙（一张卡只开一把锁）
    image,                   // 要存的值
    OBJC_ASSOCIATION_RETAIN_NONATOMIC  // 存储策略（= strong, nonatomic）
);

// 取：用同一把钥匙拿回来
UIImage *img = objc_getAssociatedObject(self, &kImageKey);
```

---

## 四、UIView+Display.h

```objc
//
//  UIView+Display.h
//  给 UIView 扩展 image 和 text 属性
//

#import <UIKit/UIKit.h>

@interface UIView (Display)

/// 设置 / 获取图片。内部创建或复用 UIImageView，图片撑满整个 UIView。
@property (nonatomic, strong, nullable) UIImage *image;

/// 设置 / 获取文字。内部创建或复用 UILabel，定位在 UIView 底部。
@property (nonatomic, copy, nullable) NSString *text;

/// 内部的 UIImageView（只读，供 SDWebImage 等外部库使用）
@property (nonatomic, strong, readonly) UIImageView *displayImageView;

/// 内部的 UILabel（只读）
@property (nonatomic, strong, readonly) UILabel *displayTextLabel;

@end
```

---

## 五、UIView+Display.m

```objc
//
//  UIView+Display.m
//

#import "UIView+Display.h"
#import <objc/runtime.h>   // ★ Associated Objects 需要这个头文件

// ─── key：每把锁一个唯一的钥匙 ───
static const void *kImageKey = &kImageKey;
static const void *kTextKey  = &kTextKey;
static const void *kImageViewKey = &kImageViewKey;
static const void *kTextLabelKey  = &kTextLabelKey;

@implementation UIView (Display)

// ============================================================
// 属性 1：image（Associated Objects 存取）
// ============================================================
- (void)setImage:(UIImage *)image {
    objc_setAssociatedObject(self, kImageKey, image, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

    if (image) {
        self.displayImageView.image = image;   // 有图就设到 ImageView 上
        self.displayImageView.hidden = NO;     // 显示
    } else {
        self.displayImageView.hidden = YES;    // 没图就隐藏
    }
}

- (UIImage *)image {
    return objc_getAssociatedObject(self, kImageKey);
}

// ============================================================
// 属性 2：text
// ============================================================
- (void)setText:(NSString *)text {
    objc_setAssociatedObject(self, kTextKey, text, OBJC_ASSOCIATION_COPY_NONATOMIC);

    if (text.length > 0) {
        self.displayTextLabel.text = text;     // 有文字就设到 Label 上
        self.displayTextLabel.hidden = NO;
    } else {
        self.displayTextLabel.hidden = YES;    // 没文字就隐藏
    }
}

- (NSString *)text {
    return objc_getAssociatedObject(self, kTextKey);
}

// ============================================================
// 内部 UIImageView（懒加载：第一次访问时创建，之后复用）
// ============================================================
- (UIImageView *)displayImageView {
    UIImageView *iv = objc_getAssociatedObject(self, kImageViewKey);
    if (!iv) {
        iv = [[UIImageView alloc] init];
        iv.contentMode = UIViewContentModeScaleAspectFill;  // 等比填满
        iv.clipsToBounds = YES;                              // 裁掉超出部分
        iv.hidden = YES;                                     // 默认隐藏
        [self addSubview:iv];

        // Masonry 撑满整个 UIView
        [iv mas_makeConstraints:^(MASConstraintMaker *make) {
            make.edges.equalTo(self);
        }];

        objc_setAssociatedObject(self, kImageViewKey, iv, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return iv;
}

// ============================================================
// 内部 UILabel（懒加载，放在底部，半透明黑底白字）
// ============================================================
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

        // Masonry：垂直居中，高 30
        [lb mas_makeConstraints:^(MASConstraintMaker *make) {
            make.center.equalTo(self);
            make.height.mas_equalTo(30);
        }];

        objc_setAssociatedObject(self, kTextLabelKey, lb, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return lb;
}

@end
```

---

## 六、调用演示

```objc
#import "UIView+Display.h"

// 创建普通 UIView
UIView *box = [[UIView alloc] init];

// 像 UIImageView 一样设图片
box.image = [UIImage imageNamed:@"test.jpg"];

// 像 UILabel 一样设文字
box.text = @"这是一段文字";
```

---

## 七、性能设计

- `displayImageView` 和 `displayTextLabel` 是**懒加载 getter**：第一次访问时才创建，之后永远返回同一个
- Associated Objects 存的就是这个创建好的子视图，不会重复创建
- Setter 只更新内容（`.image = xxx` / `.text = xxx` / `.hidden = YES/NO`），不重建视图

---

## 八、本步骤涉及的知识点

| 知识点                                                     | 在哪里体现                                              |
| ------------------------------------------------------- | -------------------------------------------------- |
| Category 限制                                             | 不能加成员变量，需要 Associated Objects                      |
| `objc_setAssociatedObject` / `objc_getAssociatedObject` | 存取值                                                |
| 存储策略                                                    | `RETAIN_NONATOMIC`（对象）/ `COPY_NONATOMIC`（NSString） |
| 懒加载 getter                                              | `displayImageView` / `displayTextLabel` 只创建一次      |
| readonly + 内部可写                                         | .h 声明 readonly，外部不能替换                              |
| Masonry 约束                                              | UIImageView 撑满、UILabel 底栏                          |
