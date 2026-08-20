# NetworkManager — 纯 AFN 版

> 增删改查 + 下载，**全部用 AFNetworking**（第三方库）。
> 增删改查用 `manager` 的 GET/POST；下载用 AFN 的 `downloadTaskWithRequest:progress:destination:completionHandler:`。
> 这是对比用版本 —— 看第三方库如何帮你省掉网络层脏活。

---

## 一、风格定位

| 项目 | 说明 |
|------|------|
| 增删改查 | `AFHTTPSessionManager` 的 `GET:` / `POST:` |
| 下载 | `downloadTaskWithRequest:progress:destination:completionHandler:` |
| 用到的类 | AFHTTPSessionManager / AFJSONRequestSerializer / AFJSONResponseSerializer |
| 需不需要第三方库 | ✅ 需要 `pod 'AFNetworking'` |

---

## 二、Podfile

```ruby
platform :ios, '12.0'
target '你的Target名' do
  use_frameworks!
  pod 'AFNetworking'
end
```

---

## 三、NetworkManager.h

```objc
//
//  NetworkManager.h
//  网络请求管理器（单例）— 纯 AFN 版
//

#import <Foundation/Foundation.h>
#import "User.h"

/// 网络请求完成回调
typedef void (^NetworkCompletionBlock)(BOOL success, id _Nullable result, NSError * _Nullable error);

/// 下载进度回调（progress: 0.0 ~ 1.0）
typedef void (^DownloadProgressBlock)(double progress);

/// 下载完成回调（filePath: 文件保存路径）
typedef void (^DownloadCompletionBlock)(BOOL success, NSString * _Nullable filePath, NSError * _Nullable error);


@interface NetworkManager : NSObject

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

## 四、NetworkManager.m

```objc
//
//  NetworkManager.m
//

#import "NetworkManager.h"
#import <AFNetworking/AFNetworking.h>   // ★ 引入 AFNetworking

/// ─── 服务器基础地址 ───
static NSString * const kBaseURL = @"http://10.17.66.196:8086";


// 类扩展：AFN 版只需要一个 manager 属性（下载也共用它）
@interface NetworkManager ()
@property (nonatomic, strong) AFHTTPSessionManager *manager;
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

- (instancetype)initPrivate {
    self = [super init];
    if (self) {
        // 建 manager，内部自动创建 NSURLSession + JSON 解析器
        _manager = [AFHTTPSessionManager manager];
        // 服务器吃 JSON body，必须用 JSON 序列化器，否则 POST 报 500
        _manager.requestSerializer = [AFJSONRequestSerializer serializer];
    }
    return self;
}

- (instancetype)init {
    @throw [NSException exceptionWithName:@"Singleton"
                                   reason:@"请使用 +sharedManager 获取单例"
                                 userInfo:nil];
}


#pragma mark - 1. 获取用户列表（GET /user/users）

- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/users", kBaseURL];
    NSLog(@"[NetworkManager] 发起 GET 请求：%@", url);

    [self.manager GET:url
           parameters:nil
              headers:nil          // 4.x 必须有，无自定义头传 nil
             progress:nil
              success:^(NSURLSessionDataTask *task, id response) {
        // AFN 已自动：解析 JSON、切回主线程。response 已是解析好的 NSDictionary

        // 业务 code 校验（AFN 不自动做，必须自己写）
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSLog(@"[NetworkManager] 业务错误：code=%ld", (long)code);
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: response[@"message"] ?: @"未知错误"}];
            if (completion) completion(NO, nil, bizError);
            return;
        }

        NSArray<User *> *users = [User usersFromArray:response[@"data"]];
        NSLog(@"[NetworkManager] 成功获取 %lu 个用户", (unsigned long)users.count);
        if (completion) completion(YES, users, nil);
    }
              failure:^(NSURLSessionDataTask *task, NSError *error) {
        NSLog(@"[NetworkManager] 请求失败：%@", error.localizedDescription);
        if (completion) completion(NO, nil, error);
    }];
}


#pragma mark - 2. 新增用户（POST /user/save + JSON Body）

- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSDictionary *params = @{
        @"name": name ?: @"",
        @"age":  age  ?: @""
    };
    NSLog(@"[NetworkManager] 发起 POST 请求：%@ Body：%@", url, params);

    [self.manager POST:url
            parameters:params
               headers:nil          // 4.x 必须有
              progress:nil
               success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSLog(@"[NetworkManager] POST 业务错误");
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: response[@"message"] ?: @"未知错误"}];
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


#pragma mark - 3. 修改用户（GET /user/update?id=&name=&age=）

- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {
    // ★ 服务器接口是 GET，传 parameters 会自动拼到 URL 上
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
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: response[@"message"] ?: @"未知错误"}];
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


#pragma mark - 4. 删除用户（GET /user/delete?id=）

- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion {
    // ★ 服务器接口是 GET，不是 HTTP DELETE
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
            NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                    code:code
                                                userInfo:@{NSLocalizedDescriptionKey: response[@"message"] ?: @"未知错误"}];
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


#pragma mark - 5. 下载文件（AFNetworking 下载）

// AFN 用 Block 一次回调，不需要 delegate、不需要存回调 Block
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

    // AFN 下载：progress 收进度、destination 返回保存路径（AFN 自动移动临时文件）、completionHandler 收结果
    NSURLSessionDownloadTask *downloadTask =
        [self.manager downloadTaskWithRequest:request
                                     progress:^(NSProgress *downloadProgress) {
        // 下载进度（子线程回调）
        double p = downloadProgress.fractionCompleted;   // 0.0 ~ 1.0
        NSLog(@"[下载进度] %.1f%%", p * 100);
        dispatch_async(dispatch_get_main_queue(), ^{
            if (progressBlock) progressBlock(p);
        });
    }
                                  destination:^NSURL *(NSURL *targetPath, NSURLResponse *response) {
        // 决定下载完把文件放哪（AFN 自动把临时文件移过来）
        NSString *filename = response.suggestedFilename;
        if (!filename.length) {
            filename = [NSString stringWithFormat:@"download_%@.file", @([[NSDate date] timeIntervalSince1970])];
        }
        NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(
            NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
        NSString *destPath = [documentsPath stringByAppendingPathComponent:filename];
        NSLog(@"[下载完成] 保存到：%@", destPath);
        return [NSURL fileURLWithPath:destPath];
    }
                            completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
        // 下载结束（AFN 默认主线程回调）
        if (error) {
            NSLog(@"[下载失败] %@", error.localizedDescription);
            if (completion) completion(NO, nil, error);
        } else {
            NSLog(@"[文件保存成功] %@", filePath.path);
            if (completion) completion(YES, filePath.path, nil);
        }
    }];

    [downloadTask resume];   // AFN 只创建不启动，必须手动 resume
}

@end
```

---

## 五、版本特点

| 方法 | 实现方式 |
|------|---------|
| fetchUsersWithCompletion: | `[self.manager GET:... success:failure:]` |
| addUserWithName:age:completion: | `[self.manager POST:...]`（JSON 序列化器已配好） |
| updateUserWithId:name:age:completion: | `[self.manager GET:...]` + parameters 自动拼 URL |
| deleteUserWithId:completion: | `[self.manager GET:...]` + parameters 自动拼 URL |
| downloadFileFromURL:progress:completion: | `[self.manager downloadTaskWithRequest:progress:destination:completionHandler:]` |

**核心特点：**
- 依赖 AFNetworking（Podfile 加 `pod 'AFNetworking'`）
- 代码量骤减：AFN 自动做 JSON 解析、切主线程、错误归并、创建 session
- **唯一自己写的**：业务 `code != 200` 校验（AFN 不知道你服务器的 code 字段）
- 下载用 AFN 一个方法全搞定（destination 决定保存路径，AFN 自动移临时文件）

---

## 六、三版横评（供对照）

| | 纯 NSURLSession 版 | 纯 AFN 版 | 混合版 |
|------|------------------|-----------|--------|
| 增删改查 | NSURLSession + dataTask | AFN GET/POST | AFN GET/POST |
| 下载 | NSURLSessionDownloadTask + delegate | AFN downloadTask | NSURLSessionDownloadTask + delegate |
| 代码量 | 最多 | 最少 | 居中 |
| 依赖第三方库 | 否 | 是 | 是 |
| 适用 | 学原理、原生要求 | 求快、求简 | 你考核要的（请求 AFN + 原生下载） |
