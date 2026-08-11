# 04 — NetworkManager 扩展：修改用户 + 删除用户（GET + URL 参数）

> 第四步：两个方法都是 GET 请求，和昨天获取列表一样。唯一的区别是 URL 后面要拼接参数。
> 本步仅新增一个知识点：URL 编码（中文参数必须先编码才能拼进 URL）。

---

## 一、先搞懂 URL 编码（本步唯一新概念）

### 为什么需要编码？

URL 有严格的字符限制——中文、空格、特殊符号不能直接出现在 URL 里，必须先"翻译"成 URL 认识的格式。

```
原始中文字符："张三"
直接拼进 URL：  http://server/api/update?name=张三    ← ❌ 请求发不出去
编码后再拼：    http://server/api/update?name=%E5%BC%A0%E4%B8%89  ← ✅ 服务器能识别
```

苹果提供了专门的方法来做这件事：

```objc
NSString *rawName = @"张三";
NSString *encodedName = [rawName stringByAddingPercentEncodingWithAllowedCharacters:
                         [NSCharacterSet URLQueryAllowedCharacterSet]];
// encodedName 的值：@"%E5%BC%A0%E4%B8%89"
```

**口诀：所有拼进 URL 参数的字符串，只要可能包含中文或特殊符号，都先编码。**

### 这个方法和我们之前在 User 类里用的 `stringWithFormat:` 完全不是一回事

```
stringWithFormat:         → 把任意类型格式化成字符串（类型转换）
stringByAddingPercentEncoding... → 把字符串转成 URL 安全的格式（编码）
```

---

## 二、NetworkManager.h — 新增两个声明

```objc
#pragma mark - 用户接口

/// 1. 获取用户列表 — GET /user/users
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;

/// 2. 新增用户 — POST /user/save（JSON Body）
- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion;

/// 3. 修改用户 — GET /user/update?id=xxx&name=xxx&age=xxx    ← 新增
- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion;

/// 4. 删除用户 — GET /user/delete?id=xxx                     ← 新增
- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion;
```

---

## 三、NetworkManager.m — 新增实现

在 POST 方法后面加以下两段代码：

```objc
#pragma mark - 3. 修改用户（GET + URL 参数）

// ============================================================
// GET /user/update?id=xxx&name=xxx&age=xxx
// 通过 URL 参数告诉服务器：改哪个用户、改成什么
// ============================================================
- (void)updateUserWithId:(NSString *)userId
                    name:(NSString *)name
                     age:(NSString *)age
              completion:(NetworkCompletionBlock)completion {

    // ─── 第 1 步：对三个参数分别进行 URL 编码 ───
    // ★ 这是本步唯一的新知识点 ★
    // 如果用户名叫"张三"，编码后变成 "%E5%BC%A0%E4%B8%89"
    NSString *encodedName = [name stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];
    NSString *encodedAge  = [age stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];
    NSString *encodedId   = [userId stringByAddingPercentEncodingWithAllowedCharacters:
                             [NSCharacterSet URLQueryAllowedCharacterSet]];

    // ─── 第 2 步：把编码后的参数拼接到 URL 上 ───
    NSString *urlString = [NSString stringWithFormat:@"%@/user/update?id=%@&name=%@&age=%@",
                           kBaseURL, encodedId, encodedName, encodedAge];
    NSURL *url = [NSURL URLWithString:urlString];

    // ─── 第 3 步：创建请求（和昨天获取列表完全一样）───
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    NSLog(@"[NetworkManager] 发起修改请求：%@", urlString);

    // ─── 第 4 步：发起请求（以下和 GET 获取列表完全一样的套路）───
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

            // 检查网络错误
            if (error) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }

            // 检查空数据
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }

            // 解析 JSON
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

            // 检查业务状态码
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

            // 成功
            NSLog(@"[NetworkManager] 修改用户成功");
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, responseDict, nil);
            });
        }];

    [task resume];
}


#pragma mark - 4. 删除用户（GET + URL 参数）

// ============================================================
// GET /user/delete?id=xxx
// 只传一个 id，告诉服务器删哪个用户
// ============================================================
- (void)deleteUserWithId:(NSString *)userId
              completion:(NetworkCompletionBlock)completion {

    // ─── 第 1 步：URL 编码 userId ───
    NSString *encodedId = [userId stringByAddingPercentEncodingWithAllowedCharacters:
                           [NSCharacterSet URLQueryAllowedCharacterSet]];

    // ─── 第 2 步：拼接 URL ───
    NSString *urlString = [NSString stringWithFormat:@"%@/user/delete?id=%@",
                           kBaseURL, encodedId];
    NSURL *url = [NSURL URLWithString:urlString];

    // ─── 第 3 步：创建请求 ───
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    request.timeoutInterval = 30.0;

    NSLog(@"[NetworkManager] 发起删除请求：%@", urlString);

    // ─── 第 4 步：发起请求（同上，不再重复注释）───
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

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
```

---

## 四、为什么 80% 的代码是重复的？

你注意到了——修改和删除的错误处理、JSON 解析、主线程切回，和获取列表**一模一样**。这不是偷懒，这是你可以思考的一个问题：

> 四层错误检查在四个方法里各写了一遍，有没有办法只写一遍？

答案是**有**——可以抽一个通用的 `parseResponse:error:` 方法。但现在先保持这样，因为每个方法的错误检查都在眼前，方便学习。等你全搞懂了，回头重构时自然会想到怎么合并。

---

## 五、测试方法

```objc
// 测试修改
[[NetworkManager sharedManager] updateUserWithId:@"1"
                                            name:@"新名字"
                                             age:@"30"
                                      completion:^(BOOL success, id result, NSError *error) {
    if (success) {
        NSLog(@"✅ 修改成功");
    } else {
        NSLog(@"❌ 修改失败：%@", error.localizedDescription);
    }
}];

// 测试删除
[[NetworkManager sharedManager] deleteUserWithId:@"1"
                                      completion:^(BOOL success, id result, NSError *error) {
    if (success) {
        NSLog(@"✅ 删除成功");
    } else {
        NSLog(@"❌ 删除失败：%@", error.localizedDescription);
    }
}];
```

---

## 六、本步骤新增的知识点

| 编号 | 知识点 | 在哪里体现 |
|------|--------|-----------|
| 8 | URL 编码 | `stringByAddingPercentEncodingWithAllowedCharacters:` |
| 15 | GET + URL 参数拼接 | `@"...?id=%@&name=%@&age=%@"` |

本质就是把昨天学的 GET 请求复用了一遍，只是 URL 末尾多了参数。

---

## 七、NetworkManager 网络层小结

到这一步，四个接口全部完成：

| # | 方法 | HTTP | URL 数据来源 |
|---|------|------|-------------|
| 1 | 获取列表 | GET | `/user/users`（无参数） |
| 2 | 新增用户 | POST | Body（JSON） |
| 3 | 修改用户 | GET | URL 参数 `?id=&name=&age=` |
| 4 | 删除用户 | GET | URL 参数 `?id=` |

网络层完成。下一步进入 UI 层——写自定义 Cell 和列表页面，把这些接口串起来。

**下一步预告：** UserCell 自定义 TableViewCell（显示 id + 姓名 + 年龄）。
