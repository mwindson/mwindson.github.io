---
title: rxjs笔记之操作符
tags: [js,rxjs]
---

## 高阶 Observable 操作符

### 合并类

高阶 Observable 的合并类操作符在 Observable 对象的操作符后面加上了 All，包括`concatAll`、`mergeAll`、`zipAll`、`combineAll`（`combineLatestAll`）。

> 高阶 Observable 操作符没有`withLatestFromAll`，因为高阶合并操作符认为内部 Observable 对象是平等的。而`withLatestFrom`会由输入 Observable 对象控制数据输出节奏，Observable 对象并不平等。

#### concatAll

`concatAll`的作用与`concat`类似，不过`concatAll`的上游对象是一个高阶 Observable 对象。它会对其中的内部 Observable 对象进行`concat`操作，即将内部多个 Observable 对象进行`concat`操作并返回结果 Observable 对象。

当内部 Observable 对象完结速度慢或永不完结时，`concatAll`的合并速度慢于上游的 Observable 对象产生内部 Observable 对象的速度，这样就会造成数据积压，进而导致内存泄漏。

#### mergeAll

`mergeAll`作用同`merge`。当上游源产生一个内部 Observable 对象时，`mergeAll`会立刻订阅，并用于合并。

#### zipAll

`zipAll`对内部 Observable 对象的作用和`zip`一样。`zipAll`必须等到上游源 Observable 对象完结后才能开始，因为`zipAll`不确定是否还有新的 Observable 对象出现，因此当上游 Observable 永不完结时，`zipAll`将一直等待。

#### combineAll

`combineAll`的作用同`combineLatest`一样。同样，`combineAll`也必须等到上游源 Observable 对象完结后才能开始。

> `zipAll`和`combineAll`会在内部流产生时，收集内部流，并且直到上游流流结束时，才会订阅收集到的内部流。这样会导致由于上游流流吐出的时间间隔而出现的内部流订阅的时间间隔消失，并且所有内部流的订阅时间点推迟到上游流流结束。这一点与`mergeAll`不同，`meregeAll`会在内部流产生时订阅。

![合并前的流](./合并前的流.png)

<center>合并前的流</center>

![mergeAll后的流](./mergeAll.png)

<center>调用mergeAll后的流</center>

![combineAll后的流](./combineAll.png)

<center>调用combineAll后的流</center>

### 处理类

#### switch

当上游流产生一个内部流时，switch 会立刻订阅新的内部流，已经订阅的内部流会被退订。

switch 产生的流只有当上游流且**当前**订阅的内部流结束时才结束

![switch](./switch.png)

#### exhaust

只有当当前订阅的内部流被耗尽时，才会切换到最新产生的内部流

exhaust 产生的流只有当上游流且**最新**订阅的内部流结束时才结束

![exhaust](./exhaust.png)

> switch 和 exhaust 产生的流只有当上游流且当前订阅的内部流结束时才结束

## 过滤操作符

过滤操作符会根据数据是否满足判定函数来进行对应的操作。

### filter

### first

### last

`last`操作符作用和 `first` 相反，同样也接受三个参数——判断函数、结果处理函数、默认值。

`last`必须等到上游流结束时，才能确定结果，吐出结果数据。

### take、tabkeLast、takeWhile 和 takeUntil

`take`系列操作符用于取多个数据

* `take` —— 相当于取多个数据的`first`
* `takeLast` —— 相当于取多个的`last`
* `takeWhile` —— `takeWhile`会一直吐出上游数据，直到**上游数据不满足判定函数**
* `takeUntil` —— `takeUntil`
