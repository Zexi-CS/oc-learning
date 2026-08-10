# 01 — NSString 拼接分类（Category + 方法复用）

> 第一步：给 NSString 扩展 6 个类方法，少参调多参，只有 6 参的真干活。

---

## 一、题目要求

给 NSString 设计一个分类，扩展 6 个方法，分别有 1~6 个 NSString 参数。返回值均为 NSString，用下划线 `_` 拼接所有参数。要求按 SDWebImage 复用方法的方式实现。

---

## 二、什么是"SDWebImage 复用方式"

6 个方法里，只有参最多的那个（6 参）真正做拼接。其余 5 个全是用 `@""` 补空缺，然后调下一个更多参的方法。

```
concat1  →  concat2  →  concat3  →  concat4  →  concat5  →  concat6（真干活）
  只有 a     a, b 补空    补空          补空          补空      全参数到齐，拼！
```

本质：**写一次逻辑，留 6 个入口。**

---

## 三、NSString+Concatenation.h

```objc
//
//  NSString+Concatenation.h
//  给 NSString 扩展 6 个拼接方法，少参调多参，6 参做地基
//

#import <Foundation/Foundation.h>

@interface NSString (Concatenation)

// ─── 6 个类方法，分别接收 1~6 个参数 ───
// 全部返回非空参数用 "_" 拼接成的字符串

+ (NSString *)concat1:(NSString *)a;
+ (NSString *)concat2:(NSString *)a _:(NSString *)b;
+ (NSString *)concat3:(NSString *)a _:(NSString *)b __:(NSString *)c;
+ (NSString *)concat4:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d;
+ (NSString *)concat5:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e;
+ (NSString *)concat6:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e _____:(NSString *)f;

@end
```

**参数名规律：** 第 N 个参数的参数名 = N-1 个下划线。
- 第 1 个：a（无下划线）
- 第 2 个：_（1 个下划线）
- 第 3 个：__（2 个下划线）
- ...

---

## 四、NSString+Concatenation.m

```objc
//
//  NSString+Concatenation.m
//

#import "NSString+Concatenation.h"

@implementation NSString (Concatenation)

// ============================================================
// 地基方法：只有这一个真正干活，所有拼接逻辑都在这里
// ============================================================
+ (NSString *)concat6:(NSString *)a _:(NSString *)b __:(NSString *)c
                   ___:(NSString *)d ____:(NSString *)e _____:(NSString *)f {

    // 把有值的都收集进可变数组
    NSMutableArray *parts = [NSMutableArray array];
    if (a.length > 0) [parts addObject:a];
    if (b.length > 0) [parts addObject:b];
    if (c.length > 0) [parts addObject:c];
    if (d.length > 0) [parts addObject:d];
    if (e.length > 0) [parts addObject:e];
    if (f.length > 0) [parts addObject:f];

    // NSArray 系统方法：用 "_" 连接所有元素
    return [parts componentsJoinedByString:@"_"];
}

// ============================================================
// 以下 5 个：缺的参数用 "" 补上，调下一个更多参的方法
// ============================================================

+ (NSString *)concat1:(NSString *)a {
    return [self concat2:a _:@""];                      // a, @"" → 调 2 参
}

+ (NSString *)concat2:(NSString *)a _:(NSString *)b {
    return [self concat3:a _:b __:@""];                 // a, b, @"" → 调 3 参
}

+ (NSString *)concat3:(NSString *)a _:(NSString *)b __:(NSString *)c {
    return [self concat4:a _:b __:c ___:@""];           // 补空 → 调 4 参
}

+ (NSString *)concat4:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d {
    return [self concat5:a _:b __:c ___:d ____:@""];    // 补空 → 调 5 参
}

+ (NSString *)concat5:(NSString *)a _:(NSString *)b __:(NSString *)c ___:(NSString *)d ____:(NSString *)e {
    return [self concat6:a _:b __:c ___:d ____:e _____:@""]; // 补空 → 调 6 参
}

@end
```

---

## 五、调用演示

```objc
[NSString concat1:@"A"]                                    → "A"
[NSString concat2:@"A" _:@"B"]                             → "A_B"
[NSString concat3:@"A" _:@"B" __:@"C"]                     → "A_B_C"
[NSString concat4:@"A" _:@"B" __:@"C" ___:@"D"]           → "A_B_C_D"
[NSString concat6:@"A" _:@"B" __:@"C" ___:@"D" ____:@"E" _____:@"F"]  → "A_B_C_D_E_F"
```

---

## 六、拆解理解

**每一步传参：**

```
concat1  @"A"
    ↓ 补 @"" 变成 2 个
concat2  @"A", @""
    ↓ 补 @"" 变成 3 个
concat3  @"A", @"", @""
    ↓
...
concat6  @"A", @"", @"", @"", @"", @""
    ↓ 遍历 → 6 个都检查 → 只有 a 有长度 → parts = [@"A"]
    ↓ componentsJoinedByString:@"_" → 只有一个元素，不加分隔符
    ↓ 返回 "A"
```

**空字符串不参与拼接：** `length > 0` 把 `@""` 全部过滤掉，最终结果里不会有连续下划线。

---

## 七、本步骤涉及的知识点

| 知识点 | 在哪里体现 |
|--------|-----------|
| Category（分类） | `@interface NSString (Concatenation)` |
| 类方法（`+` 方法） | 6 个方法全是 `+`，不用创建对象 |
| 方法链式复用 | 1~5 参全调下一级，最终到 6 参 |
| `componentsJoinedByString:` | NSArray 系统拼接方法 |
| 防御性过滤 | `length > 0` 防空字符串干扰结果 |
