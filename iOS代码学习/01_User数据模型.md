# 01 — User 数据模型

> 第一步：定义"用户"长什么样，以及怎么把服务器返回的 JSON 字典翻译成 User 对象。

---

## 一、User.h — 头文件（声明）

```objc
//
//  User.h
//  用户数据模型
//  - userId: 唯一标识（字符串类型，服务器生成，更新/删除时用）
//  - name:   用户姓名
//  - age:    年龄（字符串类型，因为服务器返回的都是字符串）
//

#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, copy) NSString *userId;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *age;

/// 从服务器返回的字典中解析出一个 User 对象
/// @param dict 服务器返回的字典，例如 @{@"id": @"1", @"name": @"张三", @"age": @"25"}
/// @return 解析成功返回 User 对象，失败返回 nil
- (nullable instancetype)initWithDictionary:(NSDictionary *)dict;

/// 将 JSON 数组批量转换为 User 对象数组
/// @param array 服务器返回的数组，例如 @[@{@"id": @"1", ...}, @{@"id": @"2", ...}]
/// @return User 对象数组（如果 array 不是数组类型则返回空数组）
+ (NSArray<User *> *)usersFromArray:(NSArray<NSDictionary *> *)array;

@end
```

### 逐行解释（User.h）

| 行 | 代码 | 解释 |
|----|------|------|
| `#import <Foundation/Foundation.h>` | 导入 Foundation 框架 | Foundation 是 OC 的基础库，NSString、NSArray、NSDictionary 都在里面。类比 C 语言的 `#include <stdio.h>` |
| `@interface User : NSObject` | 声明 User 类，继承自 NSObject | OC 中所有类的"根"都是 NSObject。继承 NSObject 意味着 User 自动拥有内存管理、`init` 初始化等基础能力 |
| `@property (nonatomic, copy) NSString *userId` | 声明 userId 属性 | **属性 = 成员变量 + 自动生成的 getter/setter**。`copy` 表示赋值时会复制一份新字符串，防止外部修改影响内部 |
| `- (nullable instancetype)initWithDictionary:` | 声明实例方法 | 前面 `-` 表示这是**实例方法**（必须用 `[[User alloc] init...]` 创建对象后才能调用）。`nullable` 表示返回值可能为 nil。`instancetype` 表示返回类型和调用者一致 |
| `+ (NSArray<User *> *)usersFromArray:` | 声明类方法 | 前面 `+` 表示这是**类方法**（直接用 `[User usersFromArray:...]` 调用，不需要先创建对象） |

**为什么年龄也用 NSString 而不是 int？** 因为服务器返回的 JSON 里数字可能是 `25` 也可能是 `"25"`，用字符串统一接收最安全，不丢精度。

---

## 二、User.m — 实现文件

```objc
//
//  User.m
//

#import "User.h"

@implementation User

// ============================================================
// 方法 1：从字典中解析出 User 对象（实例方法）
// ============================================================
- (instancetype)initWithDictionary:(NSDictionary *)dict {
    // ---- 第 1 步：调用父类 NSObject 的 init，创建基础对象 ----
    self = [super init];
    if (self) {
        // ---- 第 2 步：从字典中安全取值，赋给对应的属性 ----
        //
        // 写法解释：
        //   dict[@"id"] — 从字典里取 key 为 "id" 的值
        //   ? :            — 如果取到了，执行冒号前面的代码
        //                   — 如果取不到（nil），执行冒号后面的 @""
        //   stringWithFormat:@"%@", ... — 把任意类型都转成字符串
        //                                 哪怕是 NSNumber(25) 也会变成 @"25"

        _userId = dict[@"id"]   ? [NSString stringWithFormat:@"%@", dict[@"id"]]   : @"";
        _name   = dict[@"name"] ? [NSString stringWithFormat:@"%@", dict[@"name"]] : @"";
        _age    = dict[@"age"]  ? [NSString stringWithFormat:@"%@", dict[@"age"]]  : @"";
    }
    return self;
}

// ============================================================
// 方法 2：批量转换（类方法）
// ============================================================
+ (NSArray<User *> *)usersFromArray:(NSArray<NSDictionary *> *)array {
    // ---- 第 1 步：防御性检查 ----
    // 如果传进来的根本不是数组，或者为 nil，直接返回空数组，不崩溃
    if (!array || ![array isKindOfClass:[NSArray class]]) {
        return @[];
    }

    // ---- 第 2 步：遍历数组，每个字典转一个 User 对象 ----
    NSMutableArray<User *> *users = [NSMutableArray arrayWithCapacity:array.count];
    for (NSDictionary *dict in array) {
        // 再检查一次：数组里的每个元素是不是字典？
        if ([dict isKindOfClass:[NSDictionary class]]) {
            // 调用方法 1 创建 User 对象，加入可变数组
            [users addObject:[[User alloc] initWithDictionary:dict]];
        }
    }

    // ---- 第 3 步：返回不可变副本（防止外部修改） ----
    return [users copy];
}

// ============================================================
// 方法 3：调试用的 description（可选，但强烈建议加上）
// ============================================================
- (NSString *)description {
    return [NSString stringWithFormat:@"<User: id=%@, name=%@, age=%@>",
            self.userId, self.name, self.age];
}

@end
```

### 逐行解释（User.m）

#### initWithDictionary: 方法

这是整个 User 类最核心的方法，理解了这个，后面的网络数据解析你就全通了。

**数据流向：**

```
服务器返回的 JSON                  initWithDictionary:              你的 User 对象
{                       →                                   →
  "id": "1",                                                 userId = @"1"
  "name": "张三",                                            name   = @"张三"
  "age": "25"                                                age    = @"25"
}                                                           }
```

**关键点：`self = [super init]`**

这是 OC 初始化对象的固定套路。类比 C 语言：你先 `malloc` 一块内存，然后往里面填数据。`[super init]` 就是 OC 帮你做好了内存分配，你只需要往属性里填值。

**关键点：`_userId` vs `self.userId`**

- `_userId` — 直接访问成员变量（类内部用，绕过 setter）
- `self.userId` — 通过 getter/setter 访问（推荐外部用）

在 `init` 方法里推荐用 `_userId` 直接赋值，避免触发 setter 可能带来的副作用。

**关键点：取不到值怎么办？**

```objc
_userId = dict[@"id"] ? ... : @"";
```

如果服务器没返回 `id` 字段，就用空字符串 `@""` 兜底。这样后面 UI 显示时不会因为 nil 而崩溃。

#### usersFromArray: 方法

这是一个"批量工厂"方法。网络请求返回用户列表时，服务器给的是一整坨 JSON 数组：

```json
[
  { "id": "1", "name": "张三", "age": "25" },
  { "id": "2", "name": "李四", "age": "30" }
]
```

这个方法就是帮你把这一坨数组，逐个转成 User 对象，最后给你一个 `NSArray<User *>`。

**为什么返回 `[users copy]` 而不是直接返回 `users`？**

`users` 是 `NSMutableArray`（可变数组），外面拿到后可能会被意外修改。`copy` 一下返回的是不可变数组，更安全。

#### description 方法

这是 OC 版的"toString"。当你写 `NSLog(@"%@", user)` 时，OC 会自动调用这个方法。没有它，打印出来的就是 `<User: 0x12345678>` 这种看不懂的内存地址。

---

## 三、怎么测试这段代码？

在任意 `.m` 文件的 `viewDidLoad` 里加上：

```objc
// 模拟服务器返回的 JSON 字典
NSDictionary *fakeServerData = @{
    @"id": @"1",
    @"name": @"张三",
    @"age": @"25"
};

// 用我们的方法创建 User 对象
User *user = [[User alloc] initWithDictionary:fakeServerData];

// 打印看看
NSLog(@"用户对象：%@", user);
// 输出：<User: id=1, name=张三, age=25>
```

---

## 四、这部分涉及的 OC 语法（对照 C 语言理解）

| OC 概念 | 类比 C 语言 | 区别 |
|---------|-----------|------|
| `@interface` | `.h` 头文件中的结构体声明 | OC 用 `@interface` 声明类（= 结构体 + 函数指针表） |
| `@implementation` | `.c` 源文件中的函数实现 | OC 用 `@implementation` 实现类的方法 |
| `@property` | 结构体成员 + 手动写 get/set 函数 | OC 自动生成 getter/setter，省掉大量样板代码 |
| `NSString *` | `char *` | OC 的字符串是对象，功能远强于 C 字符串 |
| `NSDictionary` | 没有直接对应 | 键值对容器，等价于其他语言的 Map / HashMap |
| `NSArray` | `int arr[]` | OC 数组只能存对象，不能存基本类型（int 要包装成 NSNumber） |
| `nil` | `NULL` | OC 的空对象，和 NULL 本质相同，但 OC 中可以给 nil 发消息而不崩溃 |
| `@""` | `""` | OC 字符串字面量，前面的 `@` 是必须的 |

---

## 五、小结

| 你学到了什么 | 为什么重要 |
|-------------|-----------|
| 数据模型的作用 | 定义整个程序的核心数据结构，后面所有代码都围绕它写 |
| `initWithDictionary:` 模式 | 把服务器 JSON 翻译成 OC 对象的标准做法 |
| 防御性编程 | 字典字段缺失、类型不对 → 空字符串兜底，不崩溃 |
| `_成员变量` vs `self.属性` | 类内部初始化时直接赋值更安全 |
| `+` 方法 vs `-` 方法 | 类方法（[User xxx]）vs 实例方法（[user xxx]） |

---

**下一步预告：**有了 User 模型之后，我们就可以写 NetworkManager —— 一个专门负责跟服务器通信的类，用 NSURLSession 发 GET 请求拿用户列表回来。
