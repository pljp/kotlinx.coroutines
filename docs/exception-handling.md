<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/ExceptionsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class ExceptionsGuideTest {
--> 
## 目次

<!--- TOC -->

## 例外処理

<!--- INCLUDE .*/example-exceptions-([0-9]+).kt
-->

このセクションでは、例外処理と例外のキャンセルについて説明します。
キャンセルされたコルーチンは中断ポイントで[CancellationException]をスローし、コルーチンの機構では無視されることは既に知っています。しかし、キャンセル中に例外がスローされたり、同じコルーチンの複数の子が例外をスローするとどうなりますか？

### 例外の伝播

コルーチンのビルダーには、自動的に例外を伝播する（[launch]と[actor]）か、それらをユーザーに公開する（[async]と[produce]）という2つの特色があります。
前者はJavaの `Thread.uncaughExceptionHandler` と同様に未処理の例外を扱いますが、後者は例えば[await][Deferred.await]や[receive][ReceiveChannel.receive]などで最終的な例外を消費することに依存しています。（[produce]と[receive][ReceiveChannel.receive]については後ほど[チャネル](#チャネル（実験的)）セクションで説明します）。

これは[GlobalScope]で新しいコルーチンを作成する簡単な例で示すことができます。

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Thread.defaultUncaughtExceptionHandlerによってコンソールにプリントされる
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException() // 何もプリントされておらず、ユーザーのawaitコールに依存する
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-01.kt)で完全なコードを取得できます。

このコードの出力は以下の通り（[debug](#コルーチンとスレッドのデバッグ)を用いる）。

```text
Throwing exception from launch
Exception in thread "ForkJoinPool.commonPool-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

<!--- TEST EXCEPTION-->

### CoroutineExceptionHandler

しかし、コンソールにすべての例外を出力したくない場合はどうすればよいですか？
[CoroutineExceptionHandler]コンテキスト要素は、カスタムロギングまたは例外処理が行われるコルーチンの一般的な `catch` ブロックとして使用されます。
これは[`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))を使うことに似ています。

JVMでは、[CoroutineExceptionHandler]を[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)で登録することによって、すべてのコルーチンのグローバル例外ハンドラを再定義することができます。
グローバル例外ハンドラは、特定のハンドラが登録されていないときに使用される[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))に似ています。
Androidでは、 `uncaughtExceptionPreHandler`がグローバルコルーチン例外ハンドラとしてインストールされています。

[CoroutineExceptionHandler]は、ユーザが処理する予定のない例外に対してのみ呼び出されるため、[async]ビルダーなどに登録しても効果はありません。

```kotlin
fun main(args: Array<String>) = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) {
        throw ArithmeticException() // ユーザーがdeferred.await()を呼び出しても何もプリントされない
    }
    joinAll(job, deferred)
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-02.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Caught java.lang.AssertionError
```

<!--- TEST-->

### キャンセルと例外

キャンセルは例外と緊密に結びついています。
コルーチンは内部的に `CancellationException` を使用してキャンセルしますが、これらの例外はすべてのハンドラで無視されるため、 `catch` ブロックで取得できる追加のデバッグ情報のソースとしてのみ使用する必要があります。
コルーチンが理由なしで[Job.cancel]を使用して取り消されると終了しますが、その親は取り消されません。
理由なしで取り消すことは、親がキャンセルすることなく子をキャンセルする仕組みです。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-03.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Cancelling child
Child is cancelled
Parent is not cancelled
```

<!--- TEST-->

コルーチンが `CancellationException` 以外の例外を検出した場合、その例外を持つ親を取り消します。
この動作はオーバーライドできず、[CoroutineExceptionHandler]実装に依存しない[構造化同時実行性](#構造化同時実行性)の安定したコルーチン階層を提供するために使用されます。
元の例外は、すべての子が終了したときに親によって処理されます。

> これは、これらの例で[CoroutineExceptionHandler]が[GlobalScope]で作成されたコルーチンに常にインストールされている理由もあります。
メインの[runBlocking]のスコープで起動されるコルーチンに例外ハンドラをインストールするのは意味がありません。メインコルーチンは、インストールされたハンドラによらず子が例外で完了したときに常にキャンセルされるためです。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    val job = GlobalScope.launch(handler) {
        launch { // 最初の子
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // 2番目の子
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-04.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
Caught java.lang.ArithmeticException
```
<!--- TEST-->

### 例外の集約

コルーチンの複数の子が例外をスローするとどうなりますか？
一般的なルールは「最初の例外が勝つ」ので、最初にスローされた例外がハンドラに渡されます。
しかし、それは、例えばコルーチンが `finally` ブロックで例外をスローした場合などに例外が失われることがあります。
したがって、追加の例外は抑制されます。

> 解決策の1つは、各例外を別々に報告することですが、[Deferred.await]は動作の不一致を避けるために同じメカニズムを持っていて、コルーチンの実装の詳細（子に仕事の一部を委任したかどうか）がその例外ハンドラに漏洩する原因になります。

<!--- INCLUDE
import kotlinx.coroutines.experimental.exceptions.*
import kotlin.coroutines.experimental.*
import java.io.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception with suppressed ${exception.suppressed().contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                throw ArithmeticException()
            }
        }
        launch {
            throw IOException()
        }
        delay(Long.MAX_VALUE)
    }
    job.join()
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-05.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Caught java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> このメカニズムは現在、Javaバージョン1.7以降でのみ動作します。
JSおよびネイティブの制限は一時的なもので、今後修正される予定です。

キャンセル例外は、デフォルトでは透過的かつアンラップされています。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
import java.io.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught original $exception")
    }
    val job = GlobalScope.launch(handler) {
        val inner = launch {
            launch {
                launch {
                    throw IOException()
                }
            }
        }
        try {
            inner.join()
        } catch (e: JobCancellationException) {
            println("Rethrowing JobCancellationException with original cause")
            throw e
        }
    }
    job.join()
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-exceptions-06.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Rethrowing JobCancellationException with original cause
Caught original java.io.IOException
```
<!--- TEST-->
