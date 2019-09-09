<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/ExceptionsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class ExceptionsGuideTest {
--> 
**目次**

<!--- TOC -->

* [例外処理](#例外処理)
  * [例外の伝播](#例外の伝播)
  * [CoroutineExceptionHandler](#coroutineexceptionhandler)
  * [キャンセルと例外](#キャンセルと例外)
  * [例外の集約](#例外の集約)
  * [監視](#監視)
    * [監督ジョブ](#監督ジョブ)
    * [監視スコープ](#監視スコープ)
    * [監視付きコルーチンの例外](#監視付きコルーチンの例外)

<!--- END_TOC -->

## 例外処理


このセクションでは、例外処理と例外のキャンセルについて説明します。
キャンセルされたコルーチンは中断ポイントで[CancellationException]をスローし、コルーチンの機構では無視されることは既に知っています。しかし、キャンセル中に例外がスローされたり、同じコルーチンの複数の子が例外をスローするとどうなりますか？

### 例外の伝播

コルーチンのビルダーには、自動的に例外を伝播する（[launch]と[actor]）か、それらをユーザーに公開する（[async]と[produce]）という2つの特色があります。
前者はJavaの `Thread.uncaughtExceptionHandler` と同様に未処理の例外を扱いますが、後者は例えば[await][Deferred.await]や[receive][ReceiveChannel.receive]などで最終的な例外を消費することに依存しています。（[produce]と[receive][ReceiveChannel.receive]については後ほど[チャネル](channels.md)セクションで説明します）。

[GlobalScope]でコルーチンを作成する簡単な例で説明できます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
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

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-01.kt)で完全なコードを取得できます。

このコードの出力は以下の通り（[debug](coroutine-context-and-dispatchers.md#コルーチンとスレッドのデバッグ)を用いる）。

```text
Throwing exception from launch
Exception in thread "DefaultDispatcher-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

<!--- TEST EXCEPTION-->

### CoroutineExceptionHandler

しかし、コンソールにすべての例外を出力したくない場合はどうすればよいですか？
[CoroutineExceptionHandler]コンテキスト要素は、カスタムロギングまたは例外処理が行われるコルーチンの一般的な `catch` ブロックとして使用されます。
これは[`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))を使うことに似ています。

JVMでは、[CoroutineExceptionHandler]を[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)で登録することによって、すべてのコルーチンのグローバル例外ハンドラーを再定義することができます。
グローバル例外ハンドラーは、特定のハンドラーが登録されていないときに使用される[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))に似ています。
Androidでは、 `uncaughtExceptionPreHandler`がグローバルコルーチン例外ハンドラーとしてインストールされています。

[CoroutineExceptionHandler]は、ユーザが処理する予定のない例外に対してのみ呼び出されるため、[async]ビルダーなどに登録しても効果はありません。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-02.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Caught java.lang.AssertionError
```

<!--- TEST-->

### キャンセルと例外

キャンセルは例外と緊密に結びついています。
コルーチンは内部的に `CancellationException` を使用してキャンセルしますが、これらの例外はすべてのハンドラーで無視されるため、 `catch` ブロックで取得できる追加のデバッグ情報のソースとしてのみ使用する必要があります。
コルーチンが理由なしで[Job.cancel]を使用して取り消されると終了しますが、その親は取り消されません。
理由なしで取り消すことは、親がキャンセルすることなく子をキャンセルする仕組みです。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-03.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Cancelling child
Child is cancelled
Parent is not cancelled
```

<!--- TEST-->

コルーチンが `CancellationException` 以外の例外を検出した場合、その例外を持つ親を取り消します。
この動作はオーバーライドできず、[CoroutineExceptionHandler]実装に依存しない[構造化並行性](composing-suspending-functions.md#asyncでの構造化並行性)の安定したコルーチン階層を提供するために使用されます。
元の例外は、すべての子が終了したときに親によって処理されます。

> これは、これらの例で[CoroutineExceptionHandler]が[GlobalScope]で作成されたコルーチンに常にインストールされている理由もあります。
メインの[runBlocking]のスコープで起動されるコルーチンに例外ハンドラーをインストールするのは意味がありません。メインコルーチンは、インストールされたハンドラーによらず子が例外で完了したときに常にキャンセルされるためです。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-04.kt)で完全なコードを取得できます

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
一般的なルールは「最初の例外が勝つ」ので、最初にスローされた例外がハンドラーに渡されます。
しかし、それは、例えばコルーチンが `finally` ブロックで例外をスローした場合などに例外が失われることがあります。
したがって、追加の例外は抑制されます。

> 解決策の1つは各例外を別々に報告することですが、[Deferred.await]には動作の不整合を回避するための同じメカニズムが必要であり、これによりコルーチンの実装の詳細（作業の一部を子に委任したかどうか）が例外ハンドラーにリークすることになります。


<!--- INCLUDE

import kotlinx.coroutines.exceptions.*
-->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception with suppressed ${exception.suppressed.contentToString()}")
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
            delay(100)
            throw IOException()
        }
        delay(Long.MAX_VALUE)
    }
    job.join()  
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-05.kt)で完全なコードを取得できます

> 注：上記のコードは、 `suppressed` 例外をサポートするJDK7以降でのみ正常に動作します

このコードの出力は以下の通り。

```text
Caught java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> このメカニズムは現在、Javaバージョン1.7以降でのみ動作します。
JSおよびネイティブの制限は一時的なもので、今後修正される予定です。

キャンセル例外は、デフォルトでは透過的かつアンラップされています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
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
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e
        }
    }
    job.join()
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-06.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Rethrowing CancellationException with original cause
Caught original java.io.IOException
```
<!--- TEST-->

### 監視

以前に検討したように、キャンセルはコルーチン階層全体を伝播する双方向の関係です。
しかし、単方向のキャンセルが必要な場合はどうでしょうか？

そのような要件の良い例は、そのスコープで定義されたジョブを持つUIコンポーネントです。
UIの子タスクのいずれかが失敗した場合、UIコンポーネント全体を常にキャンセル（事実上強制終了）する必要はありませんが、UIコンポーネントが破壊される（およびそのジョブがキャンセルされる）場合、結果は不要になったのですべての子ジョブを失敗させる必要があります。

別の例は、複数の子ジョブを生成し、実行を_監視_し、失敗を追跡し、失敗した子ジョブのみを再起動するサーバープロセスです。

#### 監視ジョブ

これらの目的のために、[SupervisorJob][SupervisorJob()]を使用できます。通常の[Job][Job()]と似ていますが、キャンセルは下方向にのみ伝播されるという唯一の例外があります。 次の例で簡単に説明できます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // launch the first child -- its exception is ignored for this example (don't do this in practice!)
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("First child is failing")
            throw AssertionError("First child is cancelled")
        }
        // launch the second child
        val secondChild = launch {
            firstChild.join()
            // Cancellation of the first child is not propagated to the second child
            println("First child is cancelled: ${firstChild.isCancelled}, but second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // But cancellation of the supervisor is propagated
                println("Second child is cancelled because supervisor is cancelled")
            }
        }
        // wait until the first child fails & completes
        firstChild.join()
        println("Cancelling supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-01.kt)で完全なコードを取得できます

このコードの出力は次のとおりです。

```text
First child is failing
First child is cancelled: true, but second one is still active
Cancelling supervisor
Second child is cancelled because supervisor is cancelled
```
<!--- TEST-->


#### 監視スコープ

*スコープ付き*並行性の場合、同じ目的で[coroutineScope]の代わりに[supervisorScope]を使用できます。
キャンセルは一方向にのみ伝播し、失敗した場合にのみすべての子をキャンセルします。
また、[coroutineScope]と同様に、完了前にすべての子を待機します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("Child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("Child is cancelled")
                }
            }
            // Give our child a chance to execute and print using yield 
            yield()
            println("Throwing exception from scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught assertion error")
    }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-02.kt)で完全なコードを取得できます。

このコードの出力は次のとおりです。

```text
Child is sleeping
Throwing exception from scope
Child is cancelled
Caught assertion error
```
<!--- TEST-->

#### 監視付きコルーチンの例外

通常のジョブと監督ジョブのもう1つの重要な違いは、例外処理です。
すべての子は、例外処理メカニズムを介してそれ自体で例外を処理する必要があります。
この違いは、子の失敗が親に伝播されないという事実に由来します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    supervisorScope {
        val child = launch(handler) {
            println("Child throws an exception")
            throw AssertionError()
        }
        println("Scope is completing")
    }
    println("Scope is completed")
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-03.kt)で完全なコードを取得できます。

このコードの出力は次のとおりです。

```text
Scope is completing
Child throws an exception
Caught java.lang.AssertionError
Scope is completed
```
<!--- TEST-->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineExceptionHandler]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[SupervisorJob()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[supervisorScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
<!--- END -->
