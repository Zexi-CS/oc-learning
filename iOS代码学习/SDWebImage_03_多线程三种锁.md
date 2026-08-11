# 03 — 300 子线程 + 三种锁

> 第三步：开 300 个线程同时干活，用三种不同的锁保护共享数据。独立工程。

---

## 一、题目要求

在 300 个子线程中，将 UIImage 对象转换成 NSData。每张图转换完成时：
- 打印"当前是第几个子线程 + 图片体积（KB，保留两位小数）"
- 将打印的字符串添加到可变数组

最后在主线程打印这个可变数组的内容。

分别用 `dispatch_semaphore_t`、`NSLock`、`@synchronized` 实现。

---

## 二、为什么需要锁——理解最核心的问题

300 个线程同时往一个可变数组里加东西：

```
线程 1：读数组 ["A"] → 准备写 ["A", "1"]     ─┐
线程 2：读数组 ["A"] → 准备写 ["A", "2"]     ─┤ 同时读到 ["A"]
                                             │
线程 1：写 ["A", "1"]                        ├─ 两人都觉得自己是第二个
线程 2：写 ["A", "2"]   ← 把线程 1 的覆盖了！ ─┘

结果：少了数据或者崩溃
```

**锁的作用：** 任何线程要操作数组，先拿钥匙。拿到才能进，干完走人。其他人在外面排队。

```
线程 1：拿钥匙 → 读 → 写 → 放钥匙
线程 2：               排队等着 → 拿钥匙 → 读 → 写 → 放钥匙
```

三种锁写法不同，但做的事一模一样：**保证同一时刻只有一个线程在操作共享数据。**

---

## 三、方案一：`@synchronized`（最简单）

```objc
@synchronized(锁对象) {
    // 只有拿到锁的线程能进这里，其他人在外面等
    [array addObject:str];
}
```

只要把操作数组的代码包在 `@synchronized` 里，系统自动帮你排队。

### 完整代码

```objc
#import <UIKit/UIKit.h>

- (void)runSynchronizedDemo {
    UIImage *image = [UIImage imageNamed:@"test"];        // 准备一张测试图
    NSMutableArray *results = [NSMutableArray array];     // 共享数组（300 个线程都要写）
    dispatch_group_t group = dispatch_group_create();     // 等所有人干完的通知器

    for (int i = 0; i < 300; i++) {
        dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
            // ↑ 在子线程执行

            // ① 图片转 NSData
            NSData *data = UIImageJPEGRepresentation(image, 1.0);
            double sizeKB = data.length / 1024.0;

            // ② 拼字符串
            int threadNum = i + 1;
            NSString *str = [NSString stringWithFormat:@"线程%d：%.2fKB", threadNum, sizeKB];

            // ③ 加锁 → 安全写入数组
            @synchronized(results) {
                [results addObject:str];
            }
        });
    }

    // ④ 等 300 个线程全部完成 → 主线程打印
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        for (NSString *s in results) {
            NSLog(@"%@", s);
        }
        NSLog(@"总计 %lu 条", (unsigned long)results.count);
    });
}
```

### `dispatch_group` 的作用

发出去的 300 个任务不知道什么时候跑完。`dispatch_group_notify` 会等所有人干完才执行 Block——确保打印时数组已经填满了。

---

## 四、方案二：`NSLock`（手动控制）

```objc
NSLock *lock = [[NSLock alloc] init];

[lock lock];      // 拿钥匙（拿不到就等）
[array addObject:str];
[lock unlock];    // 放钥匙
```

和 `@synchronized` 一样的作用，但需要自己调 `lock` 和 `unlock`。**忘记 `unlock` 会导致死锁——所有线程永远等着。**

### 完整代码（只改锁的部分）

```objc
- (void)runNSLockDemo {
    UIImage *image = [UIImage imageNamed:@"test"];
    NSMutableArray *results = [NSMutableArray array];
    NSLock *lock = [[NSLock alloc] init];               // ★ 创建一把锁
    dispatch_group_t group = dispatch_group_create();

    for (int i = 0; i < 300; i++) {
        dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
            NSData *data = UIImageJPEGRepresentation(image, 1.0);
            double sizeKB = data.length / 1024.0;
            NSString *str = [NSString stringWithFormat:@"线程%d：%.2fKB", i + 1, sizeKB];

            [lock lock];                                // ★ 上锁
            [results addObject:str];
            [lock unlock];                              // ★ 解锁
        });
    }

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        for (NSString *s in results) {
            NSLog(@"%@", s);
        }
        NSLog(@"总计 %lu 条", (unsigned long)results.count);
    });
}
```

---

## 五、方案三：`dispatch_semaphore_t`（信号量）

信号量内部维护一个计数器。`signal` 加 1，`wait` 减 1。减到 0 后谁再 `wait` 就得等。

```objc
dispatch_semaphore_t sem = dispatch_semaphore_create(1);  // 初始值 = 1（最多 1 个人进去）

dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);  // 计数器 -1（变成 0，下一个人得等）
[array addObject:str];
dispatch_semaphore_signal(sem);                        // 计数器 +1（变回 1，下一个人可以进了）
```

### 完整代码

```objc
- (void)runSemaphoreDemo {
    UIImage *image = [UIImage imageNamed:@"test"];
    NSMutableArray *results = [NSMutableArray array];
    dispatch_semaphore_t sem = dispatch_semaphore_create(1);  // ★ 只允许 1 个线程进入
    dispatch_group_t group = dispatch_group_create();

    for (int i = 0; i < 300; i++) {
        dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
            NSData *data = UIImageJPEGRepresentation(image, 1.0);
            double sizeKB = data.length / 1024.0;
            NSString *str = [NSString stringWithFormat:@"线程%d：%.2fKB", i + 1, sizeKB];

            dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);  // ★ 等信号
            [results addObject:str];
            dispatch_semaphore_signal(sem);                        // ★ 发信号
        });
    }

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        for (NSString *s in results) {
            NSLog(@"%@", s);
        }
        NSLog(@"总计 %lu 条", (unsigned long)results.count);
    });
}
```

---

## 六、三种锁对比

| | @synchronized | NSLock | dispatch_semaphore |
|------|-------------|--------|-------------------|
| 写法 | `@synchronized(obj) { ... }` | `[lock lock]` / `[lock unlock]` | `wait` / `signal` |
| 锁对象 | 任意 OC 对象 | NSLock 实例 | 信号量（计数器） |
| 容易出错吗 | 最安全（自动解锁） | 忘解锁会死锁 | 计数值设错会错乱 |
| 使用场景 | 简单加锁 | 需要 tryLock | 控制并发数（不止锁） |
| 解锁方式 | 自动（出 {} 就解） | 手动 `unlock` | 手动 `signal` |

信号量不止能做锁——设 `dispatch_semaphore_create(3)` 就可以让 3 个线程同时进去，控制下载并发数等场景。

---

## 七、运行方式

独立工程，新建一个 ViewController，`viewDidLoad` 里任选一个方案跑：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self runSynchronizedDemo];  // 或 runNSLockDemo / runSemaphoreDemo
}
```

在 Xcode 的 Console 里看输出，最后确认 `总计 300 条`。

---

## 八、本步骤涉及的知识点

| 知识点 | 在哪里体现 |
|--------|-----------|
| GCD 多线程 | `dispatch_get_global_queue`、`dispatch_group_async` |
| `dispatch_group` 等待所有任务完成 | `dispatch_group_notify` |
| 线程安全问题 | 300 线程同时写一个数组 |
| `@synchronized` | 最简单的加锁方式 |
| `NSLock` | 手动 lock/unlock |
| `dispatch_semaphore_t` | 信号量控制并发 |
| 主线程回调 | `dispatch_get_main_queue` |
