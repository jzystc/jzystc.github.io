---
title: leetcode P169 多数元素
date: 2021-03-23 10:06:00
categories:
    - leetcode
tags:
    - tutorial
---

# 169. 多数元素
给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数大[n/2]的元素。
你可以假设数组是非空的，并且给定的数组总是存在多数元素。

## 解法1 HashMap
用HashMap存储数字的出现次数,空间复杂度为O(n),同时记录当前最大出现次数和对应的数字,遍历一遍数组,时间复杂度O(n).
```java
    public int hashMapCount(int nums[]) {
        // 计数器
        HashMap<Integer, Integer> counter = new HashMap<>();
        // 当前出现最多的数字的出现次数
        int maxAppearance = 1;
        // 当前出现最多的数字
        int majorNum = nums[0];
        // 出现次数大于n/2提前停止
        int earlyStop=nums.length/2;
        for (int i = 0; i < nums.length; i++) {
            if (counter.get(nums[i]) == null) {
                counter.put(nums[i], 1);
            } else {
                int n = counter.get(nums[i]) + 1;
                
                counter.put(nums[i], n);
                if (n > maxAppearance) {
                    maxAppearance = n;
                    majorNum = nums[i];
                    if(n>earlyStop){
                        break;
                    }
                }
            }
        }
        return majorNum;
    }
```

## 解法2 排序法

对数组进行排序, 使用快速排序,时间复杂度O(nlogn).出现次数大于n/2的数一定会出现在排序后的数组索引为n/2的位置.

```java
    Random random = new Random();
    public int partition(int[] nums, int start, int end) {
        //用随机数寻找基数
        int idx = random.nextInt(end - start) + start;
        exchange(nums, start, idx);
        //交换基数和首位元素的位置,基数不参与排序
        int pivot = nums[start];
        int left = start + 1;
        int right = end;
        while (left < right) {
            while (left < right && nums[left] <= pivot) {
                left++;
            }
            if (left != right) {
                exchange(nums, left, right);
                right--;
            }
        }
        if (left == right && nums[right] > pivot) {
            right--;
        }
        if (right != start) {
            exchange(nums, right, start);
        }
        //返回基数的位置
        return right;
    }

    public void exchange(int[] nums, int i, int j) {
        int temp = nums[i];
        nums[i] = nums[j];
        nums[j] = temp;
    }

    public void quickSort(int[] nums, int start, int end) {
        if (start >= end)
            return;
        //排到n/2位置提前停止,返回索引n/2位置的数
        if (end <= nums.length / 2)
            return;
        int middle = partition(nums, start, end);
        quickSort(nums, start, middle - 1);
        quickSort(nums, middle + 1, end);
    }

    public int findMajorityBySorting(int[] nums) {
        quickSort(nums, 0, nums.length - 1);
        return nums[nums.length / 2];
    }

```

## 解法3 摩尔投票法

由于存在出现次数大于n/2的数,不断删除一对不同的数,剩下的数一定是该数

```java
    public int majorityElement(int[] nums) {
        return mooreMajorityVote(nums);
    }

    public int mooreMajorityVote(int nums[]) {
        int count = 0;
        int result = nums[0];
        int n=nums.length/2;
        for (int i = 0; i < nums.length; i++) {
            //当前数和前一数相同则加1
            if (nums[i] == result) {
                count += 1;
                if(count>n)
                {
                    break;
                }
            } else {
            //当前数和记录的数不同则抵消,之前相同的数少1个
                count -= 1;
                if (count == 0) {
                    if (i + 1 < nums.length) {
                        result = nums[i + 1];
                    }
                }

            }
        }
        return result;
    }
```

## 解法4 随机数法

随机抽取一个数, 记录它的出现次数, 由于众数出现次数大于n/2, 期望次数$\lim\sum_{i}^{\infty}\frac{i}{2^i}=2$, 期望时间复杂度$O(n)$. 最坏情况下时间复杂度为 $O(\infty)$.

```java
    public int count(int[] nums, int n) {
        int m = 0;
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] == n) {
                m++;
            }
        }
        return m;
    }

    public int randomPick(int nums[]) {
        Random random = new Random();
        while (true) {
            int candidate = nums[random.nextInt(nums.length)];
            int n = count(nums, candidate);
            if (n > nums.length / 2) {
                return candidate;
            }
        }
    }
```