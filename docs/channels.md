<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/ChannelsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class ChannelsGuideTest {
--> 
**目次**

<!--- TOC -->

* [チャネル](#チャネル)
  * [チャネルの基礎](#チャネルの基礎)
  * [チャネルのクローズと反復](#チャネルのクローズと反復)
  * [チャネルプロデューサーの作成](#チャネルプロデューサーの作成)
  * [パイプライン](#パイプライン)
  * [パイプラインによる素数](#パイプラインによる素数)
  * [ファンアウト](#ファンアウト)
  * [ファンイン](#ファンイン)
  * [バッファーされたチャネル](#バッファーされたチャネル)
  * [チャネルは公正](#チャネルは公正)
  * [ティッカーチャネル](#ティッカーチャネル)

<!--- END_TOC -->

## チャネル

遅延値は、コルーチン間で単一の値を転送する便利な方法を提供します。
チャネルは、ストリーム値を転送する方法を提供します。

### チャネルの基礎

[Channel]は概念的には `BlockingQueue` に非常によく似ています。主な違いの1つは、ブロックする `put` 操作の代わりにサスペンド[send][SendChannel.send]、ブロックする `take` 操作の代わりにサスペンド[receive][ReceiveChannel.receive]を持っていることです。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        // これはCPU使用量が多い計算や非同期ロジックかもしれないが、ここではただ5つの平方を送るだけ
        for (x in 1..5) channel.send(x * x)
    }
    // ここで受け取った5つの整数をプリントする
    repeat(5) { println(channel.receive()) }
    println("Done!")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-01.kt)で完全なコードを取得できます

このコードの出力は以下の通りです。

```text
1
4
9
16
25
Done!
```

<!--- TEST -->

### チャネルのクローズと反復

キューとは異なり、チャネルは閉じることによってそれ以上エレメントが来ないことを示すことができます。
受信側では、通常の `for` ループを使用してチャネルから要素を受け取ると便利です。

概念的には、[close][SendChannel.close]は特別なクローズトークンをチャネルに送信するようなものです。
このクローズトークンが受信されるとすぐに反復が停止し、クローズする前に以前に送信されたすべての要素が受信されるという保証があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // 送信完了
    }
    // ここでは `for` ループを使って受け取った値をプリントします（チャネルが閉じられるまで）
    for (y in channel) println(y)
    println("Done!")
//sampleEnd
}
```

</div>

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-channel-02.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

### チャネルプロデューサーの作成

コルーチンが要素のシーケンスを生成するパターンはかなり一般的です。
これは _プロデューサー - コンシューマー_ パターンの一部であり、コンカレントコードでよく見られます。
そのようなプロデューサーを、パラメーターとしてchannelをとる関数に抽象化することはできますが、結果は関数から返さなければならないという常識とは逆になります。

プロデューサー側で簡単に実行できる[produce]という便利なコルーチンビルダーと、コンシューマー側の `for` ループを置き換える拡張関数[consumeEach]があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
//sampleStart
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-03.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

### パイプライン

パイプラインは、1つのコルーチンが無限の値のストリームを生成しているパターンです。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // 1から始まる整数の無限ストリーム
}
```

</div>

また、別のコルーチンがそのストリームを消費し、処理を行い、他の結果を生成しています。
以下の例では、数値は二乗されています。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

</div>

メインコードはパイプライン全体を開始し接続します。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val numbers = produceNumbers() // 1から始まる整数を生成する
    val squares = square(numbers) // 整数を平方にする
    for (i in 1..5) println(squares.receive()) // 最初の5つをプリントする
    println("Done!") // 完了
    coroutineContext.cancelChildren() // 子コルーチンをキャンセルする
//sampleEnd
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-04.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

> コルーチンを作成するすべての関数は、[CoroutineScope]の拡張として定義されているため、アプリケーションでグローバルなコルーチンが残っていないことを確認するために[構造化並行性](composing-suspending-functions.md#asyncでの構造化並行性)に頼ることができます。

### パイプラインによる素数

コルーチンのパイプラインを使って素数を生成する例で、パイプラインを徹底的に使ってみましょう。 無限の数列から始めます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>
 
```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // startからの無限の整数ストリーム
}
```

</div>

次のパイプラインステージでは、入力数列をフィルタリングして、指定された素数で割り切れるすべての数値を削除します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

</div>

次に、2から数値のストリームを開始し、現在のチャネルから素数を取得し、見つかった素数ごとに新しいパイプラインステージを起動して、パイプラインを構築します。
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
``` 
 
次の例では最初の10個の素数をプリントし、パイプライン全体をメインスレッドのコンテキストで実行します。
すべてのコルーチンはメインの[runBlocking]コルーチンの範囲で起動されるため、開始したすべてのコルーチンの明示的なリストを保持する必要はありません。
最初の10個の素数をプリントした後、[cancelChildren][kotlin.coroutines.CoroutineContext.cancelChildren]拡張関数を使用してすべての子コルーチンを取り消します。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    var cur = numbersFrom(2)
    for (i in 1..10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // すべての子をキャンセルしてメインを終わる
//sampleEnd    
}

fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-05.kt)で完全なコードを取得できます

このコードの出力です。

```text
2
3
5
7
11
13
17
19
23
29
```

<!--- TEST -->

標準ライブラリの [`buildIterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/iterator.html) コルーチンビルダーを使って同じパイプラインを構築できることに留意してください。
`produce` を `buildIterator`、 `send` を `yield`、 `receive` を `next`、 `ReceiveChannel` を `Iterator` で置き換え、コルーチンスコープを取り除きます。 `runBlocking` も必要ありません。
ただし、上記のようなチャネルを使用するパイプラインの利点は、[Dispatchers.Default]コンテキストで実行すると実際に複数のCPUコアを使用できることです。

どのみち、これは素数を見つけるには非常に非実用的な方法です。 
実際にはパイプラインは（リモートサービスへの非同期呼び出しのような）いくつかの他のサスペンド呼び出しを必要とします。これらのパイプラインは `sequence`/`iterator` を使用して構築することはできません 。なぜなら完全に非同期の `produce` とは異なり任意の中断を許さないためです。
 
### ファンアウト

複数のコルーチンが同じチャネルから受信し、それらの間で作業を分散することがあります。
定期的に整数（毎秒10個の数値）を生成するプロデューサーコルーチンから始めましょう。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // 1から始める
    while (true) {
        send(x++) // 次を生成
        delay(100) // 0.1秒待つ
    }
}
```

</div>

いくつかのプロセッサコルーチンを持つことができます。この例では、IDと受け取った数値をプリントします。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

今度は5つのプロセッサを起動して、それらを約1秒間動作させましょう。何が起こるか確かめてください。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // プロデューサーのコルーチンを取り消し、すべてを殺す
//sampleEnd
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-06.kt)で完全なコードを取得できます

プロセッサIDとして受け取るそれぞれの固有の整数は異なる可能性がありますが、出力は次のようになります。

```
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

<!--- TEST lines.size == 10 && lines.withIndex().all { (i, line) -> line.startsWith("Processor #") && line.endsWith(" received ${i + 1}") } -->

プロデューサーコルーチンをキャンセルするとそのチャネルが閉じられるため、最終的にはプロセッサコルーチンが実行しているチャネルでの反復が終了することに注意してください。

また、 `launchProcessor` コードでファンアウトを実行するために `for` ループを使って明示的にチャネルを反復する方法にも注意してください。
`consumeEach` とは異なり、この `for` ループパターンは、複数のコルーチンから完全に安全に使用できます。
プロセッサコルーチンの1つが失敗した場合、他のものは依然としてチャネルを処理しているのに対して、 `consumeEach` によって書かれたプロセッサは、その正常または異常終了時に常に下位のチャネルを消費（キャンセル）します。

### ファンイン

複数のコルーチンが同じチャネルに送信することがあります。
例えば、文字列のチャネルと、指定された文字列を指定された遅延でこのチャネルに繰り返し送信するサスペンド関数を持っています。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

</div>

次に、文字列を送信するコルーチンをいくつか起動した場合に何が起こるかを見てみましょう（この例では、メインコルーチンの子としてメインスレッドのコンテキストで起動します）。
</div>

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // 最初の6個を受け取る
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // すべての子をキャンセルしてメインを終わる
//sampleEnd
}

suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-07.kt)で完全なコードを取得できます

出力は、

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### バッファーされたチャネル

今までに示されたチャネルにはバッファーがありませんでした。 バッファーされていないチャネルは、送信側と受信側がお互いに出会ったときに要素を転送します（別名ランデブー）。 sendが最初に呼び出された場合、receiveが呼び出されるまで中断されます。receiveが最初に呼び出された場合、sendが呼び出されるまで中断されます。

[Channel()]ファクトリ関数と[produce]ビルダーは、_バッファーサイズ_ を指定するためのオプションの `capacity` パラメーターをとります。 バッファーは、指定された容量を持つ `BlockingQueue` と同様に、送信側が中断する前に複数の要素を送信できるようにします。これはバッファーがいっぱいになるとブロックします。

次のコードの動作を見てみましょう。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val channel = Channel<Int>(4) // バッファーされたチャネルを作る
    val sender = launch { // 送信側コルーチンを起動
        repeat(10) {
            println("Sending $it") // 各要素を送信する前にプリント
            channel.send(it) // バッファーがいっぱいになったら中断する
        }
    }
    // 何も受け取らずに待つ...
    delay(1000)
    sender.cancel() // 送信者コルーチンをキャンセルする
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-08.kt)で完全なコードを取得できます

これは容量 _4_ のバッファーされたチャネルを使って _5_ 回 "sending" を表示します。

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

最初の4つの要素はバッファーに追加され、5番目の要素を送信しようとすると送信側は中断します。


### チャネルは公正

チャネルへの操作の送信と受信は、複数のコルーチンからの呼び出しの順番に関して _公正_ です。 
それらはファーストイン・ファーストアウトの順序で提供されます。例えば `receive` を呼び出す最初のコルーチンは要素を取得します。
次の例では、2つのコルーチン「ping」と「pong」が共有「table」チャネルから「ball」オブジェクトを受け取ります。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

//sampleStart
data class Ball(var hits: Int)

fun main() = runBlocking {
    val table = Channel<Ball>() // 共有テーブル
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // ボールを供給する
    delay(1000) // 1秒遅らせる
    coroutineContext.cancelChildren() // ゲームオーバー。これらをキャンセルする
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // ループでボールを受け取る
        ball.hits++
        println("$name $ball")
        delay(300) // 少し待つ
        table.send(ball) // ボールを戻す
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-09.kt)で完全なコードを取得できます

「ping」コルーチンが最初に開始されるので、ボールを受け取るのは最初のコルーチンです。 「ping」コルーチンは、ボールをテーブルに戻した後すぐに再びボールを受け取るようになっていますが、ボールは既に受信を待っていた「pong」コルーチンによって受信さます。

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

<!--- TEST -->

チャネルによっては、使用されているエグゼキューターの性質上不公平に見える実行が生成されることがあります。
詳細については、[この問題](https://github.com/Kotlin/kotlinx.coroutines/issues/111)を参照してください。

### ティッカーチャネル

ティッカーチャネルは、このチャネルからの最後の消費から遅延が与えられるたびに `Unit` を生成する特別なランデブーチャネルです。
単独では役に立たないように見えるかもしれませんが、ウィンドウ処理やその他時間に依存した処理を行う複雑な時間ベースの[produce]パイプラインと演算子を作成するのに便利な構成要素です。
ティッカーチャンネルは[select]で 「on tick」アクションを実行するために使用できます。

このようなチャンネルを作成するには、ファクトリメソッド[ticker]を使用します。
これ以上要素が必要ないことを示すには、[ReceiveChannel.cancel]メソッドを使用します。

実際にどのように動作するかを見てみましょう。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val tickerChannel = ticker(delay = 100, initialDelay = 0) // ティッカーチャネルを作る
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // 初期の遅延時間はまだ経過していない

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() } // すべての後続要素は100ms遅延する
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    // 大きな消費の遅延をエミュレートする
    println("Consumer pauses for 150ms")
    delay(150)
    // 次の要素はすぐに利用可能
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // `receive` 呼び出しの間の休止が考慮され、次の要素がより速く到着することに注意
    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() } 
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel() // これ以上要素が必要でないことを示す
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-channel-10.kt)で完全なコードを取得できます

これは次の以下のようにプリントします。

```text
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```

<!--- TEST -->

[ticker]はコンシューマーの一時停止を認識しており、一時停止が発生した場合は生成される要素の固定レートを維持しようとし、デフォルトでは次回に生成される要素の遅延を調整します。
 
オプションとして、 `mode` パラメーターに[TickerMode.FIXED_DELAY]を指定して、要素間の遅延時間を固定することができます。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[kotlin.coroutines.CoroutineContext.cancelChildren]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/kotlin.coroutines.-coroutine-context/cancel-children.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/close.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[Channel()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel.html
[ticker]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/ticker.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[TickerMode.FIXED_DELAY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-ticker-mode/-f-i-x-e-d_-d-e-l-a-y.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
