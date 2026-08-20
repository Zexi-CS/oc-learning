# NetworkManager — 混合版（AFN 请求 + 原生下载）

> 将网络编程项目的 NetworkManager 改造成**混合实现**：
> - **数据获取（增删改查）** → **AFNetworking**（manager，用 GET/POST，简单省事）
> - **下载文件** → **原生 NSURLSessionDownloadTask + delegate**（题目要求的第一种方式）
> 核心：外部调用接口完全不变，只改内部实现。覆盖完整 5 个接口：GET 列表 / POST 新增 / GET 修改 / GET 删除 / 原生下载。

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


// ★★ 混合版类扩展 ⚠️：
//   ① manager（AFN）→ 管增删改查（数据获取）
//   ② downloadSession（原生 NSURLSession）→ 管下载（题目要求的第一种方式）
//   ③ 签 NSURLSessionDownloadDelegate 协议 → 下载靠 delegate 三个方法回调
//   ④ 两个回调 Block + resumeData → 存下载进度/完成回调，供 delegate 方法里调用（原生 delegate 需要手动存）
@interface NetworkManager () <NSURLSessionDownloadDelegate>

@property (nonatomic, strong) AFHTTPSessionManager *manager;      // AFN：增删改查用
@property (nonatomic, strong) NSURLSession *downloadSession;      // 原生：下载用

// 下载相关的回调 Block（原生 delegate 要跨方法取用，必须存成属性）
@property (nonatomic, copy) DownloadProgressBlock downloadProgressBlock;      // 存进度回调
@property (nonatomic, copy) DownloadCompletionBlock downloadCompletionBlock;  // 存完成回调
@property (nonatomic, strong) NSData *downloadResumeData;                      // 断点续传数据（可选）

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
        // ★ 混合版 ⚠️【关键】：
        //   数据获取（增删改查）→ AFNetworking（manager 内部自动建 NSURLSession）
        //   下载 → 原生 NSURLSessionDownloadTask（单独建一个 downloadSession，delegate = self）

        // ─── ① AFN：数据获取用 ───
        _manager = [AFHTTPSessionManager manager];
        _manager.requestSerializer = [AFJSONRequestSerializer serializer];   // ★ POST 必须用 JSON，否则 500

        // ─── ② 原生：下载用 ───
        //   NSURLSessionConfiguration 是底层的"配置项"（超时、缓存、证书策略等），这里用默认配置即可
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _downloadSession = [NSURLSession sessionWithConfiguration:config
                                                         delegate:self      // ★ delegate = 自己，接收下载回调
                                                    delegateQueue:nil];      // nil = 系统自动开一条串行队列
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
// ★ 混合版 ⚠️：下载用「原生 NSURLSessionDownloadTask + delegate」
//   这是题目要求的第一种方式（不是 AFNetworking 下载）
//   因为 delegate 回调会分成多个方法触发，所以要先存好回调 Block
// ============================================================
- (void)downloadFileFromURL:(NSString *)urlString
                   progress:(DownloadProgressBlock)progressBlock
                 completion:(DownloadCompletionBlock)completion {

    // ─────────────────────────────────────────────
    // 第 1 步：存好回调 Block
    //   原因：原生版下载靠 delegate 三个方法，它们不是一次调用里就能拿到回调的，
    //        进度/完成回调要等 delegate 方法触发时才用。所以先存成属性，等 delegate 里再取。
    // ─────────────────────────────────────────────
    self.downloadProgressBlock = progressBlock;
    self.downloadCompletionBlock = completion;

    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        if (completion) completion(NO, nil, [NSError errorWithDomain:@"NetworkManager" code:-1 userInfo:@{NSLocalizedDescriptionKey:@"URL 无效"}]);
        return;
    }

    NSLog(@"[下载] 开始下载：%@", urlString);

    // ── 第 2 步：创建原生下载任务（NSURLSessionDownloadTask）──
    //    注意：downloadSession 是 init 里建好的原生 session（delegate = self）
    //    下载任务用 downloadTaskWithURL:（不需要手动建 request，直接传 URL）
    NSURLSessionDownloadTask *downloadTask = [self.downloadSession downloadTaskWithURL:url];

    // ── 第 3 步：启动（原生不会自动 resume，必须手动调）──
    [downloadTask resume];
}


#pragma mark - NSURLSessionDownloadDelegate（原生下载回调，系统自动调）

// ★ 下载进度回调 — 下载过程中反复调用 ★
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten              // 本次写了多少
 totalBytesWritten:(int64_t)totalBytesWritten          // 已经写了多少
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {  // 总共多少

    if (totalBytesExpectedToWrite <= 0) return;   // 获取不到总大小，跳过

    double progress = (double)totalBytesWritten / (double)totalBytesExpectedToWrite;

    NSLog(@"[下载进度] %.1f%%", progress * 100);

    // delegate 方法在子线程调，回主线程再调 Block（更新 UI 用）
    dispatch_async(dispatch_get_main_queue(), ^{
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(progress);
        }
    });
}


// ★ 下载完成回调 — 文件在系统临时目录，需要移到 Documents ★
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location {
    //   location 是系统临时文件夹的路径（随时会被清理）

    // ---- 1. 获取文件名 ----
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)downloadTask.response;
    NSString *filename = nil;

    // 优先从 Content-Disposition 响应头里提取文件名
    NSString *contentDisposition = httpResponse.allHeaderFields[@"Content-Disposition"];
    if (contentDisposition) {
        NSRange range = [contentDisposition rangeOfString:@"filename="];
        if (range.location != NSNotFound) {
            filename = [contentDisposition substringFromIndex:range.location + range.length];
            filename = [filename stringByReplacingOccurrencesOfString:@"\"" withString:@""];
            filename = [filename stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
        }
    }

    // 备选：从 URL 最后一段取文件名
    if (!filename.length) {
        filename = downloadTask.response.suggestedFilename;
    }

    // 兜底：时间戳文件名
    if (!filename.length) {
        filename = [NSString stringWithFormat:@"download_%@.file", @([[NSDate date] timeIntervalSince1970])];
    }

    NSLog(@"[下载完成] 文件名：%@", filename);

    // ---- 2. 移动到沙盒 Documents ----
    NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *destPath = [documentsPath stringByAppendingPathComponent:filename];

    NSFileManager *fm = [NSFileManager defaultManager];

    // ---- 3. 如果目标已存在，先删掉旧的（避免 move 失败）----
    if ([fm fileExistsAtPath:destPath]) {
        [fm removeItemAtPath:destPath error:nil];
    }

    // ---- 4. 把临时文件移动到 Documents ----
    NSError *moveError = nil;
    BOOL moved = [fm moveItemAtURL:location toURL:[NSURL fileURLWithPath:destPath] error:&moveError];

    if (moved) {
        NSLog(@"[文件保存成功] %@", destPath);
        dispatch_async(dispatch_get_main_queue(), ^{
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(YES, destPath, nil);
        });
    } else {
        NSLog(@"[文件保存失败] %@", moveError.localizedDescription);
        dispatch_async(dispatch_get_main_queue(), ^{
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(NO, nil, moveError);
        });
    }
}


// ★ 下载结束（成功或失败）都调用 — 用 error 区分 ★
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error {
    // error == nil → 成功；error != nil → 失败原因

    if (error) {
        NSLog(@"[下载失败] %@", error.localizedDescription);

        // 可选的断点续传：从 error 里取出已下载数据，存起来（本次简版可忽略）
        self.downloadResumeData = error.userInfo[NSURLSessionDownloadTaskResumeData];

        dispatch_async(dispatch_get_main_queue(), ^{
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(NO, nil, error);
        });
    }
    // error == nil → 成功已经在上面的 didFinishDownloadingToURL 里处理 move，这里不需要重复调
}

@end
```

---

## 四、改动对照总结（原生版 → AFN 版）

### 4.0 混合版属性（AFN + 原生下载，两个并存）

| | 原生版 | 混合版 |
|------|--------|--------|
| 数据获取 | 每个方法内部 `[NSURLSession sharedSession]` | **有**：`@property AFHTTPSessionManager *manager` |
| 下载 | 单独建 `downloadSession` + delegate | **也有**：`@property NSURLSession *downloadSession` + 签 `NSURLSessionDownloadDelegate` |
| 下载回调 | `_downloadProgressBlock` 等 | 同样需要（delegate 要跨方法取回调） |

**类扩展需要声明的东西（和原版几乎一样，只是多了个 manager）：**

```objc
@interface NetworkManager () <NSURLSessionDownloadDelegate>
@property (nonatomic, strong) AFHTTPSessionManager *manager;   // 增删改查用
@property (nonatomic, strong) NSURLSession *downloadSession;   // 下载用
@property (nonatomic, copy) DownloadProgressBlock downloadProgressBlock;
@property (nonatomic, copy) DownloadCompletionBlock downloadCompletionBlock;
@property (nonatomic, strong) NSData *downloadResumeData;
@end
```

**为什么数据获取用 AFN、下载却用原生？** 这正是你的需求组合——增删改查是普通 JSON 接口，AFN 省网络层脏活；下载要求 NSURLSessionDownloadTask（题目指定），用原生 delegate。两个各管一段，互不干扰。

### 4.5 为什么下载用原生（不改成 AFN 下载）

**这是你的明确选择**：增删改查用 AFN，但**下载必须用题目要求的 NSURLSessionDownloadTask**。

```
错误做法：把下载也改成 AFN
  [self.manager downloadTaskWithRequest:...]  ← 这就不符合题目要求了

正确做法：下载保持原生 delegate
  [self.downloadSession downloadTaskWithURL:url] + 3 个 delegate 方法
```

| | AFN 下载（不推荐，违反题目） | 原生 NSURLSessionDownloadTask（✅ 本题要求） |
|------|------------------------|----------------------------------|
| 建下载 session | 共用 `self.manager` | 单独 `downloadSession`，delegate = self |
| 进度回调 | `progress:` Block | `URLSession:downloadTask:didWriteData:` |
| 完成回调 | `destination:` + `completionHandler:` | `didFinishDownloadingToURL:`（手动移文件） |
| 失败回调 | `completionHandler:`（error 判断） | `didCompleteWithError:` |
| 断点续传 | 需自己搞 | 原生支持（`NSURLSessionDownloadTaskResumeData`） |

**所以下载部分的代码 = 原版 07 那套（delegate 3 方法），只是嵌套在混合版 NetworkManager 里。**

### 4.1 init 初始化

| | 数据获取 | 下载 |
|------|--------|--------|
| 建什么 | `_manager = [AFHTTPSessionManager manager]` | `_downloadSession = [NSURLSession sessionWithConfiguration:delegate:self:]` |
| 配置 | `requestSerializer = [AFJSONRequestSerializer serializer]` | 默认配置即可 |

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
5. **下载必须用原生 session，别用 AFN 下载**：`[self.downloadSession downloadTaskWithURL:url]` + 3 个 delegate 方法，这才是题目要求的 NSURLSessionDownloadTask。用了 `[self.manager downloadTaskWithRequest:]` 就是 AFNetworking 下载，不符合要求。
6. **下载要存回调 Block**：原生 delegate 是分多个方法回调的，所以 `downloadFileFromURL:` 里必须先把 `progressBlock` 和 `completion` 存成属性，delegate 方法里再取出来调。类似"手动存回调 Block"。

---

## 六、头文件声明部分（.h 不用大改）

原生版的 `.h` 只声明了 `+sharedManager` 和 `fetchUsersWithCompletion:`。如果你还写了 POST/修改/删除的声明（参考 03/04 文档），保持原样即可——**方法签名一字不改**，ViewController 无感知。下载方法 `downloadFileFromURL:progress:completion:` 的两个 Block 类型（`DownloadProgressBlock` / `DownloadCompletionBlock`）声明也在 .h 里。

---

**总结：只用改 Podfile + NetworkManager.h/.m 两个文件，其余全部不用动。增删改查走 AFNetworking（manager），下载走原生 NSURLSessionDownloadTask（downloadSession + delegate），各管一段。所有业务 code 校验逻辑保留，和原生版行为一致。**
