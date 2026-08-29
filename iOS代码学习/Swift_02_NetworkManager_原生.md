# Swift_02 — NetworkManager 原生版（URLSession + 闭包）

> iOS 网络编程的 Swift 版核心：NetworkManager，用原生 `URLSession` 实现增删改查 + 下载。  
> 对应你 OC 的 `03_NetworkManager_纯NSURLSession版.md` + `05_混合版原生下载`，Swift 语法重写。  
> 目标：GET 列表 / POST 新增 / GET 修改 / GET 删除 / 原生下载 五个接口。

---

## 一、先看这文件要做什么

和 OC 版完全一样的功能，只是用 Swift 的 URLSession + 闭包写：

```
NetworkManager（单例）
  ├── fetchUsers        GET  /user/users      → 返回 [User]
  ├── addUser           POST /user/save       → 新增
  ├── updateUser        GET  /user/update?id=&name=&age=
  ├── deleteUser        GET  /user/delete?id=
  └── downloadFile      下载文件 → 进度回调 + 保存路径
```

> **下载部分**：和 OC 原题一样，用**原生 `URLSessionDownloadTask`** 实现（Swift 版没有「混合版」那套跨库，这里直接把下载做成原生）。进度用 delegate 方法 `didWriteData` 回调。

**关键对比（OC → Swift）：**

| OC                                       | Swift                               |
| ---------------------------------------- | ----------------------------------- |
| `NSURLSession`                           | `URLSession`                        |
| `NSMutableURLRequest`                    | `URLRequest`                        |
| `dataTaskWithRequest:completionHandler:` | `dataTask(with:completionHandler:)` |
| `^ (Block)`                              | { 闭包 (closure) }                    |
| `id result`                              | `Any?` / 具体类型                       |
| 单例 `sharedManager`                       | `static let shared`                 |
| 回调 `completion(NO, nil, error)`          | 闭包传多参数                              |

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

## 二点五、定义下载回调类型（对应 OC 的 DownloadProgressBlock / DownloadCompletionBlock）

下载比增删改查多两类回调：进度 + 完成路径。

```swift
// 下载进度回调：progress 0.0 ~ 1.0（对应 OC 的 DownloadProgressBlock）
typealias DownloadProgressBlock = (_ progress: Double) -> Void

// 下载完成回调：成功给文件路径（对应 OC 的 DownloadCompletionBlock）
typealias DownloadCompletionBlock = (_ success: Bool, _ filePath: String?, _ error: Error?) -> Void
```

- `progress: Double` = 下载进度（0.0 ~ 1.0），下载中反复调
- `filePath: String?` = 下载保存到沙盒的完整路径，失败是 nil

> 进度和完成是两种不同的回调，所以单独定义（和 OC 两个 typedef 对应）。等下下载方法里各用一个。

---

## 三、NetworkManager 完整代码

```swift
import Foundation

// ★★ 原生下载需要 delegate 回调 → NetworkManager 必须继承 NSObject 并遵守 URLSessionDownloadDelegate ★★
// 对应 OC：@interface NetworkManager () <NSURLSessionDownloadDelegate>
class NetworkManager: NSObject, URLSessionDownloadDelegate {

    // ─── 单例：对应 OC 的 +sharedManager ───
    static let shared = NetworkManager()

    // ─── 服务器基础地址 ───
    private let baseURL = "http://10.17.66.196:8086"

    // ─── 下载用到的回调（原生 delegate 分多个方法回调，必须先存成属性）───
    private var downloadProgressBlock: DownloadProgressBlock?
    private var downloadCompletionBlock: DownloadCompletionBlock?

    // ─── 专属下载会话（delegate = self，对应 OC 的 downloadSession）───
    private lazy var downloadSession: URLSession = {
        // 默认配置 + delegate=self，回调走下面三个 delegate 方法
        let config = URLSessionConfiguration.default
        return URLSession(configuration: config, delegate: self, delegateQueue: nil)
    }()

    // ─── 私有 init，防止外面创建新实例 ───
    private override init() {}

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
        // ★ 用 URLComponents 拼 URL —— 自动把中文/特殊字符做百分号编码
        //   不能手动 "\(baseURL)/update?name=\(name)"，name 是中文时服务器会拿乱码/匹配不到
        var comps = URLComponents(string: baseURL + "/user/update")!
        comps.queryItems = [
            URLQueryItem(name: "id", value: id),
            URLQueryItem(name: "name", value: name),
            URLQueryItem(name: "age", value: age)
        ]
        guard let url = comps.url else {
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

    // ============================================================
    // 5. 下载文件 — 原生 URLSessionDownloadTask（对应 OC 的 downloadFileFromURL:）
    //    题目要求的第一种下载方式，和 OC 原生下载完全对应
    // ============================================================
    func downloadFile(from urlString: String,
                      progress: DownloadProgressBlock?,
                      completion: @escaping DownloadCompletionBlock) {

        // ── 第 1 步：存好两个回调（原生 delegate 分多个方法调，先存属性）──
        downloadProgressBlock = progress
        downloadCompletionBlock = completion

        // ── 第 2 步：URL 校验 ──
        guard let url = URL(string: urlString) else {
            let err = NSError(domain: "NetworkManager", code: -1,
                              userInfo: [NSLocalizedDescriptionKey: "URL 无效"])
            completion(false, nil, err)
            return
        }

        print("[下载] 开始下载：\(urlString)")

        // ── 第 3 步：创建原生下载任务（不需手动建 request，直接传 URL）──
        let task = downloadSession.downloadTask(with: url)

        // ── 第 4 步：启动（原生不会自动 resume）──
        task.resume()
    }

    // ════════════════════════════════════════════════════════════
    // ★ 原生下载 delegate 三个方法（系统自动调，对应 OC 那三个方法）
    // ════════════════════════════════════════════════════════════

    // ---- ① 下载进度回调 — 下载中反复调 ----
    func urlSession(_ session: URLSession,
                    downloadTask: URLSessionDownloadTask,
                    didWriteData bytesWritten: Int64,
                    totalBytesWritten: Int64,
                    totalBytesExpectedToWrite: Int64) {

        if totalBytesExpectedToWrite <= 0 { return }   // 拿不到总大小，跳过

        let progress = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)

        print("[下载进度] \(Int(progress * 100))%")

        // delegate 在子线程调，回主线程再调回调（更新 UI 用）
        DispatchQueue.main.async {
            self.downloadProgressBlock?(progress)
        }
    }

    // ---- ② 下载完成回调 — 文件在临时目录，移到 Documents ----
    func urlSession(_ session: URLSession,
                    downloadTask: URLSessionDownloadTask,
                    didFinishDownloadingTo location: URL) {
        //  location = 系统临时文件夹路径（随时会被清理）

        // 1. 取文件名（对应 OC 从 Content-Disposition 响应头取）
        var filename: String? = nil
        if let httpResponse = downloadTask.response as? HTTPURLResponse,
           let contentDisposition = httpResponse.allHeaderFields["Content-Disposition"] as? String {
            filename = contentDisposition
                .replacingOccurrences(of: "attachment; filename=", with: "")
                .replacingOccurrences(of: "\"", with: "")
        }

        // 备选：从 URL 最后一段取
        if filename == nil || filename!.isEmpty {
            filename = downloadTask.response.suggestedFilename
        }

        // 兜底：时间戳文件名
        if filename == nil || filename!.isEmpty {
            filename = "download_\(Date().timeIntervalSince1970).file"
        }

        print("[下载完成] 文件名：\(filename!)")

        // 2. 移动到沙盒 Documents
        let documentsPath = NSSearchPathForDirectoriesInDomains(
            .documentDirectory, .userDomainMask, true).first ?? ""
        let destPath = documentsPath + "/" + filename!

        let fm = FileManager.default

        // 3. 目标已存在先删旧（避免 move 失败）
        if fm.fileExists(atPath: destPath) {
            try? fm.removeItem(atPath: destPath)
        }

        // 4. 临时文件移到 Documents
        do {
            try fm.moveItem(at: location, to: URL(fileURLWithPath: destPath))
            print("[文件保存成功] \(destPath)")
            DispatchQueue.main.async {
                self.downloadCompletionBlock?(true, destPath, nil)
            }
        } catch {
            print("[文件保存失败] \(error.localizedDescription)")
            DispatchQueue.main.async {
                self.downloadCompletionBlock?(false, nil, error)
            }
        }
    }

    // ---- ③ 下载结束（成功或失败都调）— 用 error 区分 ----
    func urlSession(_ session: URLSession,
                    task: URLSessionTask,
                    didCompleteWithError error: Error?) {
        // error == nil → 成功已在上面的 didFinishDownloadingTo 处理
        if let error = error {
            print("[下载失败] \(error.localizedDescription)")
            DispatchQueue.main.async {
                self.downloadCompletionBlock?(false, nil, error)
            }
        }
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

### 4.10 下载相关：类要继承 NSObject + 遵守下载协议

增删改查用 `URLSession.shared` 闭包回调，**不需要 delegate**。但**下载**要拿进度、要移动临时文件，靠 **delegate 方法**（对应 OC 三个下载 delegate 方法），所以 NetworkManager 必须：

```swift
class NetworkManager: NSObject, URLSessionDownloadDelegate {
    // NSObject                  → delegate 方法要求的基类
    // URLSessionDownloadDelegate → 下载进度/完成/结束三个方法的协议
}
```

对应 OC：
```objc
@interface NetworkManager () <NSURLSessionDownloadDelegate>
```

### 4.11 专属下载会话 `lazy var downloadSession`

```swift
private lazy var downloadSession: URLSession = {
    let config = URLSessionConfiguration.default
    return URLSession(configuration: config, delegate: self, delegateQueue: nil)
}()
```

- **`URLSession(configuration:delegate:delegateQueue:)`** = 自定义带 delegate 的会话（对应 OC 单独建的 `downloadSession`）
- **`delegate: self`** = 下载回调走 NetworkManager 自己实现的三个 delegate 方法
- **`delegateQueue: nil`** = 让系统决定回调线程（delegate 方法在子线程，所以方法里要 `DispatchQueue.main.async`）
- **`lazy`** = 第一次用到它才创建（对应 OC 懒加载）

### 4.12 `downloadTask(with:)` — 建下载任务

```swift
let task = downloadSession.downloadTask(with: url)
task.resume()
```

- 下载任务用 `downloadTask(with:)`（对应 OC `[downloadSession downloadTaskWithURL:url]`）
- 不用 `dataTask`（那是增删改查拿 JSON 的）；下载直接传 URL，文件落到临时目录
- 原生不自动 resume，必须手动 `task.resume()`

### 4.13 三个下载 delegate 方法（对应 OC 那三个）

| OC delegate | Swift delegate | 触发时机 |
|-------------|---------------|---------|
| `URLSession:downloadTask:didWriteData:...` | `urlSession(_:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite:)` | 下载中反复调（进度） |
| `URLSession:downloadTask:didFinishDownloadingToURL:` | `urlSession(_:downloadTask:didFinishDownloadingTo:)` | 下载完，文件到临时目录 |
| `URLSession:task:didCompleteWithError:` | `urlSession(_:task:didCompleteWithError:)` | 结束（成功/失败都调） |

Swift 的命名比 OC 简化了，参数顺序一一对应。**关键点**：
- 进度方法里：`Double(已写)/Double(总)` 算进度
- 完成方法里：拿到临时路径 `location`，用 `FileManager` 移到 Documents
- 结束方法里：`error` 非 nil 才处理（成功已在完成方法处理过，别重复调）

### 4.14 `FileManager` — 移动文件到 Documents

```swift
let fm = FileManager.default
if fm.fileExists(atPath: destPath) { try? fm.removeItem(atPath: destPath) }  // 有旧档先删
try fm.moveItem(at: location, to: URL(fileURLWithPath: destPath))            // 临时文件挪过来
```

`FileManager.default` = 文件管理器单例（对应 OC `[NSFileManager defaultManager]`）。`moveItem(at:to:)` 把 system 临时目录的下载文件挪到 Documents。

---

## 五、Swift 网络层 vs OC 网络层照搬关系

| OC 写法                                                          | Swift 写法                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------ |
| `[NSURLSession sharedSession]`                                 | `URLSession.shared`                                          |
| `dataTaskWithRequest:completionHandler:`                       | `dataTask(with:completionHandler:)`                          |
| `NSMutableURLRequest request = ...; request.HTTPMethod=@"GET"` | `var request = URLRequest(url:); request.httpMethod = "GET"` |
| `[request setValue:@... forHTTPHeaderField:@"Content-Type"]`   | `request.setValue(..., forHTTPHeaderField: "Content-Type")`  |
| `NSJSONSerialization JSONObjectWithData:`                      | `JSONSerialization.jsonObject(with:)`                        |
| `[task resume]`                                                | `task.resume()`                                              |
| `dispatch_async(dispatch_get_main_queue())`                    | `DispatchQueue.main.async {}`                                |
| `^(NSData *d, NSURLResponse *r, NSError *e)`                   | `{ data, response, error in }`                               |
| `if (error) {completion(NO,nil,error); return;}`               | `if let error = error {}`                                    |
| `dict[@"code"]`                                                | `json["code"] as? Int ?? -1`                                 |
| `downloadTaskWithURL:`（原生下载）                              | `downloadSession.downloadTask(with:)`                       |
| `URLSession:downloadTask:didWriteData:`（进度）                 | `urlSession(_:downloadTask:didWriteData:...)`               |
| `didFinishDownloadingToURL:` + 手动移文件                       | `urlSession(_:downloadTask:didFinishDownloadingTo:)` + FileManager |
| `[NSFileManager defaultManager] moveItemAtURL:`                | `FileManager.default.moveItem(at:to:)`                     |

**几乎照搬**——语法换了，逻辑一模一样。增删改查 + 原生下载全部覆盖。

---

## 六、你在 Xcode 里怎么用

1. 新建 `NetworkManager.swift`（Cocoa Touch Class，Subclass of: NSObject，或 Swift File）
2. 把上面的 Swift 版代码粘进去
3. 确保 `User.swift` 在同一个 target
4. 调用方写：`NetworkManager.shared.fetchUsers { ok, result, err in ... }`
5. **下载**调用：`NetworkManager.shared.downloadFile(from: urlString, progress: { p in ... }, completion: { ok, path, err in ... })`

---

**到这里网络层完成（含原生下载）。** 你已接触 Swift 的关键语法：单例、闭包、guard、@escaping、do-catch、URLSession、delegate、FileManager、as?、??。

下一步 `Swift_04_UserListViewController`（+ UserCell）：把网络层接到 UITableView 上，完成增删改查界面 + 下载按钮。你先把 `Swift_02` 这个 NetworkManager.swift 写好，或者我们继续。

