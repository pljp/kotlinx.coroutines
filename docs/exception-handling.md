<!--- TEST_NAME ExceptionsGuideTest -->

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

<!--- END -->

## 例外処理


このセクションでは、例外処理と例外のキャンセルについて説明します。
キャンセルされたコルーチンは中断ポイントで[CancellationException]をスローし、コルーチンの機構では無視されることは既に知っています。ここでは、キャンセル中に例外がスローされた場合、または同じコルーチンの複数の子が例外をスローした場合に何が起こるかを見ていきます。

### 例外の伝播

コルーチンのビルダーには、自動的に例外を伝播する（[launch]と[actor]）か、それらをユーザーに公開する（[async]と[produce]）という2つの特色があります。
これらのビルダーを使用して別のコルーチンの _子_ ではない _ルート_ コルーチンを作成する場合、前者のビルダーは、Javaの `Thread.uncaughtExceptionHandler` と同様に、例外を**キャッチされていない**例外として扱いますが、後者は、例えば[await][Deferred.await]または[receive][ReceiveChannel.receive]を介して最終的な例外の消費はユーザーに依存しています（[produce]と[receive][ReceiveChannel.receive]については後ほど[チャネル][Channels](channels.md)セクションで説明します）。

これは、[GlobalScope]を使用してルートコルーチンを作成する簡単な例で示すことができます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = GlobalScope.launch { // launchによるルートコルーチン
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Thread.defaultUncaughtExceptionHandlerによってコンソールにプリントされる
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // asyncによるルートコルーチン
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

**キャッチされていない**例外をコンソールに出力するデフォルトの動作をカスタマイズすることができます。
[CoroutineExceptionHandler]コンテキスト要素は、カスタムロギングまたは例外処理が行われるコルーチンの一般的な `catch` ブロックとして使用されます。
_ルート_ コルーチンの[CoroutineExceptionHandler]コンテキスト要素は、このルートコルーチンと、カスタム例外処理が行われる可能性のあるすべての子の汎用 `catch` ブロックとして使用できます。
これは[`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))に似ています。
`CoroutineExceptionHandler` の例外から回復することはできません。
コルーチンは、ハンドラーが呼び出されたときに、対応する例外の発生とともにすでに完了しています。
通常、ハンドラーは、例外のログ記録、何らかのエラーメッセージの表示、アプリケーションの終了、および/または再起動に使用されます。

JVMでは、[CoroutineExceptionHandler]を[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)で登録することによって、すべてのコルーチンのグローバル例外ハンドラーを再定義することができます。
グローバル例外ハンドラーは、特定のハンドラーが登録されていないときに使用される[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))に似ています。
Androidでは、 `uncaughtExceptionPreHandler`がグローバルコルーチン例外ハンドラーとしてインストールされています。

`CoroutineExceptionHandler` は、**キャッチされていない**例外（他の方法で処理されなかった例外）でのみ呼び出されます。
特に、すべての _子_ コルーチン（別の[Job]のコンテキストで作成されたコルーチン）は、例外の処理を親コルーチンに移譲し、同様にルートまで移譲するため、コンテキストにインストールされた `CoroutineExceptionHandler` は使用されません。
加えて、[async]ビルダーは常にすべての例外をキャッチし、結果の[Deferred]オブジェクトでそれらを表すため、その `CoroutineExceptionHandler` も効果がありません。

> 監視スコープで実行されているコルーチンは、親に例外を伝播せず、このルールから除外されます。 このドキュメントの[監視](#監視)セクションで詳細を説明します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) { // GlobalScopeで実行するルートコルーチン
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) { // これもルート、ただし launch ではなく async
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
CoroutineExceptionHandler got java.lang.AssertionError
```

<!--- TEST-->

### キャンセルと例外

キャンセルは例外と密接に関連しています。
コルーチンは内部的に `CancellationException` を使用してキャンセルしますが、これらの例外はすべてのハンドラーで無視されるため、 `catch` ブロックで取得できる追加のデバッグ情報のソースとしてのみ使用する必要があります。
コルーチンは[Job.cancel]を使用してキャンセルすると終了しますが、その親はキャンセルされません。

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

コルーチンが `CancellationException` 以外の例外を検出した場合、その例外を持つ親をキャンセルします。
この動作はオーバーライドできず、[構造化並行性](composing-suspending-functions.md#asyncでの構造化並行性)の安定したコルーチン階層を提供するために使用されます。
[CoroutineExceptionHandler]実装は、子コルーチンには使用されません。

> これらの例では、[CoroutineExceptionHandler]は常に[GlobalScope]で作成されたコルーチンにインストールされます。
メイン[runBlocking]のスコープで起動されたコルーチンに例外ハンドラーをインストールすることは意味がありません。これは、インストールされたハンドラーによらず、子が例外で完了したときにメインコルーチンが常にキャンセルされるためです。

元の例外は、すべての子が終了した場合にのみ親によって処理されます。
これは、次の例で示されています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
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
CoroutineExceptionHandler got java.lang.ArithmeticException
```
<!--- TEST-->

### 例外の集約

コルーチンの複数の子が例外で失敗した場合、一般的なルールは「最初の例外が優先される」ため、最初の例外が処理されます。
最初の例外の後に発生するすべての追加の例外は、抑制された例外として最初の例外にアタッチされます。

<!--- INCLUDE
import kotlinx.coroutines.exceptions.*
-->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // 別の兄弟がIOExceptionで失敗すると、キャンセルされる
            } finally {
                throw ArithmeticException() // 2番目の例外
            }
        }
        launch {
            delay(100)
            throw IOException() // 最初の例外
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
CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> このメカニズムは現在、Javaバージョン1.7以降でのみ機能することに注意してください。
The JS and Native restrictions are temporary and will be lifted in the future.
JSおよびネイティブの制限は一時的なものであり、今後取り除かれる予定です。

キャンセル例外は透過的であり、デフォルトでアンラップされます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        val inner = launch { // このコルーチンのスタックはすべてキャンセルされる
            launch {
                launch {
                    throw IOException() // 元の例外
                }
            }
        }
        try {
            inner.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e // キャンセル例外は再スローされるが、元のIOExceptionはハンドラーに到達する
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
CoroutineExceptionHandler got java.io.IOException
```
<!--- TEST-->

### 監視

以前に検討したように、キャンセルはコルーチンの階層全体に伝播する双方向の関係です。 一方向のキャンセルが必要な場合を見てみましょう。

このような要件の良い例は、スコープで定義されたジョブを持つUIコンポーネントです。
UIの子タスクのいずれかが失敗した場合、必ずしもUIコンポーネント全体をキャンセル（実質的に強制終了）する必要はありませんが、UIコンポーネントが破棄された（そしてそのジョブがキャンセルされた）場合、結果が不要になるためすべての子ジョブを失敗させる必要があります。

もう1つの例は、複数の子ジョブを生成し、それらの実行を _監視_ して、失敗を追跡し、失敗したジョブのみを再起動する必要があるサーバープロセスです。

#### 監視ジョブ

[SupervisorJob][SupervisorJob()]は、これらの目的のために使用できます。通常の[Job][Job()]と似ていますが、キャンセルは下方向にのみ伝播されるという唯一の例外があります。これは、次の例を使用して簡単に示すことができます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // 最初の子を起動する -- この例では例外は無視される（実際にはこれを行わないこと！）
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("The first child is failing")
            throw AssertionError("The first child is cancelled")
        }
        // 2番目の子を起動する
        val secondChild = launch {
            firstChild.join()
            // 最初の子のキャンセルは2番目の子に伝播されない
            println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // ただし、スーパーバイザーのキャンセルは伝播される
                println("The second child is cancelled because the supervisor was cancelled")
            }
        }
        // 最初の子が失敗して完了するまで待つ
        firstChild.join()
        println("Cancelling the supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-01.kt)で完全なコードを取得できます

このコードの出力は次のとおりです。

```text
The first child is failing
The first child is cancelled: true, but the second one is still active
Cancelling the supervisor
The second child is cancelled because the supervisor was cancelled
```
<!--- TEST-->


#### 監視スコープ

[coroutineScope][_coroutineScope]の代わりに、[supervisorScope][_supervisorScope]を _スコープ付き_ 並行処理に使用できます。
キャンセルを一方向にのみ伝播し、失敗した場合にのみすべての子をキャンセルします。
また、[coroutineScope][_coroutineScope]と同様に、すべての子が完了するまで待機します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("The child is cancelled")
                }
            }
            // yieldを使用して子に実行およびプリントする機会を与える
            yield()
            println("Throwing an exception from the scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught an assertion error")
    }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-02.kt)で完全なコードを取得できます。

このコードの出力は次のとおりです。

```text
The child is sleeping
Throwing an exception from the scope
The child is cancelled
Caught an assertion error
```
<!--- TEST-->

#### 監視付きコルーチンの例外

通常のジョブと監督ジョブのもう1つの重要な違いは、例外処理です。
すべての子は、例外処理メカニズムを介してそれ自体で例外を処理する必要があります。
この違いは、子の失敗が親に伝播されないという事実に由来します。
これは、[supervisorScope][_supervisorScope]内で直接起動されたコルーチンが、ルートコルーチンと同じ方法で、スコープにインストールされた[CoroutineExceptionHandler]を使用 _する_ ことを意味します（詳細については、[CoroutineExceptionHandler](#coroutineexceptionhandler)セクションを参照してください）。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    supervisorScope {
        val child = launch(handler) {
            println("The child throws an exception")
            throw AssertionError()
        }
        println("The scope is completing")
    }
    println("The scope is completed")
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-03.kt)で完全なコードを取得できます。

このコードの出力は次のとおりです。

```text
The scope is completing
The child throws an exception
CoroutineExceptionHandler got java.lang.AssertionError
The scope is completed
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
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[SupervisorJob()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[_supervisorScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
<!--- END -->
