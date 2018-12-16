---
layout: post
title: 算法入门 - 三种非比较排序
tags: Algorithm
cover: "https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/algorithm.png"
---

<!--more-->

算法中，排序是永恒的话题。排序分为比较排序和非比较排序，比较排序中， 我们比较熟悉的就是`冒泡排序`，它的时间复杂度是O( n^2 )。而非比较中有三个经典的算法：`计数排序`、`基数排序`、`桶排序`。代码可以[在这里](https://github.com/liaoyuanng/Algorithm)下载。

## 计数排序

复杂度：O ( k + n ), k 是被排序的元素的最大值。

### 原理

当我们有一组待排序的数据(下面统称 `A`)，假定为{1, 3, 9, 4, 7, 5}; 我们要对它们进行排序的话，就需要比较他们和相邻元素的大小，那么，这样就不是非比较排序了。除了比较外，我们是否可以通过一种手段，不需要比较它们之间的大小呢？答案是肯定的。那么，我们如何让他们有序呢，我们考虑下，我们常使用的数据结构中，那种结构是有序的呢？对的，数组。如果我们把这些数字当做数组的下标，然后通过下标找到对应的元素，并做个标记，然后我们再遍历这个临时数组(下面统称 `B`)，这样一来，我们是否就可以达到排序的目的了？

所以，我们首先需要知道 `A `中元素，最大的数字，作为另一个`B`的大小。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15227259340718.jpg)

并将`B`初始化为`0`。由于数组的下标是从`0`开始的，所以我们需要将最大值 + 1.

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15227261493861.jpg)

那么如何标记呢？我们把`A`中出现的元素，和`B`中的下标对应起来，并把下标对应`B`的元素 + 1.

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15227265309132.jpg)

这个时候，我们会发现一个有趣的事，就是，`B`中所有大小为`1`的元素对应的下标，刚好是`A`中出现的元素(废话~~),并且是有序的。Emmm...，因缺思厅。

这个时候，我们只需要遍历 `B`，找出不为`0`的元素对应的下标，就可以啦。

具体到代码就是：

```C

void CountingSort(int a[], int n) {

    int max = Max(a, n) + 1;
    // alloc 'max' length array.
    int *counting = calloc(max, sizeof(int));

    // 把每个元素的大小，映射在数组的下标上
    for (int i = 0; i < n; ++i) {
        counting[a[i]]++;
    }
    // 计算小于等于 c[i] 的元素个数。这样，就可以得到每个元素应该在数组(排完序后的)位置。
    // 例如这个例子。上面 counting 的 元素为：0 1 0 1 1 1 0 1 0 1.
    // 经过下面的 for 循环后，counting 的元素为：0 1 1 2 3 4 4 5 5 6.
    // 代表的意思就是，小于等于第 0 个的，有0个；小于等于第 1 个的，有 1 个；
    // 小于等于第 2 个的，有 1 个，小于等于第 3 个的，有 2 个...以此类推
    // 例如，小于等于 4 的，有 3 个，那么 4 这个元素，排完序后就应该在下标为 3 - 1 的位置(从0开始)
    for (int i = 1; i < max; ++i) {
        counting[i] += counting[i - 1];
    }

    int *temp = calloc(n, sizeof(int));

    // 倒序，为了排序的稳定性
    // 如上所述，小于等于这个元素的个数，就是该元素排序时应该在的位置。
    // 需要注意的是，数组是从0开始，所以先要减1。因为上面计算的是小于等于的，所以会包括自身。
    for (int i = n - 1; i >= 0; --i) {
        temp[--counting[a[i]]] = a[i];
    }
    for (int i = 0; i < n; ++i) {
        a[i] = temp[i];
    }
    free(temp);
    free(counting);
}

```

## 桶排序

`桶排序`其实和上面的计数排序类似，一个简单的桶，它的大小应被固定，所以，它只能排序比它最大值小的元素。比如，桶的大小是 10 ，那么你只能使用它来做 0 - 9 的排序。这样的`桶排序`，应用很有限。所以，还有另外一个比较复杂的桶，它允许每个桶里面放多个元素，然后，再针对每个桶做排序。时间复杂度为`O(n)`。

一个简单的桶的实现：

``` C

void bucketSort(int a[], int size) {
    int bucket[SIZE] = {0}; // 初始化桶
    int idx = 0;

    // 每个元素放入属于自己的桶中
    for (int i = 0; i < size; ++i) {
        bucket[a[i]]++;
    }

    printf("\nbucket array: ");
    printArray(bucket, SIZE);

    // 找到不为0的下标
    for (int i = 0; i < SIZE; i++) {
        // 处理多次出现的元素
        // 最理想的情况下，这里只会循环一次，这种情况下的时间复杂度为O(N)
        while (bucket[i]-- > 0) {
            a[idx] = i;
            idx++;
        }
    }
}

```

## 基数排序

基数排序，以固定的桶的数量，按照十分位、百分位、千分位...的顺序，以此做桶排序，因为每个分位的值的范围是 0 - 9,所以，我们只需要 10 个桶就可以。大大的减少了上面的`简单桶排序`中的空间浪费。当然，你也可以自定义基数，而并非一定以`10`为基数。

基数排序的时间复杂度为`O(k * n)`，k 是元素的位数，n 是 排序的元素个数。

核心代码如下：

```C

void RadixSort(int a[], int n) {
    int max = Max(a, n);
    // 基数，从低位开始，保证算法稳定性。
    int radix = 1;
    // 用于排序的临时数组
    int *sortedArray = calloc(n, sizeof(int));

    while (max / radix > 0) {
        int bucket[SIZE] =  { 0 };
        // 这里和桶排序一个原理
        for (int i = 0; i < n; ++i) {
            bucket[(a[i] / radix) % SIZE] ++;
        }

        // 这里和计数排序一个原理
        for (int i = 1; i < SIZE; ++i) {
            bucket[i] += bucket[i - 1];
        }

        // 倒序，保证算法的稳定性
        for (int i = n - 1; i >= 0 ; --i) {
            sortedArray[--bucket[a[i] / radix % SIZE]] = a[i];
        }

        for (int i = 0; i < n; ++i) {
            a[i] = sortedArray[i];
        }

        radix *= 10;
    }

    free(sortedArray);
}

```

## 算法的稳定性

算法的稳定性，并不是值算法的性能，而是，遇到两个相同的元素，这两个元素的前后位置是否和原序列中的相等，即前后位置没有发生改变。

从上面的例子中，我们两次提到了`稳定性`，第一次，是`基数排序低位保证稳定性`，虽然高位同样可以保证稳定性，但是需要额外的空间。为什么低位开始能保证稳定行呢，假如有两个元素`166`和`66`，在十分位和百分位我们都比较不出来大小，那么他们的位置就不会发生变化。我们称这就是稳定的。

![From Wikipedia](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15231682118788.jpg)

## 总结

这三种非比较排序算法，有很多的相似之处，都有用到`桶`的概念，但是从实现上，它们还是有很大的差别的。你可以[看这里](https://stackoverflow.com/questions/14368392/radix-sort-vs-counting-sort-vs-bucket-sort-whats-the-difference)具体了解它们的不同。

[代码下载](https://github.com/liaoyuanng/Algorithm)

# Reference

* [数据结构与算法分析](https://book.douban.com/subject/1139426/)
* [Know Thy Complexities!](http://bigocheatsheet.com/)
* [Radix sort vs Counting sort vs Bucket sort. What's the difference?](https://stackoverflow.com/questions/14368392/radix-sort-vs-counting-sort-vs-bucket-sort-whats-the-difference)



