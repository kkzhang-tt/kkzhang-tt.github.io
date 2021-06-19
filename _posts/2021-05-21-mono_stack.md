---

layout: post
title: "单调栈"
category: "algrithom"
author: "kkzhang"
---

## 定义

栈内元素保持单调有序状态的栈称为单调栈（monotonous stack）。

> 与普通栈一样，只能在栈顶进行入栈/出栈操作。

按照从栈底到栈顶的元素状态，分为递增栈，递减栈；如下：

<img src="https://raw.githubusercontent.com/kkzhang-tt/kkzhang-tt.github.io/main/_images/mono_stack_1.png" style="zoom:50%;" />

## 实现

将一个元素插入单调栈时，为了维护栈的单调性，需要在保证将该元素插入到栈顶后整个栈满足单调性的前提下弹出最少的元素。

例如，栈中自顶向下的元素为 1, 3, 5, 7, 20，插入元素 10 时为了保证单调性需要依次弹出元素 1, 3, 5, 7，操作后栈变为 10, 20。

### 递减栈

元素入栈需要满足一下条件之一：

- 栈为空
- 当前元素比栈顶元素小

如果上述条件无法满足，则依次弹出栈顶元素，直到满足条件。

```java
private static Stack<Integer> monoStack = new Stack<>();

public static void push(Integer element) {
    // 用于维护一个递减栈：自底向上递减
    // 当栈为空或者当前元素比栈顶元素小的时候，入栈
    // 否则将栈顶元素弹出，直到满足入栈条件
    while (!monoStack.empty() && monoStack.peek() < element) {
        monoStack.pop();
    }
    monoStack.push(element);
}
```

### 递增栈

元素入栈需要满足一下条件之一：

- 栈为空
- 当前元素比栈顶元素大

如果上述条件无法满足，则依次弹出栈顶元素，直到满足条件。

```java
private static Stack<Integer> monoStack = new Stack<>();

public static void push(Integer element) {
    // 用于维护一个递增栈：自底向上递增
    // 当栈为空或者当前元素比栈顶元素大的时候，入栈
    // 否则将栈顶元素弹出，直到满足入栈条件
    while (!monoStack.empty() && monoStack.peek() > element) {
        monoStack.pop();
    }
    monoStack.push(element);
}
```

## 应用

单调栈是一种定义比较简单但是应用起来比较复杂的结构，其根本用法可以总结为：**在一个一维数组中帮助我们找到某个元素左边（右边）第一个大于（小于）当前元素的数**。

如在数组 [5, 4, 3, 2, 8, 10, 12] 中，元素 5 右侧第一个比其大的数为 8；元素 4 右侧第一个比其小的数为 3。

为了解决这类问题，我们通常会维护一个单调栈（递增/递减视情况而定），依次将原数组中的元素入栈；在元素入栈的同时，可能会伴有栈顶元素的出栈，此时意味着**栈顶元素遇到了第一个比其大（或小）的数**，相应的，我们需要在栈顶元素出栈的时候维护一些数据以满足题目要求。

> 解决问题的关键在于理解出栈的含义

### 第一个较大数

给一个数组，返回一个大小相同的数组。返回的数组的第 i 个位置的值应当是，对于原数组中的第 i 个元素，至少往右走多少步，才能遇到一个比自己大的元素（如果之后没有比自己大的元素，或者已经是最后一个元素，则在返回数组的对应位置放上-1）。

简单的例子：

```java
input: 5, 3, 1, 2, 4
output: -1 3 1 1 -1
// 对于元素 5，其右侧没有比它大的数，因此返回 -1；
// 对于元素 3，其右侧第一个比它大的数是 4，需要向右走 3 步。
```

对于这题可以使用暴力法求解，但是时间复杂度为 O(n^2)。

如果使用单调栈的话，我们需要维护一个递减栈：将原数组元素依次入栈，如果栈顶元素遇到一个比它大的元素，则这个最大的元素就是右侧第一个比它大的元素（由于单调栈的性质，此时栈中的元素都比栈顶小）。使用单调栈之后，每个元素只会入栈，出栈一次，时间复杂度为 O(n)。

```java
public static int[] firstLargerElement(int[] input) {
      Stack<Integer> monoStack = new Stack<>();
      int[] result = new int[input.length];
      // init result array
      for (int i = 0; i < input.length; i++) {
          result[i] = -1;
      }
      // iterate array
      for (int i = 0; i < input.length; i++) {
          while (!monoStack.empty() && input[monoStack.peek()] < input[i]) {
              // 当前栈顶对应的值遇到第一个比它大的数值
              int top = monoStack.pop();
              result[top] = i - top;
          }
          monoStack.push(i);
      }
      return result;
  }
```

单调栈中用于存储各个元素的索引(index)，当遇到一个比栈顶元素大的数时，这个数就是当前栈顶右侧最近的一个较大数（这个数是否比栈内其他数大并不重要）；栈顶元素出栈时，记录当前较大元素与栈顶元素的间隔，并更新返回结果。

> 如果某个元素一直不出栈，则说明右侧并没有比它大的数；对于递减数组，那么所有的元素只会入栈，不会出栈

### 数帽子问题

现有一条排列整齐的队伍，队员们都戴着帽子，身高是无序的。假设每个人能看到队伍中在他前面的比他个子矮的人的帽子，（如果出现一个比这个人个子高的人挡住视线，那么此人不能看到高个子前面的任何人的帽子）。现在需要计算整个队伍中总共能看到多少个帽子（所有人看到帽子的总和）。

```java
input: 2, 1, 5, 6, 2, 3 // 代表队伍中每个人的身高，顺序为队尾到队首
output: 3 // 身高为 2 的队员能看到前面身高为 1 的队员帽子；身高为 6 的队员可以看到前面身高为 2，3 的帽子
					// 其他队员均不能看到别人的帽子
```

该问题与上面 “第一个较大元素” 类似，都需要找到某个元素右侧第一个较大值。

同样，我们维护一个递减队列，当栈顶遇到一个较大的元素时，栈顶出栈并记录当前 index 间隔表示能够看到的较矮队员的数目。

```java
public static int countHats(int[] input) {
      Stack<Integer> monoStack = new Stack<>();
      int sum = 0;
      // 为了防止原数组是递减的，导致没有元素出栈
      // 可以在原数组的尾部添加一个 Integer.MAX
      // 从而使得原数组的所有元素都会出栈
      int[] array = new int[input.length + 1];
      array[input.length] = Integer.MAX_VALUE;
      for (int i = 0; i < input.length; i++) {
          array[i] = input[i];
      }
      // iterate array
      for (int i = 0; i < array.length; i++) {
          while (!monoStack.empty() && input[monoStack.peek()] < array[i]) {
              // 当前栈顶对应的值遇到右侧第一个比它大的数值
              int top = monoStack.pop();
              sum += (i - top - 1);
          }
          monoStack.push(i);
      }
      return sum;
  }
```

> 在上一个题目中，如果有元素一直不出栈，是没什么影响的（初始化为 -1）；但是在该题目中，如果有元素不能出栈，会导致整体的计算出现偏差，为了使得所有元素都出栈，在原始数组的尾部添加一个最大值，这样原来数组的每个元素都会遇到这个比其大的值，从而出栈。

### 最大矩形面积

> [leetcode]: https://leetcode-cn.com/problems/largest-rectangle-in-histogram/?utm_source=LCUS&amp;utm_medium=ip_redirect&amp;utm_campaign=transfer2china

给定 n 个非负整数，用来表示柱状图中各个柱子的高度。每个柱子彼此相邻，且宽度为 1 。求在该柱状图中，能够勾勒出来的矩形的最大面积。

<img src="https://raw.githubusercontent.com/kkzhang-tt/kkzhang-tt.github.io/main/_images/mono_stack_3.png"/>

以上是柱状图的示例，其中每个柱子的宽度为 1，给定的高度为 [2,1,5,6,2,3]。图中阴影部分为所能勾勒出的最大矩形面积，其面积为 10 个单位。

该问题同样可以用单调栈来解决，但是需要分析下，需要理解如何找到这个最大面积矩形。这个矩形的限制条件有两个，一个是高度（也即组成矩形的最短的那根柱子高度），一个是宽度，（也即组成矩形的柱子个数）。为了找到这个全局最大值，我们遍历所有局部最优情况。

那么什么是局部最优解呢，**我们将每个柱子的高度作为包含它的矩形的高度，也即这个柱子一定是这个矩形中最低的一个柱子，那么我们下一步是求解这个矩形的宽度，显然我们只需找到这个柱子左边，右边第一个比它低的柱子，就可以求出宽度**。

> 在左边 ～ 右边第一个比其低的柱子之间，所有的柱子都不比当前柱子低，从而可以以当前柱子作为整个矩形的高。

该题维护的是递增队列：

```java
public static int largestRectangleArea(int[] input) {
      Stack<Integer> monoStack = new Stack<>();
      int maxArea = 0;
      for (int i = 0; i < input.length; i++) {
          while (!monoStack.isEmpty() && input[monoStack.peek()] > input[i]) {
              // 栈顶元素的值即为高
              int height = input[monoStack.pop()];
              // 此时栈顶元素遇到右侧第一个比其小的元素
              // 该元素即为宽的右侧值
              int right = i;
              // 栈顶元素左侧第一个比其小的值为栈中的下一个元素(栈有序)
              // 如果此时栈为空，则栈顶元素左侧没有比其小的值了
              int left = monoStack.isEmpty() ? 0 : monoStack.peek();
              int width = right - left - 1;
              maxArea = Math.max(maxArea, height * width);
          }
          monoStack.push(i);
      }
      return maxArea;
  }
```

> 该题目的关键是找出矩形面积的计算方法，依次以原数组中的每个值作为矩形的高，并计算该元素下矩形的最大宽，从而计算当前矩形的最大面积。

## 总结

单调栈这种数据结构，通常应用在一维数组上。如果遇到的问题，和前后元素之间的大小有关系的话，（例如第一题中我们要找比某个元素大的元素，第三个题目中，前后柱子的高低影响了最终矩形的计算），我们可以试图用单调栈来解决。