---
title: No189_RotateArray（环状替换 & 数组翻转）
katex: true
date: 2022-01-12 16:06:00
categories: 
- leetcode
tags:
- 环状替换
- 数组翻转
- No189
---

>转载于leetcode，此处作为笔记用，不妥删除
>原帖地址：https://leetcode-cn.com/problems/rotate-array/solution/xuan-zhuan-shu-zu-by-leetcode-solution-nipk/

#### 环状替换

方法一中使用额外数组的原因在于如果我们直接将每个数字放至它最后的位置，这样被放置位置的元素会被覆盖从而丢失。因此，从另一个角度，我们可以将被替换的元素保存在变量 $temp$ 中，从而避免了额外数组的开销。

我们从位置 *0* 开始，最初令 $temp=nums[0]$ 。根据规则，位置 *0* 的元素会放至 $(0+k)\bmod$  的位置，令  $ x=(0+k)modn $，此时交换 $ temp $  和 $ nums[x]$，完成位置 *x* 的更新。然后，我们考察位置 *x*，并交换  $temp $  和 $nums[(x+k)modn] $ ，从而完成下一个位置的更新。不断进行上述过程，直至回到初始位置 *0*。

容易发现，当回到初始位置 *0* 时，有些数字可能还没有遍历到，此时我们应该从下一个数字开始重复的过程，可是这个时候怎么才算遍历结束呢？我们不妨先考虑这样一个问题：从 *0* 开始不断遍历，最终回到起点 *0* 的过程中，我们遍历了多少个元素？

由于最终回到了起点，故该过程恰好走了整数数量的圈，不妨设为 *a* 圈；再设该过程总共遍历了 *b* 个元素。因此，我们有 *an=bk*，即 *an* 一定为 *n,k* 的公倍数。又因为我们在第一次回到起点时就结束，因此 *a* 要尽可能小，故 *an* 就是 *n,k* 的最小公倍数 $lcm(n,k) $ ，因此 *b* 就为 $lcm(n,k)/k $ 。

这说明单次遍历会访问到 $lcm(n,k)/k $  个元素。为了访问到所有的元素，我们需要进行遍历的次数为

$$
\frac{n}{\text{lcm}(n,k)/k}=\frac{nk}{\text{lcm}(n,k)}=\text{gcd}(n,k)
$$


其中 $gcd $  指的是最大公约数。

我们用下面的例子更具体地说明这个过程：
```
nums = [1, 2, 3, 4, 5, 6]
k = 2
```

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/f0493a97cdb7bc46b37306ca14e555451496f9f9c21effcad8517a81a26f30d6-image.png)

如果读者对上面的数学推导的理解有一定困难，也可以使用另外一种方式完成代码：使用单独的变量 $count $  跟踪当前已经访问的元素数量，当 $count=n $  时，结束遍历过程。


```Java [sol2-Java]
class Solution {
    public void rotate(int[] nums, int k) {
        int n = nums.length;
        k = k % n;
        int count = gcd(k, n);
        for (int start = 0; start < count; ++start) {
            int current = start;
            int prev = nums[start];
            do {
                int next = (current + k) % n;
                int temp = nums[next];
                nums[next] = prev;
                prev = temp;
                current = next;
            } while (start != current);
        }
    }

    public int gcd(int x, int y) {
        return y > 0 ? gcd(y, x % y) : x;
    }
}
```


**复杂度分析**

* 时间复杂度：*O(n)*，其中 *n* 为数组的长度。每个元素只会被遍历一次。

* 空间复杂度：*O(1)*。我们只需常数空间存放若干变量。

#### 数组翻转

该方法基于如下的事实：当我们将数组的元素向右移动 *k* 次后，尾部 $k mod n $  个元素会移动至数组头部，其余元素向后移动 $k mod n $  个位置。

该方法为数组的翻转：我们可以先将所有元素翻转，这样尾部的 $k mod n  $  个元素就被移至数组头部，然后我们再翻转 $[0,kmodn−1] $  区间的元素和 $[kmodn,n−1] $  区间的元素即能得到最后的答案。

我们以 *n=7*，*k=3* 为例进行如下展示：

| 操作                                | 结果            |
| ----------------------------------- | --------------- |
| 原始数组                            | $1~2~3~4~5~6~7$ |
| 翻转所有元素                        | $7~6~5~4~3~2~1$ |
| 翻转 $[0,kmodn−1] $  区间的元素 | $5~6~7~4~3~2~1$ |
| 翻转 $[kmodn,n−1] $  区间的元素 | $5~6~7~1~2~3~4$ |


```Java [sol3-Java]
class Solution {
    public void rotate(int[] nums, int k) {
        k %= nums.length;
        reverse(nums, 0, nums.length - 1);
        reverse(nums, 0, k - 1);
        reverse(nums, k, nums.length - 1);
    }

    public void reverse(int[] nums, int start, int end) {
        while (start < end) {
            int temp = nums[start];
            nums[start] = nums[end];
            nums[end] = temp;
            start += 1;
            end -= 1;
        }
    }
}
```

**复杂度分析**

* 时间复杂度：*O(n)*，其中 *n* 为数组的长度。每个元素被翻转两次，一共 *n* 个元素，因此总时间复杂度为 *O(2n)=O(n)*。

* 空间复杂度：*O(1)*。