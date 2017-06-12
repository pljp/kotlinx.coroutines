<!--- INCLUDE .*/example-([a-z]+)-([0-9]+)\.kt 
/*
 * Copyright 2016-2017 JetBrains s.r.o.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package guide.$$1.example$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     kotlinx-coroutines-core/src/test/kotlin/guide/.*\.kt -->
<!--- TEST_OUT kotlinx-coroutines-core/src/test/kotlin/guide/test/GuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package guide.test

import org.junit.Test

class GuideTest {
--> 

# 実例によるkotlinx.coroutinesの手引き

これは `kotlinx.coroutines` の中核的機能についての短いガイドで、一連の例を示します。

## 導入とセットアップ

言語としてのKotlinは、他の様々なライブラリがコルーチンを利用できるようにするために標準ライブラリに最小限の低レベルAPIしか提供していません。 同様の機能を持つ他の多くの言語とは異なり、 `async` と `await` はKotlinのキーワードではなく、標準ライブラリの一部でもありません。

`kotlinx.coroutines` はそのような豊富なライブラリの1つです。これには、asyncとawaitを含む、このガイドで扱う高水準のコルーチンを可能にするプリミティブが含まれています。あなたのプロジェクトでこのガイドのプリミティブを使用するには、[ここ](README.md#in-your-projects)で説明する `kotlinx-coroutines-core` モジュールに依存関係を追加する必要があります。

## 目次

<!--- TOC -->

* [コルーチンの基礎](#コルーチンの基礎)
  * [初めてのコルーチン](#初めてのコルーチン)
  * [ブロッキングとノンブロッキングの世界の橋渡し](#ブロッキングとノンブロッキングの世界の橋渡し)
  * [ジョブを待つ](#ジョブを待つ)
  * [関数抽出リファクタリング](#関数抽出リファクタリング)
  * [コルーチンは軽量](#コルーチンは軽量)
  * [コルーチンはデーモンスレッドに似ている](#コルーチンはデーモンスレッドに似ている)
* [キャンセルとタイムアウト](#キャンセルとタイムアウト)
  * [コルーチンの実行をキャンセル](#コルーチンの実行をキャンセル)
  * [キャンセルは協調的](#キャンセルは協調的)
  * [計算コードをキャンセル可能にする](#計算コードをキャンセル可能にする)
  * [finallyでリソースを閉じる](#finallyでリソースを閉じる)
  * [キャンセル不可ブロックの実行](#キャンセル不可ブロックの実行)
  * [タイムアウト](#タイムアウト)
* [サスペンド関数の作成](#サスペンド関数の作成)
  * [デフォルトではシーケンシャル](#デフォルトではシーケンシャル)
  * [asyncを使用した並列動作](#asyncを使用した並列動作)
  * [遅延して開始されるasync](#遅延して開始されるasync)
  * [Asyncスタイル関数](#Asyncスタイル関数)
* [コルーチンコンテキストとディスパッチャー](#コルーチンコンテキストとディスパッチャー)
  * [ディスパッチャーとスレッド](#ディスパッチャーとスレッド)
  * [制約なし対制約ディスパッチャー](#制約なし対制約ディスパッチャー)
  * [コルーチンとスレッドのデバッグ](#コルーチンとスレッドのデバッグ)
  * [スレッド間のジャンプ](#スレッド間のジャンプ)
  * [コンテキストにおけるジョブ](#コンテキストにおけるジョブ)
  * [コルーチンの子](#コルーチンの子)
  * [コンテキストの結合](#コンテキストの結合)
  * [デバッグのためのコルーチンの命名](#デバッグのためのコルーチンの命名)
  * [明示的なジョブによるキャンセル](#明示的なジョブによるキャンセル)
* [チャネル](#チャネル)
  * [チャネルの基礎](#チャネルの基礎)
  * [チャネルのクローズと反復](#チャネルのクローズと反復)
  * [チャネルプロデューサーの作成](#チャネルプロデューサーの作成)
  * [パイプライン](#パイプライン)
  * [パイプラインによる素数](#パイプラインによる素数)
  * [論理出力数](#論理出力数)
  * [論理入力数](#論理入力数)
  * [バッファーされたチャネル](#バッファーされたチャネル)
  * [チャネルは公正](#チャネルは公正)
* [共有ミュータブルステートと並行性](#共有ミュータブルステートと並行性)
  * [問題](#問題)
  * [Volatileは助けにならない](#Volatileは助けにならない)
  * [スレッドセーフなデータ構造](#スレッドセーフなデータ構造)
  * [細粒度のスレッド拘束](#細粒度のスレッド拘束)
  * [粗粒度のスレッド拘束](#粗粒度のスレッド拘束)
  * [排他制御](#排他制御)
  * [アクター](#アクター)
* [セレクト式](#セレクト式)
  * [チャネルからの選択](#チャネルからの選択)
  * [クローズ時の選択](#クローズ時の選択)
  * [送信の選択](#送信の選択)
  * [延期された値の選択](#延期された値の選択)
  * [延期された値のチャネルの切り替え](#延期された値のチャネルの切り替え)
* [参考文献](#参考文献)

<!--- END_TOC -->

## コルーチンの基礎

このセクションでは、基本的なコルーチンの概念について説明します。

### 初めてのコルーチン

次のコードを実行します。

```kotlin
fun main(args: Array<String>) {
    launch(CommonPool) { // 共有スレッドプールに新しいコルーチンを作成する
        delay(1000L) // 1秒間ノンブロッキング遅延 (デフォルトの時間単位はms)
        println("World!") // delayのあとでプリント
    }
    println("Hello,") // コルーチンが遅延している間、メイン関数は継続する
    Thread.sleep(2000L) // メインスレッドを2秒間ブロックしてJVMを存続させます
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-01.kt)で完全なコードを取得できます。

このコードを実行します。

```text
Hello,
World!
```

<!--- TEST -->

基本的に、コルーチンは軽量スレッドです。
それらは[launch] _コルーチンビルダー_ で起動します。
`launch(CommonPool) { ... }` を `thread { ... }` に、 `delay(...)` を `Thread.sleep(...)` に置き換えても同じ結果が得られます。試してみてください。

`launch(CommonPool)` を `thread` に置き換えて起動すると、コンパイラは次のエラーを生成します。

```
Error: Kotlin: サスペンド関数は、コルーチンまたは他のサスペンド関数からのみ呼び出すことができます
```

これは、 [delay] がスレッドをブロックせず、コルーチンを _中断_ しコルーチンからのみ使用できる特別な _サスペンド関数_ であるためです。

### ブロッキングとノンブロッキングの世界の橋渡し

最初の例では、 `main` 関数の同じコード内に、_ノンブロッキング_ `delay(...)` と _ブロッキング_ `Thread.sleep(...)` を混在させています。
迷子になるのは簡単です。
[runBlocking]を使用して、ブロッキングとノンブロッキングの世界をきれいに分離しましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> { // メインコルーチンを開始
    launch(CommonPool) { // 共有スレッドプールに新しいコルーチンを作成する
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 子が遅延している間メインコルーチンは継続する
    delay(2000L) // 2秒間ノンブロッキング遅延してJVMを存続させる
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-02.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は同じですが、このコードではノンブロッキング[delay]のみを使用しています。

`runBlocking { ... }` は、トップレベルのメインコルーチンを起動するためにここで使用されるアダプタとして機能します。
`runBlocking` の外側の通常のコードは、 `runBlocking` 内部のコルーチンがアクティブになるまで _ブロック_ します。

これはまた、サスペンド関数の単体テストを書く方法でもあります。
 
```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // ここでは、好きなアサーションスタイルを使ってサスペンド関数を使うことができます
    }
}
```

<!--- CLEAR -->
 
### ジョブを待つ

別のコルーチンが動作している間遅延させるのは良い方法ではありません。
立ち上げたバックグラウンド[Job]が完了するまで明示的に（ノンブロッキングの方法で）待ちましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { // 新しいコルーチンを作成し、そのJobへの参照を保持する
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 子コルーチンが完了するまで待つ
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-03.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は変わりませんが、メインコルーチンのコードはバックグラウンドジョブの継続時間に結びついていません。ずっと良いです。

### 関数抽出リファクタリング

`launch(CommonPool) { ... }` の中のコードブロックを別の関数に抽出しましょう。
このコードで "Extract function"リファクタリングを実行すると、 `suspend` 修飾子付きの新しい関数が得られます。
それがあなたの最初の _サスペンド関数_ です。
サスペンド関数は、通常の関数と同様にコルーチン内で使用できますが、追加機能として、この例では `delay`のような他のサスペンド関数を使用してコルーチンの実行を _中断_ することができます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { doWorld() }
    println("Hello,")
    job.join()
}

// これはあなたの最初のサスペンド関数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-04.kt)で完全なコードを取得できます

<!--- TEST
Hello,
World!
-->

### コルーチンは軽量

次のコードを実行します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = List(100_000) { // コルーチンをたくさん作り、ジョブをリストする
        launch(CommonPool) {
            delay(1000L)
            print(".")
        }
    }
    jobs.forEach { it.join() } // すべてのジョブが完了するのを待つ
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-05.kt)で完全なコードを取得できます

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

10万個のコルーチンを開始し、1分後に各コルーチンがドットをプリントします。
スレッドを使って試したらどうなるでしょうか？ （ほとんどの場合、あなたのコードはメモリ不足エラーを引き起こすでしょう）

### コルーチンはデーモンスレッドに似ている

次のコードでは、「I'm sleeping」というメッセージを毎秒2回出力し、次にある程度遅れてmain関数からリターンする長期実行のコルーチンを起動します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(CommonPool) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 遅れて終了する
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-06.kt)で完全なコードを取得できます

実行すると、3行を出力して終了することがわかります。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

アクティブなコルーチンはプロセスを生かし続けるわけではありません。それらはデーモンスレッドのようなものです。

## キャンセルとタイムアウト

このセクションでは、コルーチンのキャンセルとタイムアウトについて説明します。

### コルーチンの実行をキャンセル

小さなアプリケーションでは、"main"メソッドからのリターンは、暗黙的にすべてのコルーチンを終了させる良い考えのように思えるかもしれません。
大規模で長時間実行されるアプリケーションでは、きめ細かな制御が必要です。
[launch]関数は、実行中のコルーチンを取り消すために使用できる[job]を返します。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    delay(1300L) // 本当にキャンセルされたことを確認するために少し遅らせる
    println("main: Now I can quit.")
}
``` 

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-01.kt)で完全なコードを取得できます

次の出力が生成されます。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

<!--- TEST -->

メインが `job.cancel` を呼び出すとすぐにキャンセルされるため他のコルーチンからの出力は表示されません。

### キャンセルは協調的

コルーチンのキャンセルは _協調的_ です。コルーチンコードは取り消し可能にするために協調しなければなりません。
`kotlinx.coroutines` のサスペンド関数はすべて _キャンセル可能_ です。
これらはコルーチンのキャンセルをチェックし、キャンセルすると[CancellationException]をスローします。
ただし、コルーチンが計算で作業していて取り消しをチェックしていない場合、次の例のように取り消すことはできません。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        var nextPrintTime = 0L
        var i = 0
        while (i < 10) { // 計算ループ
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime = currentTime + 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    delay(1300L) // キャンセルされたかどうか確かめるために少し遅らせる…
    println("main: Now I can quit.")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-02.kt)で完全なコードを取得できます

実行して、キャンセル後も "I'm sleeping" とプリントし続けることを確認します。

<!--- TEST 
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm sleeping 3 ...
I'm sleeping 4 ...
I'm sleeping 5 ...
main: Now I can quit.
-->

### 計算コードをキャンセル可能にする

計算コードをキャンセル可能にするには2つの方法があります。
1つは、定期的にサスペンド関数を呼び出すことです。
その目的のために良い選択肢である[yield]関数があります。
もう1つは、キャンセルステータスを明示的にチェックすることです。後者の方法を試してみましょう。

前の例の `while (true)` を `while (isActive)` に置き換えて再実行してください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        var nextPrintTime = 0L
        var i = 0
        while (isActive) { // キャンセル可能な計算ループ
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime = currentTime + 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    delay(1300L) // キャンセルされたかどうか確かめるために少し遅らせる…
    println("main: Now I can quit.")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-03.kt)で完全なコードを取得できます

ご覧のとおり、このループはキャンセルできます。
[isActive][CoroutineScope.isActive]は、[CoroutineScope]オブジェクトを介してコルーチンのコード内で使用できるプロパティです。

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
-->

### finallyでリソースを閉じる

キャンセル可能なサスペンド関数は、キャンセルの際に通常の方法で処理できるCancellationExceptionをスローします。
たとえば、 `try {...} finally {...}` とKotlin `use` 関数は、コルーチンがキャンセルされたときに、通常の終了処理を実行します。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("I'm running finally")
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    delay(1300L) // 本当にキャンセルされたことを確認するために少し遅らせる
    println("main: Now I can quit.")
}
``` 

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-04.kt)で完全なコードを取得できます

この例は次の出力を生成します。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
main: Now I can quit.
```

<!--- TEST -->

### キャンセル不可ブロックの実行

前の例の `finally` ブロックでサスペンド関数を使用しようとすると、このコードを実行しているコルーチンがキャンセルされるため[CancellationException]が発生します。
（ファイルを閉じる、ジョブをキャンセルする、またはあらゆる種類の通信チャネルを閉じるなど）正常に動作するすべてのクローズ操作は大抵ノンブロッキングであり、サスペンド関数は含まれないため、通常これは問題ではありません。
ただし、キャンセルされたコルーチンで中断する必要があるまれなケースでは、次の例のように [run]関数と [NonCancellable]コンテキストを使用して `run(NonCancellable) {...}` に対応するコードをラップすることができます。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            run(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
``` 

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-05.kt)で完全なコードを取得できます

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
-->

### タイムアウト

コルーチンの実行を実際にキャンセルする最も明白な理由は、その実行時間がタイムアウトを超えたためです。
対応する[Job]への参照を手動で追跡し、遅延の後で追跡されたものを取り消すために別のコルーチンを起動することはできますが、それを行う[withTimeout]関数が用意されています。
次の例を見てください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-06.kt)で完全なコードを取得できます

これは次の出力を生成します。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.experimental.TimeoutException: Timed out waiting for 1300 MILLISECONDS
```

<!--- TEST STARTS_WITH -->

[withTimeout]によってスローされる `TimeoutException` は、[CancellationException]のプライベートサブクラスです。
先のコンソールにはスタックトレースが表示されていませんでした。
キャンセルされたコルーチン内の `CancellationException` がコルーチン完了の通常の理由であると考えられるためです。
しかし、この例では `main` 関数の中で `withTimeout` を使用しています。

キャンセルは単なる例外なので、すべてのリソースは通常の方法で閉じられます。
タイムアウト時に特別なアクションを追加する必要がある場合は、 `try {...} catch (e：CancellationException) {...}` ブロックでタイムアウトを使ったコードをラップすることができます。

## サスペンド関数の作成

このセクションでは、サスペンド関数のさまざまな構成方法について説明します。

### デフォルトではシーケンシャル

何らかのリモートサービスコールや計算のように有用な何かを行う他の場所で定義された2つのサスペンド関数があるとします。 実際にはこの例の目的のためにそれぞれただ1秒遅らせるだけですが、これらは有用なものとしておきます。

<!--- INCLUDE .*/example-compose-([0-9]+).kt
import kotlin.system.measureTimeMillis
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

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-01.kt)で完全なコードを取得できます

これは次のような出力をします。

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST FLEXIBLE_TIME -->

### asyncを使用した並列動作

`doSomethingUsefulOne` と `doSomethingUsefulTwo` の呼び出しの間に依存関係がなく、両方を _同時_ に行うことでより速く答えを出したいのですが？
これは[async]が助けになる場面です。

概念的には、[async]は[launch]と同じです。
これは別のコルーチンを開始します。このコルーチンは、他のすべてのコルーチンと同時に動作する軽量スレッドです。
相違点は `launch` は[Job]を返し、結果の値は持ちませんが、 `async` は結果を後で提供する約束を表す軽量でノンブロッキングなフューチャーである[Deferred]を返します。
遅延された値に対して `.await()` を使用して最終的な結果を得ることができますが、 `Deferred` も `Job` なので必要に応じてキャンセルすることができます。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool) { doSomethingUsefulOne() }
        val two = async(CommonPool) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-02.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST FLEXIBLE_TIME -->

2つのコルーチンが同時に実行されるため、これは2倍高速です。
コルーチンの並行性は常に明示的であることに注意してください。

### 遅延して開始されるasync

[CoroutineStart.LAZY]パラメータを使用して[async]にするための遅延オプションがあります。
コルーチンは、 [await][Deferred.await]または [start][Job.start]関数が呼び出されたときにその結果が必要な場合にのみ開始されます。
前の例とこのオプションだけが異なる次の例を実行します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-03.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST FLEXIBLE_TIME -->

なんと、シーケンシャルな実行に戻ってしまいました。これは、最初に `one` を開始して待ってから、`two` を開始して待つためです。
遅延にとって意図されたユースケースではありません。
これは値の計算にサスペンド関数が含まれるときに、標準の `lazy` 関数に代わるものとして設計されています。

### Asyncスタイル関数

[async]コルーチンビルダーを使用して _非同期的_ に `doSomethingUsefulOne` と `doSomethingUsefulTwo` を呼び出すasyncスタイルの関数を定義できます。
"async"接頭辞か"Async"接尾辞を付けて、常に非同期で計算を開始し、その結果得られる遅延値を使用する必要があるという事実を強調するために、このような関数の名前を付けることは良いスタイルです。

```kotlin
// The result type of asyncSomethingUsefulOne is Deferred<Int>
fun asyncSomethingUsefulOne() = async(CommonPool) {
    doSomethingUsefulOne()
}

// The result type of asyncSomethingUsefulTwo is Deferred<Int>
fun asyncSomethingUsefulTwo() = async(CommonPool)  {
    doSomethingUsefulTwo()
}
```

これらの `asyncXXX` 関数は、_サスペンド_ 関数**ではない**ことに注意してください。
これらはどこからでも使用できます。
しかし、呼び出すコードのアクションは常に非同期（ここでは _並行実行_ を意味する）実行であること意味します。

次の例は、コルーチンの外部での使用例を示しています。
 
```kotlin
// この例では、 `main` の右側に `runBlocking` がないことに注意してください
fun main(args: Array<String>) {
    val time = measureTimeMillis {
        // we can initiate async actions outside of a coroutine
        val one = asyncSomethingUsefulOne()
        val two = asyncSomethingUsefulTwo()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-04.kt)で完全なコードを取得できます

<!--- TEST FLEXIBLE_TIME
The answer is 42
Completed in 1085 ms
-->

## コルーチンコンテキストとディスパッチャー

私たちはすでに `launch(CommonPool) {...}` 、 `async(CommonPool) {...}` 、 `run(NonCancellable) {...}` などを見てきました。
これらのコードスニペット[CommonPool]と[NonCancellable]は _コルーチンコンテキスト_ です。
このセクションでは、その他の選択肢について説明します。

### ディスパッチャーとスレッド

コルーチンコンテキストには、[_コルーチンディスパッチャ_][CoroutineDispatcher]が含まれており、対応するコルーチンが実行に使用するスレッド（単独または複数）を決定します。コルーチンディスパッチャは、コルーチンの実行を特定のスレッドに限定したり、スレッドプールにディスパッチしたり、制約なしで実行させたりすることができます。 次の例を試してください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // 制約なし -- メインスレッドで動作する
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // 親(runBlockingコルーチン)のコンテキスト
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(CommonPool) { // ForkJoinPool.commonPool(または同様なもの)にディスパッチされる
        println(" 'CommonPool': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(newSingleThreadContext("MyOwnThread")) { // 独自の新しいスレッドを取得
        println("     'newSTC': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-01.kt)で完全なコードを取得できます

次の出力を生成します（おそらく異なる順序で）。

```text
 'Unconfined': I'm working in thread main
 'CommonPool': I'm working in thread ForkJoinPool.commonPool-worker-1
     'newSTC': I'm working in thread MyOwnThread
    'context': I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

The difference between parent [context][CoroutineScope.context] and [Unconfined] context will be shown later.

親[コンテキスト][CoroutineScope.context]と[Unconfined]コンテキストの違いについては、後で説明します。

### 制約なし対制約ディスパッチャー
 
[Unconfined]コルーチンディスパッチャは、最初の中断ポイントまで呼び出し元スレッドでコルーチンで実行します。
中断後、呼び出されたサスペンド関数によって完全に決定されたスレッドで再開されます。
コルーチンがCPU時間を消費しない場合や、特定のスレッドに限定された共有データ（UIなど）を更新しない場合、Unconfinedディスパッチャが適切です。

一方、[CoroutineScope]インターフェイスを介してコルーチンのブロック内で使用できる[コンテキスト][CoroutineScope.context]プロパティは、この特定のコルーチンのコンテキストへの参照です。
このようにして、親コンテキストを継承することができます。
特に、[runBlocking]のデフォルトコンテキストは呼び出し側スレッドに限定されているため、継承すると予測可能なFIFOスケジューリングを使用してこのスレッドに実行を限定するという効果があります。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // 制約なし -- メインスレッドで動作する
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println(" 'Unconfined': After delay in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // 親(runBlockingコルーチン)のコンテキスト
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("    'context': After delay in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-02.kt)で完全なコードを取得できます

次のように出力します。
 
```text
 'Unconfined': I'm working in thread main
    'context': I'm working in thread main
 'Unconfined': After delay in thread kotlinx.coroutines.ScheduledExecutor
    'context': After delay in thread main
```

<!--- TEST LINES_START -->
 
このように、 `runBlocking {...}` コルーチンの `context` を継承したコルーチンは `main` スレッドで実行し続けますが、制約なしのほうはスレッドは[delay]関数が使用しているスケジューラスレッドで再開しました。

### コルーチンとスレッドのデバッグ

コルーチンは、[Unconfined]ディスパッチャまたは[CommonPool]のようなマルチスレッドディスパッチャを使用して、あるスレッドで中断し、別のスレッドで再開できます。
シングルスレッドのディスパッチャであっても、コルーチンが何を、いつ、どこでやっていたのか把握するのは難しいかもしれません。
スレッドを使用してアプリケーションをデバッグする一般的な方法は、ログファイルの各ログステートメントにスレッド名を出力することです。
この機能は、ロギングフレームワークによって普遍的にサポートされています。 コルーチンを使用する場合、スレッド名だけではコンテキストの多くが得られないので、 `kotlinx.coroutines`にはデバッグ機能が組み込まれています。

`-Dkotlinx.coroutines.debug` JVMオプションを付けて次のコードを実行してください。

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking<Unit> {
    val a = async(context) {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async(context) {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-03.kt)で完全なコードを取得できます

`runBlocking` のメインコルーチン（#1）と、遅延値を計算する2つのコルーチン `a` （#2）と、`b` （#3）の3つのコルーチンがあります。
これらはすべて `runBlocking` のコンテキストで実行されており、メインスレッドに限定されています。
このコードの出力は次のとおりです。

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST -->

`log` 関数はスレッドの名前を角括弧でプリントし、`main` スレッドであることがわかりますが、現在実行中のコルーチンの識別子が追加されています。
この識別子は、デバッグモードがオンのときに、作成されたすべてのコルーチンに連続して割り当てられます。

デバッグ機能の詳細については、[newCoroutineContext]関数のドキュメントを参照してください。

### スレッド間のジャンプ

`-Dkotlinx.coroutines.debug` JVMオプションで次のコードを実行してください。

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) {
    val ctx1 = newSingleThreadContext("Ctx1")
    val ctx2 = newSingleThreadContext("Ctx2")
    runBlocking(ctx1) {
        log("Started in ctx1")
        run(ctx2) {
            log("Working in ctx2")
        }
        log("Back to ctx1")
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-04.kt)で完全なコードを取得できます

これは2つの新しいテクニックを実証しています。
1つは明示的に指定されたコンテキストで[runBlocking]を使用し、もう1つは[run]関数を使用してコルーチンのコンテキストを変更しながら、同じコルーチンにとどまることが以下の出力でわかります。

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

### コンテキストにおけるジョブ

コルーチンの[Job]はそのコンテキストの一部です。
コルーチンは `context[Job]` 式を使ってそれ自身のコンテキストから取り出すことができます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${context[Job]}")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-05.kt)で完全なコードを取得できます

これは次のようなものを生成します

```
My job is BlockingCoroutine{Active}@65ae6ba4
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is BlockingCoroutine{Active}@") -->

[CoroutineScope]の[isActive][CoroutineScope.isActive]は `context[Job]!!.isActive` の便利なショートカットです。

### コルーチンの子

コルーチンの[context][CoroutineScope.context]を使用して別のコルーチンを起動すると、新しいコルーチンの[Job]は親コルーチンのジョブの _子_ になります。
親コルーチンがキャンセルされると、すべての子が再帰的にキャンセルされます。
  
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // 何らかのリクエストを処理するためにコルーチンを開始する
    val request = launch(CommonPool) {
        // 他の2つのジョブが生成される。1つは別のコンテキスト
        val job1 = launch(CommonPool) {
            println("job1: I have my own context and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // もう一方は親コンテキストを継承する
        val job2 = launch(context) {
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
        // 両方のサブジョブが完了したらリクエストは完了
        job1.join()
        job2.join()
    }
    delay(500)
    request.cancel() // リクエストの処理をキャンセル
    delay(1000) // 何が起こるか確かめるために1秒遅らせる
    println("main: Who has survived request cancellation?")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-06.kt)で完全なコードを取得できます

このコードの出力は以下の通りです。

```text
job1: I have my own context and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### コンテキストの結合

コルーチンのコンテキストは、 `+` 演算子を使って組み合わせることができます。
右側のコンテキストは、左側のコンテキストの関連エントリを置き換えます。
たとえば、親コルーチンの[Job]は継承でき、ディスパッチャは次のように置き換えられます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // 何らかのリクエストを処理するためにコルーチンを開始する
    val request = launch(context) { // `runBlocking` のコンテキストを使う
        // CommonPoolでCPU集約型の子ジョブを生成する !!!
        val job = launch(context + CommonPool) {
            println("job: I am a child of the request coroutine, but with a different dispatcher")
            delay(1000)
            println("job: I will not execute this line if my parent request is cancelled")
        }
        job.join() // サブジョブが完了したらリクエストは完了
    }
    delay(500)
    request.cancel() // リクエストの処理をキャンセル
    delay(1000) // 何が起こるか確かめるために1秒遅らせる
    println("main: Who has survived request cancellation?")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-07.kt)で完全なコードを取得できます

このコードの予想される結果は次のとおりです。

```text
job: I am a child of the request coroutine, but with a different dispatcher
main: Who has survived request cancellation?
```

<!--- TEST -->

### デバッグのためのコルーチンの命名

コルーチンが頻繁にログを記録し、同じコルーチンからのログレコードを相関させるだけでよい場合は、自動的に割り当てられたIDが有効です。
しかし、コルーチンが特定の要求の処理や特定のバックグラウンドタスクの処理に縛られている場合は、デバッグの目的で明示的に名前を付ける方がよいでしょう。
[CoroutineName]はスレッド名と同じ機能を果たします。
これは、デバッグが有効になっているときにこのコルーチンを実行しているスレッド名に表示されます。

次の例は、この概念を示しています。

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // 2つのバックグラウンド値の計算を実行する
    val v1 = async(CommonPool + CoroutineName("v1coroutine")) {
        log("Computing v1")
        delay(500)
        252
    }
    val v2 = async(CommonPool + CoroutineName("v2coroutine")) {
        log("Computing v2")
        delay(1000)
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-08.kt)で完全なコードを取得できます

`-Dkotlinx.coroutines.debug` JVMオプションで出力される結果は次のようになります。
 
```text
[main @main#1] Started main coroutine
[ForkJoinPool.commonPool-worker-1 @v1coroutine#2] Computing v1
[ForkJoinPool.commonPool-worker-2 @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

<!--- TEST FLEXIBLE_THREAD -->

### 明示的なジョブによるキャンセル

コンテキスト、子、ジョブに関する知識をまとめてみましょう。
アプリケーションにライフサイクルを持つオブジェクトがあるとしますが、そのオブジェクトはコルーチンではありません。
たとえば、Androidアプリケーションを作成し、Androidアクティビティのコンテキストでさまざまなコルーチンを起動して、データのフェッチや更新、アニメーションなどの非同期操作を実行します。
メモリリークを避けるためにアクティビティが破棄されると、これらのコルーチンはすべてキャンセルされなければなりません。

アクティビティのライフサイクルに結びついた[ジョブ]のインスタンスを作成することで、コルーチンのライフサイクルを管理することができます。
次の例に示すように、[Job()] [Job.invoke]ファクトリ関数を使用してジョブインスタンスが作成されます。
すべてのコルーチンがコンテキスト内でこのジョブで開始されていることを確認してから、[Job.cancel]を1回呼び出すとすべて終了します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = Job() // ライフサイクルを管理するジョブオブジェクトを作成する
    // デモ用に10個のコルーチンを起動し、それぞれ別の時間に動作する
    val coroutines = List(10) { i ->
        // これらはすべてジョブオブジェクトの子
        launch(context + job) { // メインrunBlockingスレッドのコンテキストを使用しますが、独自のジョブオブジェクトを使用します
            delay(i * 200L) // 可変の遅延 0ms, 200ms, 400ms, ... など
            println("Coroutine $i is done")
        }
    }
    println("Launched ${coroutines.size} coroutines")
    delay(500L) // 0.5秒遅延する
    println("Cancelling job!")
    job.cancel() // ジョブをキャンセル.. !!!
    delay(1000L) // コルーチンがまだ動作中かどうかを確かめるためにさらに遅延する
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-09.kt)で完全なコードを取得できます

この例の出力は次の通り

```text
Launched 10 coroutines
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Cancelling job!
```

<!--- TEST -->

ご覧のように、最初の3つのコルーチンだけがメッセージを出力し、他は `job.cancel()` の1回の呼び出しでキャンセルされました。
したがって、私たちが仮定しているAndroidアプリケーションでは、アクティビティが作成されたときに親ジョブオブジェクトを作成し、それを子コルーチンに使用し、アクティビティが破棄されたときにキャンセルするだけです。

## チャネル

遅延値は、コルーチン間で単一の値を転送する便利な方法を提供します。
チャネルは、ストリーム値を転送する方法を提供します。

<!--- INCLUDE .*/example-channel-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
-->

### チャネルの基礎

[Channel]は概念的には `BlockingQueue` に非常によく似ています。主な違いの1つは、ブロックする `put` 操作の代わりにサスペンド[send][SendChannel.send]、ブロックする `take` 操作の代わりにサスペンド[receive][ReceiveChannel.receive]を持っていることです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        // これはCPU使用量が多い計算や非同期ロジックかもしれないが、ここではただ5つの平方を送るだけ
        for (x in 1..5) channel.send(x * x)
    }
    // ここで受け取った5つの整数をプリントする
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-01.kt)で完全なコードを取得できます

このコードの出力は以下の通りです。

```text
1
4
9
16
25
Done!
```

<!--- TEST -->

### チャネルのクローズと反復

キューとは異なり、チャネルは閉じることによってそれ以上エレメントが来ないことを示すことができます。
受信側では、通常の `for` ループを使用してチャネルから要素を受け取ると便利です。

概念的には、[close][SendChannel.close]は特別なクローズトークンをチャネルに送信するようなものです。
このクローズトークンが受信されるとすぐに反復が停止し、クローズする前に以前に送信されたすべての要素が受信されるという保証があります。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        for (x in 1..5) channel.send(x * x)
        channel.close() // 送信完了
    }
    // ここでは `for` ループを使って受け取った値をプリントします（チャネルが閉じられるまで）
    for (y in channel) println(y)
    println("Done!")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-02.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

### チャネルプロデューサーの作成

コルーチンが要素のシーケンスを生成するパターンはかなり一般的です。
これは _プロデューサー - コンシューマー_ パターンの一部であり、コンカレントコードでよく見られます。
そのようなプロデューサーを、パラメータとしてchannelをとる関数に抽象化することはできますが、結果は関数から返さなければならないという常識とは逆になります。

プロデューサ側で簡単に行うことを容易にする[produce]という便利なコルーチンビルダーと、コンシューマ側の `for` ループを置き換えることができる拡張関数[consumeEach]があります。

```kotlin
fun produceSquares() = produce<Int>(CommonPool) {
    for (x in 1..5) send(x * x)
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-03.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

### パイプライン

パイプラインは、1つのコルーチンが無限の値のストリームを生成しているパターンです。

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1
    while (true) send(x++) // 1から始まる整数の無限ストリーム
}
```

そして、別のコルーチンはそのストリームを消費し、いくつかの処理を行い、他の結果を生成しています。
以下の例では、数値は単に二乗されます。

```kotlin
fun square(numbers: ReceiveChannel<Int>) = produce<Int>(CommonPool) {
    for (x in numbers) send(x * x)
}
```

メインコードはパイプライン全体を開始し接続します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val numbers = produceNumbers() // 1からの整数を生成する
    val squares = square(numbers) // 整数を平方にする
    for (i in 1..5) println(squares.receive()) // 最初の5つをプリントする
    println("Done!") // 完了
    squares.cancel() // より大きなアプリではこれらのコルーチンをキャンセルする必要があります
    numbers.cancel()
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-04.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

[コルーチンはデーモンスレッドに似ている](#コルーチンはデーモンスレッドに似ている)ため、このサンプルアプリケーションではこれらのコルーチンをキャンセルする必要はありませんが、より大きなアプリケーションではパイプラインが必要なくなった場合に停止する必要があります。
あるいは、パイプラインコルーチンを[コルーチンの子](#コルーチンの子)として実行することもできます。

### パイプラインによる素数

コルーチンのパイプラインを使って素数を生成する例を使って、パイプラインを徹底的に見てみましょう。無限の数列から始めます。今回は明示的なコンテキストパラメータを導入して、呼び出し元がコルーチンの実行を制御できるようにします。
 
<!--- INCLUDE kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt  
import kotlin.coroutines.experimental.CoroutineContext
-->
 
```kotlin
fun numbersFrom(context: CoroutineContext, start: Int) = produce<Int>(context) {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```

次のパイプラインステージでは、入力数列をフィルタリングして、指定された素数で割り切れるすべての数値を削除します。

```kotlin
fun filter(context: CoroutineContext, numbers: ReceiveChannel<Int>, prime: Int) = produce<Int>(context) {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

2からの数列を開始し、現在のチャネルから素数を取り出し、見つかった各素数に対して新しいパイプラインステージを開始することでパイプラインを構築します。
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
``` 
 
次の例では最初の10個の素数を出力し、パイプライン全体をメインスレッドのコンテキストで実行します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    var cur = numbersFrom(context, 2)
    for (i in 1..10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(context, cur, prime)
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt)で完全なコードを取得できます

このコードの出力です。

```text
2
3
5
7
11
13
17
19
23
29
```

<!--- TEST -->

標準ライブラリの `buildIterator` コルーチンビルダーを使って同じパイプラインを構築できることに留意してください。
`produce` を `buildIterator`、 `send` を `yield`、 `receive` を `next`、 `ReceiveChannel` を `Iterator` で置き換え、コンテキストを取り除きます。 `runBlocking` も必要ありません。
ただし、上記のようなチャネルを使用するパイプラインの利点は、[CommonPool]コンテキストで実行すると実際に複数のCPUコアを使用できることです。

とにかく、これは素数を見つけるには非常に非実用的な方法です。 実際にはパイプラインは（リモートサービスへの非同期呼び出しのような）いくつかの他のサスペンド呼び出しを必要とします。これらのパイプラインは完全に非同期の `produce` とは異なり任意の中断を許さないため、`buildSeqeunce` / `buildIterator` を使用して構築することはできません。
 
### 論理出力数

複数のコルーチンが同じチャネルから受信し、それらの間で作業を分散することがあります。
定期的に整数（毎秒10個の数値）を生成するプロデューサーコルーチンから始めましょう。

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1 // 1から始める
    while (true) {
        send(x++) // 次を生成
        delay(100) // 0.1秒待つ
    }
}
```

いくつかのプロセッサコルーチンを持つことができます。この例では、IDと受け取った数値をプリントします。

```kotlin
fun launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch(CommonPool) {
    channel.consumeEach {
        println("Processor #$id received $it")
    }    
}
```

今度は5つのプロセッサを起動して、1秒間動作させましょう。何が起こるか確かめてください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(1000)
    producer.cancel() // cancel producer coroutine and thus kill them all
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-06.kt)で完全なコードを取得できます

プロセッサIDとして受け取るそれぞれの固有の整数は異なる可能性がありますが、出力は次のようになります。

```
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

<!--- TEST lines.size == 10 && lines.withIndex().all { (i, line) -> line.startsWith("Processor #") && line.endsWith(" received ${i + 1}") } -->

プロデューサのコルーチンをキャンセルするとそのチャネルが閉じられ、最終的にプロセッサのコルーチンが行っているチャネルでの繰り返しが終了することに注意してください。

### 論理入力数

複数のコルーチンが同じチャネルに送信することがあります。
たとえば、文字列のチャネルと、指定された文字列を指定された遅延でこのチャネルに繰り返し送信するサスペンド関数を持っています。

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

さて、文字列を送信するコルーチンをいくつか起動するとどうなるか見てみましょう（この例ではメインスレッドのコンテキストで起動します）。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<String>()
    launch(context) { sendString(channel, "foo", 200L) }
    launch(context) { sendString(channel, "BAR!", 500L) }
    repeat(6) { // receive first six
        println(channel.receive())
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-07.kt)で完全なコードを取得できます

出力は、

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### バッファーされたチャネル

今までに示されたチャネルにはバッファーがありませんでした。 バッファーされていないチャネルは、送信側と受信側がお互いに出会ったときに要素を転送します（別名ランデブー）。 sendが最初に呼び出された場合、receiveが呼び出されるまで中断されます。receiveが最初に呼び出された場合、sendが呼び出されるまで中断されます。

[Channel()] [Channel.invoke]ファクトリ関数と[produce]ビルダーは、_バッファーサイズ_ を指定するためのオプションの `capacity` パラメータをとります。 バッファーは、指定された容量を持つ `BlockingQueue` と同様に、送信側が中断する前に複数の要素を送信できるようにします。これはバッファーがいっぱいになるとブロックします。

次のコードの動作を見てみましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>(4) // バッファーされたチャネルを作る
    launch(context) { // 送信側コルーチンを起動
        repeat(10) {
            println("Sending $it") // 各要素を送信する前にプリント
            channel.send(it) // バッファーがいっぱいになったら中断する
        }
    }
    // 何も受け取らずに待つ...
    delay(1000)
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-08.kt)で完全なコードを取得できます

これは容量 _4_ のバッファーされたチャネルを使って _5_ 回 "sending"を表示します。

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

最初の4つの要素はバッファーに追加され、5番目の要素を送信しようとすると送信側は中断します。


### チャネルは公正

チャネルへの操作の送信と受信は、複数のコルーチンからの呼び出しの順番に関して _公正_ です。 
それらはファーストイン・ファーストアウトの順序で提供されます。例えば `receive` を呼び出す最初のコルーチンは要素を取得します。
次の例では、2つのコルーチン"ping"と"pong"が共有"table"チャネルから"ball"オブジェクトを受け取ります。

```kotlin
data class Ball(var hits: Int)

fun main(args: Array<String>) = runBlocking<Unit> {
    val table = Channel<Ball>() // 共有テーブル
    launch(context) { player("ping", table) }
    launch(context) { player("pong", table) }
    table.send(Ball(0)) // ボールを供給する
    delay(1000) // 1秒遅らせる
    table.receive() // ゲームオーバー。ボールを掴む
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // ループでボールを受け取る
        ball.hits++
        println("$name $ball")
        delay(300) // 少し待つ
        table.send(ball) // ボールを戻す
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-09.kt)で完全なコードを取得できます

"ping"コルーチンが最初に開始されるので、ボールを受け取るのは最初のコルーチンです。 "ping"コルーチンは、ボールをテーブルに戻した後すぐに再びボールを受け取るようになっていますが、ボールは既に受信を待っていた"pong"コルーチンによって受信さます。

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
ping Ball(hits=5)
```

<!--- TEST -->

## 共有ミュータブルステートと並行性

コルーチンは、[CommonPool]のようなマルチスレッドディスパッチャを使用して同時に実行できます。 これは、すべての通常の並行性の問題を提起します。
主な問題は、**共有ミュータブルステート**へのアクセスの同期です。
コルーチンの世界でのこの問題に対するいくつかの解決策は、マルチスレッドの世界の解決策と似ていますが、他は独自のものです。

### 問題

千のコルーチンを同じように千回実行してみましょう（100万回の実行）。
さらなる比較のために完了時間も測定します。

<!--- INCLUDE .*/example-sync-([0-9]+).kt
import kotlin.coroutines.experimental.CoroutineContext
import kotlin.system.measureTimeMillis
-->

<!--- INCLUDE .*/example-sync-03.kt
import java.util.concurrent.atomic.AtomicInteger
-->

<!--- INCLUDE .*/example-sync-06.kt
import kotlinx.coroutines.experimental.sync.Mutex
-->

<!--- INCLUDE .*/example-sync-07.kt
import kotlinx.coroutines.experimental.channels.*
-->

```kotlin
suspend fun massiveRun(context: CoroutineContext, action: suspend () -> Unit) {
    val n = 1000 // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch(context) {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

<!--- INCLUDE .*/example-sync-([0-9]+).kt -->

まず、マルチスレッド化された[CommonPool]コンテキストを使用して、共有ミュータブル変数をインクリメントする非常に単純なアクションから始めます。

```kotlin
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-01.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

最後に何がプリントされますか？ 千のコルーチンが同期なしで複数のスレッドから同時に `counter` をインクリメントするため、「Counter = 1000000」をプリントすることはほとんどありません。

### Volatileは助けにならない

変数を `volatile` にすると並行性の問題が解決されるという誤解が一般的です。 それを試してみましょう。

```kotlin
@Volatile // Kotlinの `volatile` はアノテーション
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-02.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

このコードはより遅く動作しますが、volatile変数は対応する変数の線形（専門用語で「アトミック」）読み書きを保証するものの、より大きなアクション（この場合はインクリメント）のアトミック性を提供しないため、最後に「Counter = 1000000」を得られません。 

### スレッドセーフなデータ構造

スレッドとコルーチンの両方で機能する一般的な解決策は、共有状態で実行する必要があるすべての操作で必ず同期を提供するスレッドセーフ（別名、同期、線形化、またはアトミック）データ構造を使用することです。
単純なカウンタの場合、アトミックな `incrementAndGet` 操作を持つ `AtomicInteger` クラスを使うことができます。

```kotlin
var counter = AtomicInteger()

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-03.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

これは、この特定の問題に対する最速の解決策です。 単純なカウンター、コレクション、キュー、その他の標準的なデータ構造とそれらの基本的な操作では機能します。 ただし、複雑な状態やすぐに使用できるスレッドセーフな実装を持たない複雑な操作には、容易に拡張できません。

### 細粒度のスレッド拘束

_スレッド拘束_ は、特定の共有状態へのすべてのアクセスが1つのスレッドに限定されている、共有ミュータブルステートの問題への提案です。
これは通常、すべてのUI状態が単一のイベントディスパッチ/アプリケーションスレッドに限定されるUIアプリケーションで使用されます。 単一スレッドのコンテキストを使用してコルーチンで簡単に適用できます。

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) { // 各コルーチンをCommonPoolで実行する
        run(counterContext) { // それぞれのインクリメントを単一スレッドのコンテキストに限定する
            counter++
        }
    }
    println("Counter = $counter")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-04.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

このコードは _細粒度_ のスレッド拘束を行うため、非常にゆっくりと動作します。
個々のインクリメントは [run] ブロックを使用してマルチスレッドの `CommonPool` コンテキストからシングルスレッドのコンテキストに切り替わります。

### 粗粒度のスレッド拘束

現実にはスレッド拘束は大きなチャンクで行われます。例えば、状態を更新するビジネスロジックの大きな部分は単一のスレッドに限定されます。
次の例では、そのようにしてシングルスレッドコンテキストで各コルーチンを起動して実行します。

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(counterContext) { // 各コルーチンをシングルスレッドコンテキストで実行する
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-05.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

これで、はるかに高速に動作し正しい結果が得られます。

### 排他制御

この問題に対する排他制御の解決策は、決して同時に実行されない _クリティカルセクション_ で共有状態のすべての変更を保護することです。
ブロックする世界では通常 `synchronized` または `ReentrantLock` を使用します。
コルーチンの代案は[Mutex]と呼ばれています。
それはクリティカルセクションを区切る[lock][Mutex.lock]と[unlock][Mutex.unlock]関数を持っています。
主な違いは、 `Mutex.lock` はサスペンド関数であることです。
これはスレッドをブロックしません。

```kotlin
val mutex = Mutex()
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        mutex.lock()
        try { counter++ }
        finally { mutex.unlock() }
    }
    println("Counter = $counter")
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-06.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

この例でのロックは細粒度なので、代償を払っています。
しかし、共有状態を定期的に変更しなければならない状況には適していますが、この状態が限定された自然なスレッドはありません。

### アクター

アクターは、コルーチン、このコルーチンに閉じ込められカプセル化された状態、および他のコルーチンと通信するためのチャネルの組み合わせです。
単純なアクターは関数として記述できますが、複雑な状態のアクターはクラスに適しています。

[actor]コルーチンビルダーがアクターのメールボックスチャネルをメッセージを受信するスコープに結合し、
結果のジョブオブジェクトに送信チャネルを結合するので、アクターへの単一の参照をそのハンドルとして持ち運ぶことができます。

```kotlin
// counterActorのメッセージ型
sealed class CounterMsg
object IncCounter : CounterMsg() // カウンターをインクリメントする一方向のメッセージ
class GetCounter(val response: SendChannel<Int>) : CounterMsg() // 返信を持ったリクエスト

// この関数は、新しいカウンタアクタを起動する
fun counterActor() = actor<CounterMsg>(CommonPool) {
    var counter = 0 // アクターの状態
    for (msg in channel) { // 受信メッセージを反復処理する
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.send(counter)
        }
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val counter = counterActor() // アクターを作る
    massiveRun(CommonPool) {
        counter.send(IncCounter)
    }
    val response = Channel<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.receive()}")
    counter.close() // アクターを終了する
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-07.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

アクター自体がどのようなコンテキストで実行されるかは（正確さにおいて）問題ではありません。
アクターはコルーチンでありコルーチンは順番に実行されるので、特定のコルーチンへの状態の閉じ込めは共有ミュータブルステートの問題に対する解決策として機能します。

この場合、常に実行する作業があり別のコンテキストに切り替える必要がないため、負荷の下ではロックよりもアクターのほうが効率的です。

> [actor]コルーチンビルダーは二重の[produce]コルーチンビルダーであることに注意してください。
  アクターはメッセージを受信するチャネルに関連付けられ、プロデューサーは要素を送信するチャネルに関連付けられます。

## セレクト式

セレクト式を使用すると複数のサスペンド関数を同時に待つことができ、利用可能になった最初のものを _選択_ することができます。

<!--- INCLUDE .*/example-select-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.selects.*
-->

### チャネルからの選択

2つの文字列のプロデューサー、 `fizz` と `buzz` があります。 `fizz` は300ミリ秒ごとに"Fizz"文字列を生成します。

<!--- INCLUDE .*/example-select-01.kt
import kotlin.coroutines.experimental.CoroutineContext
-->

```kotlin
fun fizz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // 300ミリ秒ごとに "Fizz" を送る
        delay(300)
        send("Fizz")
    }
}
```

`buzz` は500ミリ秒ごとに "Buzz!" 文字列を生成します。

```kotlin
fun buzz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // 500ミリ秒ごとに "Buzz!" を送る
        delay(500)
        send("Buzz!")
    }
}
```

[receive][ReceiveChannel.receive]サスペンド関数を使用すると、一方のチャネルから _または_ 他方のチャネルから受信することができます。
しかし、[select]式は、[onReceive][SelectBuilder.onReceive]節を使って _両方_ から同時に受け取ることができます。

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
    val fizz = fizz(context)
    val buzz = buzz(context)
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-01.kt)で完全なコードを取得できます

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

チャネルが閉じられ、対応する `select` が例外をスローすると、 `select` の[onReceive][SelectBuilder.onReceive]節は失敗します。
[onReceiveOrNull][SelectBuilder.onReceiveOrNull]節を使用して、チャネルが閉じられたときに特定のアクションを実行できます。
次の例は、 `select` が選択された節の結果を返す式であることも示しています。

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
    // この例ではメインスレッドのコンテキストを予測可能性のために使用する...
    val a = produce<String>(context) { 
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String>(context) { 
        repeat(4) { send("World $it") }
    }
    repeat(8) { // 最初の8個の結果をプリントする
        println(selectAorB(a, b))
    }
}
```

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-02.kt)で完全なコードを取得できます

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

2番目の所見は、[onReceiveOrNull][SelectBuilder.onReceiveOrNull]は、チャネルが既に閉じられているときに直ちに選択されることです。

### 送信の選択

Select式には、[onSend][SelectBuilder.onSend]節があり、選択のバイアスされた性質と組み合わせてとても有効に使用できます。

プライマリチャネルのコンシューマーが送信に追いつかないときに、その値を `side` チャネルに送る整数のプロデューサーの例を書きましょう。

```kotlin
fun produceNumbers(side: SendChannel<Int>) = produce<Int>(CommonPool) {
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
    launch(context) { // これはサイドチャネルの非常に高速なコンシューマー
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // 急がずに、消費した値をきっちり消化する
    }
    println("Done consuming")
}
``` 
 
> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-03.kt)で完全なコードを取得できます
  
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

遅延値は、[onAwait][SelectBuilder.onAwait]節を使用して選択できます。
ランダム遅延の後に遅延文字列値を返す非同期関数から始めましょう。

<!--- INCLUDE .*/example-select-04.kt
import java.util.*
-->

```kotlin
fun asyncString(time: Int) = async(CommonPool) {
    delay(time.toLong())
    "Waited for $time ms"
}
```

ランダムな遅延でこれを1ダース開始してみましょう。

```kotlin
fun asyncStringsList(): List<Deferred<String>> {
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

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-04.kt)で完全なコードを取得できます

出力は、

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 延期された値のチャネルの切り替え

次の遅延値が来るかチャネルが閉じられるまで、遅延ストリング値のチャネルを消費し、受信した遅延値を待つチャネルプロデューサー関数を書きましょう。
この例では、同じ `select` に[onReceiveOrNull][SelectBuilder.onReceiveOrNull]節と[onAwait] [SelectBuilder.onAwait]節を入れています。

```kotlin
fun switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String>(CommonPool) {
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
fun asyncString(str: String, time: Long) = async(CommonPool) {
    delay(time)
    str
}
```

メイン関数は、単に `switchMapDeferreds` の結果をプリントするコルーチンを起動し、いくつかのテストデータを送信するだけです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val chan = Channel<Deferred<String>>() // テスト用のチャネル
    launch(context) { // プリント用のコルーチンを起動する
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

> [ここ](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-05.kt)で完全なコードを取得できます

このコードの結果は次の通りです。

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

## 参考文献

* [Guide to UI programming with coroutines](ui/coroutines-guide-ui.md)
* [Guide to reactive streams with coroutines](reactive/coroutines-guide-reactive.md)
* [Coroutines design document (KEEP)](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
* [Full kotlinx.coroutines API reference](http://kotlin.github.io/kotlinx.coroutines)

<!--- SITE_ROOT https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core -->
<!--- DOCS_ROOT kotlinx-coroutines-core/target/dokka/kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/yield.html
[CoroutineScope.isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/is-active.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html
[run]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-non-cancellable/index.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/start.html
[CommonPool]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-common-pool/index.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-dispatcher/index.html
[CoroutineScope.context]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/context.html
[Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-unconfined/index.html
[newCoroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-coroutine-context.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-name/index.html
[Job.invoke]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/invoke.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/cancel.html
<!--- INDEX kotlinx.coroutines.experimental.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/unlock.html
<!--- INDEX kotlinx.coroutines.experimental.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/send.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/receive.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/close.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/consume-each.html
[Channel.invoke]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/invoke.html
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/actor.html
<!--- INDEX kotlinx.coroutines.experimental.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/select.html
[SelectBuilder.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-receive.html
[SelectBuilder.onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-receive-or-null.html
[SelectBuilder.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-send.html
[SelectBuilder.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-await.html
<!--- END -->
