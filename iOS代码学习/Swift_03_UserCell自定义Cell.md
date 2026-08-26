# Swift_03 — UserCell 自定义 Cell

> iOS 网络编程 Swift 版的 UI 第一步：自定义 Cell，显示用户的名字和年龄。
> 对应你 OC 的 `02_UserCell自定义Cell.md`，Swift + Storyboard 重写。
> ⚠️ 布局用 **SnapKit**（Masonry 的 Swift 版）。确保工程 Podfile 已加 `pod 'SnapKit'` 并有 `import SnapKit`（你已装好）。

---

## 一、先回顾你要做的

```
列表每一行（UserCell）：
┌───────────────────────────────────┐
│  张三              25岁            │
└───────────────────────────────────┘
   名字UILabel          年龄UILabel
```

和 OC 一样：`UserCell` 继承 `UITableViewCell`，里面放两个 UILabel，一个显示名字、一个显示年龄。

---

## 二、完整代码

```swift
import UIKit
import SnapKit

class UserCell: UITableViewCell {

    // 两个 UILabel：名字 + 年龄
    // UIButton/UILabel 等控件用 let（引用不变，只是内部内容变）
    let nameLabel = UILabel()
    let ageLabel = UILabel()

    // 复用标识符（对应 OC 的 static NSString * const kCellID）
    static let identifier = "UserCell"

    // 初始化，对应 OC 的 initWithStyle:reuseIdentifier:
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupSubviews()
    }

    // 从 storyboard 创建时会走到这个 init，必须实现
    required init?(coder: NSCoder) {
        fatalError("不支持从 storyboard 解码 UserCell")
    }

    // 搭建子视图
    private func setupSubviews() {
        // 名字
        nameLabel.font = .systemFont(ofSize: 16)
        addSubview(nameLabel)

        // 年龄
        ageLabel.font = .systemFont(ofSize: 14)
        ageLabel.textColor = .gray
        ageLabel.textAlignment = .right
        addSubview(ageLabel)

        // 布局：用 SnapKit（≈ Masonry，比原生锚点更直观）
        // SnapKit 会自动处理 TAMIC，不用手动写 translatesAutoresizingMaskIntoConstraints=false
        nameLabel.snp.makeConstraints { make in
            make.leading.equalTo(contentView).offset(16)   // 左边距 16
            make.centerY.equalTo(contentView)               // 垂直居中
        }
        ageLabel.snp.makeConstraints { make in
            make.trailing.equalTo(contentView).offset(-16)  // 右边距 16
            make.centerY.equalTo(contentView)               // 垂直居中
        }
    }

    // 填充数据，对应 OC 的 configureWithUser:
    func configure(with user: User) {
        nameLabel.text = user.name
        ageLabel.text = "\(user.age) 岁"   // 字符串插值
    }
}
```

---

## 三、逐段拆解 Swift 新语法（UI 才遇到的）

### 3.1 `let nameLabel = UILabel()`

属性不管是 `let` 还是 `var` 都行——这里用 `let` 因为 label 对象本身**不会换**（只是文字内容变），引用不变。

对应 OC：
```objc
@property (nonatomic, strong) UILabel *nameLabel;
// setupSubviews 里：
self.nameLabel = [[UILabel alloc] init];
```
Swift 直接在属性定义时就创建了（`= UILabel()`），不用在 init 里再 alloc。

### 3.2 `override init(style:reuseIdentifier:)`

对应 OC 的 `initWithStyle:reuseIdentifier:`。

```objc
// OC
- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier
```
```swift
// Swift
override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
    super.init(style: style, reuseIdentifier: reuseIdentifier)
    setupSubviews()
}
```
- `override` = 重写父类方法（Swift 必须显式写，不像 OC 可选）
- `super.init(...)` = 先调父类初始化（和 OC 一样的要求）
- 调用格式变成 `init(style: String?)` 这样的命名参数

### 3.3 `required init?(coder: NSCoder)`

**这是从 storyboard 创建控件时 Xcode 强制要求的 init。**

如果你用 storyboard 放了这个 Cell 或用 xib/原型 cell，系统会走这个 init。Swift 规定：子类重写了 init，如果父类有 `required init?(coder:)`，子类也必须实现。

```swift
required init?(coder: NSCoder) {
    fatalError("不支持从 storyboard 解码 UserCell")
}
```

**对你现在的情况**：如果你**纯代码**创建 cell（不拖 storyboard 的原型 cell），这个 init 永远不被调，写 `fatalError` 占位就行（真被调就会崩，提醒你"这里不该走"）。
如果你用了 storyboard 的原型 cell，那就要改为 `super.init(coder: coder)` + setupSubviews，不能用 fatalError。

> 我们的工程**纯代码布局**，注册 cell 时用 `register(UserCell.self, ...)`，所以这个 init 用 fatalError 占位是对的。

### 3.4 `static let identifier = "UserCell"`

对应 OC 的 `static NSString * const kCellID = @"UserCell"`。放 Cell 类里，方便外面 `UserCell.identifier` 引用。

### 3.5 布局：SnapKit（≈ Masonry 的 Swift 版）

SnapKit 是 Masonry 在 Swift 里的移植版，**写法几乎和 `mas_makeConstraints:` 一模一样**，你 OC 用 Masonry 会很熟。

```swift
import SnapKit   // 别忘了 import

nameLabel.snp.makeConstraints { make in
    make.leading.equalTo(contentView).offset(16)   // 左边距 16
    make.centerY.equalTo(contentView)               // 垂直居中
}
ageLabel.snp.makeConstraints { make in
    make.trailing.equalTo(contentView).offset(-16)  // 右边距 16
    make.centerY.equalTo(contentView)
}
```

**和 OC Masonry 的直接对照：**

| OC Masonry | SnapKit（Swift） | 意思 |
|-----------|-----------------|------|
| `[view mas_makeConstraints:^(MASConstraintMaker *make){...}]` | `view.snp.makeConstraints { make in ... }` | 开始加约束 |
| `make.left.equalTo(x).offset(16)` | `make.leading.equalTo(x).offset(16)` | 左边距 16 |
| `make.right.equalTo(x).offset(-16)` | `make.trailing.equalTo(x).offset(-16)` | 右边距 16 |
| `make.centerY.equalTo(x)` | `make.centerY.equalTo(x)` | 垂直居中 |
| `make.top.equalTo(x)` | `make.top.equalTo(x)` | 顶部 |
| `make.size.mas_equalTo(CGSizeMake(80,80))` | `make.size.equalTo(CGSize(width:80,height:80))` | 宽高 |

**几乎就是把 OC 的 `make.left` 换成 `make.leading`、`mas_equalTo` 换 `equalTo`。** 完全一样的思路。

**SnapKit 自动处理 TAMIC**：SnapKit（像 Masonry 一样）内部自动帮你设 `translatesAutoresizingMaskIntoConstraints = false`，所以**不用手动写那行**——这正好对应你之前学 TAMIC 的结论：**用 Masonry/SnapKit 不用手动关，用原生锚点才要手动关。**

### 3.5b 原生锚点（如果不用 SnapKit）

如果你某些地方没装 SnapKit、又必须布局，用原生锚点：

```swift
nameLabel.translatesAutoresizingMaskIntoConstraints = false   // 必须手动关 TAMIC
NSLayoutConstraint.activate([
    nameLabel.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
    nameLabel.centerYAnchor.constraint(equalTo: contentView.centerYAnchor),
])
```

比 SnapKit 啰嗦，且必须手动关 TAMIC，所以**优先用 SnapKit**。

### 3.6 `func configure(with user: User)`

对应 OC 的 `- (void)configureWithUser:(User *)user`。

```swift
func configure(with user: User) {
    nameLabel.text = user.name
    ageLabel.text = "\(user.age) 岁"   // \(变量) = 字符串插值
}
```

**`"\(user.age) 岁"` = 字符串插值**，对应 OC 的 `[NSString stringWithFormat:@"%@ 岁", user.age]`。Swift 里把变量用 `\( )` 直接包进字符串。

---

## 四、OC → Swift 对照表（UserCell）

| OC | Swift |
|----|-------|
| `@property UILabel *nameLabel` | `let nameLabel = UILabel()` |
| `initWithStyle:reuseIdentifier:` | `override init(style:reuseIdentifier:)` |
| `[super initWithStyle:...]` | `super.init(style:...)` |
| `self.nameLabel = [[UILabel alloc] init]` | 属性定义时直接创（`let x = UILabel()`） |
| `[self.contentView addSubview:...]` | `addSubview(...)`（在 Cell 里 contentView 是默认） |
| `Masonry mas_makeConstraints` | `SnapKit snp.makeConstraints`（写法几乎一致） |
| `configureWithUser:` | `func configure(with: User)` |
| `[NSString stringWithFormat:@"%@岁",age]` | `"\(user.age) 岁"` |

---

## 五、纯代码布局的坑（重要）

**1. 不要用 `self.addSubview`，用 `contentView.addSubview`（或直接 addSubview）**
OC 里 Cell 的子视图要放 contentView 上。Swift 里在 Cell 内部 `addSubview(nameLabel)` 实际就是加到 contentView 上（因为 UITableViewCell 的 addSubview 是加到 contentView）。稳妥起见 SnapKit 的参照用 `contentView`（我上面就是这么写的）。

**2. 用 SnapKit 会自动关 TAMIC，不用手动写**
SnapKit（像 Masonry）内部自动设 `translatesAutoresizingMaskIntoConstraints = false`。但如果混用了原生锚点，那部分必须手动关 TAMIC。**统一用 SnapKit 就不用操心 TAMIC。**

**3. 别忘了在 ViewController 里注册 cell**
```swift
tableView.register(UserCell.self, forCellReuseIdentifier: UserCell.identifier)
```

---

## 六、你在 Xcode 里的用法

这一份 UserCell.swift 先建好，放到项目里。下一步 `Swift_04_UserListViewController` 会用它，并调用 `configure(with:)` 填充数据。

`Swift_03` 的 UserCell 先写起来，和 OC 的 UserCell 一样是"每行显示名字+年龄"。写好后我们继续主界面。有问题随时问。
