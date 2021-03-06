## Stream

<div class="s s2"></div>

stream 是一个抽象的接口，而且在 Node.js 中被多种对象进行了继承和扩展，比如 `http.IncomingMessage` 就是一个 stream。stream 是可读、可写或可读写的。所有的 stream 都是 EventEmitter 的实例。

通过 `require('stream')` 可以加载 Stream 的基类，提供的基类包括 Readable stream / Writable stream / Duplex stream 和 Transform stream。

本节文档主要分为三个部分：

1. 第一部分介绍了开发者需要在开发中使用 steam 所涉及的 API 
2. 第二部分介绍了开发者创建自定义 stream 所需要的 API
3. 第三部分深入解析了 stream 的工作机制，包括一些内部机制

## stream 常用 API

Stream 可以是可读、可写或双工的（可读写）。

所有的 stream 都是 EventEmitter 的实例，但是它们也有一些独有的方法和属性。

如果 stream 是可读写的，那么它一定拥有以下所有的方法和事件。所以即使 Duplex 和 Transform stream 间存在差异，这一部分所介绍的 API 也会完整的描述它们的功能。

不要为了刻意使用 stream 而实现 Stream 接口，如果要实现自定义的 Stream，请参考第二部分的文档。

对于大多数的 Node.js 程序来说，无论多么简单都有可能会用到 Stream。下面是一个使用 stream 的示例：

```js
const http = require('http');

var server = http.createServer( (req, res) => {
  // req is an http.IncomingMessage, which is a Readable Stream
  // res is an http.ServerResponse, which is a Writable Stream

  var body = '';
  // we want to get the data as utf8 strings
  // If you don't set an encoding, then you'll get Buffer objects
  req.setEncoding('utf8');

  // Readable streams emit 'data' events once a listener is added
  req.on('data', (chunk) => {
    body += chunk;
  });

  // the end event tells you that you have entire body
  req.on('end', () => {
    try {
      var data = JSON.parse(body);
    } catch (er) {
      // uh oh!  bad json!
      res.statusCode = 400;
      return res.end(`error: ${er.message}`);
    }

    // write back something interesting to the user:
    res.write(typeof data);
    res.end();
  });
});

server.listen(1337);

// $ curl localhost:1337 -d '{}'
// object
// $ curl localhost:1337 -d '"foo"'
// string
// $ curl localhost:1337 -d 'not json'
// error: Unexpected token o
```

## Class: stream.Duplex

Duplex stream 是同时实现了 Readable 和 Writable 接口的 stream。

Duplex stream 的示例包括：

- [TCP socket](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [zlib stream](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto stream](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)

## Class: stream.Readable

Readable stream 接口是对数据源的抽象。换言之，数据由 Readable stream 产生。

Readable stream 并不主动开始发送数据，直到显式表明可以接收数据时才会发送数据（类似于惰性加载）。

Readable stream 拥有两种模式：流动模式（flowing mode）和暂停模式（paused mode）。在流动模式中，数据从操作系统底层获取并尽可能快的传递给开发者；在暂停模式中，开发者必须显式调用 `stream.read()` 才能获取数据。其中，默认使用暂停模式。

注意，如果没有为 `data` 事件设置监听器，没有设置 `stream.pipe()` 的输出对象，那么 stream 就会自动切换为流动模式，且数据会丢失。

开发者可以通过以下方式切换为流动模式：

- 添加 `data` 事件处理器监听数据
- 调用 `stream.resume()` 显式打开数据流
- 调用 `stream.pipe()` 将数据发送给 Writable stream

通过以下模式可以切换为暂停模式：

- 调用 `stream.pasue()` 时不传递输出对象
- 移除 `data` 事件处理器且调用 `stream.pasue()` 时不传递输出对象

注意，为了保持向后兼容性，移除 `data` 事件处理器并不会自动暂停 stream。此外，如果存在输出对象，则调用 `stream.pause()` 并不能保证输出对象为空且要求获取数据时仍保持暂停状态。

下面是一个使用 Readable stream 的示例：

- [HTTP responses, on the client](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_incomingmessage)
- [HTTP requests, on the server](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_incomingmessage)
- [fs read streams](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_class_fs_readstream)
- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)
- [TCP sockets](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [child process stdout and stderr](https://nodejs.org/dist/latest-v5.x/docs/api/child_process.html#child_process_child_stdout)
- [process.stdin](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stdin)

#### 事件：'close'

当 stream 或底层资源（比如文件描述符）关闭时就会触发该事件。该事件也用于指示之后再没有其他事件会被触发，也不会再有任何计算。

并不是所有的 stream 都会触发 `close` 事件。

#### 事件：'data'

- `chunk`，Buffer 实例或字符串，数据块

给未显式暂停的 stream 绑定 `data` 事件监听器会让 stream 切换为流动模式，数据会被可能快的传送出去。

如果开发者只是想从 stream 尽快获取所有的数据，下面是最好的方式：

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
});
```

#### 事件：'end'

当没有数据可以读取时就会触发该事件。

注意，除非所有的数据都被处理了，否则不会触发 `end` 事件。可以通过切换到流动模式或反复调用 `stream.read()` 直到结束来实现。

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
});
readable.on('end', () => {
  console.log('there will be no more data.');
});
```

#### 事件：'error'

- `Error` 实例

如果接受到的数据存在错误就会触发该事件。

#### 事件：'readable'

当可以读取 stream 中的数据块时，就会触发 `readable` 事件。

在某些情况下，如果数据还没有准备好，那么监听 `readable` 事件将会从系统底层读取数据并放入内部缓冲中：

```js
var readable = getReadableStreamSomehow();
readable.on('readable', () => {
  // there is some data to read now
});
```

一旦内部缓存为空且所有的数据准备完成时，就会再次触发 `readable` 事件。

唯一的异常是在流动模式中，stream 数据传送完成时不会触发 `readable` 事件。

`readable` 事件用于表示 stream 接收到了新的消息：要么是新数据可用了要么是 stream 的数据全部发送完毕了。对于前一种情况，`stream.read()` 将会返回新数据；对于后一种情况，`stream.read()` 将会返回 null。举例来说，在下面的代码中，`foo.txt` 是一个空文件：

```js
const fs = require('fs');
var rr = fs.createReadStream('foo.txt');
rr.on('readable', () => {
  console.log('readable:', rr.read());
});
rr.on('end', () => {
  console.log('end');
});
```

运行后的输出结果：

```bash
$ node test.js
readable: null
end
```

#### readable.isPaused()

- 返回值类型：布尔值

该方法返回一个布尔值，表示 readable stream 是否是通过客户端代码（使用 `stream.pause()`）显式暂停的。

```js
var readable = new stream.Readable

readable.isPaused() // === false
readable.pause()
readable.isPaused() // === true
readable.resume()
readable.isPaused() // === false
```

#### readable.pause()

- 返回值类型：`this`

该方法会让处于流动模式中的 stream 停止触发 `data` 事件，并切换为暂停模式。所有可用的数据都会保留在内部缓存中。

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
  readable.pause();
  console.log('there will be no more data for 1 second');
  setTimeout(() => {
    console.log('now data will start flowing again');
    readable.resume();
  }, 1000);
});
```

#### readable.pipe(destination[, options])

- `destination`，stream.Writable 的实例，被写入数据的目标对象
- `options`，对象，包含以下属性：
    - `end`，布尔值，是否读取到了结束符，默认值为 true

该方法从 readable stream 拉取所有的数据，并将其写入 `destination`，整个过程是由系统自动管理的，所以不用担心 `destination` 会被 readable stream 的吞吐量压垮。

`pipe()` 可以接受多个 `distination`。

```js
var readable = getReadableStreamSomehow();
var writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt'
readable.pipe(writable);
```

由于该方法返回目标对象 stream，所以可以执行链式调用：

```js
var r = fs.createReadStream('file.txt');
var z = zlib.createGzip();
var w = fs.createWriteStream('file.txt.gz');
r.pipe(z).pipe(w);
```

下面代码模拟了 Unix 的 `cat` 命令：

```js
process.stdin.pipe(process.stdout);
```

默认情况下当源 stream 触发了 `end` 事件之后，就会使用 `destination` 调用 `stream.end()`，所以 `distination` 将不再可写。设置 `options` 为 `{ end: false }` 可以保持 `destination` stream 的开启状态。

下面的代码会让 `writer` 一直处于开启状态，所以 `end` 事件之后还可以写入数据 `Goodbye`：

```js
reader.pipe(writer, { end: false });
reader.on('end', () => {
  writer.end('Goodbye\n');
});
```

注意，`{ end: false }` 对 `process.stderr` 和 `process.stdout` 没有效用，只有进程退出时它们才会被关闭。

#### readable.read([size])

- `size`，数值，可选参数，用于指定读取的数据量
- 返回值类型：字符串、Buffer 实例或 Null

`read()` 从内部缓存读取数据并返回该数据。如果没有可用的数据，则返回 `null`。

如果传入了 `size`，则返回指定长度的数据。如果 `size` 长的数据不可能，除非是最后的数据块（返回剩余所有），否则返回 null。

如果未指定 `size` 参数，则会返回内部缓存中的所有数据。

该方法只能在暂停模式中调用，因为在流动模式中，系统会自动调用该方法直到内部缓存为空。

```js
var readable = getReadableStreamSomehow();
readable.on('readable', () => {
  var chunk;
  while (null !== (chunk = readable.read())) {
    console.log('got %d bytes of data', chunk.length);
  }
});
```

如果该方法返回了数据块，那么它也会触发 `data` 事件。

注意，在 `end` 事件之后调用 `stream.read([size])` 会返回 `null`，而不会抛出任何运行时错误。

#### readable.resume()

- 返回值类型：`this`

该方法用于恢复 readable stream 的 `data` 事件。

该方法会将 stream 切换为流动模式。如果你不想要使用来自 stream 的数据，只想触发 `end` 事件，可以使用 `stream.resume()` 启动数据的流动模式：

```js
var readable = getReadableStreamSomehow();
readable.resume();
readable.on('end', () => {
  console.log('got to the end, but did not read anything');
});
```

#### readable.setEncoding(encoding)

- `encoding`，字符串，使用的编码格式
- 返回值类型：`this`

该方法根据指定的编码格式返回字符串，而不是返回 Buffer 对象。举例来说，如果调用 `readable.setEncoding('utf8')`，则输出的数据就会被解析为 UTF-8 数据，并以字符串的形式返回。如果调用 `readable.setEncoding('hex')`，则会返回十六进制格式的字符串数据。

该方法可以正确处理多字节字符，但如果开发者直接获取 Buffer 数据并使用 `buf.toString(encoding)` 处理，则无法正确处理多字节字符。如果你想以字符串的形式读取数据，那么最好一直使用该方法。

当然，开发者也可以以 `readable.setEncoding(null)` 的形式调用该方法并禁用任何编码格式。该方法对于处理二进制数据或大量的多字节字符串非常有用。

```js
var readable = getReadableStreamSomehow();
readable.setEncoding('utf8');
readable.on('data', (chunk) => {
  assert.equal(typeof chunk, 'string');
  console.log('got %d characters of string data', chunk.length);
});
```

#### readable.unpipe([destination])

- `destination`，stream Writeable 实例，可选参数，解除指定 stream

该方法用于移除调用 stream.pipe() 之前的钩子方法。

如果未指定 `destination`，则移除所有的 pipe。

如果指定了 `destination`，但没有对应的 pipe，则该操作无效。

```js
var readable = getReadableStreamSomehow();
var writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt',
// but only for the first second
readable.pipe(writable);
setTimeout(() => {
  console.log('stop writing to file.txt');
  readable.unpipe(writable);
  console.log('manually close the file stream');
  writable.end();
}, 1000);
```

#### readable.unshift(chunk)

- `chunk`，Buffer 实例或字符串，将数据块插入到读取队列

如果某个 stream 中的数据已经被解析过了，但是又需要解析前的数据，那么就可以使用该方法进行你操作将 stream 传递给第三方。

注意，`stream.unshift(chunk)` 不能再 `end` 事件之后调用，否则会抛出运行时错误。

如果开发者发现在程序中必须多次调用 `stream.unshift(chunk)`，那么请考虑实现一个 `Transform` stream。

```js
// Pull off a header delimited by \n\n
// use unshift() if we get too much
// Call the callback with (error, header, stream)
const StringDecoder = require('string_decoder').StringDecoder;
function parseHeader(stream, callback) {
  stream.on('error', callback);
  stream.on('readable', onReadable);
  var decoder = new StringDecoder('utf8');
  var header = '';
  function onReadable() {
    var chunk;
    while (null !== (chunk = stream.read())) {
      var str = decoder.write(chunk);
      if (str.match(/\n\n/)) {
        // found the header boundary
        var split = str.split(/\n\n/);
        header += split.shift();
        var remaining = split.join('\n\n');
        var buf = new Buffer(remaining, 'utf8');
        if (buf.length)
          stream.unshift(buf);
        stream.removeListener('error', callback);
        stream.removeListener('readable', onReadable);
        // now the body of the message can be read from the stream.
        callback(null, header, stream);
      } else {
        // still reading the header.
        header += str;
      }
    }
  }
}
```

注意，与 `stream.push(chunk)` 不同，`stream.unshift(chunk)` 不会通过重置 stream 的内部读取状态来中断读取继承。如果在读取期间调用 `unshift()`，则有可能出现未知的结果。如果在调用 `unshift()` 惠州立即调用 `stream.push('')`，将会重置读取状态，但是最好的方式还是在读取过程中不要调用 `unshift()`。

#### readable.wrap(stream)

- `stream`，Stream 实例，旧式的 readable stream

在 Node.js v0.10 之前，Stream 并没有实现完整的 Stream API。

如果你正在使用旧式的 Node.js 库，那么就只能使用 `data` 事件和 `stream.pasue()` 方法，通过 `wrap()` 方法可以创建一个使用旧式 stream 的 Readable stream。

应该尽量少用该函数，该函数存在的价值只是为了便于与旧版的 Node.js 程序和库进行交互。

```js
const OldReader = require('./old-api-module.js').OldReader;
const Readable = require('stream').Readable;
const oreader = new OldReader;
const myReader = new Readable().wrap(oreader);

myReader.on('readable', () => {
  myReader.read(); // etc.
});
```

## Class: stream.Transform

Transform stream 是 Duplex stream，其中输入是由输出计算而来的。它们都实现了 Readable 和 Writable 的接口。

下面是一些使用 Transform stream 的实例：

- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)

## Class: stream.Writable

Writable stream 接口是对接收数据的 destination 的抽象。

下面是一些使用 writable stream 的实例：

- [HTTP requests, on the client](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_clientrequest)
- [HTTP responses, on the server](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_serverresponse)
- [fs write streams](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_class_fs_writestream)
- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)
- [TCP sockets](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [child process stdin](https://nodejs.org/dist/latest-v5.x/docs/api/child_process.html#child_process_child_stdin)
- [process.stdout](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stdout)
- [process.stderr](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stderr)

#### 事件：'drain'

如果 `stream.write(chunk)` 返回 false，`drain` 事件就会通知开发者何时适合向 stream 写入更多的数据。

```js
// Write the data to the supplied writable stream one million times.
// Be attentive to back-pressure.
function writeOneMillionTimes(writer, data, encoding, callback) {
  var i = 1000000;
  write();
  function write() {
    var ok = true;
    do {
      i -= 1;
      if (i === 0) {
        // last time!
        writer.write(data, encoding, callback);
      } else {
        // see if we should continue, or wait
        // don't pass the callback, because we're not done yet.
        ok = writer.write(data, encoding);
      }
    } while (i > 0 && ok);
    if (i > 0) {
      // had to stop early!
      // write some more once it drains
      writer.once('drain', write);
    }
  }
}
```

#### 事件：'error'

- `Error` 实例

如果写入或 pipe 数据的时候出现了错误就会触发该事件。

#### 事件：'finish'

调用 `stream.end()` 并且所有数据都被刷新到系统底层后，就会触发该事件。

```js
var writer = getWritableStreamSomehow();
for (var i = 0; i < 100; i ++) {
  writer.write('hello, #${i}!\n');
}
writer.end('this is the end\n');
writer.on('finish', () => {
  console.error('all writes are now complete.');
});
```

#### 事件：'pipe'

- `src`，stream.Readable，发起 pipe 写入操作的源 stream

当 readable stream 调用 `stream.pipe()` 方法时就会触发该事件，并添加 writable stream 到 `destination` 上。

```js
var writer = getWritableStreamSomehow();
var reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.error('something is piping into the writer');
  assert.equal(src, reader);
});
reader.pipe(writer);
```

#### 事件：'unpipe'

- `src`，Readable stream，发起 unpipe 写入操作的源 stream

当 readable stream 调用 `stream.unpipe()` 方法时就会触发该事件，并移除 `destination` 中的 writable stream。

```js
var writer = getWritableStreamSomehow();
var reader = getReadableStreamSomehow();
writer.on('unpipe', (src) => {
  console.error('something has stopped piping into the writer');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer);
```

#### writable.cork()

该方法强制系统缓存所有写入的数据。

调用 `stream.uncork()` 或 `stream.end()` 时都会刷新缓存数据。

#### writable.end([chunk][, encoding][, callback])

- `chunk`，字符串或 Buffer 实例，写入的数据
- `encoding`，字符串，如果 chunk 是字符串，则该参数指定编码格式
- `callback`，函数，stream 结束时执行的回调函数

当没有数据需要写入到 stream 时可以调用该方法。如果指定了 `callback`，则该参数会被绑定为 `finish` 事件的监听器。

在 `stream.end()` 之后调用 `stream.write()` 将会触发错误。

```js
// write 'hello, ' and then end with 'world!'
var file = fs.createWriteStream('example.txt');
file.write('hello, ');
file.end('world!');
// writing more now is not allowed!
```

#### writable.setDefaultEncoding(encodign)

- `encoding`，字符串，编码格式

该方法用于设置 writable stream 的默认字符串编码格式。

#### writable.uncork()

该方法用于刷新调用 `stream.cork()` 之后缓存的所有数据。

#### writable.write(chunk[, coding][, callback])

- `chunk`，字符串或 Buffer 实例，写入的数据
- `encoding`，字符串，如果 chunk 是字符串，则该参数指定编码格式
- `callback`，函数，数据块刷新后执行的回调函数
- 返回值类型：布尔值，如果数据处理完成则返回 true

该方法用于向系统底层写入数据，并在数据完全写入后指定回调函数。如果出现了错误，将无法确定是否会执行 callback，所以为了检测错误，请监听 `error` 事件。

该方法的返回值用于通知开发者是否应该继续写入数据。如果数据已在内部缓存，那么就会返回 false，否则返回 true。

该返回值主要是建议性的，开发者还是可以继续写入数据，即使返回的是 false。不过，写入的数据会被缓存在内存中，所以最好不要这么做。相反，开发者可以等出发了 `drain` 事件之后再继续写入数据。

## 自定义 Stream 的 API

实现自定义 Stream 的模式：

1. 从恰当的父类创建你需要的子类，`util.inherits()` 方法对此很有帮助
1. 在子类的构造器中合理调用父类的构造器，确保内部机制初始化成功
1. 实现一个或多个方法，如下所示

所要扩展的类和要实现的方法取决于开发者编写的 stream 的类型：

| 用途                | 类               | 实现的方法             |
| -------------------|------------------|-----------------------|
| 只读                |Readable          |_read                  |
| 只写                |Writable          |_write, _writev        |
| 读写                |Duplex            |_read, _write, _writev |
| 写数据读结果         |Transform         |_transform, _flush     |

在开发者的实现代码里，千万不要调用第一部分的代码，否则会引起不利的副作用。

## Class: stream.Duplex

Duplex stream 是可读写的 stream，比如 TCP socket 连接。

注意，`stream.Duplex` 是一个抽象类，也是 `stream._read(size)` 和 `stream._write(chunk, encoding, callback)` 的底层基础。

因为 JavaScript 没有多原型继承机制，所以该类继承自 Readable，而又寄生于 Writable。从而允许开发者实现底层的 `stream._read(n)` 和 `stream._write(chunk, encoding, callback)` 方法来扩展 Duplex 类。

#### new stream.Duplex(options)

- `options` 是一个对象，同时传递给 Writable 和 Readable 的构造器，具有以下属性：
    - `allowHalfOpen`，布尔值，默认值为 true。如果值为 false，则当 writable 端停止时 readable 端的 stream 也会自动停止，反之异然
    - `readableObjectMode`，布尔值，默认值为 false。为 readable stream 设置 `objectMode`。如果 `objectMode === true`，则没有任何效果
    - `writableObjectMode`，布尔值，默认值为 false。为 writable stream 设置 `objectMode`。如果 `objectMode === true`，则没有任何效果

对于继承了 Duplex class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

## Class: stream.PassThrough

该类是 Transform stream 的一个实现，相对而言并不重要，只是简单的将输入的数据传送给输出。创建该类的目的主要是为了演示和测试，但也偶尔用做新型 stream 的构建基块。

## Class: stream.Readable

`stream.Readable` 是一个可扩展的底层抽象类，常服务于 `stream._read(size)` 方法。

#### new stream.Readable([options])

- `options`，对象
    - `highwatermark`，数值，表示内部缓存可以存储的最大字节量，默认值为 16384（16kb），对于 `objectMode` stream，默认值是 16
    - `encoding`，字符串，如果指定了该参数，则 Buffer 实例会被转换为指定格式的字符串，默认值为 `null`
    - `objectmode`，布尔值，表示 stream 的行为是否应该像是对象 stream。也就是说，`stream.read(n)` 返回一个单值，而不是长度为 n 的 Buffer 实例，默认值为 `false`
    - `read`，函数，`stream._read()` 方法的实现

对于继承了 Readable class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### readable._read(size)

- `size`，数值，异步读取的字节量

注意，可以实现该方法，但不要直接使用该方法。

该方法使用下划线作为前缀，这表明它是类的内部方法，只应该在 Readable 类内部使用。所有 Readable stream 的实现都应该提供一个 `_read` 方法从系统底层获取资源和数据。

当调用 `_read()` 时，如果资源数据可以使用了，则 `_read()` 的实现应该通过调用 `this.push(dataChunk)` 将数据加入到读取队列中。`_read()` 需要持续读取资源并将其添加到队列中，直到返回 `false` 则停止读取数据。

只有在数据读取停止之后再次调用 `_read()` 才能读取更多的数据并将数据添加到队列。

注意，一旦调用了 `_read()` 方法，那么只有调用 `stream.push()` 之后才会再次调用 `_read()`。

`size` 参数只具有建议性而不具有强制性，该参数只是用来通知读取的数据量。但与具体的实现无关，比如对于 TCP 和 TLS，它们可能会忽略该参数，只要数据可用就会输出数据。举例来说，调用 `stream.push(chunk)` 之前也没必要等到 `size` 长的数据块准备完毕。

#### readable.push(chunk[, encoding])

- `chunk`，Buffer 实例、字符串或 Null，添加到读取队列的数据块
- `encoding`，字符串，字符串的编码格式，必须是一个有效的 Buffer 编码格式，比如 utf8 或 ascii
- 返回值类型：布尔值，用于指定是否应该继续推送数据

注意，Readable 的实现者必须调用该方法，而不是由 Readable stream 的使用调用该方法。

如果传入的值不是 null，则 `push()` 方法将数据块推送到队列，便于随后的 stream 处理器使用。如果传入的是 null，则是向 stream 发送结束信号（EOF），之后将不会再写入数据。

当触发了 `readable` 事件之后，通过 `stream.read()` 可以拉取使用 `push()` 添加的数据。

该方法设计的非常灵活。举例来说，开发者可以使用该方法封装一个底层资源，包含了一些暂停和回复机制，以及一个数据回调函数：

```js
// source is an object with readStop() and readStart() methods,
// and an `ondata` member that gets called when it has data, and
// an `onend` member that gets called when the data is over.

util.inherits(SourceWrapper, Readable);

function SourceWrapper(options) {
  Readable.call(this, options);

  this._source = getLowlevelSourceObject();

  // Every time there's data, we push it into the internal buffer.
  this._source.ondata = (chunk) => {
    // if push() returns false, then we need to stop reading from source
    if (!this.push(chunk))
      this._source.readStop();
  };

  // When the source ends, we push the EOF-signaling `null` chunk
  this._source.onend = () => {
    this.push(null);
  };
}

// _read will be called when the stream wants to pull more data in
// the advisory size argument is ignored in this case.
SourceWrapper.prototype._read = function(size) {
  this._source.readStart();
};
```

#### 实例：计数 stream

这是一个基础的 Readable stream 实例，它从 1 到 1000000 递增的顺序触发数字直到结束：

```js
const Readable = require('stream').Readable;
const util = require('util');
util.inherits(Counter, Readable);

function Counter(opt) {
  Readable.call(this, opt);
  this._max = 1000000;
  this._index = 1;
}

Counter.prototype._read = function() {
  var i = this._index++;
  if (i > this._max)
    this.push(null);
  else {
    var str = '' + i;
    var buf = new Buffer(str, 'ascii');
    this.push(buf);
  }
};
```

#### 实例：简单协议 v1（初始版）

该实例与 [parseHeader 方法](https://nodejs.org/dist/latest-v5.x/docs/api/stream.html#stream_readable_unshift_chunk)类似，但它是一个自定义的 stream。值得注意的是，该方法不会将传入的数据转换为字符串。

实际上，更好的方式是使用 Transform stream 实现该方法，详情请查看 [SimpleProtocol v2](https://nodejs.org/dist/latest-v5.x/docs/api/stream.html#stream_example_simpleprotocol_parser_v2)。

```js
// A parser for a simple data protocol.
// The "header" is a JSON object, followed by 2 \n characters, and
// then a message body.
//
// NOTE: This can be done more simply as a Transform stream!
// Using Readable directly for this is sub-optimal. See the
// alternative example below under the Transform section.

const Readable = require('stream').Readable;
const util = require('util');

util.inherits(SimpleProtocol, Readable);

function SimpleProtocol(source, options) {
  if (!(this instanceof SimpleProtocol))
    return new SimpleProtocol(source, options);

  Readable.call(this, options);
  this._inBody = false;
  this._sawFirstCr = false;

  // source is a readable stream, such as a socket or file
  this._source = source;

  source.on('end', () => {
    this.push(null);
  });

  // give it a kick whenever the source is readable
  // read(0) will not consume any bytes
  source.on('readable', () => {
    this.read(0);
  });

  this._rawHeader = [];
  this.header = null;
}

SimpleProtocol.prototype._read = function(n) {
  if (!this._inBody) {
    var chunk = this._source.read();

    // if the source doesn't have data, we don't have data yet.
    if (chunk === null)
      return this.push('');

    // check if the chunk has a \n\n
    var split = -1;
    for (var i = 0; i < chunk.length; i++) {
      if (chunk[i] === 10) { // '\n'
        if (this._sawFirstCr) {
          split = i;
          break;
        } else {
          this._sawFirstCr = true;
        }
      } else {
        this._sawFirstCr = false;
      }
    }

    if (split === -1) {
      // still waiting for the \n\n
      // stash the chunk, and try again.
      this._rawHeader.push(chunk);
      this.push('');
    } else {
      this._inBody = true;
      var h = chunk.slice(0, split);
      this._rawHeader.push(h);
      var header = Buffer.concat(this._rawHeader).toString();
      try {
        this.header = JSON.parse(header);
      } catch (er) {
        this.emit('error', new Error('invalid simple protocol data'));
        return;
      }
      // now, because we got some extra data, unshift the rest
      // back into the read queue so that our consumer will see it.
      var b = chunk.slice(split);
      this.unshift(b);
      // calling unshift by itself does not reset the reading state
      // of the stream; since we're inside _read, doing an additional
      // push('') will reset the state appropriately.
      this.push('');

      // and let them know that we are done parsing the header.
      this.emit('header', this.header);
    }
  } else {
    // from there on, just provide the data to our consumer.
    // careful not to push(null), since that would indicate EOF.
    var chunk = this._source.read();
    if (chunk) this.push(chunk);
  }
};

// Usage:
// var parser = new SimpleProtocol(source);
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```

## Class: stream.Transform

Transform stream 是 Duplex stream，输入和输出具有因果关系，比如 zlib stream 和 crypto stream。

输入和输出没有数据块大小、数据块数量以及到达时间的要求。举例来说，一个哈希 stream 只会在结束时向输出发送一个单一的数据块；一个 zlib stream 则会生成比输入或大或小的输出结果。

Transform class 必须实现 `stream._transform()` 方法，可以选择性地实现 `stream._flush()` 方法。

#### new stream.Transform([options])

- `options`，对象，将会传递给 Writable 和 Readable class 的构造器，包含以下属性：
    - `transform`，函数，是 `stream._transform()` 的实现
    - `flush`，函数，是 `stream._flush()` 的实现

对于继承了 Transform class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### 事件：'finish' 和 'end'

`finish` 和 `end` 事件分别来自于 Writable 和 Readable class。调用 `stream.read()` 并使用 `stream._transform()` 方法处理完所有的数据块之后就会触发 `finish()` 事件；`stream._flush()` 调用完内部的回调函数并输出完所有的数据之后触发 `end` 事件。

#### transform._flush(callback)

- `callback`，函数，当开发者刷新完所有的剩余数据之后执行该回调函数

**注意，一定不要直接调用该函数**。可以在子类中实现该方法，且只允许 Transform class 的内部方法调用它。

在某些情况下爱，transform 操作需要在 stream 的最后触发额外的数据。举例来说，一个 zlib 压缩 stream 会存储一些优化压缩结果的内部状态。

在这些情况下，开发者可以实现一个 `_flush()` 方法，该方法会在所有写入的数据被处理之后、触发 `end` 事件结束 readable stream 之前被调用。与 `stream._transform()` 类似，当刷新操作完成之后，会调用 `transform.push(chunk)` 零次或多次，最后调用 `callback`。

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### transform._transform(chunk, encoding, callback)

- `chunk`，Buffer 实例或字符串，用于传输的数据块。除非 `decodeStrigns === false`，否则该参数都是 Buffer 实例
- `encoding`，字符串，如果 `chunk` 是一个字符串，则该参数指定字符串的编码格式。如果 chunk 是一个 Buffer 实例，则该参数是一个特殊值 "buffer"，在这种情况下请忽略该值。
- `callback`，函数，当处理完输入的数据块之后将会执行该回调函数

**注意，一定不要直接调用该函数**。可以在子类中实现该该方法，且只允许 Transform class 的内部方法调用它。

所有的 Transform stream 实现都必须提供一个 `_transform()` 方法接收输入并生成输出数据。

`_transform` 可以处理 Transform class 中规定的任何事情，比如处理写入的字节、将它们传给 readable stream、处理异步 I/O等任务。

调用 `transform.push(outputChunk)` 根据输入生成输出数据的次数取决于开发者想要输出的数据量。

之后当前数据块被完全处理之后才可以调用回调函数。注意，输入块或许有也或许没有对应的输出块。如果给回调函数设置了第二个参数，则该参数会被传递给 push 方法。换言之，以下代码相等：

```js
transform.prototype._transform = function (data, encoding, callback) {
  this.push(data);
  callback();
};

transform.prototype._transform = function (data, encoding, callback) {
  callback(null, data);
};
```

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### 实例：SimpleProtocol 解析器 v2

上面的简单协议解析器可以使用高阶的 Transform stream class 来实现，处理方式与 `parseHeader` 和 `SimpleProtocol v1` 类似。

在下面的代码中，并没有将输入作为参数，而是将其 pipe 进了解析器，这种方案更符合 Node.js stream 的使用习惯：

```js
const util = require('util');
const Transform = require('stream').Transform;
util.inherits(SimpleProtocol, Transform);

function SimpleProtocol(options) {
  if (!(this instanceof SimpleProtocol))
    return new SimpleProtocol(options);

  Transform.call(this, options);
  this._inBody = false;
  this._sawFirstCr = false;
  this._rawHeader = [];
  this.header = null;
}

SimpleProtocol.prototype._transform = function(chunk, encoding, done) {
  if (!this._inBody) {
    // check if the chunk has a \n\n
    var split = -1;
    for (var i = 0; i < chunk.length; i++) {
      if (chunk[i] === 10) { // '\n'
        if (this._sawFirstCr) {
          split = i;
          break;
        } else {
          this._sawFirstCr = true;
        }
      } else {
        this._sawFirstCr = false;
      }
    }

    if (split === -1) {
      // still waiting for the \n\n
      // stash the chunk, and try again.
      this._rawHeader.push(chunk);
    } else {
      this._inBody = true;
      var h = chunk.slice(0, split);
      this._rawHeader.push(h);
      var header = Buffer.concat(this._rawHeader).toString();
      try {
        this.header = JSON.parse(header);
      } catch (er) {
        this.emit('error', new Error('invalid simple protocol data'));
        return;
      }
      // and let them know that we are done parsing the header.
      this.emit('header', this.header);

      // now, because we got some extra data, emit this first.
      this.push(chunk.slice(split));
    }
  } else {
    // from there on, just provide the data to our consumer as-is.
    this.push(chunk);
  }
  done();
};

// Usage:
// var parser = new SimpleProtocol();
// source.pipe(parser)
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```

## Class: stream.Writable

`stream.Writable` 是一个可扩展的抽象类，可用于 `stream._write(chunk, encoding, callback)` 等方法的底层实现。

#### new stream.Writable([options])

- `options`，对象
    - `highwatermark`，数值，当 `stream.write()` 开始返回 `false` 时的缓存级别，默认值是 16384（16kb），对于 `objectMode` stream，默认值是 16
    - `decodeString`，布尔值，该参数决定是否在讲字符串传递给 `stream._write()` 之前将其转换为 Buffer，默认值为 true
    - `objectmode`，布尔值，决定 `stream.write(anyObj)` 是否是一个有效操作。如果值为 true，则可以写入任意类型的数据，而不只是 Buffer 和字符串数据，默认值为 `false`
    - `write`，函数，`stream._write()` 的实现
    - `writev`，函数，`stream._writev()` 的实现

对于继承了 Writable class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### writable._write(chunk, encoding, callback)

- `chunk`，Buffer 实例或字符串，写入的数据块。除非 `decodeStrings === false`，否则该参数只能是 Buffer 实例
- `encoding`，字符串，如果 `chunk` 是一个字符串，则该参数指定字符串的编码格式。如果 chunk 是一个 Buffer 实例，则该参数是一个特殊值 "buffer"，在这种情况下请忽略该值。
- `callback`，函数，当处理完输入的数据块之后将会执行该回调函数

**注意，一定不能直接调用该方法**。可以在子类中实现该方法，且只允许 Writable class 的内部方法调用它。

调用 `callback(err)` 用于通知系统数据写入完成或者出现了错误。

如果在构造器中设置了 `decodeStrings` 选项，那么 `chunk` 就只能是字符串而不能是 Buffer 实例，`encoding` 参数用于表示字符串的编码格式。这种实现是为了优化某些字符串的处理。如果没有显式设置 `decodeStrings === false`，那么系统会忽略 `encoding` 参数，并假设 `chunk` 是一个 Buffer 实例。

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### writable._writev(chunks, callback)

- `chunk`，数组，写入的数据。每一个数据块都遵循如下格式：`{ chunk: ..., encoding: ... }`
- `callback`，函数，当处理完数据块之后将会执行该回调函数

**注意，一定不能直接调用该方法**。可以在子类中实现该方法，且只允许 Writable class 的内部方法调用它

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

## 简化构造器 API

在某些简单的情况下，不通过继承创建 stream 也大有用处。

通过向构造器传递恰当的方法即可实现这一目标。

#### Duplex

```js
var duplex = new stream.Duplex({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  },
  write: function(chunk, encoding, next) {
    // sets this._write under the hood

    // An optional error can be passed as the first argument
    next()
  }
});

// or

var duplex = new stream.Duplex({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  },
  writev: function(chunks, next) {
    // sets this._writev under the hood

    // An optional error can be passed as the first argument
    next()
  }
});
```

#### Readable

```js
var readable = new stream.Readable({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  }
});
```

#### Transform

```js
var transform = new stream.Transform({
  transform: function(chunk, encoding, next) {
    // sets this._transform under the hood

    // generate output as many times as needed
    // this.push(chunk);

    // call when the current chunk is consumed
    next();
  },
  flush: function(done) {
    // sets this._flush under the hood

    // generate output as many times as needed
    // this.push(chunk);

    done();
  }
});
```

#### Writable

```js
var writable = new stream.Writable({
  write: function(chunk, encoding, next) {
    // sets this._write under the hood

    // An optional error can be passed as the first argument
    next()
  }
});

// or

var writable = new stream.Writable({
  writev: function(chunks, next) {
    // sets this._writev under the hood

    // An optional error can be passed as the first argument
    next()
  }
});
```

## Stream: 内部玄机

#### 缓存

Writable 和 Readable stream 都会在对象内部缓存数据，该对象可以通过 `_writableState.getBuffer()` 或 `_readableState.buffer` 获取。

缓存的总量取决于构造器中接收的 `highWaterMark` 配置信息。

调用 `stream.push(chunk)` 可以将数据缓存到 Readable stream 中。如果数据没有经过 `stream.read()` 处理，就会一直待在内部队列，直到被拉去和处理。

当开发者反复 `stream.write(chunk)` 时就会将数据缓存到 Writable stream 中，直到 `stream.write(chunk)` 返回 `false`。

设计 stream 的初衷，尤其是对于 `stream.pipe()` 方法，是为了在可控范围内限制数据的缓存量，所以即使输入资源和输出的目标对象之间的速度存在差异，都不会过度影响内存的使用。

#### 兼容性

在 Node.js v0.10 之前的版本，Readable stream 的接口非常简单，功能贫乏，实用性不强。

- 在老版本中，系统不会等待开发者调用 `stream.read()` 方法，而是触发 `data` 事件。如果开发者要决定如何处理数据，那么就需要手动缓存数据块
- 在老版本中，`stream.pause()` 只是建议性的方法，而不是绝对有效的。这也就是说，即使 stream 处于暂停状态，开发者仍有可能接收到 `data` 事件。

在 Node.js v0.10，添加了 Readable。为了保持向后兼容性，当添加了 `data` 事件处理器之后或调用了 `stream.resume()` 之后，Readable stream 就会切换为流动模式。这么做的好处是，即使你没有使用 `stream.read()` 或 `readable` 事件，也无需担心会丢失数据块。

虽然大多数的程序都会正常运行，但还是有必要介绍一些边缘用例：

- 没有添加任何 `data` 事件处理器
- 从没调用过 `stream.resume()` 方法
- stream 没有 pipe 到任何 writable destination

举例来说，思考一下下面的代码：

```js
// WARNING!  BROKEN!
net.createServer((socket) => {

  // we add an 'end' method, but never consume the data
  socket.on('end', () => {
    // It will never get here.
    socket.end('I got your message (but didnt read it)\n');
  });

}).listen(1337);
```

在 Node.js v0.10 之前，新传入的消息数据会被直接丢弃。不过从 Node.js v0.10 之后，socket 会保持暂停状态。

这种情况下的变通方法就是使用 `stream.resume()` 启动数据流：

```js
// Workaround
net.createServer((socket) => {

  socket.on('end', () => {
    socket.end('I got your message (but didnt read it)\n');
  });

  // start the flow of data, discarding it.
  socket.resume();

}).listen(1337);
```

为了让新的 Readable stream 切换到流动模式，在 v0.10 之前 stream 可以通过 `stream.wrap()` 包装成 Readable stream。

#### Object Mode

通常来说，stream 只适用于对字符串和 Buffer 实例的操作。

但处于 object mode 的 stream 则可以处理任何 JavaScript 值。

在 object mode 中，不论调用 `stream.read(size)` 时 `size` 是多少，Readable stream 都会返回一个单元素。

在 object mode 中，不论调用 `stream.write(data, encoding)` 时 `encoding` 是什么，Writable stream 都会忽略该参数。

特殊值 `null` 在 object mode 中仍然保持了特殊性。也就是说，对于 object mode 中的 Readable stream，如果 `stream.read()` 返回了 null，表示没有数据了；如果调用 `stream.push(null)`，表示 stream 的数据推送结束了。

Node.js 的核心 stream 没有一个是 object mode stream。object mode 只存在于用户的 stream 库中。

开发者应该在 stream 子类的构造器中设置 `objectMode` 配置信息，在其他地方设置则不安全。

对于 Duplex stream 的 `objectMode` 参数，可以通过 `readableObjectMode` 和 `writableObjectMode` 设置为 readable 或 writable。这些选项可以通过 Transform stream 来实现解析器和序列化器。

```js
const util = require('util');
const StringDecoder = require('string_decoder').StringDecoder;
const Transform = require('stream').Transform;
util.inherits(JSONParseStream, Transform);

// Gets \n-delimited JSON string data, and emits the parsed objects
function JSONParseStream() {
  if (!(this instanceof JSONParseStream))
    return new JSONParseStream();

  Transform.call(this, { readableObjectMode : true });

  this._buffer = '';
  this._decoder = new StringDecoder('utf8');
}

JSONParseStream.prototype._transform = function(chunk, encoding, cb) {
  this._buffer += this._decoder.write(chunk);
  // split on newlines
  var lines = this._buffer.split(/\r?\n/);
  // keep the last partial line buffered
  this._buffer = lines.pop();
  for (var l = 0; l < lines.length; l++) {
    var line = lines[l];
    try {
      var obj = JSON.parse(line);
    } catch (er) {
      this.emit('error', er);
      return;
    }
    // push the parsed object out to the readable consumer
    this.push(obj);
  }
  cb();
};

JSONParseStream.prototype._flush = function(cb) {
  // Just handle any leftover
  var rem = this._buffer.trim();
  if (rem) {
    try {
      var obj = JSON.parse(rem);
    } catch (er) {
      this.emit('error', er);
      return;
    }
    // push the parsed object out to the readable consumer
    this.push(obj);
  }
  cb();
};
```

#### stream.read(0)

在某些情况下，开发者只想刷新底层的 readable stream 机制，而不处理任何数据，那么就可以使用 `stream.read(0)`，该方法返回 null。

如果内部的 Buffer 数据长度小于 `highWaterMark`，且尚未被 stream 读取，那么调用 `stream.read(0)` 就调用发底层的 `stream._read()`。

一般来说没有调用该方法的必要。不过，开发者可能会发现在 Node.js 内部有这样的调用，特别是 Readable stream 的内部。

#### stream.push('')

推送一个零字节的字符串或 Buffer 实例数据会发生一些有趣的现象。因为调用了 `stream.push()`，所以会终止 `reading` 进程。不过，实际上并没有向 readable buffer 添加任何数据，也就不需要开发者处理任何数据了。

虽然现在很少有空数据的情况，但是通过调用 `stream.read(0)` 可以检查是否有任务在处理你的 stream。出于这种目的，你还是有可能调用 `stream.push('')` 的。

到目前为止，只在 `tls.CryptoStream` 类中使用过该手段，但该方法在 Node.js/io.js v1.0 已被抛弃了。如果你必须使用 `stream.push('')` 方法，请先思考是否有其他处理方式，因为这种做法会被视为发生了极其严重的错误。

<style>
.s {
    margin: 1.5rem 0;
    padding: 10px 20px;
    color: white;
    border-radius: 5px;
}
.s:before {
    display: block;
    font-size: 2rem;
    font-weight: 900;
}
.s0 {
    background-color: #C04848;
}
.s0:before {
    content: "接口稳定性: 0 - 已过时";
}
.s1 {
    background-color: #F07241;
}
.s1:before {
    content: "接口稳定性: 1 - 实验中";
}
.s2 {
    background-color: #457D97;
}
.s2:before {
    content: "接口稳定性: 2 - 稳定";
}
.s3 {
    background-color: #14C3A2;
}
.s3:before {
    content: "接口稳定性: 3 - 已锁定";
}
</style>