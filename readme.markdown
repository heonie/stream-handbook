# stream-handbook

이 문서는 [stream](http://nodejs.org/docs/latest/api/stream.html)을 이용한 [node.js](http://nodejs.org/) 프로그램 방법을 다룹니다.

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

# node packaged manuscript

이 핸드북을 npm으로 설치할 수 있습니다. 다음과 같이 해보세요:

```
npm install -g stream-handbook
```

이제 `stream-handbook` 명령어가 생기게 될 것이고, 그 명령어는 이 readme 파일을 당신의 `$PAGER`로 열게 될 것입니다.
아니면, 이 문서를 현재 보고있는 것 처럼 계속 읽을 수도 있습니다.

# 소개

```
"우리는 데이터를 다른 방식으로 변경(massage data)할 필요가 있을때, 프로그램들을 정원의 호스--스크류 처럼
다른 세그먼트에 연결할 수 있는 방법이 있어야 합니다. 이것은 IO가 하고있는 방법이기도 합니다."
```

[Doug McIlroy. October 11, 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

Stream은 [초기의 유닉스 시대](http://www.youtube.com/watch?v=tc4ROCJYbm0)부터 있어왔고,
[한 가지 일을 잘 수행하는](http://www.faqs.org/docs/artu/ch01s06.html) 작은 컴포넌트를 이용하여 대형 시스템을 구성하는 신뢰할 수 있는 방법으로서 수십년동안 스스로를 입증해왔습니다.
unix 에서, Stream은 shell에서 `|` 파이프에 의해 구현됩니다.
node 에서는, 내장되어 있는 [stream 모듈](http://nodejs.org/docs/latest/api/stream.html)이 코어 라이브러리들에서 사용되며, 또한 사용자-영역의 모듈에서도 사용될 수 있습니다.
unix와 비슷하게, node stream 모듈의 주요 결합 연산자는 `.pipe()` 라고 불리고 있고, 느리게 데이터를 소비하는 소비자를 위해 쓰기를 조절하는 배압(backpressure) 매커니즘을 제공합니다.

Stream은 [관심거리를 분리](http://www.c2.com/cgi/wiki?SeparationOfConcerns)하는데에 도움이 되는데, 그들이 구현영역을 [재사용](http://www.faqs.org/docs/artu/ch01s06.html#id2877537) 가능하도록 일관된 인터페이스로 제한하기 때문입니다.
그런 다음, 한 스트림의 출력을 다른 스트림의 입력으로 연결하고 스트림들에 대해 추상적으로 동작하는 [라이브러리를 사용](http://npmjs.org)하여 상위레벨의 흐름제어를 시작할 수 있습니다.

Stream은 [작은-프로그램 디자인](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/)과 [유닉스 철학](http://www.faqs.org/docs/artu/ch01s06.html)의 중요한 컴포넌트이지만, 생각해볼만한 많은 다른 중요한 추상개념들이 있습니다.
[기술적 부채](http://c2.com/cgi/wiki?TechnicalDebt)은 적이라는 점을 기억하고, 맞닥뜨린 문제에 대한 최고의 추상화를 추구하십시오.

![brian kernighan](http://substack.net/images/kernighan.png)

***

# 왜 stream을 써야만 하는가?

node의 I/O는 비동기이기 때문에, 디스크 그리고 네트워크와 상호작용하려면 함수에 콜백을 전달하는것을 포함하게 됩니다.
당신은 다음과 같이 디스크로부터 파일을 제공하는 코드를 작성해야 할 수도 있습니다:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        res.end(data);
    });
});
server.listen(8000);
```

이 코드는 잘 동작하지만, 부피가 크고 모든 요청에 대해 클라이언트에 결과를 쓰기 전에 `data.txt` 파일 전체를 메모리에 버퍼링합니다.
만약 `data.txt`가 매우 크다면, 특히 느린 연결을 사용하는 사용자들의 경우, 동시에 많은 사용자들에 제공하게 됨에 따라 당신의 프로그램은 많은 메모리를 잡아먹기 시작할 수 있습니다.

사용자들은 당신의 서버 메모리에 파일 전체가 버퍼되기를 기다리고 나서야 컨텐츠 받기가 시작될 수 있기 때문에 사용자 경험 또한 나빠집니다.

운좋게도 `(req, res)` 인자 둘 모두가 stream 이기 때문에, `fs.readFile()` 대신 `fs.createReadStream()`를 사용함으로서 이것을 훨씬 좋게 만들 수 있습니다:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(res);
});
server.listen(8000);
```

여기서 `.pipe()`는 `fs.createReadStream()` 으로부터 `'data'` 와 `'end'` 이벤트를 리스닝 합니다.
이 코드는 깔끔할 뿐만 아니라, 이제 `data.txt` 는 디스크로부터 수신될 때 즉시 한번에 한 덩어리(chunk)씩 클라이언트들에게 기록되어집니다.

`.pipe()`를 사용하는 것은 다른 잇점도 있습니다. 이를테면, node가 클라이언트가 정말 느리거나 연결 지연이 클 때 배압(backpressure)을 자동으로 핸들링 해서 덩어리들을 불필요하게 버퍼하지 않도록 합니다.

압축을 원하나요? 그를 위한 스트리밍 모듈도 있습니다!

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

이제 gzip이나 deflate를 지원하는 브라우저에 대해서는 우리의 파일이 압축될겁니다! 우리는 그저 [oppressor](https://github.com/substack/oppressor)가 모든 content-encoding 들을 다루도록 둘 수 있게 되었습니다.

당신이 stream api를 배우고 나면, 스트리밍을 지원하지 않는 커스텀 API들을 통해 데이터를 어떻게 푸시할지 기억해야 하는 대신, 이러한 스트리밍 모듈들을 레고 블럭이나 정원의 호스들 처럼 한데 물릴 수 있습니다.

stream은 node로 프로그래밍 하는것을 간단하고 우아하고 조립 가능하게(composable) 만들어 줍니다.

# 기본

5가지 stream 들이 있습니다: readable, writable, transform, duplex, 그리고 "classic".

## pipe

여러 종류의 스트림들이 모두 입력을 출력과 짝짓는데 `.pipe()`를 사용합니다.

`.pipe()` 는 단순히 readable 소스 streams인 `src` 를 받아, 그 출력을 쓰기 가능한 목적지 스트림 `dst`에 연결하는 함수입니다:

```
src.pipe(dst)
```

`.pipe(dst)` 는 `dst`를 반환해서 여러 `.pipe()` 호출들을 한데 묶을 수 있도록 합니다:

``` js
a.pipe(b).pipe(c).pipe(d)
```
는 아래와 같습니다:

``` js
a.pipe(b);
b.pipe(c);
c.pipe(d);
```

이것은 당신이 커맨드라인에서 프로그램들을 pipe 하던것과 매우 유사합니다:

```
a | b | c | d
```

shell 대신에 node에서 한다는것만 빼고요!

## readable 스트림

Readable 스트림은 데이터를 생산하고, `.pipe()`를 호출함으로서 그 데이터는 writable, transform 또는 duplex 스트림에 의해 소비되어집니다(fed):

``` js
readableStream.pipe(dst)
```

### readable 스트림 생성

readable 스트림을 만들어봅시다!

``` js
var Readable = require('stream').Readable;

var rs = new Readable;
rs.push('beep ');
rs.push('boop\n');
rs.push(null);

rs.pipe(process.stdout);
```

```
$ node read0.js
beep boop
```

`rs.push(null)` 은 소비자에게 `rs`가 데이터 출력을 마쳤다고 알려줍니다.

여기서 `process.stdout`에 piping하기 전에 readable 스트림인 `rs`에 컨텐트를 푸시했지만 여전히 전체 메시지가 출력된것을 보세요.

이것은 readable 스트림에 `.push()` 할 때, 푸시한 chunk들이 그것들을 consumer가 읽을 준비가 될 때 까지 버퍼되기 때문입니다.

하지만, 만약 데이터들을 모두 버퍼링하는것을 피하고 consumer가 그것을 요청할 때만 데이터를 생성할 수 있다면 여러가지 상황에서 더 좋을것입니다.


`._read` 함수를 정의함으로서 chunk를 요청이 있을때만 푸시할 수 있습니다:

``` js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97;
rs._read = function () {
    rs.push(String.fromCharCode(c++));
    if (c > 'z'.charCodeAt(0)) rs.push(null);
};

rs.pipe(process.stdout);
```

```
$ node read1.js
abcdefghijklmnopqrstuvwxyz
```

여기서 우리는 `'a'` 에서 `'z'` 까지의 문자들을 consumer가 그것들을 읽을 준비가 되었을때만 푸시합니다.

`_read` 함수는 또한 잠정적인 `size` 파라미터를 그 첫번째 인자로 받아들이고, 이것은 consumer가 몇 byte의 데이터를 읽고자 하는지를 지정합니다. 하지만 당신의 readable 스트림은 원한다면 `size`를 무시할 수 있습니다.

readable 스트림을 상속받기 위해서 `util.inherits()` 를 사용할 수도 있음에 유의하세요. 하지만 그러한 접근은 이해하기 쉬운 예제에는 잘 어울리지 않습니다.

우리의 `_read` 함수가 consumer가 요청했을 때만 호출됨을 보이기 위해, 우리의 readable 스트림 코드에 약간의 지연을 추가하도록 수정해보겠습니다:

```js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97 - 1;

rs._read = function () {
    if (c >= 'z'.charCodeAt(0)) return rs.push(null);
    
    setTimeout(function () {
        rs.push(String.fromCharCode(++c));
    }, 100);
};

rs.pipe(process.stdout);

process.on('exit', function () {
    console.error('\n_read() called ' + (c - 97) + ' times');
});
process.stdout.on('error', process.exit);
```

이 프로그램을 실행하면, 5바이트의 출력만 요청하면 `_read()` 가 5번밖에 호출되지 않음을 볼 수 있습니다:

```
$ node read2.js | head -c5
abcde
_read() called 5 times
```

setTimeout는 운영 체제가 관련한 신호를 보내서 파이프를 닫을 때까지 시간을 요하기 때문에 필요합니다.

또한, `process.stdout.on('error', fn)` 핸들러는 `head`가 더이상 우리 프로그램의 출력에 관심이 없을때 운영 체제가 SIGPIPE 를 우리 프로세스로 보내면 EPIPE 에러가 `process.stdout`에서 발생될 것이기 때문에 필요합니다.

외부의 운영 체제 파이프들과의 인터페이스를 위해서는 이러한 추가적인 복잡한것들이 필요하지만, 항상 node 스트림들과만 직접 인터페이스할때는 자동으로 이루어집니다.

문자열이나 버퍼 대신 임의의 값을 푸시하는 readable 스트림을 만들고 싶다면, readable 스트림 을 생성할 때 `Readable({ objectMode: true })` 와 같이 생성해야 함에 유의하세요.


### readable 스트림 으로부터 읽기(consuming)

대부분의 경우, readable 스트림을 다른 종류의 스트림이나 [through](https://npmjs.org/package/through)
또는 [concat-stream](https://npmjs.org/package/concat-stream) 과 같은 모듈로 생성한 스트림에 pipe하는것이 훨씬 쉽지만, 가끔은 readable 스트림을 직접 consume하는것이 유용할 수 있습니다.

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    console.dir(buf);
});
```

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js 
<Buffer 61 62 63 0a>
<Buffer 64 65 66 0a>
<Buffer 67 68 69 0a>
null
```

데이터가 사용 가능할 때, `'readable'` 이벤트가 발생되면 `.read()`를 호출하여 버퍼로부터 데이터를 읽을 수 있습니다.

스트림이 종료되면 가져올 byte가 더이상 없기 때문에 `.read()`는 null을 리턴합니다.

`n` 바이트의 데이터를 반환하도록 `.read(n)`와 같이 호출할 수도 있습니다. 많은 수의 바이트를 읽는것은 단순히 권고안일 뿐 오브젝트 스트림에서는 동작하지 않지만, 모든 코어 스트림이 그것을 지원합니다.

다음은 `.read(n)` 사용하여 stdin에서 3바이트를 버퍼하는 예 입니다:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
});
```

이 예제를 실행하면 불완전한 데이터가 나옵니다!

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
```

이것은 내부 버퍼에 데이터가 더 남아있고 node에게 "kick"해서 우리가 읽은 3바이트 외에 더 많은 데이터에 관심있음을 알려줘야 하기 때문입니다.
간단히 `.read(0)` 를 호출하면 이것을 수행하게 됩니다:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
    process.stdin.read(0);
});
```

이제 우리의 코드가 예상한대로 3-바이트 chunk로 동작합니다!

``` js
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
<Buffer 68 69 0a>
```

`.read()`가 당신이 원한것보다 더 많은 데이터를 주었을 때 `.unshift()` 를 사용하여 데이터를 다시 돌려놓아서 동일한 읽기 로직이 동작하도록 할 수도 있습니다.

`.unshift()` 를 사용하면 불필요한 버퍼 복사를 막을 수 있습니다. 다음과 같이, newline을 만났을때 split하는 readable 파서를 만들 수 있습니다.

``` js
var offset = 0;

process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    if (!buf) return;
    for (; offset < buf.length; offset++) {
        if (buf[offset] === 0x0a) {
            console.dir(buf.slice(0, offset).toString());
            buf = buf.slice(offset + 1);
            offset = 0;
            process.stdin.unshift(buf);
            return;
        }
    }
    process.stdin.unshift(buf);
});
```

```
$ tail -n +50000 /usr/share/dict/american-english | head -n10 | node lines.js 
'hearties'
'heartiest'
'heartily'
'heartiness'
'heartiness\'s'
'heartland'
'heartland\'s'
'heartlands'
'heartless'
'heartlessly'
```

하지만, 자신만의 라인-파싱 로직을 사용하기 보다는 [split](https://npmjs.org/package/split)과 같은 모듈이 있으니 이를 사용하는것이 좋습니다.

## writable 스트림

writable 스트림은 `.pipe()`의 출발점이 아니라, 대상이 되는 스트림입니다:

``` js
src.pipe(writableStream)
```

### writable 스트림의 생성

단순히 `._write(chunk, enc, next)`함수를 정의하면 readable 스트림을 거기에 pipe할 수 있습니다:

``` js
var Writable = require('stream').Writable;
var ws = Writable();
ws._write = function (chunk, enc, next) {
    console.dir(chunk);
    next();
};

process.stdin.pipe(ws);
```

```
$ (echo beep; sleep 1; echo boop) | node write0.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
```

첫 번째 인자인 `chunk`는 생산자에 의해 write된 데이터입니다.

두 번째 인자인 `enc`는 문자열 인코딩을 나타내는 문자열인데, `opts.decodeString`가 `false`이고 문자열을 write했을때만 사용합니다.

세 번째 인자인 `next(err)`는 데이터의 consumer에게 데이터를 더 write할 수 있다고 알려주는 콜백입니다.
에러 객체인 `err`를 선택적으로 넘길수도 있는데, 이는 스트림 인스턴스에 `'error'` 이벤트를 발생시킵니다.

만약 pipe를 통해 데이터를 공급받고있는 readable 스트림이 문자열들을 write하면, 당신이 `Writable({ decodeStrings: false })`을 이용하여 당신의 writable 스트림을 만들지 않는 한 그 문자열들은 `Buffer`들로 변환 될 것입니다.

만약 pipe를 통해 데이터를 공급받고있는 readable 스트림이 object들을 write하면, `Writable({ objectMode: true })`를 사용하여 당신의 writable 스트림을 생성하세요.

### writable 스트림에 기록하기

writable 스트림에 기록하기 위해서는, 당신이 기록하기를 원하는 `data`로 `.write(data)`를 호출하기만 하면 됩니다!

``` js
process.stdout.write('beep boop\n');
```

목적지인 writable 스트림에 write가 끝났다고 알려주기 위해서는, `.end()`를 호출하세요.
`.end(data)`와 같이 종료 전에 기록할 `data`를 줄 수도 있습니다:

``` js
var fs = require('fs');
var ws = fs.createWriteStream('message.txt');

ws.write('beep ');

setTimeout(function () {
    ws.end('boop\n');
}, 1000);
```

```
$ node writing1.js 
$ cat message.txt
beep boop
```

만약 high water mark와 buffering이 신경쓰인다면, 들어오는 데이터 버퍼 내에 `Writable()`에 넘겨진 `opts.highWaterMark` 옵션보다 더 많은 데이터가 있을 경우 `.write()`는 false를 리턴합니다.

만약 버퍼가 다시 비워지기를 기다리고 싶다면, `'drain'` 이벤트를 listen하세요.

## transform

Transform 스트림은 이중(duplex, 읽기와 쓰기가 둘 다 가능한) 스트림의 특정 형태입니다.
차이점은, Transform 에서는 출력이 입력으로부터 어떤 방법으로 계산되어졌다는 점입니다.
The distinction is that in Transform streams, the output is in some way calculated
from the input.

transform 스트림을 "through streams" 이라고 부르는것을 들어봤을 수도 있겠습니다.

Through 스트림은 입력을 변환하여 출력을 생성하는 단순한 readable/writable 필터입니다.

## duplex (이중)

Duplex 스트림은 읽기/쓰기가 가능하고 스트림의 양쪽 끝이 양방향 상호작용을 하도록 연결되어 마치 전화와 같이 메시지를 주고받습니다.
RPC 교환이 duplex 시스템의 좋은 예 입니다. 아래와 같은 것을 보게된다면:

``` js
a.pipe(b).pipe(a)
```

당신은 아마도 duplex 시스템을 다루고 있을것입니다.

## classic streams

Classic 스트림은 node 0.4 에서 처음 발견된 오래된 인터페이스입니다.

Classic streams are the old interface that first appeared in node 0.4.
아마도 이러한 스타일의 스트림을 오랫동안 접하게 될것이므로 어떻게 작동하는지 알면 좋습니다.
언제든 한 스트림이 `"data"` 리스너가 등록되어 가지고 있다면, `"classic"` 모드로 변경되어 오래된 API에 따라 동작하게 됩니다.

### classic readable 스트림

Classic readable 스트림은, 데이터의 소비자에게 줄 데이터를 가지고 있을때 `"data"` 이벤트를 발생시키고, 소비자에게 줄 데이터의 생성이 끝났을때는 `"end"` 이벤트를 발생시키는 이벤트 emitter일 뿐입니다.

`.pipe()`는 `stream.readable`의 참/거짓을 확인함으로서 어떤 classic 스트림이 읽기 가능한지를 확인합니다.

아래는 `A`부터 `J` 까지를 출력하는 아주 간단한 readable 스트림입니다:

``` js
var Stream = require('stream');
var stream = new Stream;
stream.readable = true;

var c = 64;
var iv = setInterval(function () {
    if (++c >= 75) {
        clearInterval(iv);
        stream.emit('end');
    }
    else stream.emit('data', String.fromCharCode(c));
}, 100);

stream.pipe(process.stdout);
```

```
$ node classic0.js
ABCDEFGHIJ
```

classic readable 스트림으로부터 읽기 위해서는 `"data"` 와 `"end"` 리스너를 등록합니다.
아래는 오래된 readable 스트림 스타일을 사용하여 `process.stdin`으로부터 읽어들이는 예제입니다:

``` js
process.stdin.on('data', function (buf) {
    console.log(buf);
});
process.stdin.on('end', function () {
    console.log('__END__');
});
```

```
$ (echo beep; sleep 1; echo boop) | node classic1.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

`"data"` 리스너를 등록하면 그 스트림을 호환성 모드로 들어가게 하여 streams2 api의 장점들을 잃게 된다는 것에 유의하세요.

더 이상은 `"data"` 와 `"end"` 핸들러를 절대 등록하지 않는것이 좋습니다.
종래의 스트림들과 상호작용할 필요가 있다면, 대신에 가급적이면 `.pipe()` 할 수 있는 라이브러리를 사용하도록 하십시오.

예를 들면, `"data"` 와 `"end"` 리스너를 설정하는것을 막기 위해서 [through](https://npmjs.org/package/through)를 사용할 수 있습니다:

``` js
var through = require('through');
process.stdin.pipe(through(write, end));

function write (buf) {
    console.log(buf);
}
function end () {
    console.log('__END__');
}
```

```
$ (echo beep; sleep 1; echo boop) | node through.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

또는 전체 스트림의 컨텐츠를 버퍼에 담도록 [concat-stream](https://npmjs.org/package/concat-stream)을 사용할 수도 있습니다:

``` js
var concat = require('concat-stream');
process.stdin.pipe(concat(function (body) {
    console.log(JSON.parse(body));
}));
```

```
$ echo '{"beep":"boop"}' | node concat.js 
{ beep: 'boop' }
```

Classic readable 스트림은 일시적으로 스트림을 중지시키기 위한 `.pause()` 와 `.resume()` 로직을 가지고 있지만 이것은 단지 권고사항일 뿐입니다.
만약 classic readable 스트림에서 `.pause()` 와 `.resume()` 를 사용하려면, 직접 기록을 하기보다는 [through](https://npmjs.org/package/through)를 사용하여 버퍼링을 다루는것이 좋습니다.

### classic writable 스트림

Classic writable 은 매우 간단합니다. `.write(buf)`, `.end(buf)` 그리고 `.destroy()` 만 정의하면 됩니다.

`.end(buf)`가 `buf`를 가져갈 수도 가져가지 않을 수도 있지만, node의 사용자들은 `stream.end(buf)`가 `stream.write(buf); stream.end()`를 의미할것으로 기대할것이에, 그들의 예상을 방해하지 않는것이 좋겠습니다.

## 더 읽을거리

* [core stream documentation](http://nodejs.org/docs/latest/api/stream.html#stream_stream)
* [readable-stream](https://npmjs.org/package/readable-stream) 모듈을 사용하여 node 0.8과 그 이하를 준수하는 당신의 streams2 코드를 만들 수 있습니다. `npm install readable-stream` 하고 `require('stream')` 대신에 `require('readable-stream')` 만 사용하면 됩니다.

***

# 내장된 스트림

아래의 스트림들은 node 자체에 내장되어 있습니다.

## process

### [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin)

이 readable 스트림은 당신의 프로그램의 표준 시스템 입력 스트림을 포함합니다.

기본적으로는 중지되어있지만, 처음 그것을 참조하게 되면 [next tick](http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback)에 암묵적으로 `.resume()`이 호출될 것입니다.

만약 process.stdin 이 tty ([`tty.isatty()`](http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd)를 확인하세요) 라면, 입력 이벤트는 line-buffer될 것입니다.
`process.stdin.setRawMode(true)`를 호출함으로서 line-buffering을 끌 수 있지만, `^C` 와 `^D` 같은 키 조합을 처리하는 기본 핸들러도 제거될것입니다.

### [process.stdout](http://nodejs.org/api/process.html#process_process_stdout)

이 writable 스트림은 당신의 프로그램의 표준 시스템 출력 스트림을 포함합니다.

stdout으로 데이터를 보내고 싶다면 그것에 `write` 하세요.

### [process.stderr](http://nodejs.org/api/process.html#process_process_stderr)

이 writable 스트림은 당신의 프로그램의 표준 시스템 에러 스트림을 포함합니다.

stderr로 데이터를 보내고 싶다면 그것에 `write` 하세요.

## child_process.spawn()

## fs

### fs.createReadStream()

### fs.createWriteStream()

## net

### [net.connect()](http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener)

이 함수는 원격 호스트에 tcp로 연결하는 [duplex 스트림]을 반환합니다.

곧바로 그 스트림에 쓰기를 시작할 수 있으며, 그 쓰여진것들은 `'connect'` 이벤트가 발생할 때 까지 버퍼됩니다.

### net.createServer()

## http

### http.request()

### http.createServer()

## zlib

### zlib.createGzip()

### zlib.createGunzip()

### zlib.createDeflate()

### zlib.createInflate()

***

# 스트림의 제어

## [through](https://github.com/dominictarr/through)

## [from](https://github.com/dominictarr/from)

## [pause-stream](https://github.com/dominictarr/pause-stream)

## [concat-stream](https://github.com/maxogden/node-concat-stream)

concat-stream 은 스트림 내용을 단일 버퍼로 버퍼링할 것입니다.
`concat(cb)` 은 단일 콜백인 `cb(body)` 을 받아버퍼링된 `body` 
`concat(cb)` just takes a single callback `cb(body)` with the buffered
`body` when the stream has finished.

예를 들어, 이 프로그램에서, `cs.end()`가 호출되면 concat 콜백은 body 문자열 `"beep boop"`과 함께 발생됩니다.
프로그램은 body 를 받아 대문자로 바꾸고 `BEEP BOOP.`를 출력하게 됩니다.


``` js
var concat = require('concat-stream');

var cs = concat(function (body) {
    console.log(body.toUpperCase());
});
cs.write('beep ');
cs.write('boop.');
cs.end();
```

```
$ node concat.js
BEEP BOOP.
```

아래는 들어오는 url-encoded 폼 데이터를 파싱하고 그 폼 파라미터들의 문자열화된 JSON버전을 답하는 concat-stream 의 사용 예 입니다:

``` js
var http = require('http');
var qs = require('querystring');
var concat = require('concat-stream');

var server = http.createServer(function (req, res) {
    req.pipe(concat(function (body) {
        var params = qs.parse(body.toString());
        res.end(JSON.stringify(params) + '\n');
    }));
});
server.listen(5005);
```

```
$ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
{"beep":"boop","dinosaur":"trex"}
```

## [duplex](https://github.com/dominictarr/duplex)

## [duplexer](https://github.com/Raynos/duplexer)

## [emit-stream](https://github.com/substack/emit-stream)

## [invert-stream](https://github.com/dominictarr/invert-stream)

## [map-stream](https://github.com/dominictarr/map-stream)

## [remote-events](https://github.com/dominictarr/remote-events)

## [buffer-stream](https://github.com/Raynos/buffer-stream)

## [event-stream](https://github.com/dominictarr/event-stream)

## [auth-stream](https://github.com/Raynos/auth-stream)

***

# meta 스트림

## [mux-demux](https://github.com/dominictarr/mux-demux)

## [stream-router](https://github.com/Raynos/stream-router)

## [multi-channel-mdm](https://github.com/Raynos/multi-channel-mdm)

***

# state 스트림

## [crdt](https://github.com/dominictarr/crdt)

## [delta-stream](https://github.com/Raynos/delta-stream)

## [scuttlebutt](https://github.com/dominictarr/scuttlebutt)

[scuttlebutt](https://github.com/dominictarr/scuttlebutt)는, 노드들이 중개자를 통해서만 연결되어질 수 있으며 모든 데이터의 신뢰할 수 있는 버전을 가진 노드가 없는 mesh 토폴로지와의 peer-to-peer 상태 동기화에 사용될 수 있습니다.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt)가 제공하는 분산 peer-to-peer 네트워크 종류는 서로다른측의 네트워크 장벽에 있는 노드들이 같은 상태를 공유하고 업데이트할 필요가 있을 때 특히 유용합니다. 이러한 네트워크 종류의 한 예는, http 서버를 통해 서로에게 메시지를 보내는 브라우저 클라이언트들과 브라우저들이 직접적으로 연결할 수 없는 백엔드 프로세스들입니다.
또 다른 use-case는 IPv4 주소가 부족하여 내부 네트워크들을 연결하는 시스템입니다.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) 는 연결된 노드들간에 메시지를 전달하는데에 [gossip 프로토콜](https://en.wikipedia.org/wiki/Gossip_protocol)을 사용하여 모든 노드들의 상태가 어디서나 같은 값으로 [eventually converge](https://en.wikipedia.org/wiki/Eventual_consistency) 되도록 합니다.

`scuttlebutt/model` 인터페이스를 사용하면, 노드들을 생성하고 그것들을 서로 연결하여 원하는 어떤 형태의 네트워크든 만들 수 있습니다:

``` js
var Model = require('scuttlebutt/model');
var am = new Model;
var as = am.createStream();

var bm = new Model;
var bs = bm.createStream();

var cm = new Model;
var cs = cm.createStream();

var dm = new Model;
var ds = dm.createStream();

var em = new Model;
var es = em.createStream();

as.pipe(bs).pipe(as);
bs.pipe(cs).pipe(bs);
bs.pipe(ds).pipe(bs);
ds.pipe(es).pipe(ds);

em.on('update', function (key, value, source) {
    console.log(key + ' => ' + value + ' from ' + source);
});

am.set('x', 555);
```

우리가 만든 네트워크는 아래와 같은 방향성 없는 그래프입니다:

```
a <-> b <-> c
      ^
      |
      v
      d <-> e
```

노드 `a` 와 `e`는 직접적으로 연결되어있지 않지만, 아래의 스크립트를 실행하였을 때:

```
$ node model.js
x => 555 from 1347857300518
```

`a`가 설정한 값은 `b`와 `d`를 거쳐 `e`로 향하게 됩니다. 여기서는 모든 노드들이 같은 프로세스에 있지만, [scuttlebutt](https://github.com/dominictarr/scuttlebutt)가 간단한 스트리밍 인터페이스를 사용하기 때문에 그 노드들은 어떤 프로세스들나 서버에 위치할 수 있고, 문자열 데이터를 다룰 수 있는 어떤 스트리밍 방식으로든 연결될 수 있습니다.

다음으로는 네트워크를 통해 연결되고 한 카운터 변수를 증가시키는 더 실제적인 예제를 만들어보겠습니다.

여기 초기 `count` 값으로 0을 설정하고 320 밀리초마다 `count ++` 하고 count가 업데이트 될 때마다 출력하는 서버가 있습니다:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
m.set('count', '0');
m.on('update', function (key, value) {
    console.log(key + ' = ' + m.get('count'));
});

var server = net.createServer(function (stream) {
    stream.pipe(m.createStream()).pipe(stream);
});
server.listen(8888);

setInterval(function () {
    m.set('count', Number(m.get('count')) + 1);
}, 320);
```

이제 이 서버에 접속하여 특정 시간마다 count를 업데이트하고 수신하는 모든 업데이트들을 출력하는 클라이언트를 만들 수 있습니다:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
var s = m.createStream();

s.pipe(net.connect(8888, 'localhost')).pipe(s);

m.on('update', function cb (key) {
    // wait until we've gotten at least one count value from the network
    if (key !== 'count') return;
    m.removeListener('update', cb);
    
    setInterval(function () {
        m.set('count', Number(m.get('count')) + 1);
    }, 100);
});

m.on('update', function (key, value) {
    console.log(key + ' = ' + value);
});
```

이 클라이언트는 약간 더 까다로운데 카운터 자신의 업데이트를 시작하기 위해 다른 누군가로부터 업데이트되기를 기다려야 하고 그렇지 않으면 카운터가 0이 되기 때문입니다.

서버와 몇몇 클라이언트가 동작하면 이와 같은 시퀀스를 볼 수 있습니다:

```
count = 183
count = 184
count = 185
count = 186
count = 187
count = 188
count = 189
```

때로는 노드 중 일부에서 다음과 같이 반복되는 값의 시퀀스를 볼 수도 있습니다:

```
count = 147
count = 148
count = 149
count = 149
count = 150
count = 151
```

이러한 값들은 [scuttlebutt](https://github.com/dominictarr/scuttlebutt)의 히스토리 기반 충돌 해결 알고리즘 때문인데, 이 알고리즘은 모든 노드들에 대해 시스템의 상태가 eventually consistent 함을 보장합니다.

이 예제의 서버는 거기에 접속하는 클라이언트들과 같은 권한을 가진 또 다른 노드일 뿐이라는것에 유의하세요. 여기서 쓰인 "클라이언트"와 "서버" 라는 용어는 상태 동기화가 어떻게 이루어지는가에 영향을 미치는것이 아니라, 누가 연결을 초기화 하는가에 대한 것일 뿐입니다. 이러한 속겅을 가진 프로토콜을 대칭 프로토콜이라고도 부릅니다.
대칭 프로토콜의 다른 예제를 보려면 [dnode](https://github.com/substack/dnode)를 보세요.

## [append-only](http://github.com/Raynos/append-only)

***

# http 스트림

## [request](https://github.com/mikeal/request)

## [oppressor](https://github.com/substack/oppressor)

## [response-stream](https://github.com/substack/response-stream)

***

# io 스트림

## [reconnect](https://github.com/dominictarr/reconnect)

## [kv](https://github.com/dominictarr/kv)

## [discovery-network](https://github.com/Raynos/discovery-network)

***

# parser 스트림

## [tar](https://github.com/creationix/node-tar)

## [trumpet](https://github.com/substack/node-trumpet)

## [JSONStream](https://github.com/dominictarr/JSONStream)

스트림으로부터 json 데이터를 파싱하고 문자열화 하려면 이 모듈을 사용하세요.

만약 큰 json 콜렉션을 느린 연결을 통해 보낼 필요가 있거나 천천히 채워질 json 객체가 있다면, 이 모듈은 데이터가 도착할 때 점진적으로 그것을 파싱할 수 있게 해줍니다.

## [json-scrape](https://github.com/substack/json-scrape)

## [stream-serializer](https://github.com/dominictarr/stream-serializer)

***

# browser 스트림

## [shoe](https://github.com/substack/shoe)

## [domnode](https://github.com/maxogden/domnode)

## [sorta](https://github.com/substack/sorta)

## [graph-stream](https://github.com/substack/graph-stream)

## [arrow-keys](https://github.com/Raynos/arrow-keys)

## [attribute](https://github.com/Raynos/attribute)

## [data-bind](https://github.com/travis4all/data-bind)

***

# html 스트림

## [hyperstream](https://github.com/substack/hyperstream)


# audio 스트림

## [baudio](https://github.com/substack/baudio)

# rpc 스트림

## [dnode](https://github.com/substack/dnode)

[dnode](https://github.com/substack/dnode) 는 어떤 종류의 스트림을 통해서든 원격 함수들을 호출하도록 해줍니다.

다음은 기본적인 dnode 서버입니다:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

이제 서버의 `.transform()` 함수를 호출하는 간단한 클라이언트를 만들 수 있습니다:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

서버를 실행하고, 클라이언트를 실행하면 다음을 볼 수 있습니다:

```
$ node client.js
beep => BOOP
```

클라이언트는 서버의 `transform()` 함수로 `'beep'` 을 보냈고, 서버는 클라이언트의 콜백을 그 결과와 함께 호출합니다. 깔끔해!

여기서 dnode가 제공하는 스트리밍 인터페이스는 클라이언트와 서버가 서로 연결될 수 있어서(`c.pipe(d).pipe(c)`) 양쪽에서 요청하고 응답하는 이중(duplex) 스트림입니다.

dnode의 난리(craziness)는 함수 인자를 조각난(stubbed) 콜백으로 전달하기 시작할때 시작됩니다. 
다음은 이전 서버를 다중 콜백을 전달하도록 업데이트한 버전입니다:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(function (n, fn) {
                var oo = Array(n+1).join('o');
                fn(s.replace(/[aeiou]{2,}/, oo).toUpperCase());
            });
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

다음은 업데이트된 클라이언트입니다:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (cb) {
        cb(10, function (s) {
            console.log('beep:10 => ' + s);
            d.end();
        });
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

서버를 돌리고, 클라이언트를 실행하면 아래의 결과를 얻습니다:

```
$ node client.js
beep:10 => BOOOOOOOOOOP
```

그냥 됩니다!™(It just works!™)

기본 아이디어는 객체에 함수를 넣고 스트림의 다른쪽에서 그것을 호출하고 그 함수는 또 다른쪽에서 stubed out되어 본래의 함수가 원래 있던곳으로 왕복하는 것입니다.
가장 좋은점은, 함수들을 인자로서 스텁된 함수에 전달할 때 그 함수들은 *다른*쪽에 스텁아웃 된다는 점입니다.

함수 인자를 재귀적으로 스터빙 하는 이 방식은 이제부터 "turtles all the way down" 수법으로 알려질겁니다.
어떤힌 함수들로부터의 반환값이든 모두 무시되고 객체들의 열거 가능한 속성들, json 스타일만 보내어질것입니다.

이게 turtles all the way down 이지요!

![turtles all the way](http://substack.net/images/all_the_way_down.png)

dnode가 node내에서나 브라우저상에서 어떠한 스트림을 통해서든 작동하기 때문에, 어디서 정의된 함수든 호출하기가 쉬우며 특히 [mux-demux](https://github.com/dominictarr/mux-demux)와 쌍을 이루어서 어떤 대량의 데이터 스트림과 함께 컨트롤을 위한 rpc스트림을 멀티플렉싱 하는데에 유용합니다.

## [rpc-stream](https://github.com/dominictarr/rpc-stream)

***

# test 스트림

## [tap](https://github.com/isaacs/node-tap)

## [stream-spec](https://github.com/dominictarr/stream-spec)

***

# power combos

## 분산 파티션-허용 채팅

[append-only](http://github.com/Raynos/append-only) 모듈은 [scuttlebutt](https://github.com/dominictarr/scuttlebutt)상에 편리한 덧붙이기만 가능한 배열을 제공해줄 수 있는데, 이것은 다른 노드들과 복제를 수행하고 네트워크 파티션에서 생존할 수 있는 최종적 일관성(eventually-consistent: 일시적으로는 consistant하지 않을 수 있으나 일정 시간 후에는 consistant한)있는 분산 채팅을 아주 쉽게 만들 수 있게 해줍니다.

TODO: the rest

## 당신만의 socket.io 를 굴려라 roll your own socket.io

우리는 socket.io 스타일의 스트림들 사이에서의 이벤트 emitter api를 이 문서의 위에서 언급한 몇 가지 라이브러리들을 사용하여 만들 수 있습니다.

먼저 우리는 [shoe](http://github.com/substack/shoe)를 사용하여 새로운 웹소켓 핸들러 서버를 만들 수 있고,
[emit-stream](https://github.com/substack/emit-stream)을 사용하여 한 스트림을 이벤트 emitter로 바꾸어 객체를 emit하도록 할 수 있습니다.
그 객체 스트림은 그 이후 [JSONStream](https://github.com/dominictarr/JSONStream)로 공급되어 그 객체들을 직렬화(serialize)할 수 있고, 거기로부터 직렬화된 스트림은 원격 브라우저로 파이프될 수도 있습니다.

``` js
var EventEmitter = require('events').EventEmitter;
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var sock = shoe(function (stream) {
    var ev = new EventEmitter;
    emitStream(ev)
        .pipe(JSONStream.stringify())
        .pipe(stream)
    ;
    ...
});
```

위에서의 shoe 콜백 내에서 우리는 `ev` 함수로 이벤트들을 emit할 수 있습니다. 여기서 우리는 단순히 특정 시간마다 다른 종류의 이벤트들을 emit할 것입니다:

``` js
var intervals = [];

intervals.push(setInterval(function () {
    ev.emit('upper', 'abc');
}, 500));

intervals.push(setInterval(function () {
    ev.emit('lower', 'def');
}, 300));

stream.on('end', function () {
    intervals.forEach(clearInterval);
});
```

마지막으로 shoe 인스턴스는 한 http 서버에 바인드 될 필요가 있습니다:

``` js
var http = require('http');
var server = http.createServer(require('ecstatic')(__dirname));
server.listen(8080);

sock.install(server, '/sock');
```

한편 브라우저측에서는 json shoe 스트림을 parse하고 그 결과인 객체를 `eventStream()`으로 전달합니다.
`eventStream()`은 단순히 서버측에서의 이벤트를 emit하는 이벤트 emitter를 반환합니다:

``` js
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var parser = JSONStream.parse([true]);
var stream = parser.pipe(shoe('/sock')).pipe(parser);
var ev = emitStream(stream);

ev.on('lower', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toLowerCase();
    document.body.appendChild(div);
});

ev.on('upper', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toUpperCase();
    document.body.appendChild(div);
});
```

[browserify](https://github.com/substack/node-browserify)를 사용하여 이러한 브라우저 소스코드를 만들어 이러한 모든 멋진 모듈들을 브라우저 측에서 `require()` 할 수 있도록 해보세요:

```
$ browserify main.js -o bundle.js
```

그런 다음, `<script src="/bundle.js"></script>`을 어떤 html안에 넣고 브라우저에서 열면 서버측의 이벤트들이 브라우저측으로 스트림되는것을 확인해보세요.

이러한 스트리밍 접근을 이용하여 스트림들을 이야기하는 방법만 알면 되는 작은 재사용한 컴포넌트들에 더욱 의존할 수 있습니다.
전역 이벤트 시스템 socket.io-style을 통해 메시지들의 경로를 정하는 대신, 당신의 어플리케이션을 더 작은 기능의 단위로 쪼개고 그것들이 하나의 일만 잘 할 수 있도록 하는데에 더욱 집중할 수 있습니다.

예를들어 이 예제에서의 JSONStream을 단순히 [stream-serializer](https://github.com/dominictarr/stream-serializer)로 바꾸면 다른 트레이드오프들을 사용한 다른 직렬화를 수행할 수 있습니다.
간단한 스트리밍 인터페이스를 사용하여 [reconnections](https://github.com/dominictarr/reconnect) 또는 heartbeat를 처리하기 위해 shoe의 꼭대기에 레이어를 볼트로 고정할 수도 있습니다.
코어안에 있는 EventEmitter 대신에 [eventemitter2](https://npmjs.org/package/eventemitter2)를 사용하여 한 스트림을 연쇠에 추가하여 네임스페이스를 사용하는 이벤트들을 사용할 수도 있습니다.

다른 방식으로 동작하는 다른 스트림을 원한다면 이 예제에서의 shoe 스트림을 mux-demux를 통해 실행하여 필요한 각각의 다른 종류의 스트림에 대해 별도의 채널들을 생성하면 간단할 것입니다.

당신의 시스템 요구사양이 점점 발전함에 따라 이러한 스트리밍 조각들 각각을 필요에 따라 "모아니면 도"의 위험 없이 바꿀 수 있으며, 그러한 위험은 더욱 독단적인 프레임웍 접근법에서 필연적으로 수반됩니다.

## 브라우저와 서버를 위한 html 스트림

클라이언트와 서버에서 동일한 html 렌더링 로직을 재사용하기 위해 어떤 스트리밍 모듈들을 사용할 수도 있습니다.
이러한 접근법은 색인 가능하고, SEO(검색엔진 최적화)친화적이며, 실시간 업데이트를 가능하게 합니다.

우리의 렌더러는 몇줄의 json을 입력으로 받고, 출력으로서 html 문자열을 반환합니다. 문자열은 만능 인터페이스이지요!

render.js:

``` js
var through = require('through');
var hyperglue = require('hyperglue');
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/static/row.html', 'utf8');

module.exports = function () {
    return through(function (line) {
        try { var row = JSON.parse(line) }
        catch (err) { return this.emit('error', err) }
        
        this.queue(hyperglue(html, {
            '.who': row.who,
            '.message': row.message
        }).outerHTML);
    });
};
```

[brfs](http://github.com/substack/brfs)를 사용하여 브라우저 코드에서 `fs.readFileSync()` 호출을 인라인 할 수 있고,
[hyperglue](https://github.com/substack/hyperglue)를 사용하여 css 셀렉터 기반으로 html을 업데이트 할 수 있습니다.
여기서 hyperglue 를 반드시 사용할 필요는 없습니다; 그 언에서 html 문자열을 반환할 수 있다면 어떠한 것이든 동작할것입니다.

여기서는 아주 간단히 `row.html`을 만들어 넣었습니다:

row.html:

``` html
<div class="row">
  <div class="who"></div>
  <div class="message"></div>
</div>
```

서버는 그저 단순히 [slice-file](https://github.com/substack/slice-file)을 사용하였고,
[slice-file](https://github.com/substack/slice-file)은 찬사받는 `tail/tail -f` api 보다 약간 낫지만,
그것의 인터페이스는 정규 결과들과 couchdb와 같은 변경 피드를 가진 데이터베이스에 잘 매핑되어 있습니다.

server.js:

``` js
var http = require('http');
var fs = require('fs');
var hyperstream = require('hyperstream');
var ecstatic = require('ecstatic')(__dirname + '/static');

var sliceFile = require('slice-file');
var sf = sliceFile(__dirname + '/data.txt');

var render = require('./render');

var server = http.createServer(function (req, res) {
    if (req.url === '/') {
        var hs = hyperstream({
            '#rows': sf.slice(-5).pipe(render())
        });
        hs.pipe(res);
        fs.createReadStream(__dirname + '/static/index.html').pipe(hs);
    }
    else ecstatic(req, res)
});
server.listen(8000);

var shoe = require('shoe');
var sock = shoe(function (stream) {
    sf.follow(-1,0).pipe(stream);
});
sock.install(server, '/sock');
```

서버의 첫 번째 부분은 `/` 라우트를 핸들링하며 `data.txt`로부터 마지막 5줄을 `#rows` div에 스트림합니다.

서버의 두 번째 부분은 [shoe](http://github.com/substack/shoe)를 이용하여 `#rows`의 업데이트를 실시간으로 핸들링하는 간단한 스트리밍 웹소켓 polyfill 입니다.

다음으로 우리는 [shoe](http://github.com/substack/shoe)로부터 실시간 업데이트를 가져와 `#rows` div에 넣는 간단한 브라우저 코드를 작성할 수 있습니다:

``` js
var through = require('through');
var render = require('./render');

var shoe = require('shoe');
var stream = shoe('/sock');

var rows = document.querySelector('#rows');
stream.pipe(render()).pipe(through(function (html) {
    rows.innerHTML += html;
}));
```

[browserify](http://browserify.org) 와 [brfs](http://github.com/substack/brfs)로 컴파일 하면:

```
$ browserify -t brfs browser.js > static/bundle.js
```

됐습니다! 이제 `data.txt`를 약간의 바보같은 데이터로 채워봅니다:

```
$ echo '{"who":"substack","message":"beep boop."}' >> data.txt
$ echo '{"who":"zoltar","message":"COWER PUNY HUMANS"}' >> data.txt
```

그다음 서버를 돌리고:

```
$ node server.js
```

그 다음 우리 컨텐트를 확인하기 위해 `localhost:8000`로 이동합니다. 약간의 컨텐트를 추가하면:

```
$ echo '{"who":"substack","message":"oh hello."}' >> data.txt
$ echo '{"who":"zoltar","message":"HEAR ME!"}' >> data.txt
```

그러면 페이지가 실시간 업데이트를 이용하여 자동적으로 업데이트 될 것입니다. 만세!

이제 우리는 클라이언트와 서버 모두에서 정확히 동일한 로직을 사용하여 SEO 친화적으로 색인 가능한 실시간 컨텐트를 서비스하게 되었습니다. 만세!
