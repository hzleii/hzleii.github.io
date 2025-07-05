---
title: "1480.一堆数组的动态和"
description: "1480.一堆数组的动态和"
date: 2020-04-18
topic: leetcode
mathjax: true
author: hzlei
# banner: /assets/vuejs/vuejs.webp
# cover: /assets/vuejs/vuejs.webp
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 1480.一堆数组的动态和 # 必选
  caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  # color: white # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...

---



## 题目描述


**给你一个数组 nums 。数组「动态和」的计算公式为：**

$$ \tt {runningSum[i] = sum(nums[0]…nums[i])} $$

**请返回$ \tt {nums} $的动态和。**


{% box child:tabs %}
{% tabs %}

<!-- tab 示例 1 -->
```
输入：nums = [1,2,3,4]
输出：[1,3,6,10]
解释：动态和计算过程为 [1, 1+2, 1+2+3, 1+2+3+4] 。
```

<!-- tab 示例 2 -->
```
输入：nums = [1,1,1,1,1]
输出：[1,2,3,4,5]
解释：动态和计算过程为 [1, 1+1, 1+1+1, 1+1+1+1, 1+1+1+1+1] 。
```

<!-- tab 示例 3 -->
```
输入：nums = [3,1,2,10,1]
输出：[3,4,6,16,17]
```

{% endtabs %}
{% endbox %}



**提示：**

- $ \tt {1 <= nums.length <= 1000} $
- $ \tt {-10^6 <= nums[i] <= 10^6} $



## 题解

通过观察和思考可得到：

1. 最终答案中，第$ \tt{1} $个值不用变，第$ \tt{n} $个值是参数中数组第$ \tt{1} $到第$ \tt{n} $个值的和
2. 不用新开一个数组，直接在原数组中操作即可
3. 只用一次遍历，要得到第$ \tt{n} $个值，只需要用第$ \tt {n-1} $个值加上第$ \tt{n} $个值即可


**代码展示：**


```java
class Solution {
    public int[] runningSum(int[] nums) {
        for (int i = 1; i < nums.length; i++) {
            nums[i] += nums[i - 1];
        }
        return nums;
    }
}
```