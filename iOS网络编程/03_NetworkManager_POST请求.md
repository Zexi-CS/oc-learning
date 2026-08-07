# 03 — NetworkManager 扩展：POST 请求（新增用户）

> 第三步：在 NetworkManager 里新增 POST 方法，向服务器提交数据创建新用户。
> 只新增一个方法 + 头文件声明，不动昨天已经写好的代码。

---

## 一、POST 和 GET 的本质区别（先理解，再看代码）

```
GET（昨天写的）：
   ┌──────────┐          GET /user/users            ┌──────────┐
   │  客户端   │  ─────────────────────────────────▶  │  服务器   │
   └──────────┘     （URL 上没参数，纯查数据）          └──────────┘


POST（今天写的）：
   ┌──────────┐       POST /user/save               ┌──────────┐
   │  客户端   │  ─────────────────────────────────▶  │  服务器   │
   │          │     Body: {"name":"张三","age":"25"}  │          │
   └──────────┘     （数据装在请求体里，不在 URL 上）    └──────────┘
```

代码上只差三处：

| 差异点 | GET | POST |
|--------|-----|------|
| HTTPMethod | `@"GET"` | `@"POST"` |
| 请求头 | 不需要 | `Content-Type: application/json` |
| 数据位置 | 拼 URL 后面 | `request.HTTPBody = JSON 数据` |

---

## 二、NetworkManager.h — 新增声明

在昨天头文件的 `#pragma mark` 区域里，**加一行声明**：

```objc
#pragma mark - 用户接口

/// 1. 获取用户列表 — GET /user/users
- (void)fetchUsersWithCompletion:(NetworkCompletionBlock)completion;

/// 2. 新增用户 — POST /user/save（JSON Body）     ← 新增这行
- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion;
```

---

## 三、NetworkManager.m — 新增实现

在昨天 .m 文件的 `#pragma mark - 1. 获取用户列表（GET）` 后面，**加以下代码**：

```objc
#pragma mark - 2. 新增用户（POST + JSON Body）

// ============================================================
// POST /user/save — 向服务器新增一个用户
// ============================================================
- (void)addUserWithName:(NSString *)name
                    age:(NSString *)age
             completion:(NetworkCompletionBlock)completion {

    // ─── 第 1 步：构造 URL ───
    NSString *urlString = [NSString stringWithFormat:@"%@/user/save", kBaseURL];
    NSURL *url = [NSURL URLWithString:urlString];

    // ─── 第 2 步：创建【可变】请求对象 ───
    // GET 用 NSURLRequest（不可变），POST 必须用 NSMutableURLRequest（可变）
    // 因为需要改 HTTPMethod 和加 HTTPBody
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";            // ★ 设为 POST（昨天 GET 写的是 @"GET"）
    request.timeoutInterval = 30.0;

    // ★ 设置请求头：告诉服务器"我发的数据是 JSON 格式"
    // 服务器收到后就知道该怎么解析
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];

    // ─── 第 3 步：把参数打包成 JSON，塞进请求体 ───
    // 服务器约定接收的格式是 JSON 对象，例如：
    // {"name": "张三", "age": "25"}
    NSDictionary *bodyDict = @{
        @"name": name ?: @"",    // 兜底：万一传了 nil，用空字符串
        @"age":  age  ?: @""
    };

    // NSJSONSerialization 的另一面：把 OC 字典 → JSON 二进制
    // 昨天用的是 JSONObjectWithData（二进制 → 字典），今天反过来
    NSError *jsonError = nil;
    NSData *bodyData = [NSJSONSerialization dataWithJSONObject:bodyDict
                                                       options:0
                                                         error:&jsonError];
    if (jsonError) {
        // JSON 序列化失败（几乎不可能，但防御一下）
        NSLog(@"[NetworkManager] JSON 序列化失败：%@", jsonError);
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) completion(NO, nil, jsonError);
        });
        return;
    }

    // 把 JSON 二进制数据赋给请求体
    request.HTTPBody = bodyData;

    NSLog(@"[NetworkManager] 发起 POST 请求：%@ Body：%@", urlString, bodyDict);

    // ─── 第 4 步：创建 session 并发起请求 ───
    // 和 GET 完全一样的流程
    NSURLSession *session = [NSURLSession sharedSession];

    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

            // ---- 4.1：检查网络错误 ----
            if (error) {
                NSLog(@"[NetworkManager] POST 请求失败：%@", error.localizedDescription);
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, error);
                });
                return;
            }

            // ---- 4.2：检查数据是否为空 ----
            if (!data || data.length == 0) {
                NSError *emptyError = [NSError errorWithDomain:@"NetworkManager"
                                                          code:-1
                                                      userInfo:@{NSLocalizedDescriptionKey: @"服务器未返回数据"}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, emptyError);
                });
                return;
            }

            // ---- 4.3：解析 JSON ----
            NSError *parseError = nil;
            NSDictionary *responseDict = [NSJSONSerialization JSONObjectWithData:data
                                                                         options:0
                                                                           error:&parseError];
            if (parseError) {
                NSLog(@"[NetworkManager] POST 响应 JSON 解析失败：%@", parseError);
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, parseError);
                });
                return;
            }

            // ---- 4.4：检查业务状态码 ----
            NSInteger code = [responseDict[@"code"] integerValue];
            if (code != 200) {
                NSString *msg = responseDict[@"message"] ?: @"未知错误";
                NSLog(@"[NetworkManager] POST 业务错误：code=%ld, message=%@", (long)code, msg);
                NSError *bizError = [NSError errorWithDomain:@"NetworkManager"
                                                        code:code
                                                    userInfo:@{NSLocalizedDescriptionKey: msg}];
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) completion(NO, nil, bizError);
                });
                return;
            }

            // ---- 4.5：成功，回到主线程通知调用方 ----
            NSLog(@"[NetworkManager] POST 新增用户成功");
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) completion(YES, responseDict, nil);
            });
        }];

    // ─── 第 5 步：启动任务 ───
    [task resume];
}
```

---

## 四、和昨天 GET 方法的关键差异点

只标出不一样的地方，其他的完全复用昨天学过的逻辑：

| 代码 | 昨天 GET | 今天 POST | 为什么不同 |
|------|---------|-----------|-----------|
| 请求对象 | `NSURLRequest` | `NSMutableURLRequest` | POST 需要修改方法和加 Body，不可变的不行 |
| HTTPMethod | `@"GET"` | `@"POST"` | 动词不同 |
| 请求头 | 没设 | `Content-Type: application/json` | 告诉服务器 Body 里的数据格式 |
| Body | 没有 | `NSJSONSerialization dataWithJSONObject:...` | 参数不在 URL，在 Body 里 |
| NSJSONSerialization 方向 | `JSONObjectWithData`（解包） | `dataWithJSONObject`（打包） | GET 是收数据，POST 是发数据 |

---

## 五、两个 NSJSONSerialization 的对称关系

```
发出去时（POST）：
  你的 OC 字典                        JSON 二进制
  @{@"name":@"张三"}     ──→     {"name": "张三"}
               dataWithJSONObject

收回来时（GET）：
  JSON 二进制                         OC 字典
  {"name": "张三"}        ──→     @{@"name": @"张三"}
               JSONObjectWithData

这就是 JSON 翻译官的两面工作：打包出去、拆包进来。
```

---

## 六、测试方法

在 ViewController.m 里加：

```objc
[[NetworkManager sharedManager] addUserWithName:@"测试用户"
                                            age:@"25"
                                     completion:^(BOOL success, id result, NSError *error) {
    if (success) {
        NSLog(@"✅ 新增成功：%@", result);
    } else {
        NSLog(@"❌ 新增失败：%@", error.localizedDescription);
    }
}];
```

新增成功后，再用昨天写的 `fetchUsersWithCompletion:` 拉一次列表，应该能看到刚才加的用户。

---

## 七、本步骤新增的知识点

| 编号 | 知识点 | 在哪里体现 |
|------|--------|-----------|
| 16 | POST 请求 | `HTTPMethod = @"POST"` |
| 17 | Content-Type 请求头 | `setValue:forHTTPHeaderField:` |
| 18 | HTTP Body（JSON） | `request.HTTPBody = bodyData` |
| 6 | NSJSONSerialization 序列化方向 | `dataWithJSONObject:`（字典 → JSON） |
| 10 | NSMutableURLRequest（可变请求） | `setValue:` / `HTTPBody` / `HTTPMethod` 在创建后才修改 |

**下一步预告：** 修改用户（GET + URL 参数 `?id=xxx&name=xxx&age=xxx`）和删除用户（GET + URL 参数 `?id=xxx`）。
