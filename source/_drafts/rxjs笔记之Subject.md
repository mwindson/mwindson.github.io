---
title: rxjs笔记之Subject
tags: [js,rxjs]
---

## Subject

### hot 流和 cold 流

`rxjs`中创建的数据流分为 hot 流和 cold 流。当 Observer 订阅数据流时，cold 流会从**数据流的起始时间点**吐出数据，而 hot 流会从**订阅的时间点**开始吐出数据。

cold 流实现的是单播，hot 流实现的是多播。

cold 流由`interval`、`range`、`of`等操作符创建。

hot 流的创建操作符包括`fromPromise`、`fromEvent`、`fromEventPattern`。这些操作符数据来源于外部，如 Promise 或 Dom 事件。

cold 流的数据在没有订阅者的情况下，数据都不会产生。可以认为，每个订阅者订阅的 cold 流是独立的数据流。

hot 流的数据产生与订阅者无关，在没有订阅者的情况下，数据仍然会产生，不过不会传入管道。可以认为，每个订阅者订阅的 hot 流是同一数据流。

### subject 扮演的角色

`subject`是一个 Observable,也是一个 Observer。

`subject`通过`subject.next`方法向 subject 中传入数据。因此`subject`可以接受另一个`Observable`的数据。

`subject`能通过调用`subject.complete`来停止数据传递，进入结束状态。

`subjecet`可以保存多个订阅自己的 Observer 列表，而`Observable`没有状态，不能记忆多个订阅者。

因此，`subject`可以通过接受一个 cold 流，并把流中的数据传递给多个订阅 `subject` 的订阅者，从而将一个 cold 流转化为一个 hot 流以实现多播。

```javascript
let s1$ = Rx.Observable.interval(1000).take(4)

let subject = new Rx.Subject()

s1$.subscribe(
  value => {
    subject.next(value)
  },
  error => {
    subject.error(error)
  },
  () => {
    subject.complete()
  }
)
// 简写
s1$.subscribe(subject)
```
