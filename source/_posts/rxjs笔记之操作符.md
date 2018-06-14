---
title: rxjs笔记之操作符
date: 2018-06-14 19:29:30
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

## 控制操作符

控制操作符主要是为了解决上下游速度不一致导致的`BackPressure`问题，对上游数据进行一定的舍弃，来实现**有损**的回压控制。

每一类控制操作符默认用另一个流来控制，而简单的以时间做控制的操作符加上了 Time 后缀。

### throtlleTime、debounceTime、auditTime

`throttleTime`限制在 duration 的时间内上游传递给下游数据个数。`throttleTime`在获得上游数据后，会先把数据传递给下游，再开始计时。新的数据不会导致时间窗口重新更新计时。

`debounceTime`限制传递给下游的两个数据之间时间不能小于给定时间。`debounceTime`在获得上游数据后，会先开始计时，如果计时内未出现新的数据，才会把缓存的窗口内第一个数据传递给下游；如果出现新的数据，那么`debounceTime`就会开启新的计时，并更新缓存的数据。

`auditTime`和`throttleTime`的原理相似。在一定时间内，`throttle`会把第一个数据传给下游，而`audit`会把最后一个传给下游。

旧时间窗口结束后，只有当上游吐出数据时，才会开始三者的新时间窗口的计时。

```javascript
let s1$ = Rx.Observable.interval(400)
  .take(2)
  .mapTo('A')
  .concat(
    Rx.Observable.interval(800)
      .take(3)
      .mapTo('B')
  )
  .concat(
    Rx.Observable.interval(300)
      .take(4)
      .mapTo('C')
  )

s1$.throttleTime(500)
s1$.debounceTime(500)
s1$.auditTime(500)
```

![throttleTime、debounceTime和auditTime](./throttleTime、debounceTime和auditTime.png)

### throttle、debounce、audit

`throttle`、`debounce`、`audit`是带 Time 后缀的通用版，操作符接受`seletor`函数参数，该函数可根据数据产生一个 Observable 对象来决定节流时间。

### sample、sampleTime

`sample`和`sampleTime`与之前的操作符有所不同。这两个操作符不管上游如何产生数据，只根据自己设定的规则参数来将符合条件的数据传递给下游。

`sampleTime`会根据设定的时间参数，从数据流**起始**处将时间划分为等长的区间。`sampleTime`会在这个时间块结束时判断这个时间块内是否有数据，如果有数据，那么会把s上游最后一个数据传给下游流，如果时间块内没有数据，那么操作符不会给下游传递数据。

`sample`接受一个notifier流，当notifier流产生数据时，`sample`就会把上游最后一个产生的数据传给下游。

![sampleTime](./sampleTime.png)

### distinct

`distinct`只会把未出现过的数据传递给下游，上游相同的数据只有第一次能传递给下游。`distinct`可以接受一个函数`keySelector`来控制对象类型数据中哪个属性用于比较。

`distinct`第二个参数是一个 Observable 对象，每当 Observable 对象产生数据时，`distinct`会清空内部用于保存只出现数据的集合，这样可以避免大量数据时的内存泄漏，也可以控制在一段时间范围内，下游数据唯一。

### distinctUntilChanged

`distinctUntilChanged`也会舍弃重复的数据，不过这个操作符只会比较当前数据和上一个数据。

源数据：

```javascript
Rx.Observable.of(0, 1, 1, 2, 0, 0, 1, 3, 3, 4).delay(500)
```

![distinct和distinctUntilChanged结果比较](./distinct和distinctUntilChanged.png)

### ignoreElements、elementAt、single

`ignoreElements`会忽略所有数据，只关注 complete 和 error 事件。

`elementAt`只返回指定下标的数据。

`single`会检查上游是否只有一个满足对应条件的数据。

## 转化操作符

转化操作符会对数据进行处理，但转化操作符不会对数据进行过滤，会对数据进行转化和组合。

### map、mapTo、pluck

`map`是最简单、最常用的转化操作符。`map`会对上游数据进行函数处理，并把返回结果传给下游流。

`mapTo`会把所有数据映射成同一个数据，并传给下游。

`pluck`类似根据传入参数 key 来取上游数据中对应 key 的值，并传给下游。`plunk`只能去一个值，而不能嵌套取值。当对应 key 不存在时，`pluck`可以把传入的第二个参数作为默认值（默认为 undefined）传给下游。

### 缓存数组类和缓存 Observable 类

和控制操作符实现的有损回压控制不同，转化操作符不会舍弃上游的数据。它会通过将多个数据放在一个数组和 Observable 对象并且一次性传给下游，从而实现**无损**的回压控制。因此可以把这类操作符分为缓存数组类和缓存 Observable 类，两者的区别只在于传给下游流的数据形式。

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

`windowToggle`、`bufferToggle`通过一个 Observable 对象和一个函数`closingSelector`来控制缓冲窗口的开关。

Observable 对象参数控制缓冲窗口的开始，而`closingSelector`函数接受 Observable 对象产生的数据，产生新的流来控制缓冲窗口的结束。

```javascript
let s1$ = Rx.Observable.interval(200)

let opening$ = Rx.Observable.interval(400)
const func = value => Rx.Observable.interval(value > 3 ? 300 : 600)

const s2$ = s1$.windowToggle(opening$, func)
```

![windowToggle和bufferToggle](./windowToggle和bufferToggle.png)

#### buffer 和 window

`buffer`和`window`只接受一个`notifer$`数据流，当`notifer$`产生数据时，结束前一个缓冲窗口，并且开始下一个缓冲窗口。

> `notifer$`完结时，会导致操作符产生的内部流和外部的高阶流都完结。

### 高阶 map 操作符

`map`操作可以通过将每个数据映射到一个内部流，从而产生一个高阶流。高阶`map`操作符，不仅仅能产生一个高阶流，而且会对内部流进行相应的组合操作，如`concatMap`等价于`map`+`concatAll`。

## 分组操作符

分组操作符会把上游流的数据按一定的规则进行分组，产生内部流，组合成一个高阶流返回。

### groupBy

`groupBy`会检查上游源数据的 key，并把相同 key 值的数据放在同一个内部流进行传递。`groupBy`可以将上游的数据交叉地传递给下游高阶流的内部流。这与连续传递的`buffer`和`window`不同。

```javascript
let s1$ = Rx.Observable.interval(1000)
const s2$ = s1$.groupBy(x => x % 2)
```

应用：可以通过`event.target.className`作为 key 值，实现对事件委托进行分组处理。
![groupBy](./groupBy.png)

### partition

`partition`会对上游流的数据进行判定，满足条件的作为一个数据流，不满足的作为另外一个数据流，返回包含这两个流的数组。

## 累计操作符

### scan

`scan`操作符和`reduce`类似，不过`scan`会在对上游流的每一个数据的累计计算都返回结果。而`reduce`只有在上游流完结时，才会把累计的**最终**结果返回。

应用：`scan`可以保存应用的状态，并根据数据流更新这些状态。

### mergeScan

`mergeScan`的规约函数会返回一个 Observable 对象，`mergeScan`会记住 Observable 对象传给下游的最后一个数据，并把它作为 accumution 参数传给规约函数。

> `mergeScan`返回的 Observable 对象最好只包括一个数据，不然难以确定最后一个数据具体是什么。
