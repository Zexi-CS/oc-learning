# Swift_02 — NetworkManager 原生版（URLSession + 闭包）

> iOS 网络编程的 Swift 版核心：NetworkManager，用原生 `URLSession` 实现增删改查。
> 对应你 OC 的 `03_NetworkManager_纯NSURLSession版.md`，Swift 语法重写。
> 目标：GET 列表 / POST 新增 / GET 修改 / GET 删除 四个接口。

---

## 一、先看这文件要做什么

和 OC 版完全一样的功能，只是用 Swift 的 URLSession + 闭包写：

```
NetworkManager（单例）
  ├── fetchUsers        GET  /user/users      → 返回 [User]
  ├── addUser           POST /user/save       → 新增
  ├── updateUser        GET  /user/update?id=&name=&age=
  └── deleteUser        GET  /user/delete?id=
```

**关键对比（OC → Swift）：**
| OC | Swift |
|----|-------|
| `NSURLSession` | `URLSession` |
| `NSMutableURLRequest` | `URLRequest` |
| `dataTaskWithRequest:completionHandler:` | `dataTask(with:completionHandler:)` |
| `^ (Block)` | { 闭包 (closure) } |
| `id result` | `Any?` / 具体类型 |
| 单例 `sharedManager` | `static let shared` |
| 回调 `completion(NO, nil, error)` | 闭包传多参数 |

---

## 二、定义回调类型（对应 OC 的 NetworkCompletionBlock）

OC 里你用 typedef 定义了一个 Block 类型：
```objc
typedef void (^NetworkCompletionBlock)(BOOL success, id result, NSError *error);
```

Swift 用 **typealias**（类型别名）+ 闭包类型：

```swift
import Foundation

// 对应 OC 的 NetworkCompletionBlock
// 参数：(是否成功, 结果, 错误)
typealias CompletionBlock = (_ success: Bool, _ result: Any?, _ error: Error?) -> Void
```

拆解：
- `typealias` = 给类型起别名（对应 OC 的 `typedef`）
- `(_ success: Bool, _ result: Any?, _ error: Error?) -> Void` = 一个闭包类型
  - `Bool` = 成功与否
  - `Any?` = 结果（可能 nil，用可选 `?`）
  - `Error?` = 错误（可能 nil）
  - `-> Void` = 无返回值（对应 OC 的 `void (^)`）

> `_` 开头表示这个参数"调用时不用写名字"。先当成 OC 的 Block 参数即可。

---

## 三、NetworkManager 完整代码

```swift
import Foundation

class NetworkManager {

    // ─── 单例：对应 OC 的 +sharedManager ───
    static let shared = NetworkManager()

    // ─── 服务器基础地址 ───
    private let baseURL = "http://10.17.66.196:8086"

    // ─── 私有 init，防止外面创建新实例 ───
    private init() {}

    // ============================================================
    // 1. 获取用户列表 — GET /user/users
    // ============================================================
    func fetchUsers(completion: @escaping CompletionBlock) {
        let urlString = baseURL + "/user/users"
        guard let url = URL(string: urlString) else {
            completion(false, nil, nil)   // URL 非法
            return
        }

        // 构造请求
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.timeoutInterval = 30

        // 发送请求 —— 核心 API：URLSession 的 dataTask(with:)
        let task = URLSession.shared.dataTask(with: request) { data, response, error in

            // 网络回调在子线程，刷新 UI 要在主线程
            DispatchQueue.main.async {

                // 1. 网络错误
                if let error = error {
                    print("请求失败: \(error.localizedDescription)")
                    completion(false, nil, error)
                    return
                }

                // 2. 空数据
                guard let data = data, !data.isEmpty else {
                    completion(false, nil, nil)
                    return
                }

                // 3. 解析 JSON
                do {
                    if let json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any] {

                        // 4. 业务 code 校验
                        let code = json["code"] as? Int ?? -1
                        if code != 200 {
                            let msg = json["message"] as? String ?? "未知错误"
                            print("业务错误 code=\(code) msg=\(msg)")
                            completion(false, nil, nil)
                            return
                        }

                        // 5. data 转 User 数组
                        let dataArray = json["data"] as? [[String: Any]] ?? []
                        let users = User.users(fromArray: dataArray)
                        print("成功获取 \(users.count) 个用户")
                        completion(true, users, nil)
                    } else {
                        completion(false, nil, nil)
                    }
                } catch {
                    print("JSON 解析失败: \(error)")
                    completion(false, nil, error)
                }
            }
        }

        task.resume()   // 启动任务
    }

    // ============================================================
    // 2. 新增用户 — POST /user/save
    // ============================================================
    func addUser(name: String, age: String, completion: @escaping CompletionBlock) {
        let urlString = baseURL + "/user/save"
        guard let url = URL(string: urlString) else {
            completion(false, nil, nil)
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.timeoutInterval = 30
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        // 参数打包成 JSON body
        let bodyDict: [String: String] = ["name": name, "age": age]
        let bodyData = try? JSONSerialization.data(withJSONObject: bodyDict, options: [])
        request.httpBody = bodyData

        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            DispatchQueue.main.async {
                if let error = error {
                    completion(false, nil, error)
                    return
                }
                guard let data = data, !data.isEmpty else {
                    completion(false, nil, nil)
                    return
                }
                do {
                    if let json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any] {
                        let code = json["code"] as? Int ?? -1
                        if code != 200 {
                            completion(false, nil, nil)
                            return
                        }
                        print("新增用户成功")
                        completion(true, json, nil)
                    }
                } catch {
                    completion(false, nil, error)
                }
            }
        }
        task.resume()
    }

    // ============================================================
    // 3. 修改用户 — GET /user/update?id=&name=&age=
    // ============================================================
    func updateUser(id: String, name: String, age: String, completion: @escaping CompletionBlock) {
        let urlString = "\(baseURL)/user/update?id=\(id)&name=\(name)&age=\(age)"
        guard let url = URL(string: urlString) else {
            completion(false, nil, nil)
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.timeoutInterval = 30

        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            DispatchQueue.main.async {
                if let error = error {
                    completion(false, nil, error)
                    return
                }
                guard let data = data, !data.isEmpty else {
                    completion(false, nil, nil)
                    return
                }
                do {
                    if let json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any] {
                        let code = json["code"] as? Int ?? -1
                        if code != 200 {
                            completion(false, nil, nil)
                            return
                        }
                        print("修改用户成功")
                        completion(true, json, nil)
                    }
                } catch {
                    completion(false, nil, error)
                }
            }
        }
        task.resume()
    }

    // ============================================================
    // 4. 删除用户 — GET /user/delete?id=
    // ============================================================
    func deleteUser(id: String, completion: @escaping CompletionBlock) {
        let urlString = "\(baseURL)/user/delete?id=\(id)"
        guard let url = URL(string: urlString) else {
            completion(false, nil, nil)
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.timeoutInterval = 30

        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            DispatchQueue.main.async {
                if let error = error {
                    completion(false, nil, error)
                    return
                }
                guard let data = data, !data.isEmpty else {
                    completion(false, nil, nil)
                    return
                }
                do {
                    if let json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any] {
                        let code = json["code"] as? Int ?? -1
                        if code != 200 {
                            completion(false, nil, nil)
                            return
                        }
                        print("删除用户成功")
                        completion(true, json, nil)
                    }
                } catch {
                    completion(false, nil, error)
                }
            }
        }
        task.resume()
    }
}
```

---

## 四、逐段拆解新语法（你 OC 没见过的）

### 4.1 `static let shared = NetworkManager()`

单例。对应 OC 的 `+sharedManager` + `dispatch_once`。
- `static let` = 类级常量，只创建一次（Swift 自动保证线程安全，不用 dispatch_once）
- 每次 `NetworkManager.shared` 拿到同一个实例

### 4.2 `private init() {}`

私有 init，防止外面 `NetworkManager()` 创建新实例（对应 OC 的 initPrivate + @throw）。

### 4.3 `URLSession.shared.dataTask(with: ...)`

核心网络 API，对应 OC 的 `[NSURLSession sharedSession] dataTaskWithRequest:completionHandler:`。

```swift
URLSession.shared.dataTask(with: request) { data, response, error in
    // 闭包体：网络回来执行
}
task.resume()   // 启动
```

- `URLSession.shared` = 共享会话（对应 `[NSURLSession sharedSession]`）
- `dataTask(with:completionHandler:)` 的完成回调是一个闭包
- 闭包三个参数：`data, response, error`

### 4.4 闭包 `{ data, response, error in ... }`

对应 OC 的 Block。OC：
```objc
^(NSData *data, NSURLResponse *response, NSError *error) { ... }
```
Swift：
```swift
{ data, response, error in
    // 闭包体（类型自动推断）
}
```
- `in` 分隔"参数"和"闭包体"

### 4.5 `guard let ... else { }`

guard = 保险栓语法，专门做"提前检查，不满足就退出"。

```swift
guard let url = URL(string: urlString) else {
    completion(false, nil, nil)   // URL 非法就退出
    return
}
// url 在这之后可以直接用（guard 解包后作用域更大）
```

对应 OC 的 `if (!url) return;`。

### 4.6 `DispatchQueue.main.async { }`

对应 OC 的 `dispatch_async(dispatch_get_main_queue(), ^{})`。切回主线程（网络回调在子线程，改 UI 必须回主线程）。

### 4.7 `@escaping` 关键字

```swift
func fetchUsers(completion: @escaping CompletionBlock)
```
- 普通闭包"同步用完就丢"
- **网络请求是异步的**，completion 在方法返回之后才被调（等网络回来）
- `@escaping` 告诉编译器"这个闭包会逃出方法外，稍后才调"

**记法**：异步回调（网络、延迟）都要加 @escaping。

### 4.8 `do { try ... } catch { }`

对应 OC 的 `NSError *jsonError; [NSJSONSerialization ... error:&jsonError]`。

```swift
do {
    let json = try JSONSerialization.jsonObject(with: data, options: [])
    // 成功继续
} catch {
    // 抛异常走这里
}
```

JSONSerialization 可能抛错，Swift 要求 `try` + `do-catch`。

### 4.9 `as? [String: Any]`、`as? Int ?? -1`

- `as? [String: Any]` 安全转字典（JSON 外层）
- `as? Int ?? -1` 转 Int，取不到用 -1（`??` = 空值合并，类似 OC `?:`）

---

## 五、Swift 网络层 vs OC 网络层照搬关系

| OC 写法 | Swift 写法 |
|--------|-----------|
| `[NSURLSession sharedSession]` | `URLSession.shared` |
| `dataTaskWithRequest:completionHandler:` | `dataTask(with:completionHandler:)` |
| `NSMutableURLRequest request = ...; request.HTTPMethod=@"GET"` | `var request = URLRequest(url:); request.httpMethod = "GET"` |
| `[request setValue:@... forHTTPHeaderField:@"Content-Type"]` | `request.setValue(..., forHTTPHeaderField: "Content-Type")` |
| `NSJSONSerialization JSONObjectWithData:` | `JSONSerialization.jsonObject(with:)` |
| `[task resume]` | `task.resume()` |
| `dispatch_async(dispatch_get_main_queue())` | `DispatchQueue.main.async {}` |
| `^(NSData *d, NSURLResponse *r, NSError *e)` | `{ data, response, error in }` |
| `if (error) {completion(NO,nil,error); return;}` | `if let error = error {}` |
| `dict[@"code"]` | `json["code"] as? Int ?? -1` |

**几乎照搬**——语法换了，逻辑一模一样。

---

## 六、你在 Xcode 里怎么用

1. 新建 `NetworkManager.swift`（Cocoa Touch Class，Subclass of: NSObject，或 Swift File）
2. 把上面的 Swift 版代码粘进去
3. 确保 `User.swift` 在同一个 target
4. 调用方写：`NetworkManager.shared.fetchUsers { ok, result, err in ... }`

---

**到这里网络层完成。** 你已接触 Swift 的关键语法：单例、闭包、guard、@escaping、do-catch、URLSession、as?、??。

下一步 `Swift_03_UserListViewController_原生.md`（+ UserCell）：把网络层接到 UITableView 上，完成增删改查界面。这是最后的大块。你先把 `Swift_02` 这个 NetworkManager.swift 写好，或者我们继续下一个文件？
