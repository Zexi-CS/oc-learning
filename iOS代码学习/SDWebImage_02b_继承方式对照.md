# 02b — 继承方式实现 UIView 显示图片和文字（对照版）

> 就是写一个 UIView 的子类——和 User、UserCell 一样的套路，没有任何 Runtime 黑魔法。
> 先看懂这个版本，再回头用分类 + Associated Objects 把它翻译一遍。

---

## 一、DisplayView.h

```objc
//
//  DisplayView.h
//  继承方式：直接 @property，编译器帮你生成成员变量

#import <UIKit/UIKit.h>

@interface DisplayView : UIView

@property (nonatomic, strong) UIImage *image;
@property (nonatomic, copy)   NSString *text;

/// 内部 UIImageView（公开只读，供 SDWebImage 下载用）
/// 因为 SDWebImage 的 sd_setImageWithURL: 必须对着 UIImageView 调，
/// 而 _imageView 原本是私有的，所以暴露出来给外面（如 ImageCell）拿去做下载
@property (nonatomic, strong, readonly) UIImageView *imageView;

@end
```

三行完事——和 User 类的头文件一模一样。

---

## 二、DisplayView.m

```objc
//
//  DisplayView.m
//

#import "DisplayView.h"
#import <Masonry/Masonry.h>

@interface DisplayView ()

// .h 里声明了 readonly imageView，类扩展里用 readwrite 覆盖（允许内部写、外部只读）
// 这样外部能拿 _imageView 做 SDWebImage 下载，但改不了属主
@property (nonatomic, strong, readwrite) UIImageView *imageView;  // 内部 UIImageView
@property (nonatomic, strong) UILabel     *textLabel;   // 内部 UILabel

@end

@implementation DisplayView

// ============================================================
// 初始化：创建子视图（只建一次）
// ============================================================
- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupSubviews];
    }
    return self;
}

- (void)setupSubviews {
    // ---- UIImageView ----
    _imageView = [[UIImageView alloc] init];
    _imageView.contentMode = UIViewContentModeScaleAspectFill;
    _imageView.clipsToBounds = YES;
    _imageView.hidden = YES;
    [self addSubview:_imageView];

    [_imageView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self);
    }];

    // ---- UILabel ----
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

// ============================================================
// 重写 image setter（外面设置图片时触发）
// ============================================================
- (void)setImage:(UIImage *)image {
    _image = image;                      // 存图片数据

    if (image) {
        _imageView.image = image;        // 设到 ImageView 上
        _imageView.hidden = NO;          // 显示
    } else {
        _imageView.hidden = YES;         // 隐藏
    }
}

// ============================================================
// 重写 text setter
// ============================================================
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

---

## 三、调用方式

```objc
DisplayView *box = [[DisplayView alloc] init];
box.image = [UIImage imageNamed:@"test.jpg"];  // 显示图片
box.text  = @"这是一段文字";                    // 底部显示文字
```

---

## 四、和分类版本的对比

| | 继承版 | 分类版 |
|------|--------|--------|
| 属性声明 | `@property` 自动生成成员变量和 getter/setter | `@property` 只生成声明，成员变量和 setter/getter 要手写 |
| 存值 | `_image = image`（直接写成员变量） | `objc_setAssociatedObject(self, kKey, image, ...)` |
| 取值 | `return _image` | `objc_getAssociatedObject(self, kKey)` |
| 子视图创建 | `init` 里一次建完 | 懒加载 getter 里创建 |
| 子视图复用 | 不用管，init 只跑一次 | 必须在懒加载里判断 `if (!iv) 创建` |
| 代码行数 | ~50 行 | ~90 行 |
| 适用场景 | 自己新建的类 | **给 UIView、NSString 等系统类加能力** |

---

## 五、怎么从这个翻译到分类版

把继承版拆成三块，逐块翻译：

| 继承版 | 翻译成分类版 |
|--------|-------------|
| `@property image` 自动生成 `_image` | 删掉自动生成 → `setImage:` 里用 `objc_setAssociatedObject` 存 |
| `init` 里 `_imageView = [[UIImageView alloc] init]` | 移到 `displayImageView` 懒加载 getter 里 |
| `_imageView` 是私有属性 | `displayImageView` 是公开 readonly 属性 + 懒加载 getter |

**核心不变：** 都是存一张图片 → 创建一个 UIImageView → 把图设上去。只是"存"和"创建"的方式换了写法而已。
