# Swift vs OC — iOS 网络编程程序语法对比

> 只针对「用 Swift 实现 iOS 网络编程那题」这一个程序，把写到的所有 OC ↔ Swift 语法差异整理成一张对照。
> 涉及文件：User / NetworkManager / UserCell / UserListViewController / 入口（SceneDelegate）。
> 不扩展到其他语法（泛型、协议扩展等与本题无关的都不讲）。

---

## 〇、总览：Swift 相比 OC 最核心的 6 个变化

| # | 变化 | OC | Swift |
|---|------|-----|-------|
| 1 | 无 `.h/.m` 分离 | Interface + Implementation 两个文件 | 一个 `.swift` 文件 |
| 2 | 类型自文档 | 数组里装什么靠记 | 类型直接写明 `[[String: Any]]` |
| 3 | 指针 `*` 消失 | `NSString *`、`id` | 无 `*`，`Any` 对应 `id` |
| 4 | 点语法 → 也是方法调用 | `obj.xxx = x` | `obj.xxx = x`（一样） |
| 5 | 可选类型 `?` | OC 对象默认可空 | 用 `?` 显式标"可能 nil" |
| 6 | 无 `@property` | `@property` | `var` / `let` |

---

## 一、模型层（User.swift）

### 1.1 类定义
```objc
// OC
@interface User : NSObject
@end
@implementation User
@end
```
```swift
// Swift：一个文件搞定，声明+实现在同一大括号
class User {   // 不强制继承 NSObject
}
```

### 1.2 属性
```objc
// OC
@property (nonatomic, copy) NSString *name;
```
```swift
// Swift：必须有初值
var name = ""
// 或 let maxAge = 100  （let 不可变，对应 OC 的 const）
```

| OC | Swift |
|----|-------|
| `@property (nonatomic, copy) NSString *name` | `var name = ""`（getter/setter 自动） |
| `@property` | `var`（可变）/ `let`（不可变） |
| `_name` | 直接 `name`（Swift 属性没有下划线） |
| 默认默认 nil | 必须给初值（`= ""`）或在 init 给 |

### 1.3 初始化方法
```objc
// OC
- (instancetype)initWithDictionary:(NSDictionary *)dict {
    self = [super init];
    if (self) {
        _userId = dict[@"id"] ?: @"";
    }
    return self;
}
```
```swift
// Swift：init 直接带参数名
init(dict: [String: Any]) {
    if let v = dict["id"] as? String { userId = v }
}
```
| OC | Swift |
|----|-------|
| `initWithDictionary:` | `init(dict:)` |
| `dict[@"id"] ?: @""` | `if let v = dict["id"] as? String { userId = v }` |
| `super init` + `if (self)` | Swift 首句不强制（数据类） |

### 1.4 字典/数组类型
```objc
// OC
NSDictionary *d = @{@"id": @"1"};
NSArray *arr = @[@"a", @"b"];
NSArray<NSDictionary*> *dictArr;
```
```swift
// Swift
let d: [String: Any] = ["id": "1"]      // 字典：冒号前key后value
let arr: [String] = ["a", "b"]          // 数组：无冒号
let dictArr: [[String: Any]]            // 数组，元素是 [String:Any] 字典
```
| OC | Swift | 判断 |
|----|-------|------|
| `NSDictionary` | `[String: Any]` | 有冒号 `:` = 字典 |
| `NSArray` | `[String]` | 无冒号 = 数组 |
| `NSArray<NSDictionary*>` | `[[String: Any]]` | 外层数组装字典 |
| `id` | `Any` | 任意类型（默认非空） |
| (OC 默认可空) | `Any?` | 可能 nil（加 `?`） |

### 1.5 类方法（+ 方法）
```objc
// OC
+ (NSArray<User *> *)usersFromArray:(NSArray *)array
```
```swift
class func users(fromArray array: [[String: Any]]) -> [User]
```
| OC | Swift |
|----|-------|
| `+ (返回类型)` | `class func` 或 `static func` |
| `方法名:参数` | `方法名(参数)` |
| `返回类型`写在方法名前 | 返回类型写在 `->` 后 |

### 1.6 外部/内部参数名
```swift
class func users(fromArray array: [[String: Any]]) -> [User]
//               └──┬──┘  └─┬┘
//             外部名  内部名
```
- 外部名 `fromArray`：调用方看（`User.users(fromArray: data)`）
- 内部名 `array`：函数体内用
- OC 的方法名本身就是"外部信息+参数"，Swift 拆成两个 name

---

## 二、网络层（NetworkManager.swift）

### 2.1 单例
```objc
// OC
+ (instancetype)sharedManager {
    static NetworkManager *instance = nil;
    static dispatch_once_t once;
    dispatch_once(&once, ^{ instance = [[self alloc] initPrivate]; });
    return instance;
}
```
```swift
// Swift：一行搞定，自动线程安全
static let shared = NetworkManager()
private init() {}   // 私有 init 防外部创建
```

### 2.2 回调类型（Block）
```objc
// OC
typedef void (^NetworkCompletionBlock)(BOOL success, id result, NSError *error);
```
```swift
// Swift：typealias + 闭包类型
typealias CompletionBlock = (_ success: Bool, _ result: Any?, _ error: Error?) -> Void
```
| OC | Swift |
|----|-------|
| `typedef void (^Block)(参数)` | `typealias Block = (参数) -> Void` |
| `^` (Block) | 闭包 `{ ... }` |
| 参数类型写在括号内 | 参数 → 类型写 `参数: 类型` |
| `void Block` | `-> Void`（无返回） |
| `_ success` | `_` = 忽略外部标签；`success` 参数名 |

### 2.3 URLSession
```objc
// OC
NSURLSession *s = [NSURLSession sharedSession];
NSURLSessionDataTask *task = [s dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *res, NSError *err) {...}];
[task resume];
```
```swift
// Swift
let task = URLSession.shared.dataTask(with: request) { data, response, error in
    // 闭包体
}
task.resume()
```
| OC | Swift |
|----|-------|
| `[NSURLSession sharedSession]` | `URLSession.shared` |
| `dataTaskWithRequest:completionHandler:` | `dataTask(with:completionHandler:)` |
| `^(NSData *d, ...){...}` | `{ data, response, error in ... }` |
| `[task resume]` | `task.resume()` |

### 2.4 请求对象
```objc
// OC
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
request.HTTPMethod = @"POST";
[request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
```
```swift
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
```

### 2.5 错误检查（if error）
```objc
// OC
if (error) { completion(NO, nil, error); return; }
```
```swift
if let error = error {
    completion(false, nil, error)
    return
}
```
- OC：`if (!err)` 判假
- Swift：`if let error = error` 可空绑定（error 有值才进）

### 2.6 guard（提前退出）+ URL 校验
```objc
// OC
if (!url) { return; }
```
```swift
guard let url = URL(string: urlString) else {
    completion(false, nil, nil)
    return
}
// url 解包后可直接用（作用域比 if let 大）
```

### 2.7 JSON 解析 + 转 User
```objc
// OC
NSError *err;
NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:&err];
if (err) {...}
```
```swift
do {
    let json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any]
} catch {
    // 解析失败
}
```
| OC | Swift |
|----|-------|
| `NSJSONSerialization ... error:&err` | `try JSONSerialization...` + `do-catch` |
| 传 `&err` 拿错误 | 方法 `throws`，Swift 强制 try |

### 2.8 切主线程
```objc
// OC
dispatch_async(dispatch_get_main_queue(), ^{ ... });
```
```swift
DispatchQueue.main.async { ... }
```

### 2.9 可选强转 as? as!
```swift
json["data"] as? [[String: Any]]   // as? 安全转，失败=nil
cell as! UserCell                  // as! 强制转，失败崩溃
```
对应 OC：`as?` ≈ 安全的强转（失败 return），OC 强转失败直接崩。

### 2.10 ?? 空值合并
```swift
let code = json["code"] as? Int ?? -1   // 取不到用 -1
```
对应 OC：`dict[@"code"] ?: @(-1)`（`??` 和 `?:` 一个意思）

### 2.11 @escaping（异步回调必须）
```swift
func fetchUsers(completion: @escaping CompletionBlock)
```
- 网络请求的 completion 在方法返回后才调，Swift 要求标 `@escaping`
- OC 没有这概念，Block 直接当参数

---

## 三、UI 层（UserCell.swift）

### 3.1 控件属性
```objc
// OC
@property (nonatomic, strong) UILabel *nameLabel;
// 创建：
self.nameLabel = [[UILabel alloc] init];
```
```swift
let nameLabel = UILabel()   // 定义属性时直接创建
```

### 3.2 初始化 Cell
```objc
// OC
- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)rid {
    self = [super initWithStyle:style reuseIdentifier:rid];
    if (self) { [self setupSubviews]; }
    return self;
}
```
```swift
override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
    super.init(style: style, reuseIdentifier: reuseIdentifier)
    setupSubviews()
}
// storyboard 要求强制加的：
required init?(coder: NSCoder) {
    fatalError("不支持从 storyboard 解码")
}
```
| OC | Swift |
|----|-------|
| `initWithStyle:reuseIdentifier:` | `init(style:reuseIdentifier:)` |
| 重写不写 `override` | **必须写 `override`** |
| `(复用ID)` | `reuseIdentifier: String?`（可空） |
| 没有 | 必须实现 `required init?(coder:)`（storyboard） |

### 3.3 布局：SnapKit（≈ Masonry）
```objc
// OC + Masonry
[nameLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    make.left.equalTo(self.contentView).offset(16);
    make.centerY.equalTo(self.contentView);
}];
```
```swift
// Swift + SnapKit
nameLabel.snp.makeConstraints { make in
    make.leading.equalTo(contentView).offset(16)
    make.centerY.equalTo(contentView)
}
```
| OC Masonry | Swift SnapKit |
|-----------|---------------|
| `[v mas_makeConstraints:^(make){...}]` | `v.snp.makeConstraints { make in ... }` |
| `make.left` | `make.leading` |
| `make.size.mas_equalTo(CGSize(...))` | `make.size.equalTo(CGSize(width:height:))` |
| `make.edges.equalTo(self.view)` | `make.edges.equalTo(view)` |

### 3.4 填充数据 + 字符串插值
```objc
// OC
- (void)configureWithUser:(User *)user {
    nameLabel.text = user.name;
    ageLabel.text  = [NSString stringWithFormat:@"%@ 岁", user.age];
}
```
```swift
func configure(with user: User) {
    nameLabel.text = user.name
    ageLabel.text = "\(user.age) 岁"   // 字符串插值
}
```
| OC | Swift |
|----|-------|
| `configureWithUser:` | `func configure(with: User)` |
| `stringWithFormat:@"%@..."` | `"\(变量) 文本"` |

---

## 四、主界面（UserListViewController.swift）

### 4.1 遵守协议（DataSource/Delegate）
```objc
// OC
@interface UserListViewController () <UITableViewDataSource, UITableViewDelegate>
// 方法写在类实现里
```
```swift
// Swift：用 extension 分块
extension UserListViewController: UITableViewDataSource { }
extension UserListViewController: UITableViewDelegate { }
```
| OC | Swift |
|----|-------|
| `@interface ... <协议>` | `extension 类: 协议 { 方法 }` |
| 协议方法在类里 | extension 大括号里 |
| `dataSource = self` | 一样（只是赋值，协议在 extension 声明） |

### 4.2 DataSource 方法
```objc
// OC
- (NSInteger)tableView:(UITableView *)tv numberOfRowsInSection:(NSInteger)sec {
    return self.users.count;
}
- (UITableViewCell *)tableView:... cellForRowAtIndexPath:(NSIndexPath *)ip {
    return cell;
}
```
```swift
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return users.count
}
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    ...
}
```
| OC | Swift |
|----|-------|
| `- (返回类型)` | `func` + `-> 返回类型` |
| 方法名 `tableView:cellForRowAtIndexPath:` | `tableView(_:cellForRowAt:)` |
| 用 `indexPath.row` | `indexPath.row`（一样） |

### 4.3 取 Cell
```objc
// OC
UserCell *cell = [tv dequeueReusableCellWithIdentifier:kCellID forIndexPath:ip];
```
```swift
let cell = tableView.dequeueReusableCell(withIdentifier: UserCell.identifier, for: indexPath) as! UserCell
```
| OC | Swift |
|----|-------|
| `dequeueReusableCellWithIdentifier:forIndexPath:` | `dequeueReusableCell(withIdentifier:for:)` |
| 自动返回 UserCell | 需 `as! UserCell` 强制转 |
| `kCellID` 全局常量 | `UserCell.identifier`（static let） |

### 4.4 target-action（按钮）
```objc
// OC
self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc]
    initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
    target:self action:@selector(addButtonTapped)];
```
```swift
navigationItem.rightBarButtonItem = UIBarButtonItem(
    barButtonSystemItem: .add,   // 枚举简写
    target: self,
    action: #selector(addButtonTapped)
)
// 方法要标 @objc
@objc private func addButtonTapped() { ... }
```
- `.add` = `UIBarButtonItem.SystemItem.add` 简写（对应 OC 的 `UIBarButtonSystemItemAdd`）
- `@selector` → `#selector`，方法前加 `@objc`

### 4.5 防循环引用（闭包捕获 self）
```objc
// OC
__weak __typeof(self) weakSelf = self;
...
if (!weakSelf) return;
```
```swift
// Swift：捕获列表 [weak self]
NetworkManager.shared.fetchUsers { [weak self] success, result, _ in
    guard let self = self else { return }
    ...
}
```
| OC | Swift |
|----|-------|
| `__weak` | `[weak self]`（闭包捕获列表） |
| `if (!weakSelf) return` | `guard let self = self else { return }` |

### 4.6 弹窗 UIAlertController
```objc
// OC
UIAlertController *alert = [... alertControllerWithTitle:@"新增用户" message:nil preferredStyle:UIAlertControllerStyleAlert];
[alert addTextFieldWithConfigurationHandler:^(UITextField *tf){...}];
[alert addAction:[UIAlertAction actionWithTitle:@"确认" style:... handler:nil]];
[self presentViewController:alert animated:YES completion:nil];
```
```swift
let alert = UIAlertController(title: "新增用户", message: nil, preferredStyle: .alert)
alert.addTextField { tf in tf.placeholder = "姓名" }
alert.addAction(UIAlertAction(title: "确认", style: .default) { _ in ... })
present(alert, animated: true)
```
| OC | Swift |
|----|-------|
| `UIAlertControllerStyleAlert` | `.alert`（枚举简写） |
| `addTextFieldWithConfigurationHandler:` | `addTextField { tf in ... }` |
| `UIAlertActionStyleDefault` | `.default` |
| `[self presentViewController:animated:completion:]` | `present(_:animated:)` |

### 4.7 左滑 UIContextualAction
```objc
// OC
UIContextualAction *del = [UIContextualAction contextualActionWithStyle:Destructive title:@"删除" handler:^(...,void(^completionHandler)(BOOL)){ completionHandler(YES); }];
```
```swift
let delete = UIContextualAction(style: .destructive, title: "删除") { _, _, completion in
    completion(true)   // 对应 completionHandler(YES)
}
```

---

## 五、入口（SceneDelegate.swift）

### 5.1 强转子类
```objc
// OC
UIWindowScene *windowScene = (UIWindowScene *)scene;
```
```swift
guard let windowScene = scene as? UIWindowScene else { return }
```
- OC：`(UIWindowScene *)scene` 强转
- Swift：`scene as? UIWindowScene` 安全转

### 5.2 创建窗口 + 根控制器
```objc
// OC
self.window = [[UIWindow alloc] initWithWindowScene:windowScene];
UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:vc];
self.window.rootViewController = nav;
[self.window makeKeyAndVisible];
```
```swift
window = UIWindow(windowScene: windowScene)
let nav = UINavigationController(rootViewController: UserListViewController())
window?.rootViewController = nav
window?.makeKeyAndVisible()
```

---

## 六、方法调用格式速记（贯穿全文）

```
OC:   [对象 方法名:参数1 参数名2:参数2]
      [self.tableView reloadData]
Swift:对象.方法名(参数1, 参数2: 参数2)
      self.tableView.reloadData()

OC:  - (返回类型)方法名:(参数类型)参数
Swift:func 方法名(参数: 类型) -> 返回类型

OC:  [NSString stringWithFormat:@"%@x", num]
Swift:"\(num)x"
```

---

## 七、本题出现的"同名但不同语义"提醒

| 词 | OC 意思 | Swift 意思 |
|----|---------|-----------|
| `/` | 方法名的一部分 | 无关 |
| `:` | 参数分隔 | 方法冒号、字典冒号、类型标注冒号 |
| `.` | 类方法/点语法 | 枚举简写 `.add`、类型名 `User.self` |
| `?` | 无 | 可选类型（`Any?`） |
| `!` | 无 | 强制解包/强制转（`as!`） |
| `let`/`var` | 无 | 常量/变量声明 |

---

**这份笔记只覆盖 iOS 网络编程程序里出现过的语法差异，全部对应当前 Swift_01~05 和对应 OC 文档。** 你写完这题，这些就是"够用"的 Swift 语法收获了。

---

## 八、追加：方法/参数命名深入（2026-08-28 补充）

### 8.1 init vs 普通方法：命名规则不同

```swift
init(dict: [String: Any])                    // init 方法名固定 = init，dict 是参数名
class func users(fromArray array: [[String:Any]])  // 方法名=users, 外部名=fromArray, 内部名=array
```

| | init | 普通 func |
|---|------|----------|
| 方法名 | 固定 `init` | `users` 等 |
| 外部参数名 | 无（就一个 `dict`） | `fromArray`（调用方看） |
| 内部参数名 | `dict` | `array`（体内用） |
| 调用 | `User(dict:)` | `User.users(fromArray:)` |
| 能加"外部标签"吗 | 不能 | 能 |

- **普通方法**：方法名 + 外部参数名(修饰参数) 组合 = OC 的长方法名拆开
- **init**：方法名固定 init，参数就一个名，无内外之分
- 结论：`init` 的 dict 是参数名（你对），`users` 是方法名、fromArray 是外部标签（不是方法名）

### 8.2 类型推断：let u = User(dict:) 的 u 怎么是 User？

- **看等号右边**：`User(dict:)` 返回 User → 左边 u 自动推断成 User，不用写类型
- `let v = dict["id"]`：v 类型**固定**为 `Any?`（由字典类型 `[String: Any]` 决定），**不是看值内容变**
- 想变成 String → 要 `as? String` 主动转

### 8.3 let vs var（循环里）

```swift
for dict in array {
    let u = User(dict: dict)   // let 或 var 都行，效果一样
    users.append(u)            // u 只被 append，没被改指向 → let 够用
}
var users = [User]()           // 循环外要 append 改数组 → 必须 var
```

**规律：变量"会不会被重新赋值/修改"决定 let/var**
- 会改（append、重赋值）→ `var`（必须）
- 不改（只创建、被别的用）→ `let`（够用，var 也行不报错）

**误导点澄清**：let/var 的"生命周期/释放"相同（都是局部变量，循环一轮结束作废、下一轮新的）。**区别只在"同一轮内能不能改指向"**，这题里 u 不改，所以两者等价。

---

## 九、追加：闭包捕获列表 weak / 可选链（2026-08-28 补充）

### 9.1 闭包头部 [weak self]（捕获列表）

```swift
NetworkManager.shared.fetchUsers { [weak self] success, result, _ in
    self?.users = ...
}
```
结构：`{ [weak self] success, result, _ in 闭包体 }`
- `[weak self]` = **捕获列表**，让 self 用弱引用，防循环引用（对应 OC `__weak`)
- `self?` = weak 后 self 变可选，用 `self?.xxx` 调用
- `success, result, _` = 闭包参数；`_` = 忽略不用的参数

### 9.2 闭包参数列表

- `{ tf in ... }`：tf 是闭包参数，**不用写 let/var**（参数天生是常量）
- **不用写类型**：类型由"闭包匹配的方法签名"决定（如 addTextField → UITextField），编译器推断
- `{ (tf: UITextField) in }` = 显式写类型，和 `{ tf in }` 等价
- `in` = 分隔符，左边参数、右边闭包体

### 9.3 下划线的两种含义

| 位置 | `_` 含义 |
|------|---------|
| `func tableView(_ tableView:...)` | 省略该参数的外部标签（协议签名要求） |
| 闭包 `{ _ , _ , completion in }` | 忽略该参数（不用就 _ 占位） |

### 9.3b 闭包下划线占位：_ 占位 vs 全部写名的区别（08-28 补充）

**`_` 是在"写闭包定义、列参数"时用的，不是在"调用时"用。**

```swift
// 定义闭包时决定：哪些参数不关心、用 _ 占位
{ [weak self] _, _, completion in
    completion(true)
}
```

**`_` 占位 vs 把参数全部写出来，两者都能正常跑**，区别：

| 写法 | 结果 |
|------|------|
| `{ action, sourceView, completion in ... }`（写了但不用） | 能跑，但产生"未使用参数"警告 |
| `{ _, _, completion in ... }`（用 _ 占位） | 能跑，无警告，语义清晰"故意不用" |

**判断标准：这个闭包提供的参数，你用不用？**
- 要用 → 起个名字（如 `completion`）
- 不用 → `_` 占位（不要起没用的名字）

**注意**：`_` 占位是"定义闭包时声明忽略"，不是"调用时做占位"。你写 `{ _, _, completion in }` 的那一行就是在定义闭包、标注忽略前两个参数。

### 9.4 可选链 self?.fetchUsers()

- `self?.xxx()` = **可选链**：self 非 nil 才执行，nil 整条跳过不崩
- 不是三目！`??` 才是空值合并（a ?? b = a 非 nil 用 a）
- 对应 OC：`if (self) { [self xxx]; }`

### 9.5 编辑/新增弹窗预填（placeholder vs text）

```swift
alert.addTextField { tf in
    tf.placeholder = "姓名"
    if let u = user { tf.text = u.name }   // 编辑预填，新增不预填
}
```
- `placeholder`：输入框空的提示字（新增显示）
- `text`：输入框已有内容（编辑预填）
- 新增/编辑共用 `showEditAlert(title:user:)`，靠 `user` 是否 nil 区分（nil=新增，有值=编辑）

---

**已更新：Swift vs OC 网络编程程序语法对比（含 08-28 补充的 init/参数命名、类型推断、let/var、weak/可选链、下划线、预填）。**
