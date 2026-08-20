# NetworkManageAFN 版 — 用 AFNetworking 重构 NetworkManager

> 将网络编程项目的 NetworkManager 从原生 NSURLSession 改造成 AFNetworking 实现。
> **核心：外部调用接口完全不变，只改内部实现。**
> 覆盖完整 5 个接口：GET 列表 / POST 新增 / GET 修改 / GET 删除 / 下载文件。

---

## ⚠️ 改动前必读

### 需要改动的文件（只有 2 个，其余全部不动）

| 文件 | 改动 |
|------|------|
| `Podfile` | 新增 `pod 'AFNetworking'` |
| `NetworkManager.h/.m` | 换成下面这份 |

### 完全不用动的文件

- `User.h/.m`（数据模型）
- `UserCell.h/.m`（Cell）
- `UserListViewController.h/.m`（主界面）
- `AppDelegate` / `SceneDelegate`
- `Info.plist`

> **为什么其他文件不用动？** ViewController 只调 `[[NetworkManager sharedManager] fetchUsersWithCompletion:]` 这个方法名，不关心底层是 NSURLSession 还是 AFNetworking。这就是分层——换发动机不换方向盘。

---

## 一、Podfile

```ruby
platform :ios, '12.0'
target '你的Target名' do
  use_frameworks!
  pod 'AFNetworking'
end
```

> **改动标记 ⚠️**：如果你的 Podfile 已有别的 pod，在这一行并排加即可。

---

## 二、NetworkManager.h — 头文件

> **改动标记 ⚠️**：头文件只删掉原来内部用到的私有声明（原本 .h 里就没有 session，所以头文件几乎不用改）。真正变化在 .m 里。

```objc
//
//  NetworkManager.h
//  网络请求管理器（单例模式） — AFNetworking 版
//

#import <Foundation/Foundation.h>
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
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;

/// 新增用户 — POST /user/save
- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion;

/// 修改用户 — GET /user/update?id=&name=&age=
- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion;

/// 删除用户 — GET /user/delete?id=
- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion;

#pragma mark - 文件下载

/// 下载进度回调（progress: 0.0 ~ 1.0）
typedef void (^DownloadProgressBlock)(double progress);

/// 下载完成回调（filePath: 文件保存路径）
typedef void (^DownloadCompletionBlock)(BOOL success, NSString * _Nullable filePath, NSError * _Nullable error);

/// 下载文件（带进度 + 保存到沙盒 Documents）
- (void)downloadFileFromURL:(NSString *)urlString
                   progress:(DownloadProgressBlock _Nullable)progressBlock
                 completion:(DownloadCompletionBlock)completion;

@end
```

> **头文件说明**：头文件只声明对外接口，四个方法签名和原生版**完全一致**，所以调用方（ViewController）一行都不用改。

---

## 三、NetworkManager.m — 实现文件（核心改动）

```objc
//
//  NetworkManager.m
//

#import "NetworkManager.h"

// ★ 改动标记 ⚠️：引入 AFNetworking（原来只 import Foundation）
#import <AFNetworking/AFNetworking.h>

/// ─── 服务器基础地址 ───
static NSString * const kBaseURL = @"http://10.17.66.196:8086";


// ★★ 关键新增 ⚠️：类扩展里声明 manager 属性（原版没有这行！）
// 原版里 session 是每个方法内部临时创建的，所以不需要成员变量
// 但 AFN 版 manager 需要在 init 里配置一次（JSON 序列化器），然后所有方法共用，
// 所以必须存成属性 → 编译器自动生成 _manager 成员变量
@interface NetworkManager ()
@property (nonatomic, strong) AFHTTPSessionManager *manager;
@end


@implementation NetworkManager

#pragma mark - 单例

// ============================================================
// 单例实现（和原生版完全一样，不用改）
// ============================================================
+ (instancetype)sharedManager {
    static NetworkManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });
    return instance;
}

// ─── 私有初始化方法 ───
- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        // ★ 改动标记 ⚠️【关键】：
        // 原生版这里注释说"后续会创建 NSURLSession"
        // AFN 版不再需要自己建 session —— AFHTTPSessionManager 内部自动创建了
        // 这里只需持有 manager，并配置成 JSON 序列化（服务器吃 JSON body）

        _manager = [AFHTTPSessionManager manager];
        _manager.requestSerializer = [AFJSONRequestSerializer serializer];   // ★ POST 必须用 JSON，否则 500
    }
    return self;
}

// ─── 禁用公开的 init（和原生版一样）───
- (instancetype)init {
    @throw [NSException exceptionWithName:@"Singleton"
                                   reason:@"请使用 +sharedManager 获取单例，不要用 [[NetworkManager alloc] init]"
                                 userInfo:nil];
}


#pragma mark - 获取用户列表（GET /user/users）

// ============================================================
// GET /user/users — 从服务器拉取所有用户
// ============================================================
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/users", kBaseURL];

    NSLog(@"[NetworkManager] 发起 GET 请求：%@", url);

    [self.manager GET:url
           parameters:nil
              headers:nil          // 4.x 必须有，无自定义头传 nil
             progress:nil          // 不监听进度
              success:^(NSURLSessionDataTask *task, id response) {
        // ★ AFNetworking 已自动：解析 JSON、切回主线程
        // response 已经是解析好的 NSDictionary

        // ---- 业务 code 校验（AFNetworking 不自动做，必须自己写）----
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSLog(@"[NetworkManager] 业务错误：code=%ld, message=%@", (long)code, msg);
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizError);
            return;
        }

        // ---- 转成 User 对象数组 ----
        NSArray<User *> *users = [User usersFromArray:response[@"data"]];
        NSLog(@"[NetworkManager] 成功获取 %lu 个用户", (unsigned long)users.count);

        if (completion) completion(YES, users, nil);
    }
              failure:^(NSURLSessionDataTask *task, NSError *error) {
        // 网络层错误（连不上、超时等），AFNetworking 已经切回主线程
        NSLog(@"[NetworkManager] 请求失败：%@", error.localizedDescription);
        if (completion) completion(NO, nil, error);
    }];
}


#pragma mark - 新增用户（POST /user/save）

// ============================================================
// POST /user/save — 向服务器新增一个用户
// ★ 必须传 JSON 序列化后的参数（manager 已在 init 里配置了 JSON serializer）
// ============================================================
- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSDictionary *params = @{
        @"name": name ?: @"",   // 兜底空字符串，和原生版一致
        @"age":  age  ?: @""
    };

    NSLog(@"[NetworkManager] 发起 POST 请求：%@ Body：%@", url, params);

    [self.manager POST:url
            parameters:params
               headers:nil          // 4.x 必须有
              progress:nil          // 不上传
               success:^(NSURLSessionDataTask *task, id response) {
        // ---- 业务 code 校验（自己写）----
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSLog(@"[NetworkManager] POST 业务错误：code=%ld, message=%@", (long)code, msg);
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizError);
            return;
        }
        NSLog(@"[NetworkManager] POST 新增用户成功");
        if (completion) completion(YES, response, nil);
    }
               failure:^(NSURLSessionDataTask *task, NSError *error) {
        NSLog(@"[NetworkManager] POST 请求失败：%@", error.localizedDescription);
        if (completion) completion(NO, nil, error);
    }];
}


#pragma mark - 修改用户（GET /user/update?id=&name=&age=）

// ============================================================
// 修改用户 — ★ 注意服务器接口是 GET（不是 HTTP PUT/POST）
// 参数走 URL 上的 ?id=&name=&age=
// AFNetworking 的 GET 传 parameters 会自动拼到 URL 上
// ============================================================
- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/update", kBaseURL];
    NSDictionary *params = @{
        @"id":   userId ?: @"",
        @"name": name   ?: @"",
        @"age":  age    ?: @""
    };

    NSLog(@"[NetworkManager] 发起修改请求：%@", url);

    [self.manager GET:url
           parameters:params
              headers:nil
             progress:nil
              success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSLog(@"[NetworkManager] 修改业务错误：code=%ld, message=%@", (long)code, msg);
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizError);
            return;
        }
        NSLog(@"[NetworkManager] 修改用户成功");
        if (completion) completion(YES, response, nil);
    }
              failure:^(NSURLSessionDataTask *task, NSError *error) {
        NSLog(@"[NetworkManager] 修改请求失败：%@", error.localizedDescription);
        if (completion) completion(NO, nil, error);
    }];
}


#pragma mark - 删除用户（GET /user/delete?id=）

// ============================================================
// 删除用户 — ★ 服务器接口是 GET /user/delete?id=（不是 HTTP DELETE）
// ============================================================
- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/delete", kBaseURL];
    NSDictionary *params = @{@"id": userId ?: @""};

    NSLog(@"[NetworkManager] 发起删除请求：%@", url);

    [self.manager GET:url
           parameters:params
              headers:nil
             progress:nil
              success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSLog(@"[NetworkManager] 删除业务错误：code=%ld, message=%@", (long)code, msg);
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizError);
            return;
        }
        NSLog(@"[NetworkManager] 删除用户成功");
        if (completion) completion(YES, response, nil);
    }
              failure:^(NSURLSessionDataTask *task, NSError *error) {
        NSLog(@"[NetworkManager] 删除请求失败：%@", error.localizedDescription);
        if (completion) completion(NO, nil, error);
    }];
}


#pragma mark - 下载文件

// ============================================================
// 下载文件到沙盒 Documents
// ★ AFNetworking 版：用 downloadTaskWithRequest:progress:destination:completionHandler:
//   比原生版的 3 个 delegate 方法简单得多（AFN 帮你处理了临时文件移动）
// ============================================================
- (void)downloadFileFromURL:(NSString *)urlString
                   progress:(DownloadProgressBlock)progressBlock
                 completion:(DownloadCompletionBlock)completion {

    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        if (completion) completion(NO, nil, [NSError errorWithDomain:@"NetworkManager" code:-1 userInfo:@{NSLocalizedDescriptionKey:@"URL 无效"}]);
        return;
    }

    NSLog(@"[下载] 开始下载：%@", urlString);

    NSURLRequest *request = [NSURLRequest requestWithURL:url];

    // ★ AFNetworking 的下载方法：
    //    progress: 下载进度，AFN 自动在子线程回调（含进度）
    //    destination: 返回"下载完成后把文件放到哪个路径"（AFN 自动移动临时文件）
    //    completionHandler: 下载结束回调（filePath = 你 destination 返回的路径）
    NSURLSessionDownloadTask *downloadTask =
        [self.manager downloadTaskWithRequest:request
                                     progress:^(NSProgress *downloadProgress) {
        // ---- 下载进度回调（子线程）----
        double p = downloadProgress.fractionCompleted;   // 0.0 ~ 1.0
        NSLog(@"[下载进度] %.1f%%", p * 100);

        // 进度回调也不在主线程，切回去再调 Block
        dispatch_async(dispatch_get_main_queue(), ^{
            if (progressBlock) progressBlock(p);
        });
    }
                                  destination:^NSURL *(NSURL *targetPath, NSURLResponse *response) {
        // ---- 决定"把这个文件放到哪" ----
        // 返回一个沙盒 Documents 里的完整路径（AFN 自动把临时文件移过来）

        // 1. 取文件名：优先 suggestedFilename（系统从 URL/响应头判断的文件名）
        NSString *filename = response.suggestedFilename;
        if (!filename.length) {
            // 兜底：时间戳文件名
            filename = [NSString stringWithFormat:@"download_%@.file", @([[NSDate date] timeIntervalSince1970])];
        }

        // 2. 拼到 Documents 目录下
        NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
            NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
        NSString *destPath = [documentsPath stringByAppendingPathComponent:filename];

        NSLog(@"[下载完成] 保存到：%@", destPath);
        return [NSURL fileURLWithPath:destPath];
    }
                            completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
        // ---- 下载结束（成功或失败）----
        // ★ AFN 已经在主线程回调（manager 默认主线程完成回调）
        if (error) {
            NSLog(@"[下载失败] %@", error.localizedDescription);
            if (completion) completion(NO, nil, error);
        } else {
            NSLog(@"[文件保存成功] %@", filePath.path);
            if (completion) completion(YES, filePath.path, nil);
        }
    }];

    [downloadTask resume];   // 别忘了启动
}

@end
```

---

## 四、改动对照总结（原生版 → AFN 版）

### 4.0 新增成员变量 `_manager`（最关键）

| | 原生版 | AFN 版 |
|------|--------|--------|
| 有没有 manager 属性 | 没有（session 是方法内临时建的） | **有**：`@interface NetworkManager ()` 里 `@property AFHTTPSessionManager *manager` |

**为什么 AFN 版必须有这个属性？**
- 原版每个方法内部 `[NSURLSession sharedSession]` 即拿即用，不用存
- AFN 版 manager 要在 init 里配置一次（JSON 序列化器），然后**五个方法（含下载）共用同一个**，所以必须存成属性
- 另外原版还有 `downloadSession`、`downloadProgressBlock`、`downloadCompletionBlock`、`downloadResumeData` 等属性，**AFN 版全部不需要**——因为 AFN 用 Block 回调，不用 delegate，也不用手动存回调 Block

### 4.5 下载方法对比（原版 3 个 delegate 方法 → AFN 1 个方法）

| | 原生版 | AFN 版 |
|------|--------|--------|
| 建下载 session | 单独建 `downloadSession`，设 delegate = self | 不需要，共用 `self.manager` |
| 进度回调 | `URLSession:downloadTask:didWriteData:`（delegate） | `downloadTaskWithRequest:progress:` 的 Block |
| 完成回调 | `didFinishDownloadingToURL:`（delegate）+ 手动移文件 | `destination:` Block（返回路径，AFN 自动移文件） |
| 失败回调 | `didCompleteWithError:`（delegate） | `completionHandler:`（error 非空即失败） |
| 手动存回调 Block | `_downloadProgressBlock` / `_downloadCompletionBlock` | 不需要，Block 直接用参数 |
| 代码量 | 约 130 行（含 3 个 delegate 方法） | 约 60 行 |

**AFN 下载最大的简化**：原版要建独立下载 session + 实现 3 个 delegate 方法 + 手动移临时文件 + 手动存回调 Block。AFN 版一个 `downloadTaskWithRequest:progress:destination:completionHandler:` 全搞定，`destination:` 决定保存路径，AFN 自动处理临时文件移动。
- `@property manager` → 编译器生成 `_manager` 成员变量 → 四个请求方法里 `self.manager` 才能用

### 4.1 init 初始化

| | 原生版 | AFN 版 |
|------|--------|--------|
| 建 session | 注释"后续会创建 NSURLSession" | **不再需要**，`_manager = [AFHTTPSessionManager manager]` 内部自带 |
| 序列化器 | 无 | `_manager.requestSerializer = [AFJSONRequestSerializer serializer]` |

### 4.2 GET 请求

| 原生手写 | AFN 自动 |
|---------|---------|
| `NSMutableURLRequest` 创建 + `HTTPMethod` | `GET:` 方法内部自动 |
| `[NSURLSession sharedSession]` | manager 自带 |
| 3 层检查（error/空/JSON解析） | 归并进 `failure:` 回调 |
| `dispatch_async` 切主线程 × 4 | success/failure 默认主线程 |
| `[task resume]` | 自动 |
| **业务 code 校验** | **❌ 自己写**（AFN 不懂你的 code 字段） |

### 4.3 POST 请求

| 原生手写 | AFN 自动 |
|---------|---------|
| `Content-Type: application/json` | 需提前配 `AFJSONRequestSerializer` |
| `NSJSONSerialization` 序列化 bodyDict | 传 `NSDictionary` 自动 |
| 其它同上 | 同上 |

### 4.4 核心差异一句话

**AFNetworking 帮你省掉的是"网络层"脏活（建请求、解析 JSON、切线程、错误归并），但"业务层"的活（code != 200 判断、data 转 User）还是你自己的。** —— 因为 AFNetworking 不知道你服务器返回格式长什么样。

---

## 五、几个关键坑（必背）

1. **POST 报 500**：必须 `manager.requestSerializer = [AFJSONRequestSerializer serializer]`，否则默认表单编码，服务器吃不了。
2. **删除/修改报 500**：你服务器是 `GET /user/delete?id=`、`GET /user/update?id=`，**不是** HTTP DELETE/PUT → 全部用 `GET:` 传 parameters。
3. **`headers:nil` 必须有**（AFNetworking 4.x 比 3.x 多这个参数），参数顺序固定：`url → parameters → headers → progress → success → failure`。
4. **业务 code 校验**：只判断 HTTP 状态码还不够，要自己检查 JSON 里的 `code != 200`。

---

## 六、头文件声明部分（.h 不用大改）

原生版的 `.h` 只声明了 `+sharedManager` 和 `fetchUsersWithCompletion:`。如果你还写了 POST/修改/删除的声明（参考 03/04 文档），保持原样即可——**方法签名一字不改**，ViewController 无感知。

---

**总结：只用改 Podfile + NetworkManager.h/.m 两个文件，其余全部不用动。所有业务 code 校验逻辑保留，和原生版行为一致。**
