<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/DispatcherGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class DispatchersGuideTest {
--> 

**目次**

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
  * [コルーチンスコープ](#コルーチンスコープ)
  * [スレッドローカルデータ](#スレッドローカルデータ)

<!--- END_TOC -->

## コルーチンコンテキストとディスパッチャー

コルーチンは、Kotlin標準ライブラリで定義されている[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/)型の値で表されるコンテキストで常に実行されます。

コルーチンコンテキストは、さまざまな要素のセットです。 主な要素は、以前に見たコルーチンの[Job]とこのセクションで取り上げるディスパッチャーです。

### ディスパッチャーとスレッド

コルーチンコンテキストには、対応するコルーチンが実行に使用するスレッドを決定する _コルーチンディスパッチャー_ （[CoroutineDispatcher]を参照）が含まれています。
コルーチンディスパッチャーは、コルーチンの実行を特定のスレッドに限定したり、スレッドプールにディスパッチしたり、制約なしで実行させたりすることができます。

[launch]や[async]のようなすべてのコルーチンビルダーは、新しいコルーチンとその他のコンテキスト要素のディスパッチャーを明示的に指定するために使用できるオプションの[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
パラメーターを受け入れます。

次の例を試してください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch { // 親(メインrunBlockingコルーチン)のコンテキスト
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // 制約なし -- メインスレッドで動作する
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // DefaultDispatcherにディスパッチされる
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // 独自の新しいスレッドを取得
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-01.kt)で完全なコードを取得できます

次の出力を生成します（もしかしたら違う順序かもしれません）。

```text
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

パラメーターなしで `launch {...}` を使用すると、起動元の[CoroutineScope]からコンテキスト（およびディスパッチャー）を継承します。
この場合、 `main` スレッドで動作するメイン `runBlocking` コルーチンのコンテキストを継承します。

[Dispatchers.Unconfined]は `main` スレッドで動作しているようにも見える特別なディスパッチャーですが、実際は後述する異なるメカニズムです。

[GlobalScope]でコルーチンが起動されるときに使用されるデフォルトのディスパッチャーは[Dispatchers.Default]で表され、共有バックグラウンドスレッドプールを使用するため、 `launch(Dispatchers.Default) {...}` は `GlobalScope.launch {...}` と同じディスパッチャーを使用します 。
  
[newSingleThreadContext]は、実行するコルーチンのスレッドを作成します。
専用のスレッドは非常に高価なリソースです。
実際のアプリケーションでは、不要になったときに[close][ExecutorCoroutineDispatcher.close]関数を使用して解放するか、トップレベル変数に格納してアプリケーション全体で再利用する必要があります。

### 制約なし対制約ディスパッチャー
 
[Dispatchers.Unconfined]コルーチンディスパッチャーは、最初の中断ポイントまで呼び出し元スレッドでコルーチンで実行します。
中断後、呼び出されたサスペンド関数によって完全に決定されるスレッド内でコルーチンを再開します。
制約のないディスパッチャーは、CPU時間を消費せず特定のスレッドに限定された共有データ（UIなど）を更新しないコルーチンに適しています。

一方、ディスパッチャーはデフォルトで外側の[CoroutineScope]から継承されます。
特に、[runBlocking]コルーチンのデフォルトディスパッチャーは呼び出し側スレッドに限定されているため、継承すると予測可能なFIFOスケジューリングを使用してこのスレッドに実行を限定するという効果があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-02.kt)で完全なコードを取得できます

次のように出力します。
 
```text
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

<!--- TEST LINES_START -->
 
このように、 `runBlocking {...}` から継承されたコンテキストを持つコルーチンは `main` スレッドで実行され続け、一方、制約のないものは[delay]関数が使用しているデフォルトのエグゼキュータースレッドで再開します。

> Unconfinedディスパッチャーは、コルーチンをすぐに実行する必要があるため、後で実行するためにコルーチンのディスパッチが不要であるか、望ましくない副作用を生じる稀なケースで役立つ高度なメカニズムです。
一般的なコードでは、Unconfinedディスパッチャーを使用しないでください。

### コルーチンとスレッドのデバッグ

コルーチンはあるスレッドで中断し、別のスレッドで再開できます。
シングルスレッドのディスパッチャーであっても、コルーチンがいつ、どこで、何を実行しているのか把握するのは難しいかもしれません。
スレッドを使用してアプリケーションをデバッグする一般的な方法は、ログファイルの各ログステートメントにスレッド名を出力することです。
この機能は、ロギングフレームワークによって普遍的にサポートされています。 コルーチンを使用する場合、スレッド名だけではコンテキストの多くが得られないので、 `kotlinx.coroutines` にはデバッグ機能が組み込まれています。

`-Dkotlinx.coroutines.debug` JVMオプションを付けて次のコードを実行してください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
//sampleStart
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-03.kt)で完全なコードを取得できます

3つのコルーチンがあります。 `runBlocking` 内のメインコルーチン (＃1) と、遅延値 `a` (#2) および `b` (#3) を計算する2つのコルーチン。
これらはすべて `runBlocking` のコンテキストで実行されており、メインスレッドに限定されています。
このコードの出力は次のとおりです。

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST FLEXIBLE_THREAD -->

`log` 関数はスレッドの名前を角括弧でプリントし、現在実行中のコルーチンの識別子が追加された `main` スレッドであることがわかります。
この識別子は、デバッグモードがオンの場合、作成されたすべてのコルーチンに連続して割り当てられます。

> JVMが `-ea` オプションを指定して実行されると、デバッグモードもオンになります。

デバッグ機能の詳細については、[DEBUG_PROPERTY_NAME]プロパティのドキュメントを参照してください。

### スレッド間のジャンプ

`-Dkotlinx.coroutines.debug` JVMオプションを付けて次のコードを実行してください（[デバッグ](#コルーチンとスレッドのデバッグ)を見てください）。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-04.kt)で完全なコードを取得できます

これはいくつかの新しいテクニックを実証しています。
ひとつは明示的に指定されたコンテキストで[runBlocking]を使用し、もうひとつは[withContext]関数を使用してコルーチンのコンテキストを変更しながら、同じコルーチンにとどまることが以下の出力でわかります。

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

この例では、Kotlin標準ライブラリの `use` 関数を使用して、[newSingleThreadContext]で作成されたスレッドが不要になったときに解放することに注目してください。

### コンテキストにおけるジョブ

コルーチンの[Job]はそのコンテキストの一部であり、 `coroutineContext[Job]` 式を使用して取得できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    println("My job is ${coroutineContext[Job]}")
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-05.kt)で完全なコードを取得できます

[デバッグモード](#コルーチンとスレッドのデバッグ)では、次のようなものが出力されます。

```
My job is "coroutine#1":BlockingCoroutine{Active}@6d311334
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is \"coroutine#1\":BlockingCoroutine{Active}@") -->

[CoroutineScope]の[isActive]は `coroutineContext[Job]?.isActive == true` の便利なショートカットです。

### コルーチンの子

コルーチンが別のコルーチンの[CoroutineScope]で起動されると、そのコルーチンは[CoroutineScope.coroutineContext]によってそのコンテキストを継承し、新しいコルーチンの[Job]は親コルーチンのジョブの _子_ になります。
親のコルーチンがキャンセルされると、すべての子が再帰的にキャンセルされます。

ただし、[GlobalScope]を使用してコルーチンを起動する場合、新しいコルーチンのジョブの親はありません。
したがって、起動されたスコープとは無関係であり、独立して動作します。
  

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    // リクエストを処理するためにコルーチンを起動する
    val request = launch {
        // 他の2つのジョブを生み出す。ひとつはGlobalScope
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
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-06.kt)で完全なコードを取得できます

このコードの出力は以下の通りです。

```text
job1: I run in GlobalScope and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### 親の責任

親コルーチンは、常にすべての子の完了を待機します。
親は、起動したすべての子を明示的に追跡する必要はなく、最後にそれらを待つために[Job.join]を使用する必要はありません。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
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
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-07.kt)で完全なコードを取得できます

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
[CoroutineName]コンテキスト要素は、スレッド名と同じ目的を果たします。
[デバッグモード](#コルーチンとスレッドのデバッグ)がオンになっている場合、このコルーチンを実行しているスレッド名に含まれます。

次の例は、この概念を示しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking(CoroutineName("main")) {
//sampleStart
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
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-08.kt)で完全なコードを取得できます

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
例えば、明示的に指定されたディスパッチャーと明示的に指定された名前を同時に使用してコルーチンを起動できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-09.kt)で完全なコードを取得できます。

`-Dkotlinx.coroutines.debug` JVMオプションを付けて実行したこのコードの出力は以下の通りです。

```text
I'm working in thread DefaultDispatcher-worker-1 @test#2
```

<!--- TEST -->

### コルーチンスコープ

コンテキスト、子、ジョブに関する知識をまとめてみましょう。
アプリケーションにライフサイクルを持つオブジェクトがあるとしますが、そのオブジェクトはコルーチンではありません。
例えば、Androidアプリケーションを作成し、Androidアクティビティのコンテキストでさまざまなコルーチンを起動して、データのフェッチや更新、アニメーションなどの非同期操作を実行します。
メモリリークを避けるためにアクティビティが破棄されると、これらのコルーチンはすべてキャンセルされなければなりません。

もちろん、コンテキストとジョブを手動で操作してアクティビティとそのコルーチンのライフサイクルを結び付けることができますが、`kotlinx.coroutines` は[CoroutineScope]をカプセル化する抽象化を提供します。
すべてのコルーチンビルダーはその拡張として宣言されているため、コルーチンスコープについてよく知っておく必要があります。

アクティビティのライフサイクルに関連付けられた[CoroutineScope]のインスタンスを作成することにより、コルーチンのライフサイクルを管理します。
`CoroutineScope` インスタンスは、[CoroutineScope()]または[MainScope()]ファクトリ関数によって作成できます。
前者は汎用スコープを作成し、後者はUIアプリケーションのスコープを作成し、デフォルトのディスパッチャーとして[Dispatchers.Main]を使用します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class Activity {
    private val mainScope = MainScope()
    
    fun destroy() {
        mainScope.cancel()
    }
    // 続く ...
```

</div>

または、この `Activity` クラスで[CoroutineScope]インターフェイスを実装できます。 それを行う最良の方法は、デフォルトのファクトリ関数でデリゲーションを使用することです。
また、目的のディスパッチャー（この例では[Dispatchers.Default]を使用）とスコープを組み合わせることもできます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
    class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {
    // 続く ...
```

</div>

これで、コンテキストを明示的に指定することなく、この `Activity` のスコープでコルーチンを起動できます。 
デモでは、異なる時間遅延する10個のコルーチンを起動します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

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

</div>

メイン関数では、アクティビティを作成し、テストの `doSomething` 関数を呼び出し、500ms後にアクティビティを破棄します。
起動されたすべてのコルーチンを取り消し、待っていてももうスクリーンに何もプリントされないことを確認できます。
これにより、　`doSomething` から起動されたすべてのコルーチンがキャンセルされます。 アクティビティを破棄した後、もう少し待ってもメッセージが出力されないことがわかります。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {

    fun destroy() {
        cancel() // CoroutineScopeの拡張関数
    }
    // 続く ...

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

fun main() = runBlocking<Unit> {
//sampleStart
    val activity = Activity()
    activity.doSomething() // テスト関数を実行
    println("Launched coroutines")
    delay(500L) // 0.5秒遅らせる
    println("Destroying activity!")
    activity.destroy() // 全てのコルーチンをキャンセルする
    delay(1000) // 動作していないことを視覚的に確認する
//sampleEnd    
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-10.kt)で完全なコードを取得できます

この例の出力は以下の通り

```text
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

<!--- TEST -->

ご覧のように、最初の2つのコルーチンだけがメッセージを出力し、他のコルーチンは `Activity.destroy()` で `job.cancel()` を1回呼び出すだけでキャンセルされました。

### スレッドローカルデータ

スレッドローカルデータをコルーチン間で受け渡しできると便利な場合があります。
ただし、それらは特定のスレッドにバインドされていないため、手動で行うとボイラープレートにつながります。

[`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)の場合、[asContextElement]拡張関数が役立ちます。
これは、指定された `ThreadLocal` の値を保持する追加のコンテキスト要素を作成し、コルーチンがコンテキストを切り替えるたびに復元します。

実際にそれを実証するのは簡単です。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

val threadLocal = ThreadLocal<String?>() // スレッドローカル変数を宣言する

fun main() = runBlocking<Unit> {
//sampleStart
    threadLocal.set("main")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
        println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
//sampleEnd    
}
```  

</div>                                                                                       

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-context-11.kt)で完全なコードを取得できます。

この例では、[Dispatchers.Default]を使用してバックグラウンドスレッドプールで新しいコルーチンを起動するため、コルーチンはスレッドプールの異なるスレッドで動作しますが、コルーチンが実行されるスレッドに関係なく `threadLocal.asContextElement(value = "launch")` で指定したスレッドローカル変数の値は保持されます。

したがって、出力（[debug](#コルーチンとスレッドのデバッグ)を用いる）は次のようになります。

```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

<!--- TEST FLEXIBLE_THREAD -->

対応するコンテキスト要素を設定することを忘れがちです。 コルーチンを実行しているスレッドが異なる場合、コルーチンからアクセスされるスレッドローカル変数に予期しない値が含まれることがあります。
このような状況を回避するには、[ensurePresent]メソッドを使用し、不適切な使用法ではフェイルファーストすることをお勧めします。

`ThreadLocal` はファーストクラスのサポートを持ち、`kotlinx.coroutines` が提供するプリミティブと一緒に使うことができます。
ただし、ひとつ重要な制限があります。スレッドローカルが変更されると、新しい値はコルーチンの呼び出し元に伝播されず（コンテキスト要素がすべての `ThreadLocal` オブジェクトへのアクセスを追跡できないため）、更新された値は次の中断時に失われます。
コルーチンのスレッドローカルの値を更新するには[withContext]を使用してください。詳細は[asContextElement]を参照してください。

あるいは、値を `class Counter(var i：Int)` のような可変ボックスに保存することもできます。この可変ボックスは、スレッドローカル変数に保存されます。
ただし、このケースでは、この可変ボックス内の変数に潜在的に並行して発生する変更を同期する責任があります。

ロギングMDC、トランザクションコンテキスト、またはデータを渡すために内部的にスレッドローカルを使用する他のライブラリとの統合などの高度な使用法については、実装する必要のある[ThreadContextElement]インターフェイスのドキュメントを参照してください。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[newSingleThreadContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html
[ExecutorCoroutineDispatcher.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-executor-coroutine-dispatcher/close.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[DEBUG_PROPERTY_NAME]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-d-e-b-u-g_-p-r-o-p-e-r-t-y_-n-a-m-e.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope.coroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-name/index.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
[MainScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-scope.html
[Dispatchers.Main]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html
[asContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.lang.-thread-local/as-context-element.html
[ensurePresent]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.lang.-thread-local/ensure-present.html
[ThreadContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-thread-context-element/index.html
<!--- END -->
