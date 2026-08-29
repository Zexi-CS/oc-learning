# Swift_06 — NetworkManager Alamofire 版（第三方库）

> iOS 网络编程 Swift 版的**对照版本**：用第三方库 **Alamofire** 实现增删改查 + 下载。  
> 对应你 OC 的 `04_NetworkManager_纯AFN版.md`（AFNetworking 版），Swift 语法重写。  
> **核心：外部调用接口和 Swift_02 原生版完全一样，只改内部实现。**  
> 目标：GET 列表 / POST 新增 / GET 修改 / GET 删除 / 下载 五个接口。



---

## 一、先看这文件要做什么

和 `Swift_02` 原生版功能一模一样，只是把 `URLSession` 换成 **Alamofire**：

```
NetworkManager（单例）
  ├── fetchUsers        GET  /user/users      → 返回 [User]
  ├── addUser           POST /user/save       → 新增
  ├── updateUser        GET  /user/update?id=&name=&age=
  ├── deleteUser        GET  /user/delete?id=
  └── downloadFile      下载文件 → 进度回调 + 保存路径
```

> **下载部分**：Alamofire 用专门的下载 API（`AF.download` + `.downloadProgress`），比原生简单得多——**不需要 delegate**，进度闭包直接给。对应你 OC 的 AFNetworking 下载（`downloadTaskWithRequest:progress:destination:completionHandler:`）。

**关键对比（原生 URLSession → Alamofire）：**

| 原生态 Swift                       | Alamofire 版 Swift              |
| ------------------------------- | ------------------------------ |
| `URLSession.shared`             | `AF`（`AF.request`）             |
| `URLRequest` + 手动设 method       | `AF.request(url, method:)`     |
| `JSONSerialization` 解析 JSON     | `responseJSON`（自动解析）           |
| `DispatchQueue.main.async` 切主线程 | **Alamofire 默认主线程回调**          |
| 手动判断 HTTP 状态码 / 业务 code         | `response` 里带状态码，业务 code 仍要自己查 |
| 4 段错误检查（冗长）                     | 一个闭包搞定（简洁）                     |

**最核心的区别**：原生版要自己"发请求 → 解析 JSON → 切主线程 → 判断错误"写一大堆；Alamofire 一个 `AF.request(...).responseJSON { }` 全包了，代码少一半。这和你 OC 里 `NSURLSession vs AFNetworking` 的对比完全对称。

---

## 二、环境准备（Alamofire 必须装 CocoaPods）

Alamofire 是**第三方库**，不能用原生方式直接用，必须先通过 **CocoaPods** 装进工程（和你 OC 装 AFNetworking 一模一样）。

### 第 1 步：确保工程有 Podfile

在工程根目录（`.xcodeproj` 同级）建 `Podfile`：

```ruby
platform :ios, '13.0'

target '你的工程名' do
  use_frameworks!
  pod 'Alamofire'      # ← 加上 Alamofire
end
```

### 第 2 步：安装

终端进到工程目录（含 Podfile 那里），执行：

```bash
pod install
```

安装完成后，**以后都要用 `.xcworkspace` 打开工程**（双击 `你的工程名.xcworkspace`），**不再用 `.xcodeproj`**，否则找不到 Alamofire。

> CocoaPods 你已经有了（装过 Masonry/AFNetworking），Alamofire 的命令完全一样，只是 `pod 'Alamofire'`。**不用重新下载别的东西**。

### 第 3 步：import

代码里要 `import Alamofire`（看到 `AF.request` 就是它在干活）。

---

## 三、定义回调类型（和原生版完全一样）

这一处**不用改**——和 `Swift_02` 一模一样，因为外部调用接口要保持一致：

```swift
import Foundation

// 和 Swift_02 原生版完全相同的回调类型
typealias CompletionBlock = (_ success: Bool, _ result: Any?, _ error: Error?) -> Void

// 下载进度回调（progress 0.0 ~ 1.0）
typealias DownloadProgressBlock = (_ progress: Double) -> Void

// 下载完成回调（success / filePath / error）
typealias DownloadCompletionBlock = (_ success: Bool, _ filePath: String?, _ error: Error?) -> Void
```

参数不变、签名不变，只是把底层的 URLSession 换成 Alamofire。下载的回调类型也和原生版一致，这样调用方两种版本都能用同一句代码。

---

## 四、NetworkManager 完整代码（Alamofire 版）

```swift
import Foundation
import Alamofire                       // ← 引入 Alamofire

class NetworkManager {

    // ─── 单例：和原生版一样 ───
    static let shared = NetworkManager()

    // ─── 服务器基础地址 ───
    private let baseURL = "http://10.17.66.196:8086"

    // ─── 私有 init ───
    private init() {}

    // ============================================================
    // 1. 获取用户列表 — GET /user/users
    // ============================================================
    func fetchUsers(completion: @escaping CompletionBlock) {
        let urlString = baseURL + "/user/users"

        // Alamofire 一步到位：AF.request + responseJSON
        AF.request(urlString, method: .get)
            .responseJSON { response in

                // ★ Alamofire 默认在主线程回调，不用手动 dispatch_async！

                // 1. 网络/请求错误
                switch response.result {
                case .failure(let error):
                    print("请求失败: \(error.localizedDescription)")
                    completion(false, nil, error)
                    return

                case .success(let value):
                    // 2. value 已经是解析好的 JSON（字典/数组）
                    guard let json = value as? [String: Any] else {
                        completion(false, nil, nil)
                        return
                    }

                    // 3. 业务 code 校验（服务器自己的规则，Alamofire 不管）
                    let code = json["code"] as? Int ?? -1
                    if code != 200 {
                        let msg = json["message"] as? String ?? "未知错误"
                        print("业务错误 code=\(code) msg=\(msg)")
                        completion(false, nil, nil)
                        return
                    }

                    // 4. data 转 User 数组（和原生版同一句）
                    let dataArray = json["data"] as? [[String: Any]] ?? []
                    let users = User.users(fromArray: dataArray)
                    print("成功获取 \(users.count) 个用户")
                    completion(true, users, nil)
                }
            }
    }

    // ============================================================
    // 2. 新增用户 — POST /user/save（参数在 body）
    // ============================================================
    func addUser(name: String, age: String, completion: @escaping CompletionBlock) {
        let urlString = baseURL + "/user/save"

        // parameters = 要发出去的数据（Alamofire 帮你转 JSON）
        let parameters: [String: Any] = ["name": name, "age": age]

        // encoding: JSONEncoding.default → 参数以 JSON 格式放 body
        AF.request(urlString,
                   method: .post,
                   parameters: parameters,
                   encoding: JSONEncoding.default)
            .responseJSON { response in

                switch response.result {
                case .failure(let error):
                    completion(false, nil, error)

                case .success(let value):
                    guard let json = value as? [String: Any] else {
                        completion(false, nil, nil)
                        return
                    }
                    let code = json["code"] as? Int ?? -1
                    if code != 200 {
                        completion(false, nil, nil)
                        return
                    }
                    print("新增用户成功")
                    completion(true, json, nil)
                }
            }
    }

    // ============================================================
    // 3. 修改用户 — GET /user/update?id=&name=&age=（参数在 URL）
    // ============================================================
    func updateUser(id: String, name: String, age: String, completion: @escaping CompletionBlock) {
        let urlString = "\(baseURL)/user/update"

        // 参数放 URL 上（?id=&name=&age=）
        // encoding: URLEncoding.queryString → 参数拼到网址后面
        let parameters: [String: Any] = ["id": id, "name": name, "age": age]

        AF.request(urlString,
                   method: .get,
                   parameters: parameters,
                   encoding: URLEncoding.queryString)
            .responseJSON { response in

                switch response.result {
                case .failure(let error):
                    completion(false, nil, error)

                case .success(let value):
                    guard let json = value as? [String: Any] else {
                        completion(false, nil, nil)
                        return
                    }
                    let code = json["code"] as? Int ?? -1
                    if code != 200 {
                        completion(false, nil, nil)
                        return
                    }
                    print("修改用户成功")
                    completion(true, json, nil)
                }
            }
    }

    // ============================================================
    // 4. 删除用户 — GET /user/delete?id=（参数在 URL）
    // ============================================================
    func deleteUser(id: String, completion: @escaping CompletionBlock) {
        let urlString = "\(baseURL)/user/delete"

        let parameters: [String: Any] = ["id": id]

        AF.request(urlString,
                   method: .get,
                   parameters: parameters,
                   encoding: URLEncoding.queryString)
            .responseJSON { response in

                switch response.result {
                case .failure(let error):
                    completion(false, nil, error)

                case .success(let value):
                    guard let json = value as? [String: Any] else {
                        completion(false, nil, nil)
                        return
                    }
                    let code = json["code"] as? Int ?? -1
                    if code != 200 {
                        completion(false, nil, nil)
                        return
                    }
                    print("删除用户成功")
                    completion(true, json, nil)
                }
            }
    }

    // ============================================================
    // 5. 下载文件 — Alamofire 下载（AF.download + downloadProgress）
    //    对应你 OC 的 AFNetworking 下载（downloadTaskWithRequest:progress:destination:completionHandler:）
    // ============================================================
    func downloadFile(from urlString: String,
                      progress: DownloadProgressBlock?,
                      completion: @escaping DownloadCompletionBlock) {

        // ── 第 1 步：发起 Alamofire 下载请求 ──
        //    AF.download 直接传 URL 就下载，Alamofire 自动把文件落到临时目录
        AF.download(urlString)
            // ── 第 2 步：进度回调（Alamofire 自动在主线程调）──
            .downloadProgress { downloadProgress in
                let p = downloadProgress.fractionCompleted   // 0.0 ~ 1.0
                print("[下载进度] \(Int(p * 100))%")
                progress?(p)
            }
            // ── 第 3 步：下载完成回调 ──
            .response { response in
                // AF.download 的 response：error 非 nil = 失败；URL = 临时文件路径

                if let error = response.error {
                    print("[下载失败] \(error.localizedDescription)")
                    completion(false, nil, error)
                    return
                }

                // 拿到临时文件路径
                guard let tempURL = response.fileURL else {
                    completion(false, nil, nil)
                    return
                }

                // 取文件名（suggestedFilename 从 URL/响应头判断）
                let filename = response.value?.lastPathComponent
                               ?? tempURL.lastPathComponent
                               ?? "download_\(Date().timeIntervalSince1970).file"

                // 移到沙盒 Documents
                let documentsPath = NSSearchPathForDirectoriesInDomains(
                    .documentDirectory, .userDomainMask, true).first ?? ""
                let destPath = documentsPath + "/" + filename

                let fm = FileManager.default
                if fm.fileExists(atPath: destPath) {
                    try? fm.removeItem(atPath: destPath)   // 有旧档先删
                }

                do {
                    try fm.moveItem(at: tempURL, to: URL(fileURLWithPath: destPath))
                    print("[文件保存成功] \(destPath)")
                    completion(true, destPath, nil)
                } catch {
                    print("[文件保存失败] \(error.localizedDescription)")
                    completion(false, nil, error)
                }
            }
    }
}
```

---

## 五、核心语法拆解（Alamofire 的新东西）

### 5.1 `AF.request(...)` —— 全网入口

`AF` 是 Alamofire 给 `Session.default` 起的**全局别名**，`AF.request(url, method:)` 就是"发起一个请求"。

```swift
AF.request(urlString, method: .get)
```

对应原生版那一长串（`URLRequest` + 设 method + `URLSession.shared.dataTask` + `resume`）被压成一行。`method: .get` 的 `.get` 是 `HTTPMethod` 枚举的简写（对应 OC 的 `HTTPMethodGET`）。

### 5.2 `parameters` + `encoding` —— 参数放哪，由 encoding 决定

这两个参数是 Alamofire 传参的关键：

```swift
let parameters: [String: Any] = ["name": name, "age": age]

AF.request(url, method: .post,
           parameters: parameters,
           encoding: JSONEncoding.default)          // body 里，JSON 格式
```

| encoding                  | 参数去哪                              | 对应原生版                                            |
| ------------------------- | --------------------------------- | ------------------------------------------------ |
| `JSONEncoding.default`    | 变成 JSON 放 **body**（POST 用）        | 手动 `JSONSerialization.data` + `request.httpBody` |
| `URLEncoding.queryString` | 拼到 **URL 后面** `?id=&name=`（GET 用） | 手动字符串插值拼 URL                                     |

**一句话**：原生版"参数放 body 还是 URL"要手动写（POST 打包 JSON、update/delete 拼 URL）；Alamofire 靠 `encoding` 一个参数就分清了。

- **POST（新增）** → `JSONEncoding.default`（参数在 body）✅
- **GET 带参（改/删）** → `URLEncoding.queryString`（参数在 URL）✅

### 5.3 `.responseJSON { response in }` —— 完成回调

`responseJSON` 自动做完成回调，且**默认在主线程**（networking 线程切回 main 由 Alamofire 包了，不用 `DispatchQueue.main.async`）。

```swift
AF.request(...).responseJSON { response in
    // 请求结束，response 里带结果
}
```

- `response.result` 是一个 `Result` 枚举：`.success(值)` / `.failure(错误)`
- **自动解析 JSON**：`value` 已经是字典/数组，不用再 `JSONSerialization.jsonObject`（原生版第 3 步省了）

### 5.4 `switch response.result` —— 错误处理的简洁写法

原生版用 `if let error` + `guard data` + `do-catch` 四段检查。Alamofire 用 `switch` 一个分支就分流：

```swift
switch response.result {
case .failure(let error):       // 网络/请求级错误
    completion(false, nil, error)
case .success(let value):       // 请求成功，value 是解析好的 JSON
    // 业务 code 校验等
}
```

**注意**：Alamofire 只处理"网络层"（连不上、超时）。**"业务 code != 200" 是服务器自己的规则，Alamofire 不管，你还是要自己 `if code != 200` 判断**——这一步和原生版一模一样，没省。

### 5.5 `Result` 枚举 `.success` / `.failure`

Swift 的 `Result` 是"成功/失败二合一"打包：

```swift
enum Result<Value> {
    case success(Value)    // 成功，附带上值
    case failure(Error)    // 失败，附带错误
}
```

`response.result` 就是 `Result<Any>`，用 `switch` 分成两路处理。

### 5.6 Alamofire 下载：`AF.download` + `.downloadProgress` + `.response`

下载在 Alamofire 里是**另一种 API**（不是 `AF.request`，是 `AF.download`）：

```swift
AF.download(urlString)
    .downloadProgress { downloadProgress in
        let p = downloadProgress.fractionCompleted   // 0.0~1.0
        progress?(p)
    }
    .response { response in
        // 完成：response.error / response.fileURL
    }
```

三段对应关系：

| Alamofire 下载            | 作用                              | 对应 OC「AFN下载」                          |
| ----------------------- | ------------------------------- | ------------------------------------- |
| `AF.download(url)`      | 发起下载，文件落到临时目录                   | `downloadTaskWithRequest:`            |
| `.downloadProgress { }` | 进度回调（闭包直接给，不用 delegate！）        | `progress:` Block                     |
| `.response { }`         | 完成回调，拿 `response.fileURL`（临时路径） | `destination:` + `completionHandler:` |

**比原生简单在哪**：

- 原生版（Swift_02）要继承 `NSObject` + 遵守 `URLSessionDownloadDelegate` + 手动实现 3 个 delegate 方法
- Alamofire 版**不需要任何 delegate**，`.downloadProgress` 闭包直接给进度，`.response` 给完成——这就是第三方库帮你封装掉了 delegate 那套。

**进度**：`downloadProgress.fractionCompleted` 是 0.0~1.0（原生版要自己算 `Double(已写)/Double(总)`，Alamofire 给你算好了）。

**完成后**：`response.fileURL` 是临时路径，还是要用 `FileManager` 移到 Documents（和原生版一样）。可以给 `AF.download(urlString, destination:)` 直接指定保存位置，但为了和原生版逻辑一致，这里用同样的"拿临时路径 → FileManager.moveItem"写法。

---

## 六、Alamofire 版 vs 原生版（完整对照）

| 步骤         | 原生版（Swift_02）                                          | Alamofire 版（本文件）                        |
| ---------- | ------------------------------------------------------ | --------------------------------------- |
| 发请求        | `URLRequest` + `URLSession.shared.dataTask` + `resume` | `AF.request(url, method:)` 一行           |
| 参数放 body   | `JSONSerialization.data` + `request.httpBody`          | `parameters` + `encoding: .json`        |
| 参数放 URL    | 字符串插值 `"\(baseURL)/update?id=..."`                     | `parameters` + `encoding: .queryString` |
| 切主线程       | `DispatchQueue.main.async {}`                          | 不用（自动）                                  |
| 解析 JSON    | `JSONSerialization.jsonObject` + `try`                 | `value` 已解析好（`.success` 里直接拿）           |
| 网络错误       | `if let error = error`                                 | `case .failure(let error)`              |
| 空数据检查      | `guard let data, !data.isEmpty`                        | 不用（Alamofire 管了）                        |
| JSON 解析错误  | `do { try } catch`                                     | 不用（Alamofire 管了）                        |
| 业务 code 校验 | `if code != 200`                                       | **一样，自己写**                              |
| 外部调用接口     | `fetchUsers { ok,result,err in }`                      | **完全一样**                                |
| 代码量        | 约 200 行                                                | 约 120 行                                 |

**结论**：Alamofire 省掉的是"请求构造 + 手动解析 JSON + 切主线程 + 三层错误处理"这些**通用体力活**，只保留了"业务逻辑"（code 校验、转 User 数组）。外部调用方**零改动**。

---

## 七、你在 Xcode 里怎么用

### 1. 换实现（要不要删原生版？）

`NetworkManager.swift` 只能有一个。两个版本二选一：

- **替换**：把原生的 NetworkManager.swift 内容删掉，粘上这份 Alamofire 版（**记得先 `pod install` 装好 Alamofire**）
- **对比练**：暂时留着原生版，另存一个 `NetworkManager_Alamofire.swift` 改名对比，跑通后再定

### 2. 调用方（UserListViewController / Swift_04）

```swift
NetworkManager.shared.fetchUsers { ok, result, err in
    if ok {
        self?.users = result as? [User] ?? []
        self?.tableView.reloadData()
    }
}
```

**这一句完全不用改**——因为 `fetchUsers/addUser/updateUser/deleteUser/downloadFile` 的签名（含 `CompletionBlock`/`DownloadProgressBlock`/`DownloadCompletionBlock`）两种版本一模一样。

**下载调用也完全一样：**

```swift
NetworkManager.shared.downloadFile(
    from: urlString,
    progress: { p in print("进度 \(Int(p*100))%") },
    completion: { ok, path, err in ... }
)
```

---

## 八、和 OC 的对应（帮你想通）

| OC 原生版                                  | OC AFNetworking 版                                  | Swift 原生版                               | Swift Alamofire 版                                    |
| --------------------------------------- | -------------------------------------------------- | --------------------------------------- | ---------------------------------------------------- |
| `NSURLSession`                          | `AFHTTPSessionManager`                             | `URLSession`                            | `AF`（`AF.request`）                                   |
| 手动解析 + 切主线程 + 错误检查                      | 一行 `GET:parameters:progress:success:failure:`      | 手动一大段                                   | `AF.request(...).responseJSON`                       |
| 下载：`downloadTaskWithURL` + 3 个 delegate | 下载：`downloadTaskWithRequest:progress:destination:` | 下载：`downloadTask(with:)` + 3 个 delegate | 下载：`AF.download` + `.downloadProgress` + `.response` |
| 代码多、学习成本高                               | 代码少、依赖第三方库                                         | 代码多、零依赖                                 | 代码少、依赖 Alamofire                                     |

**两条路完全对称**：

- OC：原生 NSURLSession vs 第三方 AFNetworking
- Swift：原生 URLSession vs 第三方 Alamofire

你 OC 学过"先原生后 AFN"，Swift 也是**先用 Swift_02 原生跑通，再用这份 Alamofire 对照**，理解最深。

---

**到这里网络层两种版本都齐了。** 你有 CocoaPods，装 `pod 'Alamofire'` 就能用。跑通后这题就有 Swift 原生 + Alamofire 两套了（和 OC 的 NSURLSession + AFN 完全对称）。
