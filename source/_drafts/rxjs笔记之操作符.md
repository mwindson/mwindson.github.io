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

`filter`即只吐出上游流符合判定函数的数据

### first

`first`接受三个参数——判定函数、结果处理函数和默认值（默认为 null）。当上游数据符合判定函数时，`first`吐出经结果处理函数处理之后的数据并结束，无需等上游流结束，若等到上游流结束时，仍没有符合条件的数据，那么会吐出经过处理的默认值并结束。

### last

`last`操作符作用和 `first` 相反，同样也接受三个参数——判断函数、结果处理函数、默认值。

`last`必须等到上游流结束时，才能确定结果，吐出结果数据。

### take、tabkeLast、takeWhile 和 takeUntil

`take`系列操作符用于取多个数据，`take`和`takeLast`会在结束时一次性返回所有符合的数据，而`takeWhile`和`takeUntil`会随上游流的节奏吐出符合的数据。

- `take` —— 相当于取多个数据的`first`
- `takeLast` —— 相当于取多个的`last`
- `takeWhile` —— `takeWhile`会一直吐出上游数据，直到**上游数据不满足判定函数**
- `takeUntil` —— `takeUntil`会一直吐出上游数据，直到**参数 notifier 流**吐出数据时，停止吐出数据。

![takeUntil](./takeUntil.png)

<center>takeUntil弹珠图</center>

### skip、skipWhile 和 skipUntil

`skip`表示跳过若干数据后，吐出和上游流一样的数据

`skipWhile`与`takeWhile`、`skipUntil`与`skipUntil`一一对应

## 控制类操作符

控制操作符主要是为了解决上下游速度不一致导致的`BackPressure`问题，对上游数据进行一定的舍弃，来实现**有损**的回压控制。

每一类控制操作符默认用另一个流来控制，而简单的以时间做控制的操作符加上了 Time 后缀。

### throttle、throtlleTime

### debounce、debounceTime

### audit、auditTime

### sample、sampleTime

## 转化类操作符

转化类操作符会对数据进行处理，但转化类操作符不会对数据进行过滤，会对数据进行转化和组合。

### map、mapTo、pluck

`map`是最简单、最常用的转化操作符。`map`会对上游数据进行函数处理，并把返回结果传给下游流。

`mapTo`会把所有数据映射成同一个数据，并传给下游。

`pluck`类似根据传入参数 key 来取上游数据中对应 key 的值，并传给下游。`plunk`只能去一个值，而不能嵌套取值。当对应 key 不存在时，`pluck`可以把传入的第二个参数作为默认值（默认为 undefined）传给下游。

### 缓存数组类和缓存 Observable 类

和控制类操作符实现的有损回压控制不同，转化类操作符不会舍弃上游的数据。它会通过将多个数据放在一个数组和 Observable 对象并且一次性传给下游，从而实现**无损**的回压控制。因此可以把这类操作符分为缓存数组类和缓存 Observable 类，两者的区别只在于传给下游流的数据形式。

- 缓存数组类 bufferTime、bufferCount、bufferWhen、bufferToggle、buffer
- 缓存 Observable 类 windowTime、windowCount、windowWhen、windowToggle、window

#### bufferTime 和 windowTime

`bufferTime`和`windowTime`接受三个参数。第一个参数是区间时间长度，操作符会将下游划分为该时间长度的时间区间，并对这段时间内的数据进行分组，以数组形式或内部流的形式传给下游。

```javascript
let s1$ = Rx.Observable.timer(0, 200)
  .take(3)
  .concat(Rx.Observable.timer(500, 300).take(4))
  .concat(Rx.Observable.interval(500).take(4))

let s2$ = s1$.bufferTime(1200)
let s3$ = s1$.windowTime(1200)
```

![bufferTime和windowTime](./bufferTime和windowTime.png)

这两个操作符的第二个参数表示每个时间区间开始的时间间隔。因此当第二个参数小于第一个参数时，时间区间会重叠，可能会导致重复的数据，而第二个参数大于第二个参数时，时间区间产生间距，可能会导致数据丢失。

第三个参数表示该区间内最多数据个数。

因此，`windowTime`产生的内部流会持续一段时间，并在**持续时间等于时间区间或达到区间数据上限**时结束，而`bufferTime`会在**区间时间结束或达到区间数据上限**时将数据数组传给下游。

#### bufferCount 和 windowCount

`windowCount`、`bufferCount`和`windowTime`、`bufferTime`类似，区间长度由数据个数决定，根据接受的数据个数来决定数组传给下游或内部流结束的时间。第二个参数表示每隔多少个数据产生区间，因此也会产生丢弃数据和数据重复的现象。

#### bufferWhen 和 windowWhen

同样的，`bufferWhen`和`windowWhen`会根据传入函数返回的 Observable 对象来划分区间，当返回的流**产生数据或完结**时，操作符会将数据数组传给下游或者结束内部流，产生新的内部流。

> `bufferWhen`和`windowWhen`没有其他参数。

#### bufferToggle 和 windowToggle

`windowToggle`、`bufferToggle`
