---
title: Node.js笔记之Stream
tags: [js,node]
---

## 什么是 stream

流（stream）是一种处理数据的思想——数据可以如水流一样流动，可以分段，具有方向。流的思想使它在操作大量数据或连续不断的数据具有优势。

## Stream 类别

Node.js 中具有四种基本类型的流：

* `Readable` - 可读流
* `Writable` - 可写的流
* `Duplex` - 可读写的流
* `Transform` - 特殊的`Duplex`流，在读写中可以修改和变换数据

### Readable 流

Readable 流中的数据并不是直接流向消费者，而是先 push 到缓存池中。当数据超过缓存池的阈值标志 highWatermark 时，push 就会失败并返回 false。

Readable 流具有两种状态：flowing 和 pause —— 前者会将数据不断 pipe 给下游流，后者会停止传输数据直到下游流显示的调用 Stream.read()。

> Readable 流默认是 pause 的。

![flowing mode](/images/flowing_mode.png)

<center><b>flowing mode</b></center>

![pause mode](/images/pause_mode.png)

<center><b>pause mode</b></center>

两者状态转换情况：

pause -> flowing:

1.  添加 data 事件订阅
2.  调用`.resume()`转换为 flowing
3.  调用`.pipe()`将数据传输给 Writable 流

flowing -> pause:

1.  未 pipe 任何流时，调用.pause()暂停
2.  已经 pipe 到其他流上时，需要移除所有 data 订阅事件，并且逐一调用`.unpipe()`解除。

### Writable 流

数据流过来的时候，Writable 流内部通过`writeOrBuffer`来判断将数据是直接提供给数据消费者还是暂时先存放到缓冲区。当写入速度过快或者无消费者时，数据流会进入缓冲池缓存起来。
![writable](/images/writable_stream.png)

### Duplex 流

Duplex 流同时具有可写和可读流，不过两个流之间没有关系，互不干扰。
![duplex](/images/duplex_stream.png)

### Transform 流

Transform 流同样具有可写和可读的能力。不过可写流会通过一次 transform 函数的转换处理，然后作为可读流，提供给消费者。

![transform](/images/transform_stream.png)

## pipe

`.pipe()`方法是 stream 中最重要的方法。它提供了数据之间流动/桥接的方法。

具体方法如下：

```javascript
// 将一个可读流的数据流向一个可写流
Readable.pipe(Writbale)
```

## 自定义流的实现

Node.js 的`stream`流模块用于简单的实现流。开发者可以通过 js 的原型继承模式或更简单的构造函数方式来实现自定义流。不同种类的流必须实现相应的方法。

### 原型继承法

```javascript
// 需要实现_write方法
class myWritableStream extends Writable {
  constructor(opts) {
    super(opts)
  }
  _write(chunk, encoding, callback) {
    console.log(chunk.toString().toUpperCase())
    callback()
  }
}
new myWritableStream().write('test', () => {
  console.log('write end')
})
// print "TEST"
```

### 构造函数法

```javascript
// transform流需要实现transform方法
const myTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase())
    callback()
  }
})

myTransform.write('aaa')
myTransform.end()
myTransform.pipe(process.stdout)
// print "AAA"
```

## Backpressure

`Backpressure`是指在数据流从上游生产者向下游消费者传输的过程中，上游生产速度大于下游消费速度，导致下游的 Buffer 溢出的一种现象。

由于消费者消费速度过慢或生产者生产速度过快， stream 内部可能会产生 Backpressure 现象。

### Readable 流

当 Readable 流处于 pause mode 或者消费速度比 push 到缓存池的速度慢时，就可能发生 Backpressure 现象。

### Writable 流

当外部生产者生产速度过快时，Writable 内部的队列池会装满，此时会发生 Backpressure 问题，同时 Writable 流无法再接受生产者数据流。只有当队列得到释放后，Writable 流会触发 drain 事件。

```javascript
const Writable = require('stream').Writable
const writer = new Writable({
  write(chunk, encoding, callback) {
    // 延迟callback
    setTimeout(() => {
      callback && callback()
    })
  }
})

writeOneMillionTimes(writer, 'simple', 'utf8', () => {
  console.log('end')
})

// 写入10000次
function writeOneMillionTimes(writer, data, encoding, callback) {
  // let i = 1000000 改为1万次
  let i = 10000
  write()
  function write() {
    let ok = true
    while (i-- > 0 && ok) {
      // 写入结束时回调
      ok = writer.write(data, encoding, i === 0 ? callback : null)
    }
    if (i > 0) {
      // 这里提前停下了，'drain' 事件触发后才可以继续写入
      console.log('drain', i)
      writer.once('drain', write)
    }
  }
}
```

结果

```javascript
drain 7268
drain 4536
drain 1804
end
```

### Duplex 流

Duplex 流是 Readable 流和 Writable 流的组合，因此内部的 Readable 流和 Writable 流的 Backpressure 问题同上。

### Tansform 流

由于 Transform 流输入与输出是相互关联，内部 Readable 和 Writable 的速度相同，因此内部不存在 Backpressure 问题，而会取决于外部生产者、消费者速度的影响。
