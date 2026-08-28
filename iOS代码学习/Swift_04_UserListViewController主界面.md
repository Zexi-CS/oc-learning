# Swift_04 — UserListViewController 主界面

> iOS 网络编程 Swift 版最后一块：主界面。把 NetworkManager + UserCell 串起来。
> 对应你 OC 的 `06/07`（主界面 + 增删改查），Swift + Storyboard 重写。
> 功能：列表显示 + 新增 + 编辑 + 删除。
> ⚠️ 布局用 **SnapKit**（和 Swift_03 一致，`make.edges.equalTo(view)` 铺满）。确保有 `import SnapKit`。

---

## 一、这个页面要做什么

```
导航栏：用户列表                    [+ 新增]
┌───────────────────────────────────┐
│  张三              25岁           │  ← UserCell
│  李四              30岁           │
│  ...                              │
└───────────────────────────────────┘
   交互：
   - 右上角 + → 弹窗输姓名年龄 → 新增
   - 左滑 → 编辑 / 删除
```

和你 OC 的 06 一模一样，只是 Swift 语法。

---

## 二、完整代码（连续一整份，直接能复制）

```swift
import UIKit
import SnapKit

class UserListViewController: UIViewController {

    // ═══ 属性 ═══
    private var users: [User] = []                          // 数据源（对应 OC 的 _users）
    private let tableView = UITableView(frame: .zero, style: .plain)   // 表格

    // ════════════════════════════════════════════════════════
    // 生命周期
    // ════════════════════════════════════════════════════════
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .white
        title = "用户列表"

        setupTableView()    // 搭表格
        setupNavBar()       // 右上角 + 按钮

        fetchUsers()        // 拉取数据
    }

    // ─── 搭表格 ───
    private func setupTableView() {
        tableView.dataSource = self    // 数据源 = 自己（协议用 extension 声明）
        tableView.delegate = self      // 委托   = 自己
        tableView.rowHeight = 60
        tableView.register(UserCell.self, forCellReuseIdentifier: UserCell.identifier)

        view.addSubview(tableView)
        // 设 SnapKit 约束铺满全屏（类似 OC 的 make.edges.equalTo(self.view)）
        tableView.snp.makeConstraints { make in
            make.edges.equalTo(view)   // 上下左右都等于 view → 铺满
        }
    }

    // ─── 右上角 + 新增按钮 ───
    private func setupNavBar() {
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addButtonTapped)   // 点击调 addButtonTapped
        )
    }

    // ════════════════════════════════════════════════════════
    // 网络 —— 拉取列表
    // ════════════════════════════════════════════════════════
    private func fetchUsers() {
        NetworkManager.shared.fetchUsers { [weak self] success, result, _ in
            // 闭包已在主线程（NetworkManager 内部切过），可直接刷 UI
            if success {
                self?.users = result as? [User] ?? []
                self?.tableView.reloadData()
            }
        }
    }

    // ════════════════════════════════════════════════════════
    // 新增 / 编辑 —— 共用同一个弹窗
    // ════════════════════════════════════════════════════════
    @objc private func addButtonTapped() {
        showEditAlert(title: "新增用户", user: nil)   // user 传 nil = 新增
    }

    private func showEditAlert(title: String, user: User?) {
        let alert = UIAlertController(title: title, message: nil, preferredStyle: .alert)

        // 姓名输入框：新增显示占位，编辑预填当前名字
        alert.addTextField { tf in
            tf.placeholder = "姓名"
            if let u = user { tf.text = u.name }   // 编辑：预填名字
        }
        // 年龄输入框：新增显示占位，编辑预填当前年龄
        alert.addTextField { tf in
            tf.placeholder = "年龄"
            tf.keyboardType = .numberPad
            if let u = user { tf.text = u.age }    // 编辑：预填年龄
        }

        // 取消
        alert.addAction(UIAlertAction(title: "取消", style: .cancel))

        // 确认
        alert.addAction(UIAlertAction(title: "确认", style: .default) { [weak self, weak alert] _ in
            guard let self = self,
                  let name = alert?.textFields?[0].text,
                  let age  = alert?.textFields?[1].text else { return }

            if let user = user {
                // 有 user = 编辑
                NetworkManager.shared.updateUser(id: user.userId, name: name, age: age) { [weak self] _,_ ,_ in
                    self?.fetchUsers()
                }
            } else {
                // 没 user = 新增
                NetworkManager.shared.addUser(name: name, age: age) { [weak self] _,_ ,_ in
                    self?.fetchUsers()
                }
            }
        })

        present(alert, animated: true)
    }
}

// ════════════════════════════════════════════════════════
// UITableViewDataSource —— 提供数据
// ════════════════════════════════════════════════════════
extension UserListViewController: UITableViewDataSource {

    // 问：有几行？→ 数据有多少个就几行
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return users.count
    }

    // 问：第 indexPath.row 行显示什么？→ 取 Cell 填数据
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: UserCell.identifier, for: indexPath) as! UserCell
        cell.configure(with: users[indexPath.row])   // 填第 row 行的用户
        return cell
    }
}

// ════════════════════════════════════════════════════════
// UITableViewDelegate —— 处理交互
// ════════════════════════════════════════════════════════
extension UserListViewController: UITableViewDelegate {

    // 左滑操作：编辑 + 删除（对应 OC 的 trailingSwipeActionsConfiguration）
    func tableView(_ tableView: UITableView,
                   trailingSwipeActionsConfigurationForRowAt indexPath: IndexPath) -> UISwipeActionsConfiguration? {
        let user = users[indexPath.row]

        // 删除按钮（红色）
        let delete = UIContextualAction(style: .destructive, title: "删除") { [weak self] _, _, completion in
            NetworkManager.shared.deleteUser(id: user.userId) { [weak self] _,_ ,_ in
                self?.fetchUsers()
            }
            completion(true)   // 收起左滑
        }

        // 编辑按钮（蓝色）
        let edit = UIContextualAction(style: .normal, title: "编辑") { [weak self] _, _, completion in
            self?.showEditAlert(title: "编辑用户", user: user)   // 弹编辑框
            completion(true)
        }
        edit.backgroundColor = .systemBlue

        return UISwipeActionsConfiguration(actions: [delete, edit])
    }
}
```

**说明**：这份代码和之前是同一个逻辑，只是把所有方法按"生命周期 → 网络 → 弹窗 → DataSource → Delegate"顺序**连续排好**，中间不停顿，你可以整个 `.swift` 直接复制。

---

## 三、整体结构图（这文件就五块）

```
UserListViewController.swift
│
├─① class UserListViewController
│     ├─ 属性（users、tableView）
│     ├─ viewDidLoad（入口）
│     ├─ setupTableView（搭表格+dataSource赋值）
│     ├─ setupNavBar（+按钮）
│     ├─ fetchUsers（拉数据）
│     └─ showEditAlert（新增/编辑共用弹窗）
│
├─② extension ... UITableViewDataSource
│     ├─ numberOfRowsInSection（几行）
│     └─ cellForRowAt（每行显示）
│
└─③ extension ... UITableViewDelegate
      └─ trailingSwipeActions（左滑 编辑/删除）
```

---

## 四、关键语法速查（Swift_04 涉及的）

| 语法 | 对应 OC | 说明 |
|------|---------|------|
| `extension 类: 协议` | `@interface ... <协议>` | 声明 + 实现协议（分块） |
| `tableView.dataSource = self` | `self.tableView.dataSource = self` | 赋值（协议在 extension 里声明） |
| `override func viewDidLoad()` | `- (void)viewDidLoad` | 重写生命周期 |
| `dequeueReusableCell(...) as! UserCell` | `dequeue...` | 取 Cell，强制转 UserCell |
| `[weak self]` | `__weak` | 防循环引用 |
| `guard let self = self` | `if(!weakSelf)return` | weak 解包成强引用 |
| `@objc func` + `#selector` | `@selector` | 按钮 target-action |
| `UIAlertController` | `UIAlertController` | 弹窗 |
| `UIContextualAction` | `UIContextualAction` | 左滑按钮 |
| `completion(true)` | `completionHandler(YES)` | 关左滑 |

---

## 五、工程里最后要做的事

1. 建 `UserListViewController.swift`（Cocoa Touch Class，Subclass of: **UIViewController**）
2. 把上面完整代码粘进去
3. 让 App 显示这个页面 + 包个 UINavigationController（才有右上角"+"和 title）：
   - 方法一：Main.storyboard 里把默认 VC 的 Class 改 `UserListViewController`，并选中它 → Editor → Embed In → Navigation Controller
   - 方法二：SceneDelegate 里代码设 `rootViewController`（包 UINavigationController）

> 记得类要在同一个 target 里（都在同一项目就行）。

---

**到这里 Swift 原生版整套齐了：**
```
User  +  NetworkManager  +  UserCell  +  UserListViewController
(01)      (02)               (03)          (04)
```

把 UserCell、UserListViewController 都建好后运行，列表 + 增删改查就通了。敲的过程中有问题随时贴给我。
