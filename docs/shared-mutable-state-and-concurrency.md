<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/SharedStateGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class SharedStateGuideTest {
--> 
**目次**

<!--- TOC -->

* [共有ミュータブルステートと並行処理](#共有ミュータブルステートと並行処理)
  * [問題](#問題)
  * [Volatileは役に立たない](#volatileは役に立たない)
  * [スレッドセーフなデータ構造](#スレッドセーフなデータ構造)
  * [細粒度のスレッド制約](#細粒度のスレッド制約)
  * [粗粒度のスレッド制約](#粗粒度のスレッド制約)
  * [排他制御](#排他制御)
  * [アクター](#アクター)

<!--- END_TOC -->

## 共有ミュータブルステートと並行処理

コルーチンは、[Dispatchers.Default]のようなマルチスレッドディスパッチャーを使用して並行に実行できます。
これは、すべての通常の並行処理の問題を提起します。
主な問題は、**共有ミュータブルステート**へのアクセスの同期です。
コルーチンの世界でのこの問題に対するいくつかの解決策は、マルチスレッドの世界の解決策と似ていますが、他は独自のものです。

### 問題

1000のコルーチンを同じように100回実行してみましょう。
さらなる比較のために完了時間も測定します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

</div> 

マルチスレッドの[Dispatchers.Default]を使用して共有ミュータブル変数をインクリメントする非常に単純なアクションから始めます。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}
//sampleStart
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd    
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-01.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

最後に何をプリントしますか？ 100個のコルーチンが同期せずに複数のスレッドから並行して `counter` をインクリメントするため、"Counter = 100000" を出力することはほとんどありません。

### Volatileは役に立たない

変数を `volatile` にすると並行処理の問題が解決されるという誤解が一般的です。 それを試してみましょう。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        coroutineScope { // scope for coroutines 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
@Volatile // Kotlinの `volatile` はアノテーション
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd    
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-02.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

このコードはより遅く動作しますが、volatile変数は対応する変数の線形（専門用語で「アトミック」）読み書きを保証するものの、より大きなアクション（この場合はインクリメント）のアトミック性を提供しないため、最後に「Counter = 100000」を得られません。 

### スレッドセーフなデータ構造

スレッドとコルーチンの両方で機能する一般的な解決策は、共有状態で実行する必要があるすべての操作で必ず同期を提供するスレッドセーフ（別名、同期、線形化、またはアトミック）データ構造を使用することです。
単純なカウンタの場合、アトミックな `incrementAndGet` 操作を持つ `AtomicInteger` クラスを使うことができます。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
var counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
//sampleEnd    
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-03.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

これは、この特定の問題に対する最速の解決策です。 単純なカウンター、コレクション、キュー、その他の標準的なデータ構造とそれらの基本的な操作では機能します。 ただし、複雑な状態やすぐに使用できるスレッドセーフな実装を持たない複雑な操作には、容易に拡張できません。

### 細粒度のスレッド制約

_スレッド制約_ は、特定の共有状態へのすべてのアクセスが1つのスレッドに限定されている、共有ミュータブルステートの問題への提案です。
これは通常、すべてのUI状態が単一のイベントディスパッチ/アプリケーションスレッドに限定されるUIアプリケーションで使用されます。 単一スレッドのコンテキストを使用してコルーチンで簡単に適用できます。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // それぞれのインクリメントをシングルスレッドコンテキストに限定する
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
//sampleEnd      
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-04.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

このコードは _細粒度_ のスレッド制約を行うため、非常にゆっくりと動作します。
個々のインクリメントは[withContext(counterContext)][withContext]ブロックを使用してマルチスレッドの[Dispatchers.Default]コンテキストからシングルスレッドのコンテキストに切り替わります。

### 粗粒度のスレッド制約

現実にはスレッド制約は大きなチャンクで行われます。例えば、状態を更新するビジネスロジックの大きな部分は単一のスレッドに限定されます。
次の例では、そのようにしてシングルスレッドコンテキストで各コルーチンを起動して実行します。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    // すべてをシングルスレッドコンテキストに限定する
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd     
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-05.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

これで、はるかに高速に動作し正しい結果が得られます。

### 排他制御

この問題に対する排他制御の解決策は、決して並行に実行されない _クリティカルセクション_ で共有状態のすべての変更を保護することです。
ブロックする世界では通常 `synchronized` または `ReentrantLock` を使用します。
コルーチンの代案は[Mutex]と呼ばれています。
それはクリティカルセクションを区切る[lock][Mutex.lock]と[unlock][Mutex.unlock]関数を持っています。
主な違いは、 `Mutex.lock()` はサスペンド関数であることです。
これはスレッドをブロックしません。

`mutex.lock(); try { ... } finally { mutex.unlock() }` パターンを表す便利な[withLock]拡張関数もあります。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // ロックで各インクリメントを保護する
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
//sampleEnd    
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-06.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

この例でのロックは細粒度なので、代償を払っています。
ただし、一部の共有状態を定期的に変更する必要がある場合に適していますが、この状態に限定される自然なスレッドはありません。

### アクター

[actor](https://en.wikipedia.org/wiki/Actor_model)は、コルーチン、このコルーチンに閉じ込められカプセル化された状態、および他のコルーチンと通信するためのチャンネルの組み合わせで構成されるエンティティです 。
単純なアクターは関数として記述できますが、複雑な状態のアクターはクラスに適しています。

[actor]コルーチンビルダーがアクターのメールボックスチャネルをメッセージを受信するスコープに結合し、
結果のジョブオブジェクトに送信チャネルを結合するので、アクターへの単一の参照をそのハンドルとして持ち運ぶことができます。

アクターを使用する最初のステップは、アクターが処理するメッセージのクラスを定義することです。
Kotlinの[シールドクラス](https://kotlinlang.org/docs/reference/sealed-classes.html)はその目的に適しています。
カウンタをインクリメントする `IncCounter` メッセージと、その値を取得する `GetCounter` メッセージを持つ `CounterMsg` シールドクラスを定義します。
後で応答を送信する必要があります。 将来知られる（通信される）単一の値を表す[CompletableDeferred]通信プリミティブは、その目的のためにここで使用されます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// counterActorのメッセージ型
sealed class CounterMsg
object IncCounter : CounterMsg() // カウンターをインクリメントする一方向のメッセージ
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 返信を持ったリクエスト
```

</div>

次に、[actor]コルーチンビルダーを使用してアクターを起動する関数を定義します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// この関数は、新しいカウンタアクターを起動する
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // アクターの状態
    for (msg in channel) { // 受信メッセージを反復処理する
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

</div>

メインコードは簡単です。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        coroutineScope { // コルーチンのスコープ
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// counterActorのメッセージ型
sealed class CounterMsg
object IncCounter : CounterMsg() // カウンターをインクリメントする一方向メッセージ
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 返信付きのリクエスト

// この関数は、新しいカウンターアクターを起動する
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // アクターの状態
    for (msg in channel) { // 着信メッセージを反復処理する
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

//sampleStart
fun main() = runBlocking {
    val counter = counterActor() // アクターを作る
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    // アクターからカウンター値を得るためのメッセージを送る
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // アクターを終了する
}
//sampleEnd    
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-sync-07.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

アクター自体がどのようなコンテキストで実行されるかは（正確さにおいて）問題ではありません。
アクターはコルーチンであり、コルーチンはシーケンシャルに実行されるため、状態を特定のコルーチンに限定することは、共有ミュータブルステートの問題の解決策として機能します。
実際、アクターは自分のプライベートな状態を変更できますが、メッセージを介してのみ相互に影響を与えることができます（ロックの必要性を回避します）。

この場合、常に実行する作業があり別のコンテキストに切り替える必要がないため、負荷の下ではロックよりもアクターのほうが効率的です。

> [actor]コルーチンビルダーは二重の[produce]コルーチンビルダーであることに注意してください。
  アクターはメッセージを受信するチャネルに関連付けられ、プロデューサーは要素を送信するチャネルに関連付けられます。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
<!--- END -->
