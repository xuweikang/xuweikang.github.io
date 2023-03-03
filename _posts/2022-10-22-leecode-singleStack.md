---
layout: post
title: 单调栈的应用
tags: 单调栈 stack
categories: leecode
---

* TOC 
{:toc}


1. 给你两个 没有重复元素 的数组 nums1 和 nums2 ，下标从 0 开始计数，其中nums1 是 nums2 的子集。
>输入：nums1 = [4, 1, 2], nums2 = [ 1, 3, 4, 2]
输出：[-1, 3, -1]
解释：找出nums1中的每项值在nums2中的下一个更大元素值，如果找不到返回-1

```js
function nextGreaterElement(num1, num2) {
    let stack = [] // 模拟一个栈
    let ans = new Array(num2.length)  // num2的每一项后面第一个大的值的项
    for(var i = num2.length - 1; i >=0; i --){ // 从num2的最后一项开始遍历
        while(stack.length && stack[stack.length -1] <= num2[i]) {
            stack.pop()
        }
        if (stack.length == 0) {
            ans[i] = -1
        } else {
            ans[i] =  stack[stack.length -1]
        }
        stack.push(num2[i])
    }
    return num1.map(item => ans[num2.indexOf(item)])
    
```

2. 给定一个循环数组 nums （ nums[nums.length - 1] 的下一个元素是 nums[0] ），返回 nums 中每个元素的 下一个更大元素 。
数字 x 的 下一个更大的元素 是按数组遍历顺序，这个数字之后的第一个比它更大的数，这意味着你应该循环地搜索它的下一个更大的数。如果不存在，则输出 -1 。
>输入：nums = [1,2,3,4,3]
输出：[2,3,4,-1,4]
解释：找出nums中的每项值的下一个更大元素值，如果找不到返回-1。注意数组是循环数组

```js
function newGreaterElement(nums){
    let stack = []
    let ret = [], len = nums.length
    nums = nums.concat(nums)
    for (let i = nums.length - 1; i >= 0; i --) {
        while(stack.length > 0 && stack[stack.length - 1] <= nums[i]) {
            stack.pop()
        }
        ret[i] = stack.length == 0 ? -1 : stack[stack.length - 1]
        stack.push(nums[i])
        
    }
    ret.splice(len)
    return ret
}
```