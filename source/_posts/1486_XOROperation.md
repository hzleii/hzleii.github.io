---
title: "1486.数组异或操作"
description: "1486.数组异或操作"
date: 2020-04-19
topic: leetcode
mathjax: true
author: hzlei
# banner: /assets/vuejs/vuejs.webp
# cover: /assets/vuejs/vuejs.webp
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 1486.数组异或操作 # 必选
  # caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  # color: white # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...

---



## 题目描述



**给定两个整数，$ \tt{n} $和$ \tt{start} $。**

数组$ \tt{nums} $定义为：$ \tt{nums[i] = start + 2 * i} $（下标从$ \tt{0} $开始）且$ \tt{n == nums.length} $。

请返回$ \tt{nums} $中所有元素按位异或（**XOR**）后得到的结果。


{% box child:tabs %}
{% tabs %}

<!-- tab 示例 1 -->
```
输入：n = 5, start = 0
输出：8
解释：数组 nums 为 [0, 2, 4, 6, 8], 其中 (0 ^ 2 ^ 4 ^ 6 ^ 8) = 8。 "^" 为按位或 XOR 运算符
```

<!-- tab 示例 2 -->
```
输入：n = 4, start = 3
输出：8
解释：数组 nums 为 [3, 5, 7, 9]，其中 (3 ^ 5 ^ 7 ^ 9) = 8
```

<!-- tab 示例 3 -->
```
输入：n = 1, start = 7
输出：7
```

<!-- tab 示例 4 -->
```
输入：n = 10, start = 5
输出：2
```

{% endtabs %}
{% endbox %}

**提示：**

- $ \tt{1 <= n <= 1000} $
- $ \tt{0 <= start <= 1000} $
- $ \tt{n == nums.length} $


## 题解


## 前言

统计$ \tt{[2, n]} $中质数的数量是一个很常见的题目，也有很多巧妙高效的做法，接下来的部分只会讲述一些常见的做法，更多的拓展内容读者可以自行搜索补充，也欢迎在评论区与大家分享交流。



## 方法一：模拟

**思路：**

按照题意模拟即可：

- 初始化$ \tt{ans = 0} $
- 遍历区间$ \tt{[0, n - 1]} $中的每一个整数$ \tt{i} $，令$ \tt{ans} $与每一个$ \tt{start + 2 \times i} $做异或运算
- 最终返回$ \tt{ans} $，即我们需要的答案


**代码展示：**


{% box child:tabs %}
{% tabs %}

<!-- tab Java -->
```java
class Solution {
    public int xorOperation(int n, int start) {
        int ans = 0;
        for (int i = 0; i < n; ++i) {
            ans ^= (start + i * 2);
        }
        return ans;
    }
}
```

<!-- tab Python -->
```python
class Solution:
    def xorOperation(self, n: int, start: int) -> int:
        ans = 0
        for i in range(n):
            ans ^= (start + i * 2)
        return ans
```

<!-- tab C -->
```c
bool isPrime(int x) {
    for (int i = 2; i * i <= x; ++i) {
        if (x % i == 0) {
            return false;
        }
    }
    return true;
}

int countPrimes(int n) {
    int ans = 0;
    for (int i = 2; i < n; ++i) {
        ans += isPrime(i);
    }
    return ans;
}
```

<!-- tab JavaScript -->
```javascript
var xorOperation = function(n, start) {
    let ans = 0;
    for (let i = 0; i < n; ++i) {
        ans ^= (start + i * 2);
    }
    return ans;
};
```

{% endtabs %}
{% endbox %}


**复杂度分析：**

- 时间复杂度：$ \rm{O(n)} $ 。这里用一重循环对$ \tt{n} $个数字进行异或。
- 空间复杂度：$ \rm{O(1)} $。这里只是用了常量级别的辅助空间。



## 方法二：数学

记$ \tt{\oplus} $为异或操作，异或操作满足以下性质：

1. $ \tt{x \oplus x = 0} $
2. $ \tt{x \oplus y = y \oplus x} $（交换律）
3. $ \tt{(x \oplus y) \oplus z = x \oplus (y \oplus z)} $（结合律）
4. $ \tt{x \oplus y \oplus y = x} $（自反性）
5. $ \tt{\forall i \in Z } $，有$ \tt{4i \oplus (4i + 1) \oplus (4i + 2) \oplus (4i + 3) = 0} $



在本题中，我们要计算

$$ \tt{start \oplus (start + 2i) \oplus (start + 4i) \oplus \cdots \oplus (start + 2(n - 1)) } $$

观察公式可以知道，这些数的奇偶性相同，因此它们的二进制表示中的最低位均为$ \tt{1} $，或者均为$ \tt{0} $。
于是我们可以把参与运算的数的二进制位的最低位提取出来单独处理，当且仅当$ \tt{start} $为奇数，且$ \tt{n} $也为奇数时，结果的二进制位的最低位才为$ \tt{1} $。

此时我们可以将公式转化为：

$$ \tt{(s \oplus (s + 1) \oplus (s + 2) \oplus \cdots \oplus (s + n - 1)) \times + \ e} $$

其中$ \tt{s = \lfloor \frac{start}{2} \rfloor} $，$ \tt{e} $表示运算结果的最低位。即我们单独处理最低位，而舍去最低位后的数列恰成为一串连续的整数。

这样我们可以描述一个函数$ \tt{sumXor(x)} $，表示$ \tt{0 \oplus 1 \oplus 2 \oplus \cdots \oplus x} $。
利用异或运算的{% mark 性质5 color:error %}，我们可以将计算该函数的复杂度降低到$ \rm{O(1)} $，因为以$ \tt{4i} $为开头的连续四个整数异或的结果为$ \tt{0} $，所以$ \tt{sumXor(x)} $可以被表示为：

$$
\tt{sumXor(x) = \begin{cases} \tt x,  && \tt x=4k, k \in Z \\\\ \tt (x-1) \oplus x, && \tt x =4k+1, k \in Z \\\\ \tt (x-2) \oplus (x-1) \oplus x, &&  \tt x=4k+2,k \in Z \\\\ \tt (x-3) \oplus (x-2) \oplus (x-1) \oplus x, && \tt x=4k+3,k \in Z \end{cases} }
$$


我么可以进一步化简改式：


$$
\tt{sumXor(x) = \begin{cases} \tt x,  && \tt x=4k, k \in Z \\\\ \tt 1, && \tt x =4k+1, k \in Z \\\\ \tt x+1, &&  \tt x=4k+2,k \in Z \\\\ \tt 0, && \tt x=4k+3,k \in Z \end{cases} }
$$



这样最后的结果即可表示为$ \tt{(sumXor(s-1) \oplus sumXor(s+n-1)) \times2 + e} $


**代码展示：**

{% box child:tabs %}
{% tabs %}

<!-- tab Java -->
```java
class Solution {
    public int xorOperation(int n, int start) {
        int s = start >> 1, e = n & start & 1;
        int ret = sumXor(s - 1) ^ sumXor(s + n - 1);
        return ret << 1 | e;
    }

    public int sumXor(int x) {
        if (x % 4 == 0) {
            return x;
        }
        if (x % 4 == 1) {
            return 1;
        }
        if (x % 4 == 2) {
            return x + 1;
        }
        return 0;
    }
}
```

<!-- tab C -->
```c
int sumXor(int x) {
    if (x % 4 == 0) {
        return x;
    }
    if (x % 4 == 1) {
        return 1;
    }
    if (x % 4 == 2) {
        return x + 1;
    }
    return 0;
}

int xorOperation(int n, int start) {
    int s = start >> 1, e = n & start & 1;
    int ret = sumXor(s - 1) ^ sumXor(s + n - 1);
    return ret << 1 | e;
}
```

<!-- tab JavaScript -->
```javascript
var xorOperation = function(n, start) {
    let s = start >> 1, e = n & start & 1;
    let ret = sumXor(s - 1) ^ sumXor(s + n - 1);
    return ret << 1 | e;
};

const sumXor = (x) => {
    if (x % 4 === 0) {
        return x;
    }
    if (x % 4 === 1) {
        return 1;
    }
    if (x % 4 === 2) {
        return x + 1;
    }
    return 0;
}
```

{% endtabs %}
{% endbox %}


**复杂度分析：**

- 时间复杂度：$ \rm{O(1)} $。我们只需要常数的时间计算出结果。
- 空间复杂度：$ \rm{O(1)} $。我们只需要常数的空间保存若干变量。