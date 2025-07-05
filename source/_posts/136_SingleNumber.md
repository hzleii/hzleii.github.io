---
title: "136.只出现一次的数字"
description: "给定一个非空整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次，找出那个只出现一次的元素"
date: 2020-04-03
topic: leetcode
mathjax: true
author: hzlei
# banner: /assets/vuejs/vuejs.webp
# cover: /assets/vuejs/vuejs.webp
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 136.只出现一次的数字 # 必选
  # caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  # color: white # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...

---


## 题目描述

**给定一个非空整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次，找出那个只出现一次的元素**


{% box child:tabs %}
{% tabs %}

<!-- tab 示例 1 -->
```
输入：[2, 2, 1]
输出：1
```

<!-- tab 示例 2 -->
```
输入：[4, 1, 2, 1, 2]
输出：4
```

<!-- tab 示例 3 -->
```
输入：n = 1
输出：0
```

{% endtabs %}
{% endbox %}

**说明：**

**你的算法应该具有线性时间复杂度。你可以不使用额外空间来实现吗？**



## 题解

## 方法一：位运算

如果没有时间复杂度和空间复杂度的限制，这道题有很多解法，可能的解法有以下几种：

- 使用集合存储数字。遍历数组中的每个数字，如果集合中没有该数字，则将该数字加入集合，如果集合中已经有该数字，则将该数字从集合中删除，最后剩下的数字就是只出现一次的数字。
- 使用哈希表存储每个数字和该数字出现的次数。遍历数组即可得到每个数字出现的次数，并更新哈希表，最后遍历哈希表，得到只出现一次的数字。
- 使用集合存储数组中出现的所有数字，并计算数组中的元素之和。由于集合保证元素不重复，因此计算集合中的所有元素之和的两倍，即为每个元素出现两次的情况下的元素之和。由于数组中只有一个元素出现一次，其余元素都出现两次，因此用集合中的元素之和的两倍减去数组中的元素之和，剩下的就是数组中只出现一次的数字。

上述三种解法都需要额外使用$ \rm O(n) $的空间。其中$ \tt{n} $是数组长度。如果要求使用线性时间复杂度和常数空间复杂度，上述三种解法显然都不满足要求。那么，如何才能做到线性时间复杂度和常数空间复杂度呢？

答案是使用**位运算**。对于这道题，可使用异或运算$ \tt{\oplus} $。异或运算有一下三个性质。

- 任何数和$ \tt{0} $做异或运算，结果仍然是原来的数，即$ \tt{a \oplus 0 = a} $。
- 任何数和自身做异或运算，结果是$ \tt{0} $，即$ \tt{a \oplus a = 0} $。
- 异或运算满足交换律和结合律，即$ \tt{a \oplus b \oplus a=b \oplus a \oplus a=b \oplus (a \oplus a)=b \oplus0=b} $。

假设数组有$ \tt{2m + 1} $个数，其中有$ \tt{m} $个数各出现两次，一个数出现一次。令$ \tt{a_{1}、a{2}、\ldots、a{m}} $为出现两次的 $ \tt{m} $个数，$ \tt{a_{m+1}} $为出现一次的数。根据{% mark 性质3 color:error %}，数组中的全部元素的异或运算结果总是可以写成如下形式：

$$ \tt{(a_{1} \oplus a_{1}) \oplus (a_{2} \oplus a_{2}) \oplus \cdots \oplus (a_{m} \oplus a_{m}) \oplus a_{m+1}} $$

根据{% mark 性质2 color:error %}和{% mark 性质1 color:error %}，上式可简化和计算**得到如下结果**：

$$ \tt{0 \oplus 0 \oplus \cdots \oplus 0 \oplus a_{m+1} = a_{m+1}} $$

**因此，数组中的全部元素的异或运算结果即为数组中只出现一次的数字。**

**代码展示：**

{% box child:tabs %}
{% tabs %}

<!-- tab Java -->
```java
class Solution {
    public int singleNumber(int[] nums) {
        int single = 0;
        for (int num : nums) {
            single ^= num;
        }
        return single;
    }
}
```

<!-- tab Python -->
```python
class Solution:
    def singleNumber(self, nums: List[int]) -> int:
        return reduce(lambda x, y: x ^ y, nums)
```

<!-- tab C++ -->
```c++
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        int ret = 0;
        for (auto e: nums) ret ^= e;
        return ret;
    }
};
```

{% endtabs %}
{% endbox %}


**复杂度分析：**

- 时间复杂度：$ \rm{O(n)} $ ，其中$ \tt{m} $是数组的长度。只需要对数组遍历一次。
- 空间复杂度：$ \rm{O(1)} $。