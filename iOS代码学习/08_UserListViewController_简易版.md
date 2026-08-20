# UserListViewController — 简易版

> 比规范版更精简：点掉确认就做完，不弹成功/失败弹窗，block 里只刷列表。
> **考核专用**——默认请求一定会成功，能快速通过考核即可。

---

## 需求简述

- **列表**：显示用户（名字 + 年龄）
- **新增**：右上角"+" → 弹窗输姓名年龄 → 确认 → 直接添加并刷新
- **编辑**：左滑 → "编辑" → 弹窗（预填当前值）→ 确认 → 直接改并刷新
- **删除**：左滑 → "删除" → 直接删除并刷新
- **下载**：底部"下载文件"按钮 → 调 NetworkManager 下载 → 控制台打印进度和结果
- 增删改查全部不弹成功/失败提示，block 里永远只写 `[self fetchUsers]`
- 所有控件用 **Masonry** 布局（不直接用 frame）

---

## UserListViewController.m（完整）

```objc
//
//  UserListViewController.m — 简易版
//

#import "UserListViewController.h"
#import "User.h"
#import "NetworkManager.h"
#import "UserCell.h"
#import <Masonry/Masonry.h>            // ★ Masonry 布局

@interface UserListViewController () <UITableViewDataSource, UITableViewDelegate>

@property (nonatomic, strong) UITableView *tableView;           // 列表
@property (nonatomic, strong) NSMutableArray<User *> *users;    // 数据源
@property (nonatomic, strong) UIButton *downloadButton;         // 底部下载按钮

@end


@implementation UserListViewController

// ============================================================
// 生命周期 + 一次性搭建 UI
// ============================================================
- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"用户列表";                 // 导航栏标题
    self.view.backgroundColor = [UIColor whiteColor];
    self.users = [NSMutableArray array];     // 初始化空数组

    [self setupTableView];                   // 创建列表（Masonry 布局）
    [self setupBottomBar];                   // 创建底部下载按钮

    // ─── 右上角"+"新增按钮 ───
    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc]
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
        target:self
        action:@selector(addUser)];

    // 首次进入拉取列表
    [self fetchUsers];
}

// ─── TableView（用 Masonry 定位）───
- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
    self.tableView.dataSource = self;
    self.tableView.delegate = self;
    self.tableView.rowHeight = 60;
    [self.tableView registerClass:[UserCell class]
           forCellReuseIdentifier:kUserCellIdentifier];
    [self.view addSubview:self.tableView];

    // Masonry：顶部撑满，左右贴边，底部接下载按钮上方（留 10pt 间距）
    [self.tableView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.left.right.equalTo(self.view);
        // bottom 约束在 setupBottomBar 里补（因为要先有 downloadButton 才能引用）
    }];
}

// ─── 底部下载按钮（用 Masonry 定位）───
- (void)setupBottomBar {
    self.downloadButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.downloadButton setTitle:@"下载文件" forState:UIControlStateNormal];
    self.downloadButton.backgroundColor = [UIColor systemBlueColor];
    [self.downloadButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    [self.downloadButton addTarget:self
                            action:@selector(downloadButtonTapped)
                  forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.downloadButton];

    [self.downloadButton mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.right.equalTo(self.view).insets(UIEdgeInsetsMake(0, 20, 0, 20));
        make.height.mas_equalTo(44);
        make.bottom.equalTo(self.view).offset(-20);               // 底部间距
    }];

    // 两个方向闭环：
    // 按钮顶上 = 列表底部 + 10
    // 列表底部 = 按钮顶上 - 10
    [self.downloadButton mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.tableView.mas_bottom).offset(10);   // 贴列表下边
    }];

    [self.tableView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.bottom.equalTo(self.downloadButton.mas_top).offset(-10);
    }];
}


// ============================================================
// 网络层 —— 拉数据 / 增删改，block 一律只刷新列表
// ============================================================

// 拉取用户列表（连接服务器，塞进数组）
- (void)fetchUsers {
    [[NetworkManager sharedManager] fetchUsersWithCompletion:
        ^(BOOL success, id result, NSError *error) {
            // 考试默认成功，不判断成败
            self.users = [NSMutableArray arrayWithArray:result];
            [self.tableView reloadData];      // 刷新列表
        }];
}

// 新增用户（右上角"+"触发）
- (void)addUser {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"新增用户"
        message:nil                                                  // 只要标题，不要描述
        preferredStyle:UIAlertControllerStyleAlert];

    // 两个输入框：姓名、年龄
    [alert addTextFieldWithConfigurationHandler:nil];
    [alert addTextFieldWithConfigurationHandler:nil];

    [alert addAction:[UIAlertAction actionWithTitle:@"取消"
                                               style:UIAlertActionStyleCancel handler:nil]];

    [alert addAction:[UIAlertAction actionWithTitle:@"确认"
                                               style:UIAlertActionStyleDefault
                                             handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields[0].text ?: @"";
        NSString *age  = alert.textFields[1].text ?: @"";

        [[NetworkManager sharedManager] addUserWithName:name age:age
                                             completion:
            ^(BOOL success, id result, NSError *error) {
                [self fetchUsers];          // 确认完直接刷新，不弹提示
            }];
    }]];

    [self presentViewController:alert animated:YES completion:nil];
}


// ============================================================
// 左滑操作 —— 编辑 + 删除
// ============================================================

// 系统问：左滑显示哪些按钮？
- (UISwipeActionsConfiguration *)tableView:(UITableView *)tableView
trailingSwipeActionsConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath {

    User *user = self.users[indexPath.row];

    // ─── 删除按钮（红色）───
    UIContextualAction *deleteAction = [UIContextualAction
        contextualActionWithStyle:UIContextualActionStyleDestructive
        title:@"删除"
        handler:^(UIContextualAction *action, UIView *sourceView,
                  void (^completionHandler)(BOOL)) {
            [[NetworkManager sharedManager] deleteUserWithId:user.userId
                                                 completion:
                ^(BOOL success, id result, NSError *error) {
                    [self fetchUsers];      // 左滑删除 → 直接重拉列表
                }];
            completionHandler(YES);         // 收起左滑
        }];

    // ─── 编辑按钮（蓝色）───
    UIContextualAction *editAction = [UIContextualAction
        contextualActionWithStyle:UIContextualActionStyleNormal
        title:@"编辑"
        handler:^(UIContextualAction *action, UIView *sourceView,
                  void (^completionHandler)(BOOL)) {
            [self editUser:user];           // 弹编辑框
            completionHandler(YES);
        }];
    editAction.backgroundColor = [UIColor systemBlueColor];

    return [UISwipeActionsConfiguration configurationWithActions:@[deleteAction, editAction]];
}

// 编辑用户弹窗（左滑"编辑"触发）
- (void)editUser:(User *)user {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"编辑用户"
        message:nil                              // 只要标题
        preferredStyle:UIAlertControllerStyleAlert];

    // 输入框预填当前姓名、年龄
    [alert addTextFieldWithConfigurationHandler:^(UITextField *tf) {
        tf.text = user.name;
    }];
    [alert addTextFieldWithConfigurationHandler:^(UITextField *tf) {
        tf.text = user.age;
    }];

    [alert addAction:[UIAlertAction actionWithTitle:@"取消"
                                               style:UIAlertActionStyleCancel handler:nil]];

    [alert addAction:[UIAlertAction actionWithTitle:@"确认"
                                               style:UIAlertActionStyleDefault
                                             handler:^(UIAlertAction *action) {
        NSString *newName = alert.textFields[0].text ?: @"";
        NSString *newAge  = alert.textFields[1].text ?: @"";

        [[NetworkManager sharedManager] updateUserWithId:user.userId
                                                    name:newName
                                                     age:newAge
                                             completion:
            ^(BOOL success, id result, NSError *error) {
                    [self fetchUsers];          // 确认完直接刷新
            }];
    }]];

    [self presentViewController:alert animated:YES completion:nil];
}


// ============================================================
// UITableViewDataSource
// ============================================================

// 几行 = 数据源几条
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

// 每行显示什么
- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserCell *cell = [tableView dequeueReusableCellWithIdentifier:kUserCellIdentifier
                                                     forIndexPath:indexPath];
    [cell configureWithUser:self.users[indexPath.row]];   // 填数据
    return cell;
}


// ============================================================
// 文件下载（底部按钮触发）
// ============================================================
- (void)downloadButtonTapped {
    NSString *urlString = @"https://quan.duoyioa.com/upload/image/20190927/origin5351595200009b00009b523e23e72ce7.png";

    [[NetworkManager sharedManager] downloadFileFromURL:urlString
        progress:^(double progress) {
            // progress 在子线程回调，且这里只 NSLog 不碰 UI，可不切主线程
            NSLog(@"[下载进度] %.1f%%", progress * 100);
        }
        completion:^(BOOL success, NSString *filePath, NSError *error) {
            // completionHandler 默认主线程，可放心打印
            if (success) {
                NSLog(@"[下载完成] 文件路径：%@", filePath);
            } else {
                NSLog(@"[下载失败] %@", error.localizedDescription);
            }
        }];
}

@end
```

---

## 和规范版的区别（懒人对照）

| 规范版 | 简易版 |
|------|--------|
| 拆 3 个 UI 方法 + Masonry | 保留 setupTableView / setupBottomBar，全部用 Masonry |
| 列表全屏 frame | `CGRectZero` + Masonry，底部接下载按钮 |
| 弹窗带 message | 只留 title，message 传 nil |
| 每个请求都判断 success/error + 弹成功/失败框 | block 里只写 `[self fetchUsers]`，不弹框 |
| performAdd/performUpdate/performDelete 各拆一个方法 | 直接内联进弹窗 handler，不再拆 |
| 下载按钮 + 进度条 | 保留下载按钮，进度用 NSLog（不含进度条） |
| 通用 showAlert: 方法 | 已去掉 |

**核心简化：** 网络回调的 block 里不再写 `if (success) [self showAlert:@"成功"] else ...`，永远只写 `[self fetchUsers]`。block 的"壳"保留（因为 NetworkManager 方法签名强制要求），但内容压缩到一行。

---

## 需要的依赖

- `User.h`（数据模型）
- `NetworkManager.h`（5 个方法：fetch/add/update/delete + downloadFileFromURL:，block 签名不变）
- `UserCell.h`（自定义 Cell + `kUserCellIdentifier` + `configureWithUser:`）
- `Masonry`（pod 'Masonry'，所有控件都用 `mas_makeConstraints:` 布局）
- 下载方法 `downloadFileFromURL:progress:completion:` 在 NetworkManager.h/.m 里必须有声明和实现，否则编译报错

### ★ 关于下载方式（重要，别搞混）

这里的 `downloadFileFromURL:` 用的是 **NSURLSessionDownloadTask（苹果原生）**，不是 AFNetworking 下载。

```
ViewController 调用                  NetworkManager 底层
downloadFileFromURL:progress:completion:
        ↓                                ↓
  只传两个 Block                    NSURLSessionDownloadTask + delegate
  （一个收进度一个收结果）           （didWriteData 打印进度 / didFinishDownloading 存文件）
```

- **为什么是 NSURLSessionDownloadTask**：因为题目要求下载用原生方式。ViewModel 的 `downloadFileFromURL:progress:completion:` 签名在三版 NetworkManager 里都一致，所以它兼容三版；但要满足"原生下载"的考核要求，NetworkManager 必须用**混合版**（AFN 请求 + 原生下载）或**纯 NSURLSession 版**。
- **三版 NetworkManager 都兼容这个 ViewController**（因为 `downloadFileFromURL:progress:completion:` 签名一样）：
  - 配「混合版」→ 下载是 NSURLSessionDownloadTask ✅（考核推荐）
  - 配「纯 NSURLSession 版」→ 下载也是 NSURLSessionDownloadTask ✅（纯原生，无第三方）
  - 配「纯 AFN 版」→ 下载变 AFNetworking ❌（不符合题目要求）
- **NSURLSessionDownloadTask ≠ AFNetworking 下载**：
  - NSURLSessionDownloadTask = 苹果自带，delegate 回调（3 个方法），断点续传/后台下载原生支持
  - AFNetworking 下载 = 第三方库，Block 回调（destination:/completionHandler:）

> 若想完全不用下载，把 `downloadButtonTapped` 方法、底部按钮相关代码、NetworkManager 里的下载方法都去掉即可。
