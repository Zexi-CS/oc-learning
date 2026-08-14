# NSURLSession vs AFNetworking — 同功能代码对比

> 左边是你上周手写的原生代码，右边是 AFNetworking 等价写法。
> 对比感受一下：少了什么？什么被自动处理了？

---

## 一、GET 请求 — 获取用户列表

| | 原生 NSURLSession | AFNetworking |
|------|------------------|-------------|
| 代码行数 | **~45 行** | **~19 行** |

```objc
// ─── 原生：45 行 ───

- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {
    NSString *urlString = [NSString stringWithFormat:@"%@/user/users", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    [[self.session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }
            if (!data) {
                NSError *emptyErr = [NSError errorWithDomain:@"..." code:-1 ...];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyErr);
                });
                return;
            }
            NSError *jsonErr = nil;
            NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data ... error:&jsonErr];
            if (jsonErr) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, jsonErr);
                });
                return;
            }
            NSInteger code = [dict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = dict[@"message"] ?: @"未知错误";
                NSError *bizErr = [NSError errorWithDomain:@"..." code:code ...];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizErr);
                });
                return;
            }
            NSArray *arr = [User usersFromArray:dict[@"data"]];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, arr, nil);
            });
        }] resume];
}
```

```objc
// ─── AFNetworking：~22 行 ───

- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/users", kBaseURL];

    [[AFHTTPSessionManager manager] GET:url
                             parameters:nil
                                headers:nil            // ← 4.x 必须有，不要自定义头就传 nil
                               progress:nil            // ← 下载进度，不要就传 nil
                                success:^(NSURLSessionDataTask *task, id response) {
        // response 已经是解析好的 NSDictionary，在主线程

        // ★ 业务 code 校验 —— AFNetworking 不自动做，必须自己写
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSError *bizErr = [NSError errorWithDomain:@"Network" code:code
                                              userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizErr);
            return;
        }

        NSArray *arr = [User usersFromArray:response[@"data"]];
        if (completion) completion(YES, arr, nil);
    }
                                failure:^(NSURLSessionDataTask *task, NSError *error) {
        // 网络层错误（连不上、超时、404 等）
        if (completion) completion(NO, nil, error);
    }];
}
```

> ⚠️ **参数顺序固定**：`url → parameters → headers → progress → success → failure`，一个不能漏，不要的传 `nil` 占位。AFNetworking 4.x 比 3.x 多了 `headers` 参数，照上面写。

**AFNetworking 自动帮你做了：**
- 创建 NSURLSession（`[AFHTTPSessionManager manager]` 内部自带）
- 建 NSMutableURLRequest
- JSON 反序列化（response 已经是字典）
- 网络层错误检查（error、空 data、JSON 解析失败 → 归并进 failure）
- 切回主线程（success/failure 回调默认在主线程）

**AFNetworking 不帮你做的（必须自己写）：**
- 业务 `code` 字段校验 —— AFNetworking 只判断 HTTP 状态码（404/500），不知道你服务器 JSON 里 `code` 字段的含义，所以 `code != 200` 的判断要自己补

---

## 二、POST 请求 — 新增用户

| | 原生 NSURLSession | AFNetworking |
|------|------------------|-------------|
| 代码行数 | **~55 行** | **~8 行** |

```objc
// ─── 原生：55 行 ───

- (void)addUserWithName:(NSString *)name age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {
    NSString *urlString = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    request.timeoutInterval = 30.0;
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];

    NSDictionary *bodyDict = @{@"name": name ?: @"", @"age": age ?: @""};
    NSError *bodyErr = nil;
    NSData *bodyData = [NSJSONSerialization dataWithJSONObject:bodyDict options:0 error:&bodyErr];
    if (bodyErr) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(NO, nil, bodyErr);
        });
        return;
    }
    request.HTTPBody = bodyData;

    [[self.session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            // 四层错误检查（和 GET 完全一样，省略）
            // ...
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, dict, nil);
            });
        }] resume];
}
```

```objc
// ─── AFNetworking：~19 行 ───

- (void)addUserWithName:(NSString *)name age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSDictionary *params = @{@"name": name ?: @"", @"age": age ?: @""};

    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    // ★ 关键：默认是表单编码（application/x-www-form-urlencoded），
    //   你服务器吃 JSON body，必须换成 JSON 序列化器，否则报 500
    manager.requestSerializer = [AFJSONRequestSerializer serializer];

    [manager POST:url
       parameters:params
          headers:nil            // ← 4.x 必须有
         progress:nil            // ← 上传进度，不要就传 nil
          success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSError *bizErr = [NSError errorWithDomain:@"Network" code:code
                                              userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizErr);
            return;
        }
        if (completion) completion(YES, response, nil);
    }
          failure:^(NSURLSessionDataTask *task, NSError *error) {
        if (completion) completion(NO, nil, error);
    }];
}
```

> ⚠️ **POST 报 500 的坑**：AFNetworking 的 POST 默认用表单编码（`AFHTTPRequestSerializer`），而原生代码设的是 `Content-Type: application/json`。你服务器接口吃 JSON，就必须 `manager.requestSerializer = [AFJSONRequestSerializer serializer]`，否则请求体格式不对，服务器返回 500。

**AFNetworking 自动帮你做了：**
- NSJSONSerialization 序列化 bodyDict（需先换成 AFJSONRequestSerializer）
- 检查序列化错误
- 同上 GET 的所有自动处理

---

## 三、GET 带参数 — 修改用户

| | 原生 NSURLSession | AFNetworking |
|------|------------------|-------------|
| 代码行数 | **~50 行** | **~6 行** |

```objc
// ─── 原生：50 行（URL 编码 + 拼接 + 请求） ───

- (void)updateUserWithId:(NSString *)userId name:(NSString *)name age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {
    NSString *encodedName = [name stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];
    NSString *encodedAge = [age stringByAddingPercentEncodingWithAllowedCharacters:
                            [NSCharacterSet URLQueryAllowedCharacterSet]];
    NSString *urlString = [NSString stringWithFormat:@"%@/user/update?id=%@&name=%@&age=%@",
                           kBaseURL, userId, encodedName, encodedAge];
    // ... 建 request、发请求、四层检查（和 GET 一样）
}
```

```objc
// ─── AFNetworking：~16 行 ───

- (void)updateUserWithId:(NSString *)userId name:(NSString *)name age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/update", kBaseURL];
    NSDictionary *params = @{@"id": userId, @"name": name, @"age": age};

    [[AFHTTPSessionManager manager] GET:url
                             parameters:params
                                headers:nil
                               progress:nil
                                success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSError *bizErr = [NSError errorWithDomain:@"Network" code:code
                                              userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizErr);
            return;
        }
        if (completion) completion(YES, response, nil);
    }
                                failure:^(NSURLSessionDataTask *task, NSError *error) {
        if (completion) completion(NO, nil, error);
    }];
}
```

**AFNetworking 自动帮你做了：**
- URL 参数编码（中文自动转 `%E5%BC%A0`）
- `?id=1&name=张三&age=25` 拼接
- 同上所有自动处理

---

## 四、删除用户

| | 原生 NSURLSession | AFNetworking |
|------|------------------|-------------|
| 代码行数 | **~45 行** | **~5 行** |

```objc
// ─── 原生：45 行 ───

- (void)deleteUserWithId:(NSString *)userId completion:(NetworkCompletionBlock)completion {
    NSString *urlString = [NSString stringWithFormat:@"%@/user/delete?id=%@", kBaseURL, userId];
    // ... 建 request、发请求、四层检查（和 GET 一样）
}
```

```objc
// ─── AFNetworking：~14 行 ───

- (void)deleteUserWithId:(NSString *)userId completion:(NetworkCompletionBlock)completion {
    NSString *url = [NSString stringWithFormat:@"%@/user/delete", kBaseURL];
    NSDictionary *params = @{@"id": userId};

    // ★ 注意：你服务器接口是 GET /user/delete?id=xxx，不是 HTTP DELETE 方法
    //   所以这里用 GET，不是 DELETE
    [[AFHTTPSessionManager manager] GET:url
                             parameters:params
                                headers:nil
                               progress:nil
                                success:^(NSURLSessionDataTask *task, id response) {
        NSInteger code = [response[@"code"] integerValue];
        if (code != 200) {
            NSString *msg = response[@"message"] ?: @"未知错误";
            NSError *bizErr = [NSError errorWithDomain:@"Network" code:code
                                              userInfo:@{NSLocalizedDescriptionKey: msg}];
            if (completion) completion(NO, nil, bizErr);
            return;
        }
        if (completion) completion(YES, response, nil);
    }
                                failure:^(NSURLSessionDataTask *task, NSError *error) {
        if (completion) completion(NO, nil, error);
    }];
}
```

> ⚠️ **删除报 500 的坑**：你原生删除用的是 `GET /user/delete?id=xxx`（HTTP 方法 = GET），不是 HTTP 的 DELETE 方法。改 AFNetworking 时如果用了 `DELETE:`，方法不对，服务器直接 500。必须用 `GET:` 传 parameters（AFNetworking 自动拼到 URL 上）。

---

## 五、总结：AFNetworking 替你省掉了什么

| 原生的操作 | AFNetworking 是否自动 |
|-----------|---------------------|
| `NSMutableURLRequest` 创建 | ✅ 自动 |
| `HTTPMethod` 设置 | ✅ 自动（GET/POST/DELETE 对应） |
| `Content-Type: application/json` | ✅ 自动（POST 时） |
| JSON 序列化（客户端送 Body） | ✅ 自动（传 NSDictionary 即可） |
| JSON 反序列化（服务器返回） | ✅ 自动（response 已是字典） |
| URL 参数编码（中文→%XX） | ✅ 自动 |
| 四层错误检查 | ✅ 自动归并进 success/failure |
| `dispatch_async` 切主线程 | ✅ 成功/失败回调默认主线程 |
| `[task resume]` | ✅ 自动 |
| `timeoutInterval` 设置 | ✅ 可配（默认 60s） |
| 业务 `code` 字段校验（code != 200） | ❌ 需自己写（AFNetworking 只判断 HTTP 状态码，不懂你 JSON 里的业务码） |

---

## 六、什么时候用哪个

| 场景 | 用什么 |
|------|--------|
| 学习网络底层、理解 HTTP 协议 | 原生 NSURLSession |
| 所有正式项目、快速开发 | AFNetworking |
| 需要极简依赖、App 包体严格限制 | 原生 NSURLSession |
| 下载大文件+进度的复杂场景 | AFNetworking 也能做，但用原生更灵活 |
