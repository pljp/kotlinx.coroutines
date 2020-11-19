<!--- TEST_NAME SelectGuideTest -->

**目次**

<!--- TOC -->

* [セレクト式（実験的）](#セレクト式セレクト式（実験的）)
  * [チャネルからの選択](#チャネルからの選択)
  * [クローズ時の選択](#クローズ時の選択)
  * [送信の選択](#送信の選択)
  * [延期された値の選択](#延期された値の選択)
  * [延期された値のチャネルの切り替え](#延期された値のチャネルの切り替え)

<!--- END -->

## セレクト式（実験的）

セレクト式を使用すると複数のサスペンド関数を同時に待つことができ、利用可能になった最初のものを _選択_ することができます。

> セレクト式は `kotlinx.coroutines` の実験的な機能です。 これらのAPIは、 `kotlinx.coroutines` ライブラリの今後のアップデートで、潜在的に大きな変化を伴って進化することが期待されています。

### チャネルからの選択

2つの文字列のプロデューサー、 `fizz` と `buzz` があります。 `fizz` は300ミリ秒ごとに"Fizz"文字列を生成します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // 300ミリ秒ごとに "Fizz" を送る
        delay(300)
        send("Fizz")
    }
}
```

</div>

`buzz` は500ミリ秒ごとに "Buzz!" 文字列を生成します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // 500ミリ秒ごとに "Buzz!" を送る
        delay(500)
        send("Buzz!")
    }
}
```

</div>

[receive][ReceiveChannel.receive]サスペンド関数を使用すると、一方のチャネルから _または_ 他方のチャネルから受信することができます。
しかし、[select]式は、[onReceive][ReceiveChannel.onReceive]節を使って _両方_ から同時に受け取ることができます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit>はこのセレクト式が結果を生成しないことを意味します
        fizz.onReceive { value ->  // 最初のセレクト節
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // 2番目のセレクト節
            println("buzz -> '$value'")
        }
    }
}
```

</div>

これを全部で7回実行しましょう。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.fizz() = produce<String> {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}

suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren() // cancel fizz & buzz coroutines
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-select-01.kt)で完全なコードを取得できます

このコードの結果は次のとおりです。

```text
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
buzz -> 'Buzz!'
```

<!--- TEST -->

### クローズ時の選択

チャネルが閉じられると `select` の[onReceive][ReceiveChannel.onReceive]節が失敗し、対応する `select` が例外をスローします。
[onReceiveOrNull][onReceiveOrNull]節を使用して、チャンネルが閉じられたときに特定のアクションを実行できます。
次の例は、 `select` が選択された節の結果を返す式であることも示しています。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'a' is closed"
            else
                "a -> '$value'"
        }
        b.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'b' is closed"
            else
                "b -> '$value'"
        }
    }
```

</div>

[onReceiveOrNull][onReceiveOrNull]は、非nullの要素を持つチャネルに対してのみ定義される拡張関数であるため、閉じたチャネルとnull値の間に偶然の混乱がないことに注意してください。

"Hello" 文字列を4回生成するチャネル `a` と "World" を4回生成するチャネル `b` で使用してみましょう。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'a' is closed"
            else
                "a -> '$value'"
        }
        b.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'b' is closed"
            else
                "b -> '$value'"
        }
    }

fun main() = runBlocking<Unit> {
//sampleStart
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    repeat(8) { // 最初の8個の結果をプリントする
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()
//sampleEnd
}
```

</div>

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-select-02.kt)で完全なコードを取得できます

このコードの結果は非常に興味深いので、それをモードの詳細で分析します。

```text
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

<!--- TEST -->

それにはいくつかの所見があります。

まず、 `select` は最初の節に _偏って_ います。
複数の節が同時に選択可能な場合、最初の節が選択されます。
ここでは、両方のチャネルが常に文字列を生成しているので、チャネル `a` はselectの最初の節であり、勝ちます。
しかし、バッファされていないチャネルを使用しているので、 `a` は[send][SendChannel.send]呼び出しで時々中断し、 `b` にも送信する機会を与えます。

2番目の所見は、[onReceiveOrNull][onReceiveOrNull]は、チャネルが既に閉じられているときに直ちに選択されることです。

### 送信の選択

Select式には、[onSend][SendChannel.onSend]節があり、選択のバイアスされた性質と組み合わせてとても有効に使用できます。

プライマリチャネルのコンシューマーが送信に追いつかないときに、その値を `side` チャネルに送る整数のプロデューサーの例を書きましょう。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // 1から10までの10個の数を生成する
        delay(100) // 100ミリ秒ごと
        select<Unit> {
            onSend(num) {} // プライマリチャネルに送る
            side.onSend(num) {} // またはサイドチャネルに送る
        }
    }
}
```

</div>

コンシューマーはかなり遅くして、各数値を処理するのに250ミリ秒かけることにします。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val side = Channel<Int>() // サイドチャネルを割り当てる
    launch { // これはサイドチャネルの非常に高速なコンシューマー
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach {
        println("Consuming $it")
        delay(250) // 急がずに、消費した値をきっちり消化する
    }
    println("Done consuming")
    coroutineContext.cancelChildren()
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-select-03.kt)で完全なコードを取得できます

では、何が起こるか見てみましょう。

```text
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

<!--- TEST -->

### 延期された値の選択

遅延値は、[onAwait][Deferred.onAwait]節を使用して選択できます。
ランダム遅延の後に遅延文字列値を返す非同期関数から始めましょう。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```

</div>

ランダムな遅延でこれを1ダース開始してみましょう。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

</div>

メイン関数は最初のasyncString関数が完了するのを待って、まだアクティブな遅延値の数を数えます。
`select` 式はKotlin DSLであるため、任意のコードを使って節を提供することができることに留意してください。
この場合、各遅延値に対して `onAwait` 節を提供するために遅延値のリストを反復します。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*
import java.util.*

fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}

fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val list = asyncStringsList()
    val result = select<String> {
        list.withIndex().forEach { (index, deferred) ->
            deferred.onAwait { answer ->
                "Deferred $index produced answer '$answer'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-select-04.kt)で完全なコードを取得できます

出力は、

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 延期された値のチャネルの切り替え

次の遅延値が来るかチャネルが閉じられるまで、遅延ストリング値のチャネルを消費し、受信した遅延値を待つチャネルプロデューサー関数を書きましょう。
この例では、同じ `select` に[onReceiveOrNull][onReceiveOrNull]節と[onAwait][Deferred.onAwait]節を入れています。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // 最初に受け取った遅延値から開始する
    while (isActive) { // キャンセルまたは閉じられない限りループする
        val next = select<Deferred<String>?> { // このselectから次の遅延値またはnullを返す
            input.onReceiveOrNull { update ->
                update // 待機する次の値を置き換える
            }
            current.onAwait { value ->
                send(value) // 現在の遅延が発生した値を送信する
                input.receiveOrNull() // 入力チャネルから次の遅延を使用する
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // ループを出る
        } else {
            current = next
        }
    }
}
```

</div>

これをテストするために、指定した時間後に指定された文字列に解決される単純な非同期関数を使用します。


<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```

</div>

メイン関数は、単に `switchMapDeferreds` の結果をプリントするコルーチンを起動し、いくつかのテストデータを送信するだけです。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}

fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}

fun main() = runBlocking<Unit> {
//sampleStart
    val chan = Channel<Deferred<String>>() // テスト用のチャネル
    launch { // プリント用のコルーチンを起動する
        for (s in switchMapDeferreds(chan))
            println(s) // 受信した文字列をプリントする
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // "BEGIN" が生成されるのに十分な時間
    chan.send(asyncString("Slow", 500))
    delay(100) // slowが生成されるには不十分な時間
    chan.send(asyncString("Replace", 100))
    delay(500) // 最後のものの前に時間を与える
    chan.send(asyncString("END", 500))
    delay(1000) // 処理に時間を与える
    chan.close() // チャネルを閉じる ...
    delay(500) // 終了させるためにしばらく待つ
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-select-05.kt)で完全なコードを取得できます

このコードの結果は次の通りです。

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Deferred.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/on-await.html
<!--- INDEX kotlinx.coroutines.channels -->
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[ReceiveChannel.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive.html
[onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/on-receive-or-null.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[SendChannel.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/on-send.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
