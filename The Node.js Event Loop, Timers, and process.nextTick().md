# [日本語訳]Node.jsにおけるイベントループ、タイマー、process.nextTick()について -The Node.js Event Loop, Timers, and process.nextTick()-
> *Original Article*: https://nodejs.org/ja/docs/guides/event-loop-timers-and-nexttick/
> 
> This article is translated by [@mox692](https://twitter.com/mox692). Please contact me if any problem.


## Node.jsのイベントループ、タイマー、process.nextTick() イベントループとは？
イベントループは、Node.js が JavaScript がシングルスレッドであるにもかかわらず、可能な限りシステムカーネルに処理をオフロードすることで、ノンブロッキング I/O 操作を実行できるようにするものです。  

最近のカーネルはマルチスレッドなので、バックグラウンドで実行される複数のオペレーションを処理することができます。これらの操作の1つが完了すると、カーネルは Node.js に伝え、適切なコールバックが最終的に実行されるようにポーリングキューに追加されるようにします。  
これについては、このトピックの後半でさらに詳しく説明します。

## イベントループの説明
Node.js が起動すると、イベントループを初期化し、提供された入力スクリプトを処理します（または REPL にドロップします、これはこのドキュメントではカバーされません）。 次の図は、イベントループの処理順序の概要を簡略化して示しています。

<img width="755" alt="スクリーンショット 2022-06-22 21 37 33" src="https://user-images.githubusercontent.com/55653825/175030699-5346a185-ddb4-4933-8470-721477527326.png">   

(各ボックスは、イベントループの「フェーズ」と呼ぶことにします。)

各フェーズには、実行するコールバックのFIFOキューがあります。各フェーズはそれ自体で特別ですが、一般に、イベントループが所定のフェーズに入ると、そのフェーズに固有のあらゆるオペレーションを実行し、その後、キューを使い切るかコールバックの最大数が実行されるまでそのフェーズのキューでコールバックを実行します。  
これらの操作のいずれかがより多くの操作をスケジュールする可能性があり、pollフェーズで処理される新しいイベントはカーネルによってキューに入れられるため、ポーリングイベントが処理されている間にキューに入れられることがあります。  
その結果、長く実行されるコールバックによって、ポーリングフェーズはタイマーの閾値よりもはるかに長く実行されるようになります。詳細はこの記事のタイマとポールのセクションを参照してください。  
(WindowsとUnix/Linuxの実装には若干の不一致がありますが、この記事では重要ではありません。)


## フェーズの概要
* timers: このフェーズでは、setTimeout()およびsetInterval()によってスケジュールされたコールバックを実行します。
* pending callbacks: 次のループ反復に延期されたI/Oコールバックを実行します。
* idle、prepare：内部でのみ使用されます。
* ポーリング：新しいI/Oイベントを取得し、I/O関連のコールバック（クローズコールバック、タイマーでスケジュールされたもの、setImmediate()を除くほとんどすべて）を実行します。
* check: setImmediate()コールバックはここで呼び出される。
* close callbacks: socket.on('close', ...) のようなcloseコールバック。

イベントループの各実行間で、Node.js は非同期 I/O やタイマーを待っているかどうかをチェックし、何もなければキレイにシャットダウンしています。

## 各フェーズの詳細

### タイマー
タイマーは、提供されたコールバックが実行される正確な時間ではなく、その後に実行される可能性のある閾値を指定します。タイマーのコールバックは、指定された時間が経過した後、スケジュール可能な限り早く実行されますが、オペレーティングシステムのスケジューリングや他のコールバックの実行により遅れることがあります。

技術的には、ポーリング・フェーズはタイマーを実行するタイミングを制御します。

例えば、100msの閾値の後に実行するようにタイムアウトをスケジュールした場合、スクリプトは95msかかるファイルの読み取りを非同期で開始します。

```javascript
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);

// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});

```

イベントループがポーリングフェーズに入ると、キューは空っぽ（fs.readFile()が完了していない）なので、最早タイマーの閾値に達するまで残りms数だけ待つことになります。95ms経過を待っている間に、fs.readFile()はファイルの読み込みを終了し、10msかかるそのコールバックをポーリングキューに追加して実行します。コールバックが終了すると、キューにはもうコールバックは存在しないので、イベントループは最も早いタイマーの閾値に到達したことを確認し、タイマーのコールバックを実行するためにタイマーフェーズに折り返されます。この例では、タイマーがスケジュールされてからそのコールバックが実行されるまでの合計遅延が105msであることがわかります。

(ポーリングフェーズがイベントループを飢えさせないように、libuv（Node.jsのイベントループとプラットフォームのすべての非同期動作を実装するCライブラリ）には、さらなるイベントのためにポーリングを停止する前にハード上限（システム依存）があります。)

### pending callbacks
このフェーズは、TCPエラー通知など、いくつかのシステム操作のコールバックを実行する。  
例えば、TCPソケットが接続しようとしたときにECONNREFUSEDを受け取った場合、 *nixシステムの中には、そのエラーを報告するのを待ちたいものがあります。これは、保留中のコールバック・フェーズで実行するためにキューに入れられる。



### poll
pollフェーズには2つの主要な機能があります。

1. ブロックしてI/Oのためにポーリングすべき時間を計算する。
2. ポーリングキューにあるイベントを処理すること。  

イベントループがポーリングフェーズに入り、タイマがスケジュールされていない場合、2つのうちの1つが起こります。

* ポーリングキューが空でない場合、イベントループはコールバックのキューを繰り返し、キューを使い切るか、システム依存のハードリミットに達するまで同期的に実行されます。
* ポーリングキューが空の場合、さらに2つのうちの1つが起こります。
  * スクリプトが setImmediate() によってスケジュールされている場合、イベントループは poll フェーズを終了し、check フェーズに移行してスケジュールされたスクリプトを実行します。
  * スクリプトが setImmediate() によってスケジュールされていない場合、イベントループはコールバックがキューに追加されるのを待ち、その後すぐにそれらを実行します。

ポーリングキューが空になると、イベントループは時間閾値に到達したタイマーをチェックします。1つ以上のタイマが準備できた場合、イベントループはタイマフェーズに戻り、それらのタイマのコールバックを実行します。

### check
このフェーズでは、poll フェーズが完了した直後にコールバックを実行することができます。poll フェーズがアイドル状態になり、スクリプトが setImmediate() でキューに入れられた場合、イベントループは待たずにpollフェーズから check フェーズに移行することができます。  
setImmediate() は、実際にはイベントループの別のフェーズで実行される特別なタイマーです。これは libuv API を使用し、poll フェーズが完了した後に実行されるコールバックをスケジュールします。  
一般に、コードが実行されると、イベントループは最終的にポーリングフェーズに到達し、そこで着信接続やリクエストなどを待ちます。しかし、コールバックが setImmediate() でスケジュールされ、ポーリング フェーズがアイドルになると、ポーリング イベントを待つのではなく、終了してチェック フェーズに移行します。

### close callbacks
ソケットやハンドルが突然閉じられた場合(例： socket.destroy() )、このフェーズで 'close' イベントが発行されます。そうでない場合は、process.nextTick()を経由して発行されます。


## setImmediate vs setTimeout
setImmediate() と setTimeout() は似ていますが、呼び出されるタイミングによって動作が異なります。

setImmediate() は、現在のポーリング フェーズが完了した時点でスクリプトを実行するように設計されています。
setTimeout() は、最小の閾値 (ms) が経過した後にスクリプトが実行されるようスケジュールします。
タイマーを実行する順序は、それらが呼び出されたコンテキストによって異なります。両方がメインモジュール内から呼び出された場合、タイミングはプロセスの性能（マシン上で実行されている他のアプリケーションの影響を受ける可能性がある）に拘束される。

例えば、I/Oサイクル内ではない箇所で（つまり、メインモジュールで）次のスクリプトを実行すると、2つのタイマーの実行順序は、プロセスの性能に拘束されるため、非決定的である。


```javascript
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```bash
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

しかし、I/Oサイクル内で2つの呼び出しを移動させた場合、即時コールバックが常に先に実行される。

```javascript
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});

```

```bash
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

setTimeout() と比較して setImmediate() を使用する主な利点は、タイマーの数に関係なく、I/O サイクル内でスケジュールされた場合、 setImmediate() が常にどのタイマーよりも前に実行されることです。

## process.nextTick()
### process.nextTick()を理解する
process.nextTick()が非同期APIの一部であるにもかかわらず、図に表示されていないことにお気づきでしょうか。  
これは、process.nextTick()が技術的にはイベントループの一部ではないことに起因します。その代わり、イベントループの現在のフェーズに関係なく、現在のオペレーションが完了した後にnextTickQueueが処理されます。ここで、「操作」とは、基礎となる C/C++ ハンドラからの移行と、実行が必要な JavaScript の処理として定義されています。

前述の図を振り返ると、あるフェーズで process.nextTick() を呼び出すと、イベントループが継続する前に process.nextTick() に渡されたすべてのコールバックが解決されることになります。これは、再帰的にprocess.nextTick()を呼び出すことでI/Oを「飢え」させ、イベントループがpollフェーズに到達するのを妨げるため、いくつかの悪い状況を生み出す可能性があります。

### どうしてこの機能が必要なのか？
なぜこのようなものがNode.jsに含まれるのでしょうか？その理由の一つは、APIは必要でない場合でも常に非同期であるべきだという設計思想です。例えば、このコードを見てください。

```javascript
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(
      callback,
      new TypeError('argument should be string')
    );
}
```
このスニペットは引数チェックを行い、もし正しくない場合はエラーをコールバックに渡します。APIが最近更新され、process.nextTick()に引数を渡せるようになりました。これにより、コールバックの後に渡された引数がコールバックの引数として伝搬されるので、関数をネストする必要がありません。

私たちが行っているのは、ユーザーにエラーを返すことですが、それはユーザーのコードの残りを実行できるようにした後でのみ行われます。process.nextTick()を使うことで、apiCall()のコールバックが常にユーザーの残りのコードの後で、かつイベントループの(次のフェーズへの)進行の前に実行されることを保証しています。これを実現するために、JSコールスタックは巻き戻され、提供されたコールバックを直ちに実行することができ、v8からの「RangeError:最大コールスタックサイズを超えました。」というErrorに達することなくprocess.nextTick()への再帰的な呼び出しを行うことができます。


この考え方は、いくつかの潜在的に問題のある状況につながる可能性があります。例えば、次のスニペットを見てみましょう。

```javascript
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) {
  callback();
}

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall hasn't completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```
ユーザーはsomeAsyncApiCall()を非同期シグネチャを持つように定義しているが、実際には同期的に動作する。呼び出されたとき、someAsyncApiCall()に提供されたコールバックは、イベントループの同じフェーズで呼び出されます。その結果、スクリプトが完了するまで実行できていないため、コールバックは、まだその変数をスコープに持っていないかもしれないにもかかわらず、barを参照しようとする。

コールバックを process.nextTick() に配置すると、スクリプトは完了まで実行され、コールバックが呼び出される前にすべての変数や関数などが初期化されます。また、イベントループを継続させないという利点もあります。イベントループを継続させる前に、ユーザーにエラーを知らせることができるのは便利です。以下は、process.nextTick()を使った先ほどの例です。

```javascript
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;

```
下記が実際の例です


```javascript
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

ポートだけが渡された場合、そのポートは直ちにバインドされる。したがって、'listening' コールバックはすぐに呼び出すことができる。問題は、その時点では .on('listening') コールバックが設定されていないことである。

これを回避するために、'listening' イベントを nextTick() でキューに入れ、スクリプトが完了するまで実行されるようにする。これにより、ユーザは好きなイベントハンドラを設定することができます。

## process.nextTick() vs setImmediate()
ユーザーに関する限り似たような2つのコールがありますが、その名前は紛らわしいです。

* process.nextTick()は、同じフェーズですぐに実行されます。
* setImmediate() は、イベントループの次の反復または「tick」時に発生します。
process.nextTick() は setImmediate() よりも即座に起動しますが、これは過去の遺物であり、変更されることはないでしょう。この切り替えを行うと、npm上のパッケージの大部分が壊れることになります。毎日、新しいモジュールが追加されているので、毎日待つごとに、より多くの潜在的な破損が発生することを意味します。紛らわしいとはいえ、名前そのものは変わりません。

開発者には、どんな場合でも setImmediate() を使うことをおすすめします。そのほうが理屈が通りやすいからです。

## どうして process.nextTick() を使用するか
主な理由は2つあります。

1. ユーザーがエラーを処理したり、不要になったリソースをクリーンアップしたり、 あるいはイベントループが継続する前にリクエストを再試行したりできるように するためです。
2. 時には、コールスタックが巻き戻された後、イベントループが継続する前に、 コールバックを実行させることが必要な場合がある。

その一例として、ユーザーの期待に応えることが挙げられます。簡単な例です。

```javascript
const server = net.createServer();
server.on('connection', (conn) => {});

server.listen(8080);
server.on('listening', () => {});
```

listen() はイベントループの先頭で実行されますが、 listen コールバックは setImmediate() の中に置かれるとします。ホスト名が渡されない限り、ポートへのバインディングはすぐに行われます。イベントループを続行するには、poll フェーズに入る必要があります。これは、listen イベントの前に connection イベントが発生するように、接続を受信する可能性がゼロではないことを意味します。

もう一つの例は、EventEmitterを継承し、コンストラクタの中でイベントを発行するものです。

```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();
    this.emit('event');
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

コンストラクタからすぐにイベントを発生させることはできません。なぜなら、ユーザーがそのイベントにコールバックを割り当てるまでスクリプトが処理されていないためです。そこで、コンストラクタ内で process.nextTick() を使用してコールバックを設定し、コンストラクタが終了した後にイベントを発生させると、期待通りの結果を得ることができます。

```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();

    // use nextTick to emit the event once a handler is assigned
    process.nextTick(() => {
      this.emit('event');
    });
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```
