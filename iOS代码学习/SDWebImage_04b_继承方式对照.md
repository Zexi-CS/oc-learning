# 04b — 任务 4 继承方式对照（UIView 继承版）

> 图片和文字都用 **继承 UIView 的 DisplayView** 来显示（02b 那个）——完全符合题目"用 UIView 的继承"。
> ImageCell 里放两个 DisplayView（一个管图、一个管字），SDWebImage 下载对着 `imageView`（DisplayView 公开的只读属性）进行。

---

## 文件结构

```
├── View/
│   ├── DisplayView.h/.m      ← 继承 UIView，有 image + text + 公开 imageView（02b 那个）
│   └── ImageCell.h/.m        ← 继承 UITableViewCell，里面放两个 DisplayView
```

---

## DisplayView.h（02b 的，多了一行公开 imageView）

```objc
#import <UIKit/UIKit.h>

@interface DisplayView : UIView

@property (nonatomic, strong, nullable) UIImage *image;
@property (nonatomic, copy,   nullable) NSString *text;

/// 内部 UIImageView（公开只读）——SDWebImage 下载必须对着 UIImageView 调
@property (nonatomic, strong, readonly) UIImageView *imageView;

@end
```

## DisplayView.m（02b 的，只在类扩展里把 imageView 声明成 readwrite）

```objc
#import "DisplayView.h"
#import <Masonry/Masonry.h>

@interface DisplayView ()
@property (nonatomic, strong, readwrite) UIImageView *imageView;  // readwrite 覆盖 .h 的 readonly
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
    [_imageView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self);
    }];

    _textLabel = [[UILabel alloc] init];
    _textLabel.font = [UIFont systemFontOfSize:14];
    _textLabel.textColor = [UIColor whiteColor];
    _textLabel.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5];
    _textLabel.textAlignment = NSTextAlignmentCenter;
    _textLabel.hidden = YES;
    [self addSubview:_textLabel];
    [_textLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.center.equalTo(self);
        make.height.mas_equalTo(30);
    }];
}

- (void)setImage:(UIImage *)image {
    _image = image;
    if (image) { _imageView.image = image; _imageView.hidden = NO; }
    else { _imageView.hidden = YES; }
}

- (void)setText:(NSString *)text {
    _text = text;
    if (text.length > 0) { _textLabel.text = text; _textLabel.hidden = NO; }
    else { _textLabel.hidden = YES; }
}
@end
```

## ImageCell.m（改这三个方法，核心差异）

```objc
#import "ImageCell.h"
#import "ImageItem.h"
#import "DisplayView.h"              // ← 用 DisplayView（继承 UIView）
#import "NSString+Concatenation.h"
#import <Masonry/Masonry.h>
#import <SDWebImage/SDWebImage.h>

@interface ImageCell ()
@property (nonatomic, strong) DisplayView *imageArea;   // ← DisplayView，管图
@property (nonatomic, strong) DisplayView *textArea;    // ← DisplayView，管字
@end

@implementation ImageCell

+ (CGFloat)cellHeight { return 100; }

- (instancetype)initWithStyle:(UITableViewCellStyle)style
              reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) { [self setupSubviews]; }
    return self;
}

- (void)setupSubviews {
    // ---- 左侧：图片 DisplayView（80x80，继承 UIView）----
    self.imageArea = [[DisplayView alloc] init];       // ← DisplayView，不是裸 UIView
    self.imageArea.backgroundColor = [UIColor lightGrayColor];
    self.imageArea.clipsToBounds = YES;
    [self.contentView addSubview:self.imageArea];
    [self.imageArea mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.contentView).offset(10);
        make.centerY.equalTo(self.contentView);
        make.size.mas_equalTo(CGSizeMake(80, 80));
    }];

    // ---- 右侧：文字 DisplayView（继承 UIView）----
    self.textArea = [[DisplayView alloc] init];
    self.textArea.backgroundColor = [UIColor clearColor];
    [self.contentView addSubview:self.textArea];
    [self.textArea mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.imageArea.mas_right).offset(15);
        make.centerY.equalTo(self.contentView);
        make.right.equalTo(self.contentView).offset(-15);
        make.height.mas_equalTo(60);
    }];
}

- (void)configureWithItem:(ImageItem *)item {
    NSURL *url = [NSURL URLWithString:item.imageURL];
    if (!url) return;

    // 下载对着 DisplayView 公开的 imageView 调
    [self.imageArea.imageView sd_setImageWithURL:url
        placeholderImage:nil
        options:(SDWebImageAvoidAutoSetImage | SDWebImageRefreshCached)
        progress:^(NSInteger received, NSInteger expected, NSURL *targetURL) {
            double receivedKB = received / 1024.0;
            double expectedKB = expected / 1024.0;
            double percent = (expected > 0) ? (received * 100.0 / expected) : 0;

            NSString *part = [NSString concat3:
                [NSString stringWithFormat:@"%.2fKB", receivedKB]
                _:[NSString stringWithFormat:@"%.2fKB", expectedKB]
                __:[NSString stringWithFormat:@"%.2f%%", percent]];

            NSString *text = [NSString stringWithFormat:@"%ld_%@", (long)item.index, part];

            dispatch_async(dispatch_get_main_queue(), ^{
                self.textArea.text = text;             // ← 走 DisplayView 的 text
            });
        }
        completed:^(UIImage *image, NSError *error, SDImageCacheType type, NSURL *url) {
            if (image) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.imageArea.image = image;      // ← 走 DisplayView 的 image
                });
            }
        }];
}

- (void)prepareForReuse {
    [super prepareForReuse];
    [self.imageArea.imageView sd_cancelCurrentImageLoad];   // ← 用公开 imageView 取消下载
    self.imageArea.image = nil;
    self.textArea.text = nil;
}

@end
```

---

## 核心对比（分类版 04 vs 继承版 04b）

```objc
// 分类版（04）                             // 继承版（04b）
self.imageArea = [[UIView alloc] init];     self.imageArea = [[DisplayView alloc] init];
self.textArea  = [[UIView alloc] init];     self.textArea  = [[DisplayView alloc] init];
// 图片需要绕 displayImageView：
self.imageArea.displayImageView             self.imageArea.imageView   // DisplayView 公开的 readonly
// 文字：
self.textArea.text                          self.textArea.text          // DisplayView 自己的 text
// 图：
self.imageArea.image                        self.imageArea.image        // DisplayView 自己的 image
```

**继承版不需要 UIView+Display 分类**——DisplayView 本身就是"能显示图和文字"的 UIView 子类，`image`/`text` 是它自己的属性，`imageView` 公开出来给 SDWebImage 用。

---

## ImageItem.h（继承版也要加 index）

```objc
@property (nonatomic, assign) NSInteger index;

// ImageItem.m — defaultItems 里：
item.index = i + 1;   // 1, 2, 3...7
```
