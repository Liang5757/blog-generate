---
title: Leetcode239. 滑动窗口最大值
date: 2021-01-04 00:25:15
tags:
  - leetcode
categories:
  - 算法
---
> 题目链接: https://leetcode-cn.com/problems/sliding-window-maximum/
> 主要是记录一下分块做法，没想懂为什么比双向单调队列快，想懂了回来补充

给你一个整数数组 `nums`，有一个大小为 `k` 的滑动窗口从数组的最左侧移动到数组的最右侧。你只可以看到在滑动窗口内的 `k` 个数字。滑动窗口每次只向右移动一位。

返回滑动窗口中的最大值。
<!--more-->
**示例 1：**

```
输入：nums = [1,3,-1,-3,5,3,6,7], k = 3
输出：[3,3,5,5,6,7]
解释：
滑动窗口的位置                最大值
---------------               -----
[1  3  -1] -3  5  3  6  7       3
 1 [3  -1  -3] 5  3  6  7       3
 1  3 [-1  -3  5] 3  6  7       5
 1  3  -1 [-3  5  3] 6  7       5
 1  3  -1  -3 [5  3  6] 7       6
 1  3  -1  -3  5 [3  6  7]      7
```

**示例 2：**

```
输入：nums = [1], k = 1
输出：[1]
```

**示例 3：**

```
输入：nums = [1,-1], k = 1
输出：[1,-1]
```

**示例 4：**

```
输入：nums = [9,11], k = 2
输出：[11]
```

**示例 5：**

```
输入：nums = [4,-2], k = 2
输出：[4]
```

**提示：**

- `1 <= nums.length <= 105`
- `-104 <= nums[i] <= 104`
- `1 <= k <= nums.length`

## 题解一：双端单调队列

维护一个双端单调队列，队尾存最大值的索引，如果当前遍历的数比队尾大，则一直弹出直到队列重新满足单调队列的性质，则**滑动窗口的最大值为队尾元素**，然后因为需要的只是当前窗口的最大值，所以每次循环需要判断队头是不是在当前窗口上，如果不是则弹出队头

```js
var maxSlidingWindow = function(nums, k) {
    if(nums == null || nums.length < 2) return nums;
    // 双向队列 保存当前窗口最大值的数组位置 保证队列中数组位置的数值按从大到小排序
    let queue = [];
    // 结果数组
    let result = [];
    // 遍历nums数组
    for(let i = 0; i < nums.length; i++){
        // 保证从大到小 如果前面数小则需要依次弹出，直至满足要求
        while(queue.length !== 0 && nums[queue[queue.length - 1]] <= nums[i]){
            queue.pop();
        }
        // 添加当前值对应的数组下标
        queue.push(i);
        // 判断当前队列中队首的值是否有效
        if(queue[0] <= i - k){
            queue.shift();   
        } 
        // 当窗口长度为k时 保存当前窗口中最大值
        if(i + 1 >= k){
            result[i+1-k] = nums[queue[0]];
        }
    }
    return result;
};
```

![image-20210103235509444](image-20210103235509444.png)

## 题解二：优先队列

和单调队列基本一个道理，不过键是索引，值是映射到正向区间的nums[i]，优先队列的队头是最大值，当最大值的索引不属于当前的滑动窗口时，则出队。

```js
var maxSlidingWindow = function(nums, k) {
    let q = new MaxPriorityQueue, r = new Int16Array(nums.length - k + 1), i = -1
    while (++i < k) q.enqueue(i, nums[i] + 10001)
    r[0] = q.front().priority - 10001, i--
    while (++i < nums.length) {
        q.enqueue(i, nums[i] + 10001)
        while (q.front().element <= i - k) q.dequeue()
        r[i - k + 1] = q.front().priority - 10001
    }
    return r;
};
```

![image-20210103235832538](image-20210103235832538.png)

## 题解三：分块

将数组分成`k`块

指针`j `→ 统计每块内从`块开头`到`j`最大值
指针`i `← 统计每块内从`块结尾`到`i`最大值
滑动区间[i, i + k - 1]最大值 = `某块结尾`到`i`最大值 与 `某块开头`到`i + k - 1`最大值 取大

```js
var maxSlidingWindow = function(nums, k) {
    let n = nums.length, p = new Int16Array(n), 
        s = new Int16Array(n), r = new Int16Array(n - k + 1), i = n, j = -1;
    while (i--) {
        p[++j] = j % k ? Math.max(p[j - 1], nums[j]) : nums[j]
        s[i]   = i % k ? Math.max(s[i + 1], nums[i]) : nums[i]
    }
    while (i++ < n - k) r[i] = Math.max(s[i], p[i + k - 1])
    return r
}; 
```

![image.png](1609569283-BTeNfl-image.png)

ps：离谱，为什么这个比双端队列要快的

![image-20210104000919976](image-20210104000919976.png)



