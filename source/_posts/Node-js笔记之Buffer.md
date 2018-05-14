---
title: Node.js笔记之Buffer
date: 2018-04-22 18:40:46
tags: [js,node]
---
## Buffer概述
`Buffer`类用于在 TCP 流或文件系统操作等场景中处理二进制数据流。其大小会在创建时确定，无法调整。

## 创建Buffer
在V6之后的Node.js中，原有的`new Buffer`创建方法由于不可靠和不安全，已经被废弃。现有常用的Buffer创建办法包括：

* `Buffer.from()`
* `Buffer.alloc()`
* `Buffer.allocUnsafe()` 

```javascript
// buf1:  <Buffer 00 00 00 00 00 00 00 00 00 00>
const buf1 = Buffer.alloc(10)
// buf2:  <Buffer 01 01 01 01 01 01 01 01 01 01>
const buf2 = Buffer.alloc(10, 1)
// buf3  <Buffer 01 02 03 04>
const buf3 = Buffer.from([1, 2, 3, 4])
// buf4  <Buffer 74 65 73 74>
const buf4 = Buffer.from('test')
// 由于allocUnsafe方法会导致Buffer含有旧数据，因此结果不确定
// 内容必须被初始化，用buf.fill()来初始化
const buf5 = Buffer.allocUnsafe(10)
```
> `Buffer.allocUnsafe()` 速度快，但不安全

> 当使用 `Buffer.allocUnsafe()` 分配新建的 Buffer 时，当分配的内存小于 4KB 时，默认会从一个单一的预分配的 Buffer 切割出来。 这使得应用程序可以避免垃圾回收机制因创建太多独立分配的 Buffer。


## 编码
Node.js 目前支持的字符编码包括：
* 'ascii' - 仅支持 7 位 ASCII 数据。如果设置去掉高位的话，这种编码是非常快的。
* 'utf8' - 多字节编码的 Unicode 字符。许多网页和其他文档格式都使用 UTF-8 。
* 'utf16le' - 2 或 4 个字节，小字节序编码的 Unicode 字符。支持代理对（U+10000 至 U+10FFFF）。
* 'ucs2' - 'utf16le' 的别名。
* 'base64' - Base64 编码。
* 'latin1' - 一种把 Buffer 编码成一字节编码的字符串的方式
* 'binary' - 'latin1' 的别名。
* 'hex' - 将每个字节编码为两个十六进制字符。

字符串进行编码的方式有两种：
* 通过`Buffer.from(string[, encoding])`进行
```javascript
// buf6  <Buffer 74 65 73 74>
const buf6 = Buffer.from('test', 'ascii')
```
* 通过`buf.toString([encoding[, start[, end]]])`进行
```javascript
const buf6 = Buffer.from('test')
// 74657374
buf6.toString('hex')
```
## 其他类方法
还有一些 `Buffer` 常用的 API
* `Buffer.isBuffer`：判断对象是否为 Buffer
* `Buffer.isEncoding`：判断 Buffer 对象编码

## 实例方法
`Buffer`实例的API和数组有些相似，包括:
* `buf.length`：返回 内存为此 Buffer 实例所申请的字节数，并不是 Buffer 实例内容的字节数
* `buf.indexOf`：和数组的 indexOf 类似，返回某字符串、acsii 码或者 buf 在改 buf 中的位置
* `buf.copy`：将一个 buf 的（部分）内容复制到另外一个 buf 中
* `buf.equals`:如果 buf 与 otherBuffer 具有完全相同的字节，则返回 true，否则返回 false
* `buf.toJSON`：将buf转换成JSON格式的数据
```javascript
const buf6 = Buffer.from('test')
console.log(buf6.toJSON())
// 输出 { type: 'Buffer', data: [ 116, 101, 115, 116 ] }
```
* 一些指定字节格式读写的API

另外，`Buffer`实例可以用`for..of`进行遍历。当然，也存在`buf.values()` 、`buf.keys()` 和 `buf.entries()` 方法