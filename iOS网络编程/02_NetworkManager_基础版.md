# 02 — NetworkManager（基础版：单例 + GET 请求）

> 第二步：创建一个网络管理器，用单例模式管理所有网络通信。  
> 本步只写最核心的部分：单例结构 + 一个 GET 请求获取用户列表。

---

## 一、NetworkManager.h — 头文件

```objc
//
//  NetworkManager.h
//  网络请求管理器（单例模式）
//
//  本步骤实现：
//    1. 单例入口 +sharedManager
//    2. 获取用户列表 GET /user/users
//

#import <Foundation/Foundation.h>

// ─── 导入我们上一步写的 User 模型 ───
#import "User.h"

/// 网络请求完成回调
/// 参数说明：
///   success: 请求是否成功
///   result:  成功时是 NSArray<User *> 数组，失败时是 nil
///   error:   失败时是错误信息，成功时是 nil
typedef void (^NetworkCompletionBlock)(BOOL success, id _Nullable result, NSError * _Nullable error);


@interface NetworkManager : NSObject

/// 获取单例对象（全局唯一）
+ (instancetype)sharedManager;

/// 获取用户列表 — GET /user/users
/// @param completion 完成回调，在主线程执行，返回 User 对象数组
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;

@end
```

### 头文件逐行解释

**`typedef void (^NetworkCompletionBlock)(...)` — 自定义 Block 类型**

这是给 Block "起别名"。对比 C 语言的 typedef：

```c
// C 语言：给函数指针起别名
typedef void (*FuncPtr)(int, char*);

// OC：给 Block 起别名
typedef void (^NetworkCompletionBlock)(BOOL success, id result, NSError *error);
```

为什么起别名？看看不用别名和用别名的区别：

```objc
// 不用别名：每次都要写完整 Block 签名（又长又乱）
- (void)fetchUsersWithCompletion:(void (^)(BOOL, id, NSError *))completion;

// 用了别名：简洁清晰
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;
```

**`+ (instancetype)sharedManager` — 单例入口**

`+` 表示类方法，不需要先创建对象。`instancetype` 表示返回类型和调用者一致（调 `[NetworkManager sharedManager]` 就返回 NetworkManager 类型）。

---

## 二、NetworkManager.m — 实现文件

```objc
//
//  NetworkManager.m
//

#import "NetworkManager.h"

/// ─── 服务器基础地址（写死为常量，方便修改）───
static NSString * const kBaseURL = @"http://10.17.66.196:8086";


@implementation NetworkManager

#pragma mark - 单例

// ============================================================
// 单例实现（dispatch_once 保证只执行一次）
// ============================================================
+ (instancetype)sharedManager {
    // static 局部变量：生命周期是整个程序运行期间，只初始化一次
    static NetworkManager *instance = nil;

    // dispatch_once：保证大括号里的代码在整个程序生命周期里只执行一次
    // 即使多线程同时调用 sharedManager，也只会创建一个实例
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });+

    return instance;
}

// ─── 私有的初始化方法（外部调不到） ───
- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        // 这里暂时不需要初始化什么，后续会创建 NSURLSession
    }
    return self;
}

// ─── 禁用公开的 init，防止外部误用破坏单例 ───
- (instancetype)init {
    @throw [NSException exceptionWithName:@"Singleton"
                                   reason:@"请使用 +sharedManager 获取单例，不要用 [[NetworkManager alloc] init]"
                                 userInfo:nil];
}


#pragma mark - 获取用户列表（GET 请求）

// ============================================================
// GET /user/users — 从服务器拉取所有用户
// ============================================================
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {

    // ─── 第 1 步：构造 URL（拼地址字符串） ─── 
    NSString *urlString = [NSString stringWithFormat:@"%@/user/users", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];

    // ─── 第 2 步：创建请求对象 ───
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";           // 设为 GET 方法
    request.timeoutInterval = 30.0;        // 超时 30 秒

    NSLog(@"[NetworkManager] 发起 GET 请求：%@", urlString);

    // ─── 第 3 步：创建 NSURLSession（网络会话） ───
    NSURLSession *session = [NSURLSession sharedSession];

    // ─── 第 4 步：创建 data task 并发起请求 ───
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData * _Nullable data,
                            NSURLResponse * _Nullable response,
                            NSError * _Nullable error) {

            // ~~~~~~ 以下是网络回调，运行在后台线程 ~~~~~~

            // ---- 第 4.1 步：检查网络错误（没网 / 超时等）----
            if (error) {
                NSLog(@"[NetworkManager] 请求失败：%@", error.localizedDescription);
                // 回到主线程通知调用方
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return; // 直接结束，不往下走
            }

            // ---- 第 4.2 步：检查数据是否为空 ----
            if (!data || data.length == 0) {
                NSLog(@"[NetworkManager] 服务器返回空数据");
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }

            // ---- 第 4.3 步：解析 JSON ----
            NSError *jsonError = nil;
            NSDictionary *responseDict = [NSJSONSerialization JSONObjectWithData:data
                                                                         options:0
                                                                           error:&jsonError];
            if (jsonError) {
                NSLog(@"[NetworkManager] JSON 解析失败：%@", jsonError);
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, jsonError);
                });
                return;
            }

            // ---- 第 4.4 步：检查业务状态码 ----
            // 服务器统一返回格式：{"code": 200, "data": [...], "message": "success"}
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSLog(@"[NetworkManager] 业务错误：code=%ld, message=%@", (long)code, msg);
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }

            // ---- 第 4.5 步：提取 data 字段，转成 User 对象数组 ----
            // responseDict[@"data"] 是一个数组，例如：
            // [{"id":"1","name":"张三","age":"25"}, {"id":"2","name":"李四","age":"30"}]
            NSArray<User *> *users = [User usersFromArray:responseDict[@"data"]];
            NSLog(@"[NetworkManager] 成功获取 %lu 个用户", (unsigned long)users.count);

            // ---- 第 4.6 步：回到主线程，把结果传给调用方 ----
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, users, nil);
            });

        }]; // ← completionHandler Block 结束

    // ─── 第 5 步：启动任务（task 默认是暂停状态，必须 resume） ───
    [task resume];
}

@end
```

---

## 三、核心概念详解

### 3.1 单例模式 — 为什么全局只用一个 NetworkManager？

```
普通类：     每次 [[xxx alloc] init] 都创建一个新对象
             → 10 个页面创建 10 个 NetworkManager → 浪费内存，配置重复

单例类：     不管多少次 [NetworkManager sharedManager]，拿到的都是同一个对象
             → 全局共用，状态统一，节省资源
```

类比：一个公司只有一个前台，所有人找前台办事都是找同一个人。不需要每层楼设一个前台。

**dispatch_once 保证线程安全**：哪怕 100 个线程同时调用 `sharedManager`，也只会创建一个实例。

### 3.2 NSURLSession — "快递公司"

NSURLSession 不负责具体的一个请求，而是管理一批请求：

```
NSURLSession（快递公司）
    ├── dataTask → 寄信（GET/POST 请求）
    ├── downloadTask → 寄大件包裹（下载文件）
    └── uploadTask → 寄快递（上传文件）
```

我们用的 `[NSURLSession sharedSession]` 是系统提供的一个共享会话，最简单，不需要配置。

### 3.3 dataTask 的工作流程（一张图看懂）

```
你的代码                  NSURLSession              服务器
   │                          │                        │
   │ 1. 构造 URL + Request    │                        │
   │─────────────────────────▶│                        │
   │                          │ 2. 发送 HTTP 请求       │
   │                          │───────────────────────▶│
   │                          │                        │
   │                          │ 3. 服务器返回数据        │
   │                          │◀───────────────────────│
   │                          │                        │
   │ 4. 回调 Block 执行        │                        │
   │◀─────────────────────────│                        │
   │   (运行在后台线程)         │                        │
```

### 3.4 四层错误检查 — 为什么要层层把关？

```
第 1 层：error != nil        → 网络层面出问题（没网、超时、DNS 失败）
第 2 层：data == nil         → 服务器回是回了，但内容是空的
第 3 层：JSON 解析失败        → 服务器返回的不是 JSON（可能是 HTML 错误页）
第 4 层：code != 200         → 业务层面失败（比如参数不对）
```

跳过任何一层都可能崩溃。这是网络编程最基本的职业习惯。

### 3.5 `dispatch_async(dispatch_get_main_queue(), ^{ ... })`

网络回调默认在后台线程运行。iOS 规定：**所有 UI 操作必须在主线程**。

```objc
// 如果用 NSLog 打印数据 → 后台线程可以
// 如果改 UILabel.text → 必须回到主线程
dispatch_async(dispatch_get_main_queue(), ^{
    // 这里面 → 主线程 → 安全更新 UI
    self.label.text = @"数据到了";
});
```

### 3.6 `[task resume]` — 别忘了！\*\*

task 创建后默认是**暂停**状态，必须调用 `resume` 才会真正发起网络请求。这是新手最容易忘记的一行代码。

---

## 四、数据流完整路径（结合第一步的 User 模型）

```
1. 外部调用
   [NetworkManager.sharedManager fetchUsersWithCompletion:^(BOOL ok, NSArray *users, NSError *err) {
       // 拿到 User 对象数组
   }];

2. NetworkManager 内部
   GET http://10.17.66.196:8086/user/users
       ↓
   服务器返回 NSData（二进制 JSON）
       ↓
   NSJSONSerialization → NSDictionary
       ↓
   [User usersFromArray:responseDict[@"data"]]
       ↓
   返回 NSArray<User *>

3. 调用方拿到
   一个装满 User 对象的数组，每个 User 都有 userId、name、age
```

---

## 五、测试方法

在 ViewController.m 的 viewDidLoad 里加：

```objc
#import "NetworkManager.h"  // 别忘了导入

- (void)viewDidLoad {
    [super viewDidLoad];

    [[NetworkManager sharedManager] fetchUsersWithCompletion:^(BOOL success, id result, NSError *error) {
        if (success) {
            NSArray<User *> *users = (NSArray<User *> *)result;
            NSLog(@"✅ 获取到 %lu 个用户", (unsigned long)users.count);
            for (User *user in users) {
                NSLog(@"  用户：ID=%@, 姓名=%@, 年龄=%@", user.userId, user.name, user.age);
            }
        } else {
            NSLog(@"❌ 失败：%@", error.localizedDescription);
        }
    }];
}
```

---

## 六、本步骤涉及的知识点小结

| 编号 | 知识点                  | 在哪里体现                                            |
| -- | -------------------- | ------------------------------------------------ |
| 3  | 单例模式 dispatch_once   | `+sharedManager`                                 |
| 9  | NSURL                | `[NSURL URLWithString:]`                         |
| 10 | NSMutableURLRequest  | 构造请求、设置 HTTPMethod、timeoutInterval               |
| 11 | NSURLSession         | `[NSURLSession sharedSession]`                   |
| 12 | NSURLSessionDataTask | `dataTaskWithRequest:completionHandler:`         |
| 15 | GET 请求               | `HTTPMethod = @"GET"`                            |
| 6  | NSJSONSerialization  | `JSONObjectWithData:options:error:`              |
| 19 | 主线程切回                | `dispatch_async(dispatch_get_main_queue(), ...)` |
| 1  | Block 语法             | completionHandler 就是一个 Block                     |
| 2  | typedef Block        | `NetworkCompletionBlock`                         |

---

**下一步预告：** 写完 GET 请求后，接着写 POST（新增用户），然后逐步加上修改、删除、文件下载。
