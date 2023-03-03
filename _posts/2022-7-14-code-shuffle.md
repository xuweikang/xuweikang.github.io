---
layout: post
title: 打乱一个数组
tags: 洗牌算法 打乱数组
categories: Practice
---

* TOC 
{:toc}

### 这种解法有问题！！
```js
[12,4,16,3].sort(function() {
    return 5 - Math.random();
});
```
> v8 在处理 sort 方法时，使用了插入排序和快排两种方案。当目标数组长度小于10时，使用插入排序；反之，使用快排。

通俗的说，其实我们使用 array.sort 进行乱序，理想的方案或者说纯乱序的方案是：数组中每两个元素都要进行比较，这个比较有 50% 的交换位置概率。如此一来，总共比较次数一定为 n(n-1)。

而不管用什么排序方法，大多数排序算法的时间复杂度介于 O(n) 到 O(n2) 之间，元素之间的比较次数通常情况下要远小于 n(n-1)/2，


### Fisher–Yates shuffle 洗牌算法
```js
function shuffle(arr) {
  let l = arr.length, count = 1000000, result = new Array(l).fill(0)
  while(count --) {
    while (l) {
      let i = Math.floor(Math.random() * l--);
      [arr[l],arr[i]] = [arr[i],arr[l]]
    }
    l = arr.length
    // 最后一个元素分别出现a,b,c,d,e的次数
    if (arr[arr.length-1] == 'a') result[0] ++
    if (arr[arr.length-1] == 'b') result[1] ++
    if (arr[arr.length-1] == 'c') result[2] ++
    if (arr[arr.length-1] == 'd') result[3] ++
    if (arr[arr.length-1] == 'e') result[4] ++
  }
  console.log(result)
  return arr
}
console.log(shuffle(['a','b','c','d','e']))

// 结果：
// [ 200179, 200092, 200499, 199400, 199830 ]
// [ 'e', 'a', 'd', 'b', 'c' ]
```