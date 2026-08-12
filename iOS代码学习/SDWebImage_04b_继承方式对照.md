# 04b — 任务 4 继承方式对照

> 不用分类——ImageCell 里直接 `@property UIImageView` + `@property UILabel`，和上周 UserCell 一样简单。

---

## 和分类版的区别

| 分类版（04） | 继承版（04b） |
|-------------|-------------|
| `self.imageArea.image = xxx` | `self.displayImageView.image = xxx` |
| `self.imageArea.displayImageView → SDWebImage` | `self.displayImageView → SDWebImage` |
| 需要 UIView+Display 分类 | 零依赖，纯 UIKit |
| Runtime 代码：大量 | Runtime 代码：零 |

---

## ImageCell.m（只改这个文件）

```objc
#import "ImageCell.h"
#import "ImageItem.h"
#import "NSString+Concatenation.h"
#import <Masonry/Masonry.h>
#import <SDWebImage/SDWebImage.h>

@interface ImageCell ()

@property (nonatomic, strong) UIImageView *displayImageView;   // 左侧正方形图片
@property (nonatomic, strong) UILabel     *progressLabel;       // 右侧进度文字

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
    // ---- 左侧：正方形图片（80x80）----
    self.displayImageView = [[UIImageView alloc] init];
    self.displayImageView.contentMode = UIViewContentModeScaleAspectFill;
    self.displayImageView.clipsToBounds = YES;
    self.displayImageView.backgroundColor = [UIColor lightGrayColor];
    [self.contentView addSubview:self.displayImageView];
    [self.displayImageView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.contentView).offset(10);
        make.centerY.equalTo(self.contentView);
        make.size.mas_equalTo(CGSizeMake(80, 80));
    }];

    // ---- 右侧：进度文字 ----
    self.progressLabel = [[UILabel alloc] init];
    self.progressLabel.font = [UIFont systemFontOfSize:14];
    self.progressLabel.textColor = [UIColor blackColor];
    self.progressLabel.numberOfLines = 0;
    [self.contentView addSubview:self.progressLabel];
    [self.progressLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.displayImageView.mas_right).offset(15);
        make.centerY.equalTo(self.contentView);
        make.right.equalTo(self.contentView).offset(-15);
    }];
}

- (void)configureWithItem:(ImageItem *)item {
    NSURL *url = [NSURL URLWithString:item.imageURL];
    if (!url) return;

    [self.displayImageView sd_setImageWithURL:url
        placeholderImage:nil
        options:SDWebImageAvoidAutoSetImage
        progress:^(NSInteger received, NSInteger expected, NSURL *targetURL) {
            double receivedKB = received / 1024.0;
            double expectedKB = expected / 1024.0;
            double percent = (expected > 0) ? (received * 100.0 / expected) : 0;

            // 任务 1：拼三段
            NSString *part = [NSString concat3:
                [NSString stringWithFormat:@"%.2fKB", receivedKB]
                _:[NSString stringWithFormat:@"%.2fKB", expectedKB]
                __:[NSString stringWithFormat:@"%.2f%%", percent]];

            NSString *text = [NSString stringWithFormat:@"%ld_%@", (long)item.index, part];

            dispatch_async(dispatch_get_main_queue(), ^{
                self.progressLabel.text = text;
            });
        }
        completed:^(UIImage *image, NSError *error, SDImageCacheType type, NSURL *url) {
            if (image) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.displayImageView.image = image;   // ← 100% 才显示图
                });
            }
        }];
}

- (void)prepareForReuse {
    [super prepareForReuse];
    [self.displayImageView sd_cancelCurrentImageLoad];
    self.displayImageView.image = nil;
    self.progressLabel.text = nil;
}

@end
```

---

## ImageItem.h（继承版也要加 index）

```objc
@property (nonatomic, assign) NSInteger index;

// ImageItem.m — defaultItems 里：
item.index = i + 1;   // 1, 2, 3...7
```

---

## 核心对比

```objc
// 分类版                              // 继承版
self.imageArea.image = img;            self.displayImageView.image = img;
self.imageArea.displayImageView        self.displayImageView
    → sd_setImageWithURL:...               → sd_setImageWithURL:...
```

继承版不用 `.displayImageView` 绕一层——`self.displayImageView` 本身就是 UIImageView，SDWebImage 直接往上招呼。整个文件不依赖 UIView+Display 分类。
