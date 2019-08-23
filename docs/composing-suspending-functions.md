<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/ComposingGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class ComposingGuideTest {
--> 

## 目次

<!--- TOC -->

* [サスペンド関数の作成](#サスペンド関数の作成)
  * [デフォルトではシーケンシャル](#デフォルトではシーケンシャル)
  * [asyncを使用した並列処理](#asyncを使用した並列処理)
  * [遅延して開始されるasync](#遅延して開始されるasync)
  * [Asyncスタイル関数](#asyncスタイル関数)
  * [asyncでの構造化された並列処理](#asyncでの構造化された並列処理)

<!--- END_TOC -->

## サスペンド関数の作成

このセクションでは、サスペンド関数のさまざまな構成方法について説明します。

### デフォルトではシーケンシャル

何らかのリモートサービスコールや計算のように有用な何かを行う他の場所で定義された2つのサスペンド関数があるとします。 実際にはこの例の目的のためにそれぞれただ1秒遅らせるだけですが、これらは有用なものとしておきます。

<!--- INCLUDE .*/example-compose-([0-9]+).kt
import kotlin.system.*
-->

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

<!--- INCLUDE .*/example-compose-([0-9]+).kt -->

最初に `doSomethingUsefulOne` を実行してから `doSomethingUsefulTwo` を実行し、その結果の合計を計算します。これを _連続して_ 呼び出す必要がある場合はどうすればよいですか。
実際には、第1の関数の結果を使用して、第2の関数を呼び出す必要があるかどうか、あるいは呼び出し方法を決定するために、これを行います。

コルーチンのコードは通常のコードと同様にデフォルトでは _シーケンシャル_ なので、通常の順次呼び出しを使用します。
次の例は、両方のサスペンド関数を実行するのにかかる合計時間を測定することによってそれを示しています。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-01.kt)で完全なコードを取得できます

これは次のような出力をします。

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST ARBITRARY_TIME -->

### asyncを使用した並列処理

`doSomethingUsefulOne` と `doSomethingUsefulTwo` の呼び出しの間に依存関係がなく、両方を _同時_ に行うことでより速く答えを出したいのですが？
これは[async]が助けになる場面です。

概念的には、[async]は[launch]と同じです。
これは別のコルーチンを開始します。このコルーチンは、他のすべてのコルーチンと同時に動作する軽量スレッドです。
相違点は `launch` は[Job]を返し、結果の値は持ちませんが、 `async` は結果を後で提供する約束を表す軽量でノンブロッキングなフューチャーである[Deferred]を返します。
遅延された値に対して `.await()` を使用して最終的な結果を得ることができますが、 `Deferred` も `Job` なので必要に応じてキャンセルすることができます。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-02.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

2つのコルーチンが同時に実行されるため、これは2倍高速です。
コルーチンの並列処理は常に明示的であることに注意してください。

### 遅延して開始されるasync

オプションの `start` パラメーターに[CoroutineStart.LAZY]値を使用する[async]の遅延オプションがあります。
コルーチンは、 [await][Deferred.await]または [start][Job.start]関数が呼び出されたときにその結果が必要な場合にのみ開始されます。
次の例を実行します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // いくつかの計算
        one.start() // 最初のものを開始
        two.start() // 2番目のものを開始
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-03.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

ここでは2つのコルーチンが定義されていますが、前の例のようには実行されません。[start][Job.start]を呼び出す実行開始の制御はプログラマに与えられます。
最初に `one` を開始し、次に `two` を開始し、個々のコルーチンが終了するのを待ちます。

`println` で個々のコルーチンの[await][Deferred.await]を呼び出し、[start][Job.start]を省略した場合、シーケンシャルに[await][Deferred.await]コルーチンの実行を開始し、実行が終了するのを待ちます。これは、遅延の意図された使用例ではありません。
`async(start = CoroutineStart.LAZY)` のユースケースは、値の計算にサスペンド関数が含まれる場合に標準 `lazy` 関数の代わりになります。

### Asyncスタイル関数

明示的な[GlobalScope]参照で[async]コルーチンビルダーを使用して _非同期的_ に  `doSomethingUsefulOne` と ` doSomethingUsefulTwo` を呼び出す非同期スタイルの関数を定義できます。
そのような関数には「Async」接尾辞を付けた名前を付け、非同期計算を開始だけしてその結果を得るために遅延値を使用する必要があるという事実を強調します。

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

これらの `xxxAsync` 関数は、_サスペンド_ 関数**ではない**ことに注意してください。
これらはどこからでも使用できます。
しかし、呼び出すコードのアクションは常に非同期（ここでは _並列実行_ を意味する）実行であること意味します。

次の例は、コルーチンの外部での使用例を示しています。
 
```kotlin
// この例では、 `main` の右側に `runBlocking` がないことに注意してください
fun main(args: Array<String>) {
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
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-04.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
The answer is 42
Completed in 1085 ms
-->

> 非同期関数を使用したこのプログラミングスタイルは、他のプログラミング言語で一般的なスタイルであるため、ここでは説明のためにのみ提供されています。 Kotlinコルーチンでこのスタイルを使用することは、以下で説明する理由から **強く推奨されません** 。

`val one = somethingUsefulOneAsync()` 行と `one.await()` 式の間に何らかの論理エラーがあり、プログラムが例外をスローし、プログラムによって実行されていた操作が異常終了した場合を考えてみましょう。
通常、グローバルエラーハンドラはこの例外をキャッチし、開発者にエラーを記録して報告することができますが、そうでなければプログラムは他の操作を続行できます。
しかし、ここではバックグラウンドで実行されている `somethingUsefulOneAsync` を開始した操作は中止されます。 この問題は、以下のセクションで示すように、構造化された並列処理では発生しません。

### asyncでの構造化された並列処理

[asyncを使用した並列処理](#asyncを使用した並列処理)の例をとり、 `doSomethingUsefulOne` と `doSomethingUsefulTwo` を同時に実行し、その結果の合計を返す関数を抽出しましょう。
[async]コルーチンビルダーは[CoroutineScope]の拡張として定義されているため、スコープ内に配置する必要があります。つまり、[coroutineScope]関数が提供するものです。

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
     awaitAll(one, two)
     one.await() + two.await()
}
```

ここで、`concurrentSum` 関数のコードの中で何かがうまくいかず、例外がスローされた場合、そのスコープで起動されたコルーチンはすべて取り消されます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-05.kt)で完全なコードを取得できます。

上記のメイン関数の出力から明らかなように、まだ両方の操作の同時実行性があります。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

キャンセルは常にコルーチンの階層構造を介して伝播されます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
    
    awaitAll(one, two)
    one.await() + two.await()
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-compose-06.kt)で完全なコードを取得できます。

最初の `async` と待っている親が1つの子の失敗でどのように取り消されるかに注目してください。

```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

<!--- TEST -->

