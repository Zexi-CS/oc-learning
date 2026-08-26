# Swift_01 — User 数据模型（class）

> iOS 网络编程的 Swift 版第一步：数据模型 User。
> 对应你 OC 的 `01_User数据模型.md`，用 Swift 重写。
> 目标：存 id / name / age + 从 JSON 转模型。

---

## 一、先对比 OC 和 Swift 的写法（你熟的那版）

```objc
// ═══ OC 版（01 那个）═══
@interface User : NSObject
@property (nonatomic, copy) NSString *userId;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *age;
+ (NSArray<User *> *)usersFromArray:(NSArray *)array;
@end

@implementation User
- (instancetype)initWithDictionary:(NSDictionary *)dict {
    self = [super init];
    if (self) {
        _userId = dict[@"id"] ?: @"";
        _name   = dict[@"name"] ?: @"";
        _age    = dict[@"age"]  ?: @"";
    }
    return self;
}
+ (NSArray<User *> *)usersFromArray:(NSArray *)array {
    NSMutableArray *users = [NSMutableArray array];
    for (NSDictionary *dict in array) {
        User *u = [[User alloc] initWithDictionary:dict];
        [users addObject:u];
    }
    return users;
}
@end
```

```swift
// ═══ Swift 版 ═══
import Foundation

class User {

    // 属性：Swift 用 var（可变），没有 * 指针，没有 @property
    var userId: String = ""
    var name: String = ""
    var age: String = ""

    // 从字典转一个 User —— 对应 OC 的 initWithDictionary:
    init(dict: [String: Any]) {
        if let v = dict["id"]   as? String { userId = v }
        if let v = dict["name"] as? String { name   = v }
        if let v = dict["age"]  as? String { age    = v }
    }

    // 从数组转多个 User —— 对应 OC 的 usersFromArray:
    class func users(fromArray array: [[String: Any]]) -> [User] {
        var users = [User]()
        for dict in array {
            let u = User(dict: dict)
            users.append(u)
        }
        return users
    }
}
```

---

## 二、逐行拆解 Swift 新手必然懵的语法

### 2.1 `import Foundation`

对应 OC 的 `#import <Foundation/Foundation.h>`。Swift 里大部分情况只用 `import Foundation`（有些界面要 UIKit，那句后面讲）。Foundation 提供基本类型（String、字典、数组等）。

### 2.2 `class User { }`

定义类。没有 `.h/.m` 分离——一个 `.swift` 文件里声明和实现都在 `class { }` 大括号里。对应 OC 的 `@interface User : NSObject` + `@implementation User` 合并成一个。

### 2.3 属性 `var userId: String = ""`

对比 OC：
```objc
@property (nonatomic, copy) NSString *userId;
```
```swift
var userId: String = ""
```

拆开看「类型标注」：
| 部分 | OC | Swift |
|------|-----|-------|
| 关键字 | `@property` | `var`（可变）/ `let`（不可变） |
| 类型 | `NSString *` | `String` |
| = 初始值 | 无，默认 nil | 必须给一个，如 `= ""` |
| 指针/星号 | 有 `*` | **没有** |

**关键：Swift 属性必须有初始值**（或在 init 里给）。OC 默认 nil，Swift 默认不给会报错。这里用 `= ""` 兜底（对应你 OC 的 `?: @""`）。

### 2.4 `init(dict: [String: Any])`

`init` 就是构造函数，对应 OC 的 `- (instancetype)initWithDictionary:`。Swift 里 **creator 签名直接定义在 init 后**：

```swift
init(dict: [String: Any]) { ... }
```
- `dict:` = 参数名（外部调用时写 `User(dict: xxx)`）
- `[String: Any]` = 字典类型（key 是 String，value 是任意类型 Any）—— 对应 OC 的 `NSDictionary`

调用：
```swift
let u = User(dict: 某个字典)
```

> 注意：类（class）里 `init` 不需要调 `super.init` 因为它是独立的数据类（不继承）。如果继承 UIView/UIViewController，才需要 `super.init(...)`（后面写 Cell/VC 会讲）。

### 2.5 `if let v = dict["id"] as? String { userId = v }`

这是 **Swift 可选类型 + 安全解包**，是本文件最核心的新语法。对应 OC 的 `dict[@"id"] ?: @""`。

由内到外拆：

**① `dict["id"]` 返回可选值**
```objc
dict[@"id"]          // OC：可能是 nil，也可能有值
dict["id"]           // Swift：返回的是 String? （可选），可能是 nil 或字符串
```

**② `as? String` 安全的类型转换**
```swift
dict["id"] as? String   // 如果 dict["id"] 是 String 就成功，否则返回 nil
```
对应 OC 的强转 `(NSString *)`，但 `as?` 更安全——不是这个类型就返回 nil，**不崩**。OC 强转类型不对直接崩，Swift 的 `as?` 不会。

**③ `if let v = ... { }` 可选绑定（optional binding）**
```swift
if let v = dict["id"] as? String {
    // 只有 dict["id"] 转成 String 成功（v 有值）才进这里
    userId = v
}
// 如果转换失败（v 是 nil），直接跳过 if，不执行
```

**结论：`if let` = "如果有值才取用"。** 对应你 OC 的三元 `?: @""`，但语义更安全——nil 时不赋值，属性保持 `= ""` 的默认值。

### 2.6 `class func users(fromArray array: [[String: Any]]) -> [User]`

对应 OC 的 `+ (NSArray<User *> *)usersFromArray:`。
- `class func` = 类方法（`+` 方法），不用创建实例直接调
- 返回类型 `-> [User]`（User 数组）
- `[[String: Any]]` = 字典数组（对应 OC 的 `NSArray *array`）

### 2.7 `var users = [User]()`

创建空 User 数组。对应 OC 的 `NSMutableArray *users = [NSMutableArray array];`
- `[User]()` = 空数组（元素类型 User）
- 等于 `var users: [User] = []`

### 2.8 `for dict in array` + `users.append(u)`

```swift
for dict in array {        // 遍历数组，对应 OC 的 for (NSDictionary *dict in array)
    let u = User(dict: dict)   // 构造一个 User
    users.append(u)            // 添加到数组，对应 [users addObject:u]
}
```

`let u` = 不可变常量（定义后不能改）。这里每个 u 是一次循环的局部变量，用 let 合适。

---

## 三、完整对照表（OC → Swift）

| OC | Swift |
|----|-------|
| `@interface User : NSObject` | `class User {` |
| `@property (nonatomic, copy) NSString *userId;` | `var userId: String = ""` |
| `@implementation` | `} // class 结束`（合并进同一个花括号） |
| `- (instancetype)initWithDictionary:` | `init(dict:)` |
| `dict[@"id"] ?: @""` | `if let v = dict["id"] as? String { userId = v }` |
| `+ (NSArray *)usersFromArray:` | `class func users(fromArray:) -> [User]` |
| `[NSMutableArray array]` | `var users = [User]()` |
| `for (... in array)` + `[users addObject:]` | `for dict in array` + `users.append()` |

---

## 四、为什么 User 用 class 而不是 struct

你已确定用 class。原因回顾：
- 属性 `var userId: String = ""` 用 var 可变
- 类是引用类型，多个地方共享同一个 User 对象时改一处全变（数据一致性）
- 对应你 OC 的习惯（模型就是类）

> 补充：Swift 里更常见用 struct 做数据模型（更快、代码更少）。但你说用 class、想和 OC 对齐，那就 class。两种都能实现这题，class 更能对上你已有的认知。

---

## 五、你要在 Xcode 里怎么用

1. 新建 Swift 工程（建议选 "iOS App"，语言选 Swift）
2. 新建文件 `User.swift`，把上面的 Swift 版代码粘进去
3. 之后写 NetworkManager 时会引用它

> 暂时没有 main 调用，先把这个类写好放着，下一步 NetworkManager 就会用到。

---

**到这里，User 模型完成。** 你已经接触了 Swift 几个最核心的语法：`class`、`var`、`init`、`[String: Any]`、`as?`、`if let`、`class func`、数组遍历。

下一步写 `Swift_02_NetworkManager_原生.md`（URLSession + 闭包 + 增删改查），那是最重要、也是你要的"原生实现"核心。你可以在别的电脑上先敲好 User.swift 试试，或者我们继续往下走第一个文件。
