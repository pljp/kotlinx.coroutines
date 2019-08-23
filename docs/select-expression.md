<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.selects.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/SelectGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class SelectGuideTest {
--> 


## 目次

<!--- TOC -->

* [セレクト式（実験的）](#セレクト式セレクト式（実験的）)
  * [チャネルからの選択](#チャネルからの選択)
  * [クローズ時の選択](#クローズ時の選択)
  * [送信の選択](#送信の選択)
  * [延期された値の選択](#延期された値の選択)
  * [延期された値のチャネルの切り替え](#延期された値のチャネルの切り替え)

<!--- END_TOC -->


## セレクト式（実験的）

セレクト式を使用すると複数のサスペンド関数を同時に待つことができ、利用可能になった最初のものを _選択_ することができます。

> セレクト式は `kotlinx.coroutines` の実験的な機能です。 これらのAPIは、 `kotlinx.coroutines` ライブラリの今後のアップデートで、潜在的に大きな変化を伴って進化することが期待されています。

### チャネルからの選択

2つの文字列のプロデューサー、 `fizz` と `buzz` があります。 `fizz` は300ミリ秒ごとに"Fizz"文字列を生成します。

<!--- INCLUDE
import kotlinx.coroutines.experimental.*
import kotlin.coroutines.experimental.*
-->

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // 300ミリ秒ごとに "Fizz" を送る
        delay(300)
        send("Fizz")
    }
}
```

`buzz` は500ミリ秒ごとに "Buzz!" 文字列を生成します。

```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // 500ミリ秒ごとに "Buzz!" を送る
        delay(500)
        send("Buzz!")
    }
}
```

[receive][ReceiveChannel.receive]サスペンド関数を使用すると、一方のチャネルから _または_ 他方のチャネルから受信することができます。
しかし、[select]式は、[onReceive][ReceiveChannel.onReceive]節を使って _両方_ から同時に受け取ることができます。

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

これを全部で7回実行しましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren() // cancel fizz & buzz coroutines    
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-select-01.kt)で完全なコードを取得できます

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
[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]節を使用して、チャンネルが閉じられたときに特定のアクションを実行できます。
次の例は、 `select` が選択された節の結果を返す式であることも示しています。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

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

"Hello" 文字列を4回生成するチャネル `a` と "World" を4回生成するチャネル `b` を使用しましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
```

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

2番目の所見は、[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]は、チャネルが既に閉じられているときに直ちに選択されることです。

### 送信の選択

Select式には、[onSend][SendChannel.onSend]節があり、選択のバイアスされた性質と組み合わせてとても有効に使用できます。

プライマリチャネルのコンシューマーが送信に追いつかないときに、その値を `side` チャネルに送る整数のプロデューサーの例を書きましょう。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

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

コンシューマーはかなり遅くして、各数値を処理するのに250ミリ秒かけることにします。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
``` 
 
> [ここ](../core/kotlinx-coroutines-core/test/guide/example-select-03.kt)で完全なコードを取得できます
  
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

<!--- INCLUDE .*/example-select-04.kt
import java.util.*
-->

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```

ランダムな遅延でこれを1ダース開始してみましょう。

```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

メイン関数は最初のasyncString関数が完了するのを待って、まだアクティブな遅延値の数を数えます。
`select` 式はKotlin DSLであるため、任意のコードを使って節を提供することができることに留意してください。
この場合、各遅延値に対して `onAwait` 節を提供するために遅延値のリストを反復します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-select-04.kt)で完全なコードを取得できます

出力は、

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 延期された値のチャネルの切り替え

次の遅延値が来るかチャネルが閉じられるまで、遅延ストリング値のチャネルを消費し、受信した遅延値を待つチャネルプロデューサー関数を書きましょう。
この例では、同じ `select` に[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]節と[onAwait][Deferred.onAwait]節を入れています。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

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

これをテストするために、指定した時間後に指定された文字列に解決される単純な非同期関数を使用します。

```kotlin
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```

メイン関数は、単に `switchMapDeferreds` の結果をプリントするコルーチンを起動し、いくつかのテストデータを送信するだけです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-select-05.kt)で完全なコードを取得できます

このコードの結果は次の通りです。

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/select.html
<!--- END -->
