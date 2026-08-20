# NetworkManager — 纯 NSURLSession 版

> 增删改查 + 下载，**全部用苹果原生 NSURLSession**，不使用任何第三方库。
> 整合自原 02/03/04/07 四步文档，合并成一份完整可用的最终版。

---

## 一、风格定位

| 项目 | 说明 |
|------|------|
| 增删改查 | `NSURLSession` + `dataTaskWithRequest:completionHandler:` |
| 下载 | `NSURLSessionDownloadTask` + 3 个 delegate 方法 |
| 用到的类 | NSURLSession / NSURLSessionDataTask / NSURLSessionDownloadTask / NSJSONSerialization |
| 需不需要第三方库 | ❌ 不需要，纯系统框架 |

---

## 二、NetworkManager.h

```objc
//
//  NetworkManager.h
//  网络请求管理器（单例）— 纯 NSURLSession 版
//

#import <Foundation/Foundation.h>
#import "User.h"

/// 网络请求完成回调
///   success: 是否成功
///   result:  成功时是 NSArray<User *> 数组，失败时是 nil
///   error:   失败时是错误信息，成功时是 nil
typedef void (^NetworkCompletionBlock)(BOOL success, id _Nullable result, NSError * _Nullable error);

/// 下载进度回调（progress: 0.0 ~ 1.0）
typedef void (^DownloadProgressBlock)(double progress);

/// 下载完成回调（filePath: 文件保存路径）
typedef void (^DownloadCompletionBlock)(BOOL success, NSString * _Nullable filePath, NSError * _Nullable error);


@interface NetworkManager : NSObject

/// 获取单例对象（全局唯一）
+ (instancetype)sharedManager;

/// 获取用户列表 — GET /user/users
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;

/// 新增用户 — POST /user/save（JSON Body）
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

/// 下载文件（带进度 + 保存到沙盒 Documents）
- (void)downloadFileFromURL:(NSString *)urlString
                   progress:(DownloadProgressBlock _Nullable)progressBlock
                 completion:(DownloadCompletionBlock)completion;

@end
```

---

## 三、NetworkManager.m

```objc
//
//  NetworkManager.m
//

#import "NetworkManager.h"

/// ─── 服务器基础地址 ───
static NSString * const kBaseURL = @"http://10.17.66.196:8086";


// 类扩展：下载靠 delegate 回调，需要签协议 + 存回调 Block + 建下载 session
@interface NetworkManager () <NSURLSessionDownloadDelegate>

@property (nonatomic, strong) NSURLSession *downloadSession;            // 下载专用 session（delegate = self）
@property (nonatomic, copy) DownloadProgressBlock downloadProgressBlock;      // 存下载进度回调
@property (nonatomic, copy) DownloadCompletionBlock downloadCompletionBlock;  // 存下载完成回调
@property (nonatomic, strong) NSData *downloadResumeData;                      // 断点续传数据（可选）

@end


@implementation NetworkManager

#pragma mark - 单例

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
        // 建下载专用 session，delegate = self，接收下载回调
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _downloadSession = [NSURLSession sessionWithConfiguration:config
                                                         delegate:self
                                                    delegateQueue:nil];  // nil = 系统自动开串行队列
    }
    return self;
}

// ─── 禁用公开 init，防止破坏单例 ───
- (instancetype)init {
    @throw [NSException exceptionWithName:@"Singleton"
                                   reason:@"请使用 +sharedManager 获取单例"
                                 userInfo:nil];
}


#pragma mark - 1. 获取用户列表（GET /user/users）

- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {
    NSString *urlString = [NSString stringWithFormat:@"%@/user/users", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];

    // 创建请求
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    NSLog(@"[NetworkManager] 发起 GET 请求：%@", urlString);

    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

            // 网络回调在后台线程，改 UI 必须回主线程（completion 里处理）

            // 1. 网络错误
            if (error) {
                NSLog(@"[NetworkManager] 请求失败：%@", error.localizedDescription);
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }

            // 2. 空数据
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }

            // 3. 解析 JSON
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

            // 4. 业务 code 校验（服务器统一返回 {"code": 200, "data": [...]}）
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }

            // 5. 转成 User 对象数组
            NSArray<User *> *users = [User usersFromArray:responseDict[@"data"]];
            NSLog(@"[NetworkManager] 成功获取 %lu 个用户", (unsigned long)users.count);

            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, users, nil);
            });
        }];

    [task resume];   // 别忘了启动
}


#pragma mark - 2. 新增用户（POST /user/save + JSON Body）

- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {
    NSString *urlString = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];

    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    request.timeoutInterval = 30.0;
    // 告诉服务器"发的数据是 JSON 格式"
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];

    // 参数打包成 JSON 塞进 body
    NSDictionary *bodyDict = @{
        @"name": name ?: @"",
        @"age":  age  ?: @""
    };
    NSError *jsonError = nil;
    NSData *bodyData = [NSJSONSerialization dataWithJSONObject:bodyDict
                                                       options:0
                                                         error:&jsonError];
    if (jsonError) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(NO, nil, jsonError);
        });
        return;
    }
    request.HTTPBody = bodyData;

    NSLog(@"[NetworkManager] 发起 POST 请求：%@ Body：%@", urlString, bodyDict);

    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            // 四层错误检查（和 GET 一样）：网络错误 → 空数据 → JSON解析 → 业务 code
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }
            NSError *parseError = nil;
            NSDictionary *responseDict = [NSJSONSerialization JSONObjectWithData:data
                                                                         options:0
                                                                           error:&parseError];
            if (parseError) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, parseError);
                });
                return;
            }
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }
            NSLog(@"[NetworkManager] POST 新增用户成功");
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, responseDict, nil);
            });
        }];

    [task resume];
}


#pragma mark - 3. 修改用户（GET /user/update?id=&name=&age=）

- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {
    // ★ 服务器接口是 GET，参数拼 URL：?id=&name=&age=
    NSString *urlString = [NSString stringWithFormat:@"%@/user/update?id=%@&name=%@&age=%@",
                           kBaseURL, userId, name, age];
    // URL 里的中文会乱码？用 stringByAddingPercentEncodingWithAllowedCharacters 转义，示例省略
    NSURL *url = [NSURL URLWithString:urlString];

    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    NSLog(@"[NetworkManager] 发起修改请求：%@", urlString);

    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            // 四层错误检查（和 GET 一样）
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }
            NSError *parseError = nil;
            NSDictionary *responseDict = [NSJSONSerialization JSONObjectWithData:data
                                                                         options:0
                                                                           error:&parseError];
            if (parseError) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, parseError);
                });
                return;
            }
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }
            NSLog(@"[NetworkManager] 修改用户成功");
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, responseDict, nil);
            });
        }];

    [task resume];
}


#pragma mark - 4. 删除用户（GET /user/delete?id=）

- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion {
    // ★ 服务器接口是 GET，不是 HTTP DELETE
    NSString *urlString = [NSString stringWithFormat:@"%@/user/delete?id=%@", kBaseURL, userId];
    NSURL *url = [NSURL URLWithString:urlString];

    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    NSLog(@"[NetworkManager] 发起删除请求：%@", urlString);

    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            // 四层错误检查（和 GET 一样）
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }
            NSError *parseError = nil;
            NSDictionary *responseDict = [NSJSONSerialization JSONObjectWithData:data
                                                                         options:0
                                                                           error:&parseError];
            if (parseError) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, parseError);
                });
                return;
            }
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }
            NSLog(@"[NetworkManager] 删除用户成功");
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, responseDict, nil);
            });
        }];

    [task resume];
}


#pragma mark - 5. 下载文件（NSURLSessionDownloadTask + delegate）

// 因为 delegate 是分多个方法回调的，所以先存好回调 Block，等 delegate 方法里再取出来调
- (void)downloadFileFromURL:(NSString *)urlString
                   progress:(DownloadProgressBlock)progressBlock
                 completion:(DownloadCompletionBlock)completion {

    // 1. 存回调 Block
    self.downloadProgressBlock = progressBlock;
    self.downloadCompletionBlock = completion;

    NSURL *url = [NSURL URLWithString:urlString];
    if (!url) {
        if (completion) completion(NO, nil, [NSError errorWithDomain:@"NetworkManager" code:-1 userInfo:@{NSLocalizedDescriptionKey:@"URL 无效"}]);
        return;
    }

    NSLog(@"[下载] 开始下载：%@", urlString);

    // 2. 创建下载任务（直接用 URL，不需要手动建 request）
    NSURLSessionDownloadTask *downloadTask = [self.downloadSession downloadTaskWithURL:url];

    // 3. 启动（原生不会自动 resume，必须手动调）
    [downloadTask resume];
}


#pragma mark - NSURLSessionDownloadDelegate（下载回调，系统自动调）

// ★ 下载进度 — 反复调用（每收一段数据调一次）
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {

    if (totalBytesExpectedToWrite <= 0) return;   // 获取不到总大小则跳过

    double progress = (double)totalBytesWritten / (double)totalBytesExpectedToWrite;
    NSLog(@"[下载进度] %.1f%%", progress * 100);

    dispatch_async(dispatch_get_main_queue(), ^{
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(progress);
        }
    });
}

// ★ 下载完成 — 文件在临时目录，移到 Documents
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location {
    // location = 系统临时文件夹路径（随时会被清）

    // 1. 取文件名
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)downloadTask.response;
    NSString *filename = httpResponse.suggestedFilename;
    if (!filename.length) {
        filename = [NSString stringWithFormat:@"download_%@.file", @([[NSDate date] timeIntervalSince1970])];
    }
    NSLog(@"[下载完成] 文件名：%@", filename);

    // 2. 目标路径 = Documents 目录 + 文件名
    NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *destPath = [documentsPath stringByAppendingPathComponent:filename];

    NSFileManager *fm = [NSFileManager defaultManager];

    // 3. 目标已存在则先删（避免 move 失败）
    if ([fm fileExistsAtPath:destPath]) {
        [fm removeItemAtPath:destPath error:nil];
    }

    // 4. 把临时文件移到 Documents
    NSError *moveError = nil;
    BOOL moved = [fm moveItemAtURL:location toURL:[NSURL fileURLWithPath:destPath] error:&moveError];

    dispatch_async(dispatch_get_main_queue(), ^{
        if (moved) {
            NSLog(@"[文件保存成功] %@", destPath);
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(YES, destPath, nil);
        } else {
            NSLog(@"[文件保存失败] %@", moveError.localizedDescription);
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(NO, nil, moveError);
        }
    });
}

// ★ 下载结束（成功或失败都调）— 用 error 区分
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error {
    if (error) {
        NSLog(@"[下载失败] %@", error.localizedDescription);
        self.downloadResumeData = error.userInfo[NSURLSessionDownloadTaskResumeData];  // 断点续传数据（可选）
        dispatch_async(dispatch_get_main_queue(), ^{
            if (self.downloadCompletionBlock) self.downloadCompletionBlock(NO, nil, error);
        });
    }
    // error == nil：成功已在 didFinishDownloadingToURL 处理，不重复调
}

@end
```

---

## 四、版本特点

| 方法 | 实现方式 |
|------|---------|
| fetchUsersWithCompletion: | `sharedSession` + `dataTaskWithRequest:` + 四层错误检查 |
| addUserWithName:age:completion: | POST + JSON Body（`dataWithJSONObject:`） |
| updateUserWithId:name:age:completion: | GET + URL 参数 `?id=&name=&age=` |
| deleteUserWithId:completion: | GET + URL 参数 `?id=` |
| downloadFileFromURL:progress:completion: | `downloadSession` + `downloadTaskWithURL:` + 3 个 delegate 方法 |

**核心特点：**
- 不依赖任何第三方库，纯系统框架
- 每个请求都要手写：建请求 → JSON 序列化 → 四层错误检查 → 切主线程 → resume
- 下载依赖 delegate（要签 `NSURLSessionDownloadDelegate`、存回调 Block、建 downloadsSession）

**文件结构：**
```
Model/     User.h/.m
Network/   NetworkManager.h/.m   （本文件）
View/      UserCell.h/.m
Controller/UserListViewController.h/.m
```
