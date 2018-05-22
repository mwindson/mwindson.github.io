---
title: Node.js笔记之cluster
tags: [js,node]
---

由于单个 Node 进程在运行时只能使用其中一个内核，因此简单的 Node 程序无法充分利用多核 CPU。为了解决这个问题，Node 提供了 cluster(集群)API。

### cluster 概述

`cluster`提供了简单的创建**共享服务器端口**的子进程。

```javascript
const cluster = require('cluster')
const http = require('http')
const numCPUs = require('os').cpus().length

if (cluster.isMaster) {
  console.log('CPU数', numCPUs)
  console.log('master is running')
  // 根据CPU的核数进行子进程的创建
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork()
  }
} else {
  http
    .createServer(function(req, res) {
      res.writeHead(200)
      res.end(`a worker running in process ${process.pid}`)
    })
    .listen(3000)
}
```

### 进程创建、销毁

#### 进程创建

`cluster.fork`可以创建一个子进程并返回。`fork`行为可以通过`cluster.setupMaster`设置并修改。

当子进程被 fork 后，`cluster`主进程的**fork**、**online**事件会依次触发。
#### 进程销毁

子进程可以通过`disconnect`和`exit`销毁。当子进程退出时，子进程会断开与主进程的IPC管道，然后退出。主进程的**disconnect**、**exit**事件会依次触发。

```javascript
const cluster = require('cluster')
const http = require('http')
const numCPUs = require('os').cpus().length

if (cluster.isMaster) {
  console.log('CPU数', numCPUs)
  console.log('master is running')
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork()
  }
  cluster.on('fork', worker => {
    console.log(`worker ${worker.id} is forked`)
  })
  cluster.on('online', worker => {
    console.log(`worker ${worker.id} is running`)
  })
  cluster.on('listening', (worker, address) => {
    console.log(`A worker ${worker.id} is now connected to ${address.address}:${address.port}`)
  })
  cluster.on('disconnect', worker => {
    console.log(`worker ${worker.id} has disconnected`)
  })
  cluster.on('exit', function(worker, code, signal) {
    console.log(`Worker ${worker.id} exit`)
  })
} else {
  http
    .createServer(function(req, res) {
      res.writeHead(200)
      res.end(`I am a worker running in process ${process.pid}`)
    })
    .listen(3000)
  setTimeout(() => process.disconnect(), 1000)
}
```

### 进程通信

在`cluster`模块中，父子进程通过[IPC](https://zh.wikipedia.org/wiki/%E8%A1%8C%E7%A8%8B%E9%96%93%E9%80%9A%E8%A8%8A)的方式进行通信。

* 父进程消息发送：`worker.send()`或`ChildProcess.send()`
* 父进程消息接受：`cluster`监听 message
* 子进程消息发送：`process`监听 message

父子进程相互通信并且遍历`cluster.workers`查看运行 worker 的 id

```javascript
const cluster = require('cluster')
const http = require('http')
const numCPUs = require('os').cpus().length

if (cluster.isMaster) {
  console.log('[master] ' + 'start master...')

  for (let i = 0; i < numCPUs; i++) {
    const wk = cluster.fork()
    wk.send('[master] ' + 'hi worker' + wk.id)
  }
  cluster.on('message', (worker, msg, handle) => {
    console.log(`[worker] worker ${worker.id} : ${msg}`)
  })
  cluster.on('exit', (worker, code, signal) => {
    console.log(`[worker] ${worker.id} disconnected`)
    for (let id in cluster.workers) {
      console.log(`woker ${id} is working`)
    }
    cluster.fork()
  })
} else {
  process.on('message', function(msg) {
    console.log('[worker] ' + msg)
    process.send('[worker] worker' + cluster.worker.id + ' received!')
    process.disconnect()
  })
  http.createServer().listen(3000)
}
```
### 负载均衡

`cluster`模块的调度策略通过`cluster.schedulingPolicy`来设置。根据Node文档，除Windows外的操作系统中，`cluster`模块默认使用round-robin算法进行负载均衡。

在Windows系统中需要设置`cluster.schedulingPolicy = cluster.SCHED_RR`来启用RR算法来负载均衡

### pm2

PM2是一个用于Node.js应用的进程管理程序，通过`-i选项来设置来开启cluster模式。另外，PM2还具有无间断重启、增加进程个数的优势。
```javascript
$ pm2 start app.js -i 4
```