<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/ComposingGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class ComposingGuideTest {
--> 

**目次**

<!--- TOC -->

* [サスペンド関数の作成](#サスペンド関数の作成)
  * [デフォルトではシーケンシャル](#デフォルトではシーケンシャル)
  * [asyncを使用した並行処理](#asyncを使用した並行処理)
  * [遅延して開始されるasync](#遅延して開始されるasync)
  * [Asyncスタイル関数](#asyncスタイル関数)
  * [asyncでの構造化並行性](#asyncでの構造化並行性)

<!--- END_TOC -->

## サスペンド関数の作成

このセクションでは、サスペンド関数のさまざまな構成方法について説明します。

### デフォルトではシーケンシャル

何らかのリモートサービスコールや計算のように有用な何かを行う他の場所で定義された2つのサスペンド関数があるとします。 実際にはこの例の目的のためにそれぞれただ1秒遅らせるだけですが、これらは有用なものとしておきます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 何か有用なことをしているふり
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // これも、何か有用なことをしているふり
    return 29
}
```

</div>


最初に `doSomethingUsefulOne` 、_次に_ `doSomethingUsefulTwo` を _シーケンシャルに_ 呼び出して、それらの結果の合計を計算する必要がある場合はどうしますか？
実際には、最初の関数の結果を使用して2番目の関数を呼び出す必要があるかどうか、あるいは呼び出し方法を決定する場合にこれを行います。

コルーチンのコードは通常のコードと同様にデフォルトでは _シーケンシャル_ なので、通常の順次呼び出しを使用します。
次の例は、両方のサスペンド関数を実行するのにかかる合計時間を測定することによってそれを示しています。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-01.kt)で完全なコードを取得できます

これは次のように出力します。

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST ARBITRARY_TIME -->

### asyncを使用した並行処理

`doSomethingUsefulOne` と `doSomethingUsefulTwo` の呼び出しの間に依存関係がなく、両方を _並行に_ 行うことでより速く答えを出したい場合はどうでしょうか？
これは[async]が役立つ場面です。

概念的には、[async]は[launch]と同じです。
これは、他のすべてのコルーチンと並行に動作する軽量スレッドである別のコルーチンを開始します。
相違点は `launch` は[Job]を返し、結果の値は持ちませんが、 `async` は結果を後で提供する約束を表す軽量でノンブロッキングなフューチャーである[Deferred]を返します。
遅延された値に対して `.await()` を使用して最終的な結果を得ることができますが、 `Deferred` も `Job` なので必要に応じてキャンセルすることができます。
 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-02.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

2つのコルーチンが並行に実行されるため、これは2倍高速です。
コルーチンの並行性は常に明示的であることに注意してください。

### 遅延して開始されるasync

オプションとして、`start` パラメーターを[CoroutineStart.LAZY]に設定することにより、[async]を遅延させることができます。
このモードでは、[await][Deferred.await]で結果が必要な場合、またはその `Job` の[start][Job.start]関数が呼び出された場合にのみ、コルーチンを開始します。

次の例を実行します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // いくつかの計算
        one.start() // 最初のものを開始
        two.start() // 2番目のものを開始
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-03.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

ここでは2つのコルーチンが定義されていますが、前の例のようには実行されません。[start][Job.start]を呼び出す実行開始の制御はプログラマに与えられます。
最初に `one` を開始し、次に `two` を開始し、個々のコルーチンが終了するのを待ちます。

最初に個々のコルーチンで[start][Job.start]を呼び出さずに `println`で[await][Deferred.await]を呼び出すと、[await][Deferred.await]がコルーチンの実行とその終了を待機するためシーケンシャルな動作につながることに注意してください。これは遅延の意図したユースケースではありません。
`async(start = CoroutineStart.LAZY)` のユースケースは、値の計算にサスペンド関数が含まれる場合に標準 `lazy` 関数の代わりになります。

### Asyncスタイル関数

明示的に[GlobalScope]を参照して[async]コルーチンビルダーを使用することで _非同期的_ に `doSomethingUsefulOne` と `doSomethingUsefulTwo` を呼び出す非同期スタイルの関数を定義できます。
そのような関数には「～Async」接尾辞を付けた名前を付け、非同期計算を開始だけしてその結果を得るために遅延値を使用する必要があるという事実を強調します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// somethingUsefulOneAsyncの結果の型は、Deferred<Int>
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// somethingUsefulTwoAsyncの結果の型は、Deferred<Int>
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

</div>

これらの `xxxAsync` 関数は、_サスペンド_ 関数**ではない**ことに注意してください。
これらはどこからでも使用できます。
しかし、呼び出すコードのアクションは常に非同期（ここでは _並行_ を意味する）実行であること意味します。

次の例は、コルーチンの外部での使用例を示しています。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

//sampleStart
// この例では、 `main` の右側に `runBlocking` がないことに注意してください
fun main() {
    val time = measureTimeMillis {
        // コルーチンの外部で非同期アクションを開始できる
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // 結果を待つにはサスペンドまたはブロックする必要がある。
        // ここでは `runBlocking { ... }` を使用して、結果を待つ間メインスレッドをブロックする
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
//sampleEnd

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-04.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
The answer is 42
Completed in 1085 ms
-->

> 非同期関数を使用したこのプログラミングスタイルは、他のプログラミング言語で一般的なスタイルであるため、ここでは説明のためにのみ提供されています。 Kotlinコルーチンでこのスタイルを使用することは、以下で説明する理由から **強く推奨されません** 。

`val one = somethingUsefulOneAsync()` 行と `one.await()` 式の間に何らかの論理エラーがあり、プログラムが例外をスローし、プログラムによって実行されていた操作が異常終了した場合を考えてみましょう。
通常、グローバルエラーハンドラーはこの例外をキャッチし、開発者にエラーを記録して報告することができますが、そうでなければプログラムは他の操作を続行できます。
しかしここでは、それを開始した操作が中止されたにもかかわらず、`somethingUsefulOneAsync` がバックグラウンドで実行されています。
この問題は、以下のセクションで示すように、構造化並行性では発生しません。

### asyncでの構造化並行性

[asyncを使用した並行処理](#asyncを使用した並行処理)の例をとり、 `doSomethingUsefulOne` と `doSomethingUsefulTwo` を並行に実行し、その結果の合計を返す関数を抽出しましょう。
[async]コルーチンビルダーは[CoroutineScope]の拡張として定義されているため、スコープ内に配置する必要があります。つまり、[coroutineScope]関数が提供するものです。

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

</div>

このように、 `concurrentSum` 関数のコード内で何かがうまくいかず、例外をスローした場合、そのスコープで起動されたすべてのコルーチンはキャンセルされます。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-05.kt)で完全なコードを取得できます。

上記の `main` 関数の出力から明らかなように、まだ両方の操作の並行実行性があります。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

キャンセルは常にコルーチンの階層構造を介して伝播されます。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // 非常に長い計算をエミュレート
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-compose-06.kt)で完全なコードを取得できます。

子の1つ（つまり、 `two` ）が失敗したとき、最初の `async` と待機中の親の両方がどのようにキャンセルされるかに注目してください。
```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/start.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- END -->
