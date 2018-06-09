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

### 多播操作符

#### multicast

`multicast`是一个实例操作符，它可以通过接受一个 `Subject` 对象，将上游 cold 流转化为 hot 流。

```javascript
const hotSource$ = coldSource$.multicast(new Subject())
```

1.  connect

    当`Observer`去订阅上述代码返回的 hot 数据流时，`hotSource$`并不会吐出数据给订阅者。

    如果`multicast`操作符返回的是一个常规 hot 流，那么当无`Observer`订阅时，hot 流依然会吐出数据。为了防止数据丢失，Rxjs 使返回的 hot 流必须在调用`hotSource$.connect()`后，才会吐出数据并向订阅者传递。因此`connect`能够如闸门一样控制数据是否流出。

2.  refCount

    `refCount`能够实现一种功能：当有`Observer`订阅数 > 1 时，`multicast`返回的 hot 流开始传递数据，而订阅数减少为 0 时，自动退订上游流。

    注意，如果要使用`refCount`，那么`multicast`需要接受返回一个`Subject`对象的函数作为参数。这是为了防止传入一个固定`Subject`对象后，一旦该`Subject`对象结束并不再接受数据，那么之后的`Observer`将不能获得数据。

    ```javascript
    const cold$ = Observable.interval(1000).take(3)
    // fixed Subejct
    let hot$ = cold$.multicast(new Subject()).refCount()
    // Subject function
    hot$ = cold$.multicast(() => new Subject()).refCount()

    hot$.subscribe(value => console.log('Observer 1: ' + value))
    setTimeout(() => {
      hot$.subscribe(value => console.log('Observer 2: ' + value))
    }, 1500)
    setTimeout(() => {
      hot$.subscribe(value => console.log('Observer 2: ' + value))
    }, 5000)
    ```

3.  selector 参数

    [todo]

### 高级 Subject

#### publishLast 和 AsyncSubject

`publishLast`只多播上游的最后一个数据，当上游的 cold 流完结时，才会把最后一个数据传递给订阅者。

`publishLast`返回的多播流是可重用的，即使内部的`AsyncSubject`结束，新的`Observer`依然可以获得最后一个数据。

#### publishReplay 和 ReplaySubject

`publishReplay`接受两个参数：`bufferSize`和`windowTime`，分别代表 Replay 的数据个数和时间长度。

`publishReplay`会存储部分 Replay 的数据，并把这些数据一次性传递给接下来的订阅者。

```javascript
// rxjs v6将链式调用改为传入pipe
let cold$ = interval(1000).pipe(take(4))
// replay 最新的两个数据
let hot$ = cold$.pipe(
  publishReplay(2),
  refCount()
)
// 订阅
hot$.subscribe(value => console.log('observer 1: ' + value))
setTimeout(() => {
  hot$.subscribe(value => console.log('observer 2: ' + value))
}, 3000)
setTimeout(() => {
  hot$.subscribe(value => console.log('observer 3: ' + value))
}, 5000)
setTimeout(() => {
  hot$.subscribe(value => console.log('observer 4: ' + value))
}, 6000)

// observer 1: 0
// observer 1: 1
// observer 1: 2
// observer 2: 1
// observer 2: 2
// observer 1: 3
// observer 2: 3
// observer 3: 2
// observer 3: 3
// observer 4: 2
// observer 4: 3
```

#### publishBehavior
