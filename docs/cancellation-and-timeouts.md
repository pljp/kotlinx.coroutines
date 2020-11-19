<!--- TEST_NAME CancellationGuideTest -->

**目次**

<!--- TOC -->

* [キャンセルとタイムアウト](#キャンセルとタイムアウト)
  * [コルーチンの実行をキャンセルする](#コルーチンの実行をキャンセルする)
  * [キャンセルは協調的](#キャンセルは協調的)
  * [計算コードをキャンセル可能にする](#計算コードをキャンセル可能にする)
  * [`finally`でリソースを閉じる](#finallyでリソースを閉じる)
  * [キャンセル不可ブロックの実行](#キャンセル不可ブロックの実行)
  * [タイムアウト](#タイムアウト)
  * [非同期タイムアウトとリソース](#非同期タイムアウトとリソース)

<!--- END -->

## キャンセルとタイムアウト

このセクションでは、コルーチンのキャンセルとタイムアウトについて説明します。

### コルーチンの実行をキャンセルする

長時間実行しているアプリケーションでは、バックグラウンドコルーチンをきめ細かく制御する必要があります。
例えば、あるユーザがコルーチンを起動したページを閉じて、その結果が不要になり、その作業を取り消されるかもしれません。
[launch]関数は、実行中のコルーチンをキャンセルするために使用できる[job]を返します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    job.join() // ジョブの完了を待つ
    println("main: Now I can quit.")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-01.kt)で完全なコードを取得できます

次の出力が生成されます。

```text
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

<!--- TEST -->

メインが `job.cancel` を呼び出すとすぐにキャンセルされるため他のコルーチンからの出力は表示されません。
[cancel][Job.cancel]と[join][Job.join]の呼び出しを組み合わせた[Job]の拡張関数[cancelAndJoin]もあります。

### キャンセルは協調的

コルーチンのキャンセルは _協調的_ です。コルーチンコードは取り消し可能にするために協調しなければなりません。
`kotlinx.coroutines` のサスペンド関数はすべて _キャンセル可能_ です。
これらはコルーチンのキャンセルをチェックし、キャンセルすると[CancellationException]をスローします。
ただし、コルーチンが計算で作業していて取り消しをチェックしていない場合、次の例のように取り消すことはできません。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) { // CPUを浪費するだけの計算ループ
            // 1秒間に2回メッセージをプリントする
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了するのを待つ
    println("main: Now I can quit.")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-02.kt)で完全なコードを取得できます

実行して、キャンセル後も "I'm sleeping" とプリントし続けることを確認します。

<!--- TEST
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
job: I'm sleeping 4 ...
main: Now I can quit.
-->

### 計算コードをキャンセル可能にする

計算コードをキャンセル可能にするには2つの方法があります。
1つは、キャンセルをチェックするサスペンド関数を定期的に呼び出すことです。
[yield]関数はその目的に適しています。
もう1つは、キャンセルステータスを明示的にチェックすることです。後者の方法を試してみましょう。

前の例の `while (i < 5)` を `while (isActive)` に置き換えて再実行してください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // キャンセル可能な計算ループ
            // 1秒間に2回メッセージをプリントする
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了するのを待つ
    println("main: Now I can quit.")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-03.kt)で完全なコードを取得できます

ご覧のとおり、このループはキャンセルされました。
[isActive]は、[CoroutineScope]オブジェクトを介してコルーチン内で使用できる拡張プロパティです。

<!--- TEST
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
-->

### `finally`でリソースを閉じる

キャンセル可能なサスペンド関数は、キャンセル時に通常の方法で処理できる[CancellationException]をスローします。
例えば、 `try {...} finally {...}` 式とKotlin `use` 関数は、コルーチンがキャンセルされたときに、通常の終了処理を実行します。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("job: I'm running finally")
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了を待つ
    println("main: Now I can quit.")
//sampleEnd
}
```

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-04.kt)で完全なコードを取得できます

[join][Job.join]も[cancelAndJoin]も、すべてのファイナライズ・アクションが完了するまで待機するため、上記の例では次の出力が生成されます。

```text
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
main: Now I can quit.
```

<!--- TEST -->

### キャンセル不可ブロックの実行

前の例の `finally` ブロックでサスペンド関数を使用しようとすると、このコードを実行しているコルーチンがキャンセルされるため[CancellationException]が発生します。
（ファイルを閉じる、ジョブをキャンセルする、またはあらゆる種類の通信チャネルを閉じるなど）正常に動作するすべてのクローズ操作は大抵ノンブロッキングであり、サスペンド関数は含まれないため、通常これは問題ではありません。
ただし、キャンセルされたコルーチンで中断する必要がある稀なケースでは、次の例のように [withContext]関数と [NonCancellable]コンテキストを使用して `withContext(NonCancellable) {...}` に対応するコードをラップすることができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了を待つ
    println("main: Now I can quit.")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-05.kt)で完全なコードを取得できます

<!--- TEST
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
job: And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
-->

### タイムアウト

コルーチンの実行をキャンセルする最も明白で実際的な理由は、コルーチンの実行時間がタイムアウトを超えたことによるものです。
対応する[Job]への参照を手動で追跡し、遅延の後で追跡されたものを取り消すために別のコルーチンを起動することはできますが、それを行う[withTimeout]関数が用意されています。
次の例を見てください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-06.kt)で完全なコードを取得できます

これは次の出力を生成します。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms
```

<!--- TEST STARTS_WITH -->

[withTimeout]によってスローされる `TimeoutCancellationException` は、[CancellationException]のサブクラスです。
先のコンソールにはスタックトレースが表示されていませんでした。
キャンセルされたコルーチン内の `CancellationException` がコルーチン完了の通常の理由であると考えられるためです。
しかし、この例では `main` 関数の中で `withTimeout` を使用しています。

キャンセルは単なる例外なので、すべてのリソースは通常の方法で閉じられます。
タイムアウト時に特に追加のアクションが必要な場合は、タイムアウトするコードを `try {...} catch (e: TimeoutCancellationException) {...}` ブロックでラップできます。または、[withTimeout]に似た[withTimeoutOrNull]関数を使用します。これは例外をスローする代わりにタイムアウト時に `null` を返します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // この結果が出る前にキャンセルされます
    }
    println("Result is $result")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-07.kt)で完全なコードを取得できます

このコードを実行しても例外は出ません。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

<!--- TEST -->

### 非同期タイムアウトとリソース

<!--
  NOTE: Don't change this section name. It is being referenced to from within KDoc of withTimeout functions.
-->

[withTimeout]のタイムアウトイベントは、そのブロックで実行されているコードに関して非同期であり、タイムアウトブロック内から戻る直前であっても、いつでも発生する可能性があります。
ブロック内でリソースを開いたり取得したりして、ブロック外で閉じるか解放する必要がある場合は、このことに注意してください。

たとえば、ここでは、 `Resource` クラスを使用してクローズ可能なリソースを模倣します。このクラスは、`acquired` カウンターをインクリメントし、このカウンターを `close` 関数からデクリメントすることで、作成された回数を追跡します。
小さなタイムアウトで多くのコルーチンを実行してみましょう。少し遅れて `withTimeout` ブロックの内側からこのリソースを取得し、外側から解放してみてください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

//sampleStart
var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch {
                val resource = withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    Resource() // Acquire a resource and return it from withTimeout block
                }
                resource.close() // Release the resource
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-08.kt)で完全なコードを取得できます

<!--- CLEAR -->

上記のコードを実行すると、常にゼロが出力されるとは限りません。マシンのタイミングによって異なる場合があり、実際にゼロ以外の値を表示するには、この例のタイムアウトを微調整する必要があります。

> ここでの `acquired` カウンターの10万コルーチンからのインクリメントとデクリメントは、常に同じメインスレッドから発生するため、完全に安全であることに注意してください。
> これについては、コルーチンのコンテキストに関する次の章で詳しく説明します。

この問題を回避するには、 `withTimeout` ブロックからリソースを返すのではなく、変数にリソースへの参照を格納します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
//sampleStart
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch {
                var resource: Resource? = null // Not acquired yet
                try {
                    withTimeout(60) { // Timeout of 60 ms
                        delay(50) // Delay for 50 ms
                        resource = Resource() // Store a resource to the variable if acquired
                    }
                    // We can do something else with the resource here
                } finally {
                    resource?.close() // Release the resource if it was acquired
                }
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-cancel-09.kt)で完全なコードを取得できます。

この例では、常にゼロが出力されます。 リソースがリークすることはありません。

<!--- TEST
0
-->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[cancelAndJoin]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/cancel-and-join.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-non-cancellable.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html
<!--- END -->
