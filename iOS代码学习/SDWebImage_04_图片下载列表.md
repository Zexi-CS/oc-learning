# 04 — SDWebImage 图片下载进度列表

> 第四步：集成任务 1（NSString 拼接）+ 任务 2（UIView 显示）+ SDWebImage 下载。

---

## 一、题目要求

使用 SDWebImage 从网络请求图片，实现 UITableView 列表：
- 每行显示图片 + 进度文字（已下载KB_总KB_百分比）
- 使用任务 1 的 NSString 分类拼接文字
- 使用任务 2 的 UIView 分类显示图片
- 需考虑列表滚动流畅性

**7 张测试图 URL：**
```
https://oss-test-down.2980.com/63/45/6345517ead2f2776f680ec598e4df599-1896240-4288x2848..jpeg
https://oss-test-down.2980.com/2d/c1/2dc1a2fc2aa833b00b92dc4388a86139-2604768-4288x2848..jpeg
https://oss-test-down.2980.com/9f/62/9f620878e06d28774406017480a59fd4-2505426-3000x2002..jpeg
https://oss-test-down.2980.com/7f/0c/7f0c36257c70e2a497909337bb94a850-1268382-1668x2500..jpeg
https://oss-test-down.2980.com/2e/83/2e838298882840700f92d77b1f5dcc1f-1852262-3000x2002..jpeg
https://oss-test-down.2980.com/7a/ee/7aee3f9ae940b55a1a227c56598c3b90-4159953-4032x3024..jpeg
https://oss-test-down.2980.com/a0/c4/a0c44c798fb93e57a4e77ffdb2747a9d-1909388-7087x2636..jpeg
```

---

## 二、项目搭建

CocoaPods 引入 SDWebImage + Masonry：

```ruby
# Podfile
platform :ios, '12.0'
target 'SDWebImageDemo' do
  pod 'SDWebImage'
  pod 'Masonry'
end
```

文件清单：

```
├── Categories/
│   ├── NSString+Concatenation.h/.m    ← 任务 1
│   └── UIView+Display.h/.m            ← 任务 2
├── Model/
│   └── ImageItem.h/.m
├── View/
│   └── ImageCell.h/.m
└── Controller/
    └── ImageListViewController.h/.m
```

---

## 三、ImageItem — 数据模型

### ImageItem.h

```objc
#import <Foundation/Foundation.h>

@interface ImageItem : NSObject

@property (nonatomic, copy)   NSString  *imageURL;
@property (nonatomic, assign) NSInteger  index;       // 序号 1~7

+ (NSArray<ImageItem *> *)defaultItems;

@end
```

### ImageItem.m

```objc
#import "ImageItem.h"

@implementation ImageItem

+ (NSArray<ImageItem *> *)defaultItems {
    NSArray *urls = @[
        @"https://oss-test-down.2980.com/63/45/6345517ead2f2776f680ec598e4df599-1896240-4288x2848..jpeg",
        @"https://oss-test-down.2980.com/2d/c1/2dc1a2fc2aa833b00b92dc4388a86139-2604768-4288x2848..jpeg",
        @"https://oss-test-down.2980.com/9f/62/9f620878e06d28774406017480a59fd4-2505426-3000x2002..jpeg",
        @"https://oss-test-down.2980.com/7f/0c/7f0c36257c70e2a497909337bb94a850-1268382-1668x2500..jpeg",
        @"https://oss-test-down.2980.com/2e/83/2e838298882840700f92d77b1f5dcc1f-1852262-3000x2002..jpeg",
        @"https://oss-test-down.2980.com/7a/ee/7aee3f9ae940b55a1a227c56598c3b90-4159953-4032x3024..jpeg",
        @"https://oss-test-down.2980.com/a0/c4/a0c44c798fb93e57a4e77ffdb2747a9d-1909388-7087x2636..jpeg"
    ];

    NSMutableArray *items = [NSMutableArray arrayWithCapacity:urls.count];
    for (NSInteger i = 0; i < urls.count; i++) {
        ImageItem *item = [[ImageItem alloc] init];
        item.imageURL = urls[i];
        item.index = i + 1;   // 1, 2, 3...7
        [items addObject:item];
    }
    return [items copy];
}

@end
```

---

## 四、ImageCell — 自定义 Cell

**布局：左侧 80x80 正方形图片 + 右侧垂直居中进度文字。图片 100% 才显示。**

### ImageCell.h

```objc
#import <UIKit/UIKit.h>

@class ImageItem;

@interface ImageCell : UITableViewCell

+ (CGFloat)cellHeight;
- (void)configureWithItem:(ImageItem *)item;

@end
```

### ImageCell.m

```objc
#import "ImageCell.h"
#import "ImageItem.h"
#import "UIView+Display.h"
#import "NSString+Concatenation.h"
#import <Masonry/Masonry.h>
#import <SDWebImage/SDWebImage.h>

@interface ImageCell ()

@property (nonatomic, strong) UIView  *imageArea;      // 左侧图（任务 2 的 image）
@property (nonatomic, strong) UIView  *textArea;        // 右侧进度（任务 2 的 text）

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
    // ---- 左侧：正方形图片区域 ----
    self.imageArea = [[UIView alloc] init];
    self.imageArea.backgroundColor = [UIColor lightGrayColor];
    self.imageArea.clipsToBounds = YES;
    [self.contentView addSubview:self.imageArea];
    [self.imageArea mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.contentView).offset(10);
        make.centerY.equalTo(self.contentView);
        make.size.mas_equalTo(CGSizeMake(80, 80));
    }];

    // ---- 右侧：进度文字（任务 2 的 text）----
    self.textArea = [[UIView alloc] init];
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

    [self.imageArea.displayImageView sd_setImageWithURL:url
        placeholderImage:nil
        options:(SDWebImageAvoidAutoSetImage | SDWebImageRefreshCached)
        progress:^(NSInteger received, NSInteger expected, NSURL *targetURL) {
            double receivedKB = received / 1024.0;
            double expectedKB = expected / 1024.0;
            double percent = (expected > 0) ? (received * 100.0 / expected) : 0;

            // 任务 1：用 concat3 拼三段
            NSString *part = [NSString concat3:
                [NSString stringWithFormat:@"%.2fKB", receivedKB]
                _:[NSString stringWithFormat:@"%.2fKB", expectedKB]
                __:[NSString stringWithFormat:@"%.2f%%", percent]];

            // 序号 + 拼接结果
            NSString *text = [NSString stringWithFormat:@"%ld_%@", (long)item.index, part];

            dispatch_async(dispatch_get_main_queue(), ^{
                self.textArea.text = text;                    // ← 任务 2 的 text
            });
        }
        completed:^(UIImage *image, NSError *error, SDImageCacheType type, NSURL *url) {
            if (image) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.imageArea.image = image;       // ← 100% 才显示图
                });
            }
        }];
}

- (void)prepareForReuse {
    [super prepareForReuse];
    [self.imageArea.displayImageView sd_cancelCurrentImageLoad];
    self.imageArea.image = nil;
    self.textArea.text = nil;
}

@end
```

---

## 五、ImageListViewController — 主界面

```objc
#import "ImageListViewController.h"
#import "ImageItem.h"
#import "ImageCell.h"
#import <Masonry/Masonry.h>

@interface ImageListViewController () <UITableViewDataSource, UITableViewDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSArray<ImageItem *> *items;

@end

@implementation ImageListViewController

static NSString * const kCellID = @"ImageCell";

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"图片下载列表";
    self.view.backgroundColor = [UIColor whiteColor];
    self.items = [ImageItem defaultItems];

    [self setupTableView];
}

- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.rowHeight = [ImageCell cellHeight];
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleNone;

    [self.tableView registerClass:[ImageCell class] forCellReuseIdentifier:kCellID];

    [self.view addSubview:self.tableView];
    [self.tableView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self.view);
    }];
}

- (NSInteger)tableView:(UITableView *)tv numberOfRowsInSection:(NSInteger)section {
    return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tv cellForRowAtIndexPath:(NSIndexPath *)ip {
    ImageCell *cell = [tv dequeueReusableCellWithIdentifier:kCellID forIndexPath:ip];
    [cell configureWithItem:self.items[ip.row]];
    return cell;
}

@end
```

---

## 六、关键知识点

| 知识点 | 位置 |
|--------|------|
| SDWebImage 下载 + 进度回调 | `sd_setImageWithURL:progress:completed:` |
| `SDWebImageAvoidAutoSetImage` | 图片 100% 才手动设 |
| Cell 复用取消下载 | `prepareForReuse` → `sd_cancelCurrentImageLoad` |
| 子线程切主线程 | `dispatch_async(dispatch_get_main_queue())` |
| 任务 1 拼接 | `[NSString concat3:a _:b __:c]` |
| 任务 2 显示 | `imageArea.image` / `imageArea.displayImageView` |
| `__block` 不必要的原因 | 进度回调直接用参数 `received`/`expected`，不需要外部变量 |
