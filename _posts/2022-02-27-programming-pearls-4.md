---

layout: post
title: "编程珠玑: 编写正确的程序"
author: "kkzhang"
category: algrithom

---

## 二分搜索

二分搜索的目标是确定在排序数组 x[0...n-1] 中是否存在目标元素 t，如果存在则返回对应的索引 p，否则返回 -1。

该算法实现思路为：通过**持续跟踪数组中包含元素 t 的范围**（如果 t 存在数组中）。

1. 一开始，该范围是整个数组
2. 通过将 t 与数组的中间项进行比较并抛弃一半的范围来缩小目标范围
3. 持续步骤 2，直到在数组中找到 t 或者确定包含 t 的范围为空时为止

在包含 n 个元素的数组中，二分搜索需要执行 $logn$ 次比较操作。

> 线性搜索需要 $n/2$ 次比较
> 

## 编写程序

二分搜索的思想为：**如果 t 在 x[0...n-1] 中，那么 t 一定存在于 x 的某个特定范围内**。

通过使用**循环不变式** `mustbe(range)` 来表示：如果 t 在数组中，那么其一定在 range 内。

```java
init range to 0...n-1
loop 
    {invariant: mustbe(range)}
    if range is empty, 
        break and return -1;
    cumpute m, the middle of the range
    use m as a probe to shrink the range
        if t is found during the shrinking process, break and report its position
```

循环不变式是程序状态的断言（assertion），在循环迭代之前和之后，该断言都为真。

range 用下标`（l, u）`表示，`mustbe(l,u)` 表示：如果 t 在数组中，则一定在 `x[l...u]` 内。

```java
l = 0; u = n-1;
loop:
    {invariant: mustbe(l,u)}
    if l > u
        p = -1; break;
    m = (l + u)/2
    case:
        x[m] < t: l = m + 1;
        x[m] = t: return m;
        x[m] > t: u = m - 1;
```

在上面的伪代码中，如果 `x[m] < t`，那么 `x[0] ≤ x[1] ... ≤ x[m] < t`，所以 t 不会存在 `x[0...m]` 范围内。将该结论与已知条件 t 存在 `x[l...u]` 之内，可知 t 一定在 `x[m+1, u]` 之内，即为 `mustbe(m+1, u)`。因此，通过将 l 设为 m+1 可以再次确立不变式 `mustbe(l, u)`。

## 理解程序

上述代码的正确性分为 3 个部分，每部分都与循环不变式相关：

1. **初始化**
    
    循环初次执行时，不变式为真。
    
2. **保持**
    
    如果某次迭代开始的时候 & 循环体执行的时候，不变式都为真，那么，循环体执行完毕的时候不变式仍为真。
    
3. **终止**
    
    循环能够终止，并且可以得到期望的结果（需要用到不变式所确立的事实）。
    

## 代码

```java
// 迭代
private static int binarySearch(int[] x, int t) {
    int l = 0;
    int u = x.length - 1;
    while (l <= u) {
        // mustbe(l, u)
        int m = (l + u) / 2;
        if (x[m] > t) {
            u = m - 1;
        } else if (x[m] == t) {
            return m;
        } else {
            l = m + 1;
        }
    }
    return -1;
}

// 递归
private static int binarySearch(int[] x, int t) {
    return binarySearch(x, t, 0, x.length - 1);
}

private static int binarySearch(int[] x, int t, int l, int u) {
    if (l > u) {
        return -1;
    }

    int m = (l + u) / 2;
    if (x[m] == t) {
        return m;
    } else if (x[m] > t) {
        return binarySearch(x, t, l, m - 1);
    } else {
        return binarySearch(x, t, m + 1, u);
    }
}
```

