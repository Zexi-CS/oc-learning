# 06 — UserListViewController 主界面

> 第六步：把所有东西串起来的页面控制器。TableView + 增删改查 + 文件下载。

---

## 一、这个页面要做什么（先看全局）

```
┌──────────────────────────────────────────┐
│  导航栏：用户列表                  [+] 新增 │  ← UINavigationController + UIBarButtonItem
├──────────────────────────────────────────┤
│  ID:1    张三                   25岁     │  ← UserCell（左滑：编辑 / 删除）
│  ID:2    李四                   30岁     │
│  ID:3    王五                   22岁     │
│  ...                                     │
├──────────────────────────────────────────┤
│  [ 下载文件（NSURLSessionDownloadTask）]   │  ← UIButton
│  ████████████░░░░░░░░ 进度：45%           │  ← UIProgressView + UILabel
└──────────────────────────────────────────┘
```

所有之前的代码在这里汇合：

- User 模型 → 存数据
- NetworkManager → 拿数据、发数据
- UserCell → 显示数据

---

## 二、UserListViewController.h

```objc
//
//  UserListViewController.h
//

#import <UIKit/UIKit.h>

@interface UserListViewController : UIViewController

@end
```

头文件极其简单，因为所有实现细节都在 .m 里封装好了。

---

## 三、UserListViewController.m — 属性声明

```objc
//
//  UserListViewController.m
//  主界面：TableView + 增删改查 + 文件下载
//

#import "UserListViewController.h"
#import "User.h"
#import "NetworkManager.h"
#import "UserCell.h"
#import <Masonry/Masonry.h>


@interface UserListViewController () <UITableViewDataSource, UITableViewDelegate>
//                                    ↑                            ↑
//                         数据源协议：提供数据              代理协议：处理交互

// ─── 数据源 ───
@property (nonatomic, strong) NSMutableArray<User *> *users;  // 存所有用户数据

// ─── UI 控件 ───
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) UIButton *downloadButton;
@property (nonatomic, strong) UIProgressView *progressView;
@property (nonatomic, strong) UILabel *progressLabel;

@end
```

### 两个协议

| 协议                      | 职责   | 问的问题                     |
| ----------------------- | ---- | ------------------------ |
| `UITableViewDataSource` | 提供数据 | "几行？""每行显示什么？"           |
| `UITableViewDelegate`   | 处理交互 | "行高多少？""点击了哪行？""左滑怎么处理？" |

必须写 `<UITableViewDataSource, UITableViewDelegate>`，否则 TableView 不知道找谁问这些问题。

---

## 四、UserListViewController.m — 生命周期 + UI 搭建

```objc
@implementation UserListViewController

#pragma mark - 生命周期

- (void)viewDidLoad {
    [super viewDidLoad];

    self.title = @"用户列表";           // 导航栏标题
    self.view.backgroundColor = [UIColor whiteColor];
    self.users = [NSMutableArray array]; // 初始化空数组

    [self setupNavigationBar];           // 导航栏右上角"新增"按钮
    [self setupTableView];               // 创建 TableView
    [self setupBottomBar];               // 创建底部下载按钮 + 进度条

    // ★ 首次进入页面，自动拉取用户列表
    [self fetchUsers];
}


#pragma mark - UI 搭建

// ─── 导航栏：右上角加号按钮 ───
- (void)setupNavigationBar {
    UIBarButtonItem *addBtn = [[UIBarButtonItem alloc]
        initWithBarButtonSystemItem:UIBarButtonSystemItemAdd   // 系统自带的"+"图标
        target:self                                               // 谁响应点击 → self
        action:@selector(addUserButtonTapped)];                   // 点击后调哪个方法
    self.navigationItem.rightBarButtonItem = addBtn;
}

// ─── TableView ───
- (void)setupTableView {
    self.tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
    self.tableView.dataSource = self;   // 数据源 = 我自己
    self.tableView.delegate = self;     // 交互代理 = 我自己
    self.tableView.rowHeight = 60;      // 每行高度 60pt

    // ★ 注册 Cell 类型：告诉 TableView "UserCell 长这样"
    // 这样系统才知道回收池里该造什么类型的 Cell
    [self.tableView registerClass:[UserCell class]
           forCellReuseIdentifier:kUserCellIdentifier];

    self.tableView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.tableView];
}

// ─── 底部下载区域 ───
- (void)setupBottomBar {
    // 下载按钮
    self.downloadButton = [UIButton buttonWithType:UIButtonTypeSystem];
    [self.downloadButton setTitle:@"下载文件（NSURLSessionDownloadTask）"
                         forState:UIControlStateNormal];
    self.downloadButton.backgroundColor = [UIColor colorWithRed:0.2 green:0.6 blue:1.0 alpha:1.0];
    [self.downloadButton setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    self.downloadButton.layer.cornerRadius = 8;  // 圆角
    self.downloadButton.translatesAutoresizingMaskIntoConstraints = NO;
    [self.downloadButton addTarget:self
                            action:@selector(downloadButtonTapped)
                  forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.downloadButton];

    // 进度条
    self.progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    self.progressView.progress = 0.0;
    self.progressView.hidden = YES;  // 默认隐藏，点了下载才显示
    self.progressView.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.progressView];

    // 进度文字
    self.progressLabel = [[UILabel alloc] init];
    self.progressLabel.font = [UIFont systemFontOfSize:12];
    self.progressLabel.textColor = [UIColor grayColor];
    self.progressLabel.textAlignment = NSTextAlignmentCenter;
    self.progressLabel.hidden = YES;
    self.progressLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self.view addSubview:self.progressLabel];

    // ─── Masonry 布局：整页 ───
    // Masonry 自动关闭 translatesAutoresizingMaskIntoConstraints，不用手写

    // TableView 顶部和左右撑满
    [self.tableView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.left.right.equalTo(self.view);
    }];

    // 底部按钮：左右各留 20
    [self.downloadButton mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view).offset(20);
        make.right.equalTo(self.view).offset(-20);
        make.top.equalTo(self.tableView.mas_bottom).offset(10);
        make.height.mas_equalTo(44);
    }];

    // 进度条
    [self.progressView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view).offset(20);
        make.right.equalTo(self.view).offset(-20);
        make.top.equalTo(self.downloadButton.mas_bottom).offset(5);
    }];

    // 进度文字
    [self.progressLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view).offset(20);
        make.right.equalTo(self.view).offset(-20);
        make.top.equalTo(self.progressView.mas_bottom).offset(2);
        make.bottom.equalTo(self.view).offset(-10);
    }];
}
```

---

## 五、数据拉取 — 第一条从网络到 UI 的桥梁

```objc
#pragma mark - 数据拉取

- (void)fetchUsers {
    [[NetworkManager sharedManager] fetchUsersWithCompletion:^(BOOL success, id result, NSError *error) {
        if (success) {
            // 1. 清空旧数据
            [self.users removeAllObjects];
            // 2. 填入新数据
            [self.users addObjectsFromArray:(NSArray<User *> *)result];
            // 3. 告诉 TableView 刷新
            [self.tableView reloadData];
        } else {
            [self showAlert:@"获取用户列表失败" message:error.localizedDescription];
        }
    }];
}
```

**关键链路：**

```
fetchUsers
  → NetworkManager 发 GET 请求
  → 服务器返回 JSON
  → NetworkManager 解析 → 转成 NSArray<User *>
  → Block 回调拿到 users
  → 存入 self.users
  → [self.tableView reloadData]
  → TableView 重新问 dataSource："现在几行？每行显示什么？"
  → 列表刷新
```

---

## 六、UITableViewDataSource — 提供数据

```objc
#pragma mark - UITableViewDataSource

// 问：有几行？
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;  // 数据源有多少个用户，就显示多少行
}

// 问：第 indexPath.row 行显示什么？
- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {

    // ★ 这就是你之前问的那个方法 ★
    // 从回收池拿一个空闲 Cell（没有就自动创建）
    UserCell *cell = [tableView dequeueReusableCellWithIdentifier:kUserCellIdentifier
                                                     forIndexPath:indexPath];

    // 取出这一行对应的 User 数据
    User *user = self.users[indexPath.row];

    // 把数据填进 Cell
    [cell configureWithUser:user];

    return cell;
}
```

---

## 七、新增用户 — UIAlertController 带输入框

```objc
#pragma mark - 新增用户

- (void)addUserButtonTapped {
    // 创建带两个输入框的弹窗
    UIAlertController *alert = [UIAlertController
        alertControllerWithTitle:@"新增用户"
        message:@"请输入用户信息"
        preferredStyle:UIAlertControllerStyleAlert];

    // 输入框 1：姓名
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"请输入姓名（如：张三）";
    }];

    // 输入框 2：年龄
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"请输入年龄（0~100）";
        textField.keyboardType = UIKeyboardTypeNumberPad;  // 弹出数字键盘
    }];

    // 取消按钮
    [alert addAction:[UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:nil]];

    // 确认按钮
    [alert addAction:[UIAlertAction actionWithTitle:@"确认新增"
                                              style:UIAlertActionStyleDefault
                                            handler:^(UIAlertAction *action) {
        NSString *name = alert.textFields[0].text ?: @"";
        NSString *age  = alert.textFields[1].text ?: @"";

        if (name.length == 0 || age.length == 0) {
            [self showAlert:@"提示" message:@"姓名和年龄不能为空"];
            return;
        }

        [self performAddUserWithName:name age:age];
    }]];

    [self presentViewController:alert animated:YES completion:nil];
}

- (void)performAddUserWithName:(NSString *)name age:(NSString *)age {
    [[NetworkManager sharedManager] addUserWithName:name age:age
        completion:^(BOOL success, id result, NSError *error) {
            if (success) {
                [self showAlert:@"新增成功" message:[NSString stringWithFormat:@"用户「%@」已添加", name]];
                [self fetchUsers];  // ★ 新增后重新拉取列表
            } else {
                [self showAlert:@"新增失败" message:error.localizedDescription];
            }
        }];
}
```

---

## 八、左滑操作 — 编辑 + 删除

```objc
#pragma mark - 左滑操作（编辑 + 删除）

// 系统问：第 indexPath 行左滑后显示什么按钮？
- (UISwipeActionsConfiguration *)tableView:(UITableView *)tableView
trailingSwipeActionsConfigurationForRowAtIndexPath:(NSIndexPath *)indexPath {

    User *user = self.users[indexPath.row];

    // ─── 删除按钮（红色）───
    UIContextualAction *deleteAction = [UIContextualAction
        contextualActionWithStyle:UIContextualActionStyleDestructive   // 红色样式
        title:@"删除"
        handler:^(UIContextualAction *action, UIView *sourceView,
                  void (^completionHandler)(BOOL)) {
            [self deleteUser:user];           // 弹确认框
            completionHandler(YES);           // 告诉系统"操作完成，按钮可以收起"
        }];

    // ─── 编辑按钮（蓝色）───
    UIContextualAction *editAction = [UIContextualAction
        contextualActionWithStyle:UIContextualActionStyleNormal
        title:@"编辑"
        handler:^(UIContextualAction *action, UIView *sourceView,
                  void (^completionHandler)(BOOL)) {
            [self editUser:user];             // 弹编辑框
            completionHandler(YES);
        }];
    editAction.backgroundColor = [UIColor systemBlueColor];

    // 组合返回（数组里靠前的显示在最右边）
    UISwipeActionsConfiguration *config = [UISwipeActionsConfiguration
        configurationWithActions:@[deleteAction, editAction]];
    config.performsFirstActionWithFullSwipe = NO;  // 禁止全滑触发（防误删）
    return config;
}


// ─── 编辑用户弹窗 ───
- (void)editUser:(User *)user {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"编辑用户"
        message:[NSString stringWithFormat:@"当前用户 ID：%@", user.userId]
        preferredStyle:UIAlertControllerStyleAlert];

    // 输入框 1：预填当前姓名
    [alert addTextFieldWithConfigurationHandler:^(UITextField *tf) {
        tf.text = user.name;
    }];
    // 输入框 2：预填当前年龄
    [alert addTextFieldWithConfigurationHandler:^(UITextField *tf) {
        tf.text = user.age;
        tf.keyboardType = UIKeyboardTypeNumberPad;
    }];

    [alert addAction:[UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:nil]];
    [alert addAction:[UIAlertAction actionWithTitle:@"确认修改" style:UIAlertActionStyleDefault
        handler:^(UIAlertAction *action) {
            NSString *newName = alert.textFields[0].text ?: @"";
            NSString *newAge  = alert.textFields[1].text ?: @"";
            if (newName.length == 0 || newAge.length == 0) {
                [self showAlert:@"提示" message:@"姓名和年龄不能为空"];
                return;
            }
             [self performUpdateUserWithId:user.userId name:newName age:newAge];
        }]];

    [self presentViewController:alert animated:YES completion:nil];
}

- (void)performUpdateUserWithId:(NSString *)userId name:(NSString *)name age:(NSString *)age {
    [[NetworkManager sharedManager] updateUserWithId:userId name:name age:age
        completion:^(BOOL success, id result, NSError *error) {
            if (success) {
                [self showAlert:@"修改成功" message:nil];
                [self fetchUsers];
            } else {
                [self showAlert:@"修改失败" message:error.localizedDescription];
            }
        }];
}


// ─── 删除用户确认 ───
- (void)deleteUser:(User *)user {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"确认删除"
        message:[NSString stringWithFormat:@"确定要删除用户「%@」（ID: %@）吗？", user.name, user.userId]
        preferredStyle:UIAlertControllerStyleAlert];

    [alert addAction:[UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:nil]];
    [alert addAction:[UIAlertAction actionWithTitle:@"删除" style:UIAlertActionStyleDestructive
        handler:^(UIAlertAction *action) {
            [self performDeleteUserWithId:user.userId];
        }]];

    [self presentViewController:alert animated:YES completion:nil];
}

- (void)performDeleteUserWithId:(NSString *)userId {
    [[NetworkManager sharedManager] deleteUserWithId:userId
        completion:^(BOOL success, id result, NSError *error) {
            if (success) {
                [self showAlert:@"删除成功" message:@"用户已从服务器中移除"];
                [self fetchUsers];
            } else {
                [self showAlert:@"删除失败" message:error.localizedDescription];
            }
        }];
}
```

---

## 九、文件下载按钮

```objc
#pragma mark - 文件下载

- (void)downloadButtonTapped {
    // 要下载的图片地址
    NSString *urlString = @"https://quan.duoyioa.com/upload/image/20190927/origin5351595200009b00009b523e23e72ce7.png";

    // 显示进度条
    self.progressView.hidden = NO;
    self.progressLabel.hidden = NO;
    self.progressView.progress = 0.0;
    self.progressLabel.text = @"进度：0%";

    // 下载期间按钮不可点（防重复点击）
    self.downloadButton.enabled = NO;
    self.downloadButton.alpha = 0.6;
    [self.downloadButton setTitle:@"正在下载..." forState:UIControlStateNormal];

    [[NetworkManager sharedManager] downloadFileFromURL:urlString
        progress:^(double progress) {
            self.progressView.progress = progress;
            self.progressLabel.text = [NSString stringWithFormat:@"进度：%.1f%%", progress * 100];
        }
        completion:^(BOOL success, NSString *filePath, NSError *error) {
            self.downloadButton.enabled = YES;
            self.downloadButton.alpha = 1.0;
            [self.downloadButton setTitle:@"下载文件（NSURLSessionDownloadTask）" forState:UIControlStateNormal];

            if (success) {
                [self showAlert:@"下载完成"
                        message:[NSString stringWithFormat:@"文件已保存到：\n%@", filePath]];
            } else {
                [self showAlert:@"下载失败" message:error.localizedDescription];
                self.progressLabel.text = @"下载失败";
                self.progressLabel.textColor = [UIColor redColor];
            }
        }];

    // ⚠️ 注意：downloadFileFromURL: 这个方法 NetworkManager 里还没写！
    // 这是下一步的内容，暂时先占位，写了 NetworkManager 下载部分就能调通
}
```

---

## 十、通用弹窗

```objc
#pragma mark - 通用工具方法

- (void)showAlert:(NSString *)title message:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:title
        message:message preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:nil]];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

## 十一、完整数据流演示

```
① App 启动 → viewDidLoad → fetchUsers
                              │
           ┌──────────────────┘
           ▼
② NetworkManager 发 GET /user/users
           │
           ▼
③ 服务器返回 JSON → 解析 → NSArray<User *>
           │
           ▼
④ 赋值 self.users → [self.tableView reloadData]
           │
           ▼
⑤ TableView 调 numberOfRowsInSection → 3 行
           │
           ▼
⑥ TableView 调 cellForRowAtIndexPath（每行调一次）
     → dequeueReusableCell（从回收池拿 Cell）
     → [cell configureWithUser:user]（填数据）
     → return cell（显示到屏幕上）
```

增删改也完全是这个套路：调 NetworkManager 方法 → 成功回调里调 `[self fetchUsers]` → 列表自动刷新。

---

## 十二、完整的文件清单

```
项目根目录/
├── Model/
│   └── User.h / .m                      ← 第 01 步
├── Network/
│   └── NetworkManager.h / .m            ← 第 02~04 步（含下载待补）
├── View/
│   └── UserCell.h / .m                  ← 第 05 步
├── Controller/
│   └── UserListViewController.h / .m    ← 第 06 步（本步）
├── Supporting Files/
│   ├── AppDelegate.h / .m
│   ├── main.m
│   └── Info.plist（ATS 配置）
```

---

## 十三、本步骤涉及的知识点

| 编号 | 知识点                                      | 在哪里体现                                                   |
| -- | ---------------------------------------- | ------------------------------------------------------- |
| 20 | UITableView + DataSource + Delegate      | `numberOfRowsInSection:` / `cellForRowAtIndexPath:`     |
| 21 | registerClass / dequeue                  | `registerClass:` + `dequeueReusableCellWithIdentifier:` |
| 23 | UIAlertController 带输入框                   | 新增 / 编辑弹窗                                               |
| 22 | UISwipeActionsConfiguration              | 左滑编辑 + 删除                                               |
| 25 | UIProgressView                           | 底部进度条                                                   |
| 26 | UIButton + target-action                 | 下载按钮                                                    |
| 27 | UINavigationController + UIBarButtonItem | 导航栏 + 右上角"+"按钮                                          |
| -- | Masonry 布局 | 整页布局（TableView + 底部按钮，全部 `mas_makeConstraints:`） |

**下一步预告：** NetworkManager 补充下载方法 + AppDelegate 入口配置。
