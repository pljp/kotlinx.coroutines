<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/DispatcherGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class DispatchersGuideTest {
--> 

## 目次

<!--- TOC -->

* [コルーチンコンテキストとディスパッチャー](#コルーチンコンテキストとディスパッチャー)
  * [ディスパッチャーとスレッド](#ディスパッチャーとスレッド)
  * [制約なし対制約ディスパッチャー](#制約なし対制約ディスパッチャー)
  * [コルーチンとスレッドのデバッグ](#コルーチンとスレッドのデバッグ)
  * [スレッド間のジャンプ](#スレッド間のジャンプ)
  * [コンテキストにおけるジョブ](#コンテキストにおけるジョブ)
  * [コルーチンの子](#コルーチンの子)
  * [親の責任](#親の責任)
  * [デバッグのためのコルーチンの命名](#デバッグのためのコルーチンの命名)
  * [コンテキスト要素の結合](#コンテキスト要素の結合)
  * [明示的なジョブによるキャンセル](#明示的なジョブによるキャンセル)
  * [スレッドローカルデータ](#スレッドローカルデータ)

<!--- END_TOC -->

## コルーチンコンテキストとディスパッチャー

コルーチンは、Kotlin標準ライブラリで定義されている[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/-coroutine-context/)型の値で表される何らかのコンテキストで常に実行されます。
コルーチンコンテキストは、さまざまな要素のセットです。 主な要素は、以前に見たコルーチンの[Job]とこのセクションで取り上げたディスパッチャーです。


### ディスパッチャーとスレッド

コルーチンコンテキストには、対応するコルーチンが実行に使用するスレッドを決定する _コルーチンディスパッチャー_ （[CoroutineDispatcher]を参照）が含まれています。
コルーチンディスパッチャーは、コルーチンの実行を特定のスレッドに限定したり、スレッドプールにディスパッチしたり、制約なしで実行させたりすることができます。

[launch]や[async]のようなすべてのコルーチンビルダーは、新しいコルーチンとその他のコンテキスト要素のディスパッチャーを明示的に指定するために使用できるオプションの[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/-coroutine-context/) 
パラメーターを受け入れます。

次の例を試してください。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch { // 親(メインrunBlockingコルーチン)のコンテキスト
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // 制約なし -- メインスレッドで動作する
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // ForkJoinPool.commonPool(または同様なもの)にディスパッチされる
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // 独自の新しいスレッドを取得
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-01.kt)で完全なコードを取得できます

次の出力を生成します（もしかしたら違う順序かもしれません）。

```text
Unconfined            : I'm working in thread main
Default               : I'm working in thread CommonPool-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

`launch { ... }` がパラメーターなしで使用されるとき、それは起動されている[CoroutineScope]からコンテキスト（そして結果としてディスパッチャ）を継承します。 この場合、 `main` スレッドで動作するメイン `runBlocking` コルーチンのコンテキストを継承します。

[Dispatchers.Unconfined]は `main` スレッドで動作しているようにも見える特別なディスパッチャですが、実際は後述する異なるメカニズムです。

[GlobalScope]でコルーチンを起動するときに使用されるデフォルトのディスパッチャは、[Dispatchers.Default]で表され、共有バックグラウンド スレッドプールを使用するため、 `launch（Dispatchers.Default）{...}` は `GlobalScope.launch {...}` と同じディスパッチャを使用します。
  
[newSingleThreadContext]は、コルーチンを実行するための新しいスレッドを作成します。
専用のスレッドは非常に高価なリソースです。
実際のアプリケーションでは、不要になったときに[close][ThreadPoolDispatcher.close]関数を使用して解放するか、トップレベル変数に格納してアプリケーション全体で再利用する必要があります。

### 制約なし対制約ディスパッチャー
 
[Dispatchers.Unconfined]コルーチンディスパッチャは、最初の中断ポイントまで呼び出し元スレッドでコルーチンで実行します。
中断後、呼び出されたサスペンド関数によって完全に決定されたスレッドで再開されます。
コルーチンがCPU時間を消費しない場合や、特定のスレッドに限定された共有データ（UIなど）を更新しない場合、Unconfinedディスパッチャが適切です。

一方、デフォルトでは、外側の[CoroutineScope]のディスパッチャは継承されています。
特に、[runBlocking]コルーチンのデフォルトディスパッチャーは呼び出し側スレッドに限定されているため、継承すると予測可能なFIFOスケジューリングを使用してこのスレッドに実行を限定するという効果があります。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // 制約なし -- メインスレッドで動作する
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // 親(メインrunBlockingコルーチン)のコンテキスト
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-02.kt)で完全なコードを取得できます

次のように出力します。
 
```text
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

<!--- TEST LINES_START -->
 
このように、 `runBlocking {...}` コルーチンのコンテキストを継承したコルーチンは `main` スレッドで実行し続けますが、制約なしのほうはスレッドは[delay]関数が使用しているデフォルトエグゼキューターのスレッドで再開しました。

> Unconfinedディスパッチャは、コルーチンをすぐに実行する必要があるため、後で実行するためにコルーチンのディスパッチが不要であるか、望ましくない副作用を生じる稀なケースで役立つ高度なメカニズムです。
一般的なコードでは、Unconfinedディスパッチャを使用しないでください。

### コルーチンとスレッドのデバッグ

コルーチンはあるスレッドで中断し、別のスレッドで再開できます。
シングルスレッドのディスパッチャであっても、コルーチンがいつ、どこで、何を実行しているのか把握するのは難しいかもしれません。
スレッドを使用してアプリケーションをデバッグする一般的な方法は、ログファイルの各ログステートメントにスレッド名を出力することです。
この機能は、ロギングフレームワークによって普遍的にサポートされています。 コルーチンを使用する場合、スレッド名だけではコンテキストの多くが得られないので、 `kotlinx.coroutines` にはデバッグ機能が組み込まれています。

`-Dkotlinx.coroutines.debug` JVMオプションを付けて次のコードを実行してください。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking<Unit> {
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-03.kt)で完全なコードを取得できます

`runBlocking` のメインコルーチン（#1）と、遅延値を計算する2つのコルーチン `a` （#2）と、`b` （#3）の3つのコルーチンがあります。
これらはすべて `runBlocking` のコンテキストで実行されており、メインスレッドに限定されています。
このコードの出力は次のとおりです。

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST FLEXIBLE_THREAD -->

`log` 関数はスレッドの名前を角括弧でプリントし、`main` スレッドであることがわかりますが、現在実行中のコルーチンの識別子が追加されています。
この識別子は、デバッグモードがオンのときに、作成されたすべてのコルーチンに連続して割り当てられます。

デバッグ機能の詳細については、[newCoroutineContext]関数のドキュメントを参照してください。

### スレッド間のジャンプ

`-Dkotlinx.coroutines.debug` JVMオプションを付けて次のコードを実行してください（[デバッグ](#コルーチンとスレッドのデバッグ)を見てください）。

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) {
    newSingleThreadContext("Ctx1").use { ctx1 ->
        newSingleThreadContext("Ctx2").use { ctx2 ->
            runBlocking(ctx1) {
                log("Started in ctx1")
                withContext(ctx2) {
                    log("Working in ctx2")
                }
                log("Back to ctx1")
            }
        }
    }
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-04.kt)で完全なコードを取得できます

これはいくつかの新しいテクニックを実証しています。
1つは明示的に指定されたコンテキストで[runBlocking]を使用し、もう1つは[withContext]関数を使用してコルーチンのコンテキストを変更しながら、同じコルーチンにとどまることが以下の出力でわかります。

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

この例では、Kotlin標準ライブラリの `use` 関数を使ってもう必要でなくなったときに[newSingleThreadContext]で作成されたスレッドを解放していることに注目してください。

### コンテキストにおけるジョブ

コルーチンの[Job]はそのコンテキストの一部です。
コルーチンは `coroutineContext[Job]` 式を使ってそれ自身のコンテキストから取り出すことができます。

<!--- INCLUDE  
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-05.kt)で完全なコードを取得できます

[デバッグモード](#コルーチンとスレッドのデバッグ)で実行すると、次のようなものが生成されます。

```
My job is "coroutine#1":BlockingCoroutine{Active}@6d311334
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is \"coroutine#1\":BlockingCoroutine{Active}@") -->

[CoroutineScope]の[isActive]は `coroutineContext[Job]?.isActive == true` の便利なショートカットです。

### コルーチンの子

コルーチンが別のコルーチンの[CoroutineScope]で起動されると、そのコルーチンは[CoroutineScope.coroutineContext]によってそのコンテキストを継承し、新しいコルーチンの[Job]は親コルーチンのジョブの _子_ になります。
親のコルーチンがキャンセルされると、すべての子が再帰的にキャンセルされます。

ただし、[GlobalScope]を使用してコルーチンを起動すると、起動されたスコープに結びついておらず、独立して動作します。
  
<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // 何らかのリクエストを処理するためにコルーチンを起動する
    val request = launch {
        // 他の2つのジョブを生み出す。1つはGlobalScope
        GlobalScope.launch {
            println("job1: I run in GlobalScope and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // もう一方は親コンテキストを継承する
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    delay(500)
    request.cancel() // リクエストの処理をキャンセル
    delay(1000) // 何が起こるか確かめるために1秒遅らせる
    println("main: Who has survived request cancellation?")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-06.kt)で完全なコードを取得できます

このコードの出力は以下の通りです。

```text
job1: I run in GlobalScope and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### 親の責任

親コルーチンは常にすべての子の完了を待ちます。
親は起動するすべての子を明示的に追跡する必要はなく、これらを待つために最後に[Job.join]を使用する必要はありません。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // 要求を処理するためにコルーチンを起動する
    val request = launch {
        repeat(3) { i -> // 小数の子ジョブを起動する
            launch  {
                delay((i + 1) * 200L) // 可変の遅延 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() // すべての子を含む要求の完了を待つ
    println("Now processing of the request is complete")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-07.kt)で完全なコードを取得できます

結果は次のようになります。

```text
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```

<!--- TEST -->

### デバッグのためのコルーチンの命名

コルーチンが頻繁にログを記録し、同じコルーチンからのログレコードを相関させるだけでよい場合は、自動的に割り当てられたIDが有効です。
しかし、コルーチンが特定の要求の処理や特定のバックグラウンドタスクの処理に縛られている場合は、デバッグの目的で明示的に名前を付ける方がよいでしょう。
[CoroutineName]コンテキスト要素はスレッド名と同じ機能を果たします。
これは、[デバッグモード](#コルーチンとスレッドのデバッグ)が有効になっているときにこのコルーチンを実行しているスレッド名に表示されます。

次の例は、この概念を示しています。

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // 2つのバックグラウンド値の計算を実行する
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-08.kt)で完全なコードを取得できます

`-Dkotlinx.coroutines.debug` JVMオプションで出力される結果は次のようになります。
 
```text
[main @main#1] Started main coroutine
[main @v1coroutine#2] Computing v1
[main @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

<!--- TEST FLEXIBLE_THREAD -->

### コンテキスト要素の結合

時々、コルーチンコンテキストのために複数の要素を定義する必要があります。これに `+` 演算子を使うことができます。
例えば、同時にディスパッチャと名前を明示的に指定したコルーチンを起動することができます。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-09.kt)で完全なコードを取得できます。

`-Dkotlinx.coroutines.debug` JVMオプションを付けて実行したこのコードの出力は以下の通りです

```text
I'm working in thread CommonPool-worker-1 @test#2
```

<!--- TEST -->

### 明示的なジョブによるキャンセル

コンテキスト、子、ジョブに関する知識をまとめてみましょう。
アプリケーションにライフサイクルを持つオブジェクトがあるとしますが、そのオブジェクトはコルーチンではありません。
例えば、Androidアプリケーションを作成し、Androidアクティビティのコンテキストでさまざまなコルーチンを起動して、データのフェッチや更新、アニメーションなどの非同期操作を実行します。
メモリリークを避けるためにアクティビティが破棄されると、これらのコルーチンはすべてキャンセルされなければなりません。

アクティビティのライフサイクルに結びついた[Job]のインスタンスを作成することで、コルーチンのライフサイクルを管理します。
ジョブインスタンスは、アクティビティが作成されたときに[Job()]ファクトリ関数を使用して作成され、次のようにアクティビティが破棄されたときにキャンセルされます。


<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
class Activity : CoroutineScope {
    lateinit var job: Job

    fun create() {
        job = Job()
    }

    fun destroy() {
        job.cancel()
    }
    // 続く ...
```

この `Actvity` クラスでは、[CoroutineScope]インターフェイスを実装しています。 
スコープで起動されたコルーチンのコンテキストを指定するために、[CoroutineScope.coroutineContext]プロパティのオーバーライドを行うだけで済みます。
目的のディスパッチャ（この例では[Dispatchers.Default]を使用）とジョブを結合します。

```kotlin
    // Activityクラスの続き
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job
    // 続く ...
```

これで、コンテキストを明示的に指定することなく、この `Activity` のスコープでコルーチンを起動することができます。
デモでは、異なる時間遅延する10個のコルーチンを起動します。

```kotlin
    // Activityクラスの続き
    fun doSomething() {
        // デモのために10個のコルーチンを起動し、それぞれ別の時間作業する
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // 可変の遅延 200ms, 400ms, ... など
                println("Coroutine $i is done")
            }
        }
    }
} // Activityクラス終わり
``` 

メイン関数では、アクティビティを作成し、テストの `doSomething` 関数を呼び出し、500ms後にアクティビティを破棄します。
起動されたすべてのコルーチンを取り消し、待っていてももうスクリーンに何もプリントされないことを確認できます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val activity = Activity()
    activity.create() // アクティビティを作成
    activity.doSomething() // テスト関数を実行
    println("Launched coroutines")
    delay(500L) // 0.5秒遅らせる
    println("Destroying activity!")
    activity.destroy() // 全てのコルーチンをキャンセルする
    delay(1000) // 動作していないことを視覚的に確認する
}
```

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-10.kt)で完全なコードを取得できます

この例の出力は以下の通り

```text
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

<!--- TEST -->

ご覧のように、最初の2つのコルーチンだけがメッセージを出力し、他のコルーチンは `Activity.destroy()` の `job.cancel()` を1回呼び出すだけでキャンセルされました。

### スレッドローカルデータ

スレッドローカルデータを渡す機能を持たせるのが便利な場合もありますが、特定のスレッドにバインドされていないコルーチンの場合は、多くの定型文を記述することなく手動で達成することは困難です。

[`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)に対して、[asContextElement]拡張関数が助けになります。
これは、与えられた `ThreadLocal` の値を保持しコルーチンがコンテキストを切り替えるたびに復元する追加のコンテキスト要素を作成します。

実際にそれを実証するのは簡単です。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
val threadLocal = ThreadLocal<String?>() // スレッドローカル変数を宣言する

fun main(args: Array<String>) = runBlocking<Unit> {
    threadLocal.set("main")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
       println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
}
```                                                                                         

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-context-11.kt)で完全なコードを取得できます。

この例では、[Dispatchers.Default]のスレッドプールでバックグラウンドの新しいコルーチンを起動するのでコルーチンはスレッドプールの異なるスレッドで動作しますが、コルーチンがどのスレッドで実行されても `threadLocal.asContextElement(value = "launch")` で指定したスレッドローカル変数の値は保持しています。
したがって、出力（[debug](#コルーチンとスレッドのデバッグ)を用いる）は次のようになります。

```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[CommonPool-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[CommonPool-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

<!--- TEST FLEXIBLE_THREAD -->

`ThreadLocal` はファーストクラスのサポートを持ち、`kotlinx.coroutines` が提供するプリミティブと一緒に使うことができます。
これには1つの重要な制限があります。スレッドローカルが変更された場合、新しい値はコルーチンの呼び出し元に伝播されず（コンテキスト要素はすべての `ThreadLocal` オブジェクトへのアクセスを追跡できないため）、更新された値は次の一時停止時に失われます。
コルーチンのスレッドローカルの値を更新するには[withContext]を使用してください。詳細は[asContextElement]を参照してください。

あるいは、 `class Counter(var i: Int)` のような変更可能なボックスに値を格納することもできます。これはスレッドローカル変数に格納されます。
ただし、このケースでは、この変更可能なボックス内の変数に潜在的に同時に発生する変更を同期する責任があります。

高度な使い方、例えば、ロギングMDC、トランザクションコンテキスト、またはデータを渡すためにスレッドローカルを内部的に使用する他のライブラリとの統合などについては、実装する必要がある[ThreadContextElement]インターフェイスのドキュメントを参照してください。
