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
  * [非限定対限定ディスパッチャー](#非限定対限定ディスパッチャー)
  * [コルーチンとスレッドのデバッグ](#コルーチンとスレッドのデバッグ)
  * [スレッド間のジャンプ](#スレッド間のジャンプ)
  * [コンテキストにおけるジョブ](#コンテキストにおけるジョブ)
  * [コルーチンの子](#コルーチンの子)
  * [コンテキストの結合](#コンテキストの結合)
  * [デバッグのためのコルーチンの命名](#デバッグのためのコルーチンの命名)
  * [明示的なジョブのキャンセル](#明示的なジョブのキャンセル)
* [チャネル](#チャネル)
  * [チャネルの基礎](#チャネルの基礎)
  * [チャネルでのクローズと反復](#チャネルでのクローズと反復)
  * [チャネルプロデューサの作成](#チャネルプロデューサの作成)
  * [パイプライン](#パイプライン)
  * [パイプラインによる素数](#パイプラインによる素数)
  * [出力数](#出力数)
  * [入力数](#入力数)
  * [バッファされたチャネル](#バッファされたチャネル)
  * [チャネルは公正](#チャネルは公正)
* [共有されたミュータブルステートと並行性](#共有されたミュータブルステートと並行性)
  * [問題](#問題)
  * [Volatileは助けにならない](#Volatileは助けにならない)
  * [スレッドセーフなデータ構造](#スレッドセーフなデータ構造)
  * [細かい処理のスレッドへの閉じ込め](#細かい処理のスレッドへの閉じ込め)
  * [粗粒の処理のスレッドへの閉じ込め](#粗粒の処理のスレッドへの閉じ込め)
  * [排他制御](#排他制御)
  * [アクター](#アクター)
* [セレクト式](#セレクト式)
  * [チャネルからの選択](#チャネルからの選択)
  * [クローズの選択](#クローズの選択)
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
    delay(1300L) // それが本当にキャンセルされたことを確認するために少し遅れさせる
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

実行して、キャンセル後も「私は眠っています」とプリントし続けることを確認します。

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
[isActive] [CoroutineScope.isActive]は、[CoroutineScope]オブジェクトを介してコルーチンのコード内で使用できるプロパティです。

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

## Coroutine context and dispatchers

We've already seen `launch(CommonPool) {...}`, `async(CommonPool) {...}`, `run(NonCancellable) {...}`, etc.
In these code snippets [CommonPool] and [NonCancellable] are _coroutine contexts_. 
This section covers other available choices.

### Dispatchers and threads

Coroutine context includes a [_coroutine dispatcher_][CoroutineDispatcher] which determines what thread or threads 
the corresponding coroutine uses for its execution. Coroutine dispatcher can confine coroutine execution 
to a specific thread, dispatch it to a thread pool, or let it run unconfined. Try the following example:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // context of the parent, runBlocking coroutine
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(CommonPool) { // will get dispatched to ForkJoinPool.commonPool (or equivalent)
        println(" 'CommonPool': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("     'newSTC': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-01.kt)

It produces the following output (maybe in different order):

```text
 'Unconfined': I'm working in thread main
 'CommonPool': I'm working in thread ForkJoinPool.commonPool-worker-1
     'newSTC': I'm working in thread MyOwnThread
    'context': I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

The difference between parent [context][CoroutineScope.context] and [Unconfined] context will be shown later.

### Unconfined vs confined dispatcher
 
The [Unconfined] coroutine dispatcher starts coroutine in the caller thread, but only until the
first suspension point. After suspension it resumes in the thread that is fully determined by the
suspending function that was invoked. Unconfined dispatcher is appropriate when coroutine does not
consume CPU time nor updates any shared data (like UI) that is confined to a specific thread. 

On the other side, [context][CoroutineScope.context] property that is available inside the block of any coroutine 
via [CoroutineScope] interface, is a reference to a context of this particular coroutine. 
This way, a parent context can be inherited. The default context of [runBlocking], in particular,
is confined to be invoker thread, so inheriting it has the effect of confining execution to
this thread with a predictable FIFO scheduling.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println(" 'Unconfined': After delay in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // context of the parent, runBlocking coroutine
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("    'context': After delay in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-02.kt)

Produces the output: 
 
```text
 'Unconfined': I'm working in thread main
    'context': I'm working in thread main
 'Unconfined': After delay in thread kotlinx.coroutines.ScheduledExecutor
    'context': After delay in thread main
```

<!--- TEST LINES_START -->
 
So, the coroutine that had inherited `context` of `runBlocking {...}` continues to execute in the `main` thread,
while the unconfined one had resumed in the scheduler thread that [delay] function is using.

### Debugging coroutines and threads

Coroutines can suspend on one thread and resume on another thread with [Unconfined] dispatcher or 
with a multi-threaded dispatcher like [CommonPool]. Even with a single-threaded dispatcher it might be hard to
figure out what coroutine was doing what, where, and when. The common approach to debugging applications with 
threads is to print the thread name in the log file on each log statement. This feature is universally supported
by logging frameworks. When using coroutines, the thread name alone does not give much of a context, so 
`kotlinx.coroutines` includes debugging facilities to make it easier. 

Run the following code with `-Dkotlinx.coroutines.debug` JVM option:

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-03.kt)

There are three coroutines. The main coroutine (#1) -- `runBlocking` one, 
and two coroutines computing deferred values `a` (#2) and `b` (#3).
They are all executing in the context of `runBlocking` and are confined to the main thread.
The output of this code is:

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST -->

The `log` function prints the name of the thread in square brackets and you can see, that it is the `main`
thread, but the identifier of the currently executing coroutine is appended to it. This identifier 
is consecutively assigned to all created coroutines when debugging mode is turned on.

You can read more about debugging facilities in the documentation for [newCoroutineContext] function.

### Jumping between threads

Run the following code with `-Dkotlinx.coroutines.debug` JVM option:

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-04.kt)

It demonstrates two new techniques. One is using [runBlocking] with an explicitly specified context, and
the second one is using [run] function to change a context of a coroutine while still staying in the 
same coroutine as you can see in the output below:

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

### Job in the context

The coroutine [Job] is part of its context. The coroutine can retrieve it from its own context 
using `context[Job]` expression:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${context[Job]}")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-05.kt)

It produces somethine like

```
My job is BlockingCoroutine{Active}@65ae6ba4
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is BlockingCoroutine{Active}@") -->

So, [isActive][CoroutineScope.isActive] in [CoroutineScope] is just a convenient shortcut for `context[Job]!!.isActive`.

### Children of a coroutine

When [context][CoroutineScope.context] of a coroutine is used to launch another coroutine, 
the [Job] of the new coroutine becomes
a _child_ of the parent coroutine's job. When the parent coroutine is cancelled, all its children
are recursively cancelled, too. 
  
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(CommonPool) {
        // it spawns two other jobs, one with its separate context
        val job1 = launch(CommonPool) {
            println("job1: I have my own context and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        val job2 = launch(context) {
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
        // request completes when both its sub-jobs complete:
        job1.join()
        job2.join()
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-06.kt)

The output of this code is:

```text
job1: I have my own context and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### Combining contexts

Coroutine context can be combined using `+` operator. The context on the right-hand side replaces relevant entries
of the context on the left-hand side. For example, a [Job] of the parent coroutine can be inherited, while 
its dispatcher replaced:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(context) { // use the context of `runBlocking`
        // spawns CPU-intensive child job in CommonPool !!! 
        val job = launch(context + CommonPool) {
            println("job: I am a child of the request coroutine, but with a different dispatcher")
            delay(1000)
            println("job: I will not execute this line if my parent request is cancelled")
        }
        job.join() // request completes when its sub-job completes
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-07.kt)

The expected outcome of this code is: 

```text
job: I am a child of the request coroutine, but with a different dispatcher
main: Who has survived request cancellation?
```

<!--- TEST -->

### Naming coroutines for debugging

Automatically assigned ids are good when coroutines log often and you just need to correlate log records
coming from the same coroutine. However, when coroutine is tied to the processing of a specific request
or doing some specific background task, it is better to name it explicitly for debugging purposes.
[CoroutineName] serves the same function as a thread name. It'll get displayed in the thread name that
is executing this coroutine when debugging more is turned on.

The following example demonstrates this concept:

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // run two background value computations
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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-08.kt)

The output it produces with `-Dkotlinx.coroutines.debug` JVM option is similar to:
 
```text
[main @main#1] Started main coroutine
[ForkJoinPool.commonPool-worker-1 @v1coroutine#2] Computing v1
[ForkJoinPool.commonPool-worker-2 @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

<!--- TEST FLEXIBLE_THREAD -->

### Cancellation via explicit job

Let us put our knowledge about contexts, children and jobs together. Assume that our application has
an object with a lifecycle, but that object is not a coroutine. For example, we are writing an Android application
and launch various coroutines in the context of an Android activity to perform asynchronous operations to fetch 
and update data, do animations, etc. All of these coroutines must be cancelled when activity is destroyed
to avoid memory leaks. 
  
We can manage a lifecycle of our coroutines by creating an instance of [Job] that is tied to 
the lifecycle of our activity. A job instance is created using [Job()][Job.invoke] factory function
as the following example shows. We need to make sure that all the coroutines are started 
with this job in their context and then a single invocation of [Job.cancel] terminates them all.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = Job() // create a job object to manage our lifecycle
    // now launch ten coroutines for a demo, each working for a different time
    val coroutines = List(10) { i ->
        // they are all children of our job object
        launch(context + job) { // we use the context of main runBlocking thread, but with our own job object 
            delay(i * 200L) // variable delay 0ms, 200ms, 400ms, ... etc
            println("Coroutine $i is done")
        }
    }
    println("Launched ${coroutines.size} coroutines")
    delay(500L) // delay for half a second
    println("Cancelling job!")
    job.cancel() // cancel our job.. !!!
    delay(1000L) // delay for more to see if our coroutines are still working
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-09.kt)

The output of this example is:

```text
Launched 10 coroutines
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Cancelling job!
```

<!--- TEST -->

As you can see, only the first three coroutines had printed a message and the others were cancelled 
by a single  invocation of `job.cancel()`. So all we need to do in our hypothetical Android 
application is to create a parent job object when activity is created, use it for child coroutines,
and cancel it when activity is destroyed.

## Channels

Deferred values provide a convenient way to transfer a single value between coroutines.
Channels provide a way to transfer a stream of values.

<!--- INCLUDE .*/example-channel-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
-->

### Channel basics

A [Channel] is conceptually very similar to `BlockingQueue`. One key difference is that
instead of a blocking `put` operation it has a suspending [send][SendChannel.send], and instead of 
a blocking `take` operation it has a suspending [receive][ReceiveChannel.receive].

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-01.kt)

The output of this code is:

```text
1
4
9
16
25
Done!
```

<!--- TEST -->

### Closing and iteration over channels 

Unlike a queue, a channel can be closed to indicate that no more elements are coming. 
On the receiver side it is convenient to use a regular `for` loop to receive elements 
from the channel. 
 
Conceptually, a [close][SendChannel.close] is like sending a special close token to the channel. 
The iteration stops as soon as this close token is received, so there is a guarantee 
that all previously sent elements before the close are received:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-02.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

### Building channel producers

The pattern where a coroutine is producing a sequence of elements is quite common. 
This is a part of _producer-consumer_ pattern that is often found in concurrent code. 
You could abstract such a producer into a function that takes channel as its parameter, but this goes contrary
to common sense that results must be returned from functions. 

There is a convenience coroutine builder named [produce] that makes it easy to do it right on producer side,
and an extension function [consumeEach], that can replace a `for` loop on the consumer side:

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-03.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

### Pipelines

Pipeline is a pattern where one coroutine is producing, possibly infinite, stream of values:

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

And another coroutine or coroutines are consuming that stream, doing some processing, and producing some other results.
In the below example the numbers are just squared:

```kotlin
fun square(numbers: ReceiveChannel<Int>) = produce<Int>(CommonPool) {
    for (x in numbers) send(x * x)
}
```

The main code starts and connects the whole pipeline:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val numbers = produceNumbers() // produces integers from 1 and on
    val squares = square(numbers) // squares integers
    for (i in 1..5) println(squares.receive()) // print first five
    println("Done!") // we are done
    squares.cancel() // need to cancel these coroutines in a larger app
    numbers.cancel()
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-04.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

We don't have to cancel these coroutines in this example app, because
[coroutines are like daemon threads](#coroutines-are-like-daemon-threads), 
but in a larger app we'll need to stop our pipeline if we don't need it anymore.
Alternatively, we could have run pipeline coroutines as 
[children of a coroutine](#children-of-a-coroutine).

### Prime numbers with pipeline

Let's take pipelines to the extreme with an example that generates prime numbers using a pipeline 
of coroutines. We start with an infinite sequence of numbers. This time we introduce an 
explicit context parameter, so that caller can control where our coroutines run:
 
<!--- INCLUDE kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt  
import kotlin.coroutines.experimental.CoroutineContext
-->
 
```kotlin
fun numbersFrom(context: CoroutineContext, start: Int) = produce<Int>(context) {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```

The following pipeline stage filters an incoming stream of numbers, removing all the numbers 
that are divisible by the given prime number:

```kotlin
fun filter(context: CoroutineContext, numbers: ReceiveChannel<Int>, prime: Int) = produce<Int>(context) {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

Now we build our pipeline by starting a stream of numbers from 2, taking a prime number from the current channel, 
and launching new pipeline stage for each prime number found:
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
``` 
 
The following example prints the first ten prime numbers, 
running the whole pipeline in the context of the main thread:

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt)

The output of this code is:

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

Note, that you can build the same pipeline using `buildIterator` coroutine builder from the standard library. 
Replace `produce` with `buildIterator`, `send` with `yield`, `receive` with `next`, 
`ReceiveChannel` with `Iterator`, and get rid of the context. You will not need `runBlocking` either.
However, the benefit of a pipeline that uses channels as shown above is that it can actually use 
multiple CPU cores if you run it in [CommonPool] context.

Anyway, this is an extremely impractical way to find prime numbers. In practice, pipelines do involve some
other suspending invocations (like asynchronous calls to remote services) and these pipelines cannot be
built using `buildSeqeunce`/`buildIterator`, because they do not allow arbitrary suspension, unlike
`produce` which is fully asynchronous.
 
### Fan-out

Multiple coroutines may receive from the same channel, distributing work between themselves.
Let us start with a producer coroutine that is periodically producing integers 
(ten numbers per second):

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```

Then we can have several processor coroutines. In this example, they just print their id and
received number:

```kotlin
fun launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch(CommonPool) {
    channel.consumeEach {
        println("Processor #$id received $it")
    }    
}
```

Now let us launch five processors and let them work for a second. See what happens:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(1000)
    producer.cancel() // cancel producer coroutine and thus kill them all
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-06.kt)

The output will be similar to the the following one, albeit the processor ids that receive
each specific integer may be different:

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

Note, that cancelling a producer coroutine closes its channel, thus eventually terminating iteration
over the channel that processor coroutines are doing.

### Fan-in

Multiple coroutines may send to the same channel.
For example, let us have a channel of strings, and a suspending function that 
repeatedly sends a specified string to this channel with a specified delay:

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

Now, let us see what happens if we launch a couple of coroutines sending strings 
(in this example we launch them in the context of the main thread):

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-07.kt)

The output is:

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### Buffered channels

The channels shown so far had no buffer. Unbuffered channels transfer elements when sender and receiver 
meet each other (aka rendezvous). If send is invoked first, then it is suspended until receive is invoked, 
if receive is invoked first, it is suspended until send is invoked.

Both [Channel()][Channel.invoke] factory function and [produce] builder take an optional `capacity` parameter to 
specify _buffer size_. Buffer allows senders to send multiple elements before suspending, 
similar to the `BlockingQueue` with a specified capacity, which blocks when buffer is full.

Take a look at the behavior of the following code:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>(4) // create buffered channel
    launch(context) { // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    // don't receive anything... just wait....
    delay(1000)
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-08.kt)

It prints "sending" _five_ times using a buffered channel with capacity of _four_:

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

The first four elements are added to the buffer and the sender suspends when trying to send the fifth one.


### Channels are fair

Send and receive operations to channels are _fair_ with respect to the order of their invocation from 
multiple coroutines. They are served in first-in first-out order, e.g. the first coroutine to invoke `receive` 
gets the element. In the following example two coroutines "ping" and "pong" are 
receiving the "ball" object from the shared "table" channel. 

```kotlin
data class Ball(var hits: Int)

fun main(args: Array<String>) = runBlocking<Unit> {
    val table = Channel<Ball>() // a shared table
    launch(context) { player("ping", table) }
    launch(context) { player("pong", table) }
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    table.receive() // game over, grab the ball
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        delay(300) // wait a bit
        table.send(ball) // send the ball back
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-09.kt)

The "ping" coroutine is started first, so it is the first one to receive the ball. Even though "ping"
coroutine immediately starts receiving the ball again after sending it back to the table, the ball gets
received by the "pong" coroutine, because it was already waiting for it:

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
ping Ball(hits=5)
```

<!--- TEST -->

## Shared mutable state and concurrency

Coroutines can be executed concurrently using a multi-threaded dispatcher like [CommonPool]. It presents
all the usual concurrency problems. The main problem being synchronization of access to **shared mutable state**. 
Some solutions to this problem in the land of coroutines are similar to the solutions in the multi-threaded world, 
but others are unique.

### The problem

Let us launch a thousand coroutines all doing the same action thousand times (for a total of a million executions). 
We'll also measure their completion time for further comparisons:

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
    val n = 1000 // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
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

We start with a very simple action that increments a shared mutable variable using 
multi-threaded [CommonPool] context. 

```kotlin
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-01.kt)

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

What does it print at the end? It is highly unlikely to ever print "Counter = 1000000", because a thousand coroutines 
increment the `counter` concurrently from multiple threads without any synchronization.

### Volatiles are of no help

There is common misconception that making a variable `volatile` solves concurrency problem. Let us try it:

```kotlin
@Volatile // in Kotlin `volatile` is an annotation 
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-02.kt)

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

This code works slower, but we still don't get "Counter = 1000000" at the end, because volatile variables guarantee
linearizable (this is a technical term for "atomic") reads and writes to the corresponding variable, but
do not provide atomicity of larger actions (increment in our case).

### Thread-safe data structures

The general solution that works both for threads and for coroutines is to use a thread-safe (aka synchronized,
linearizable, or atomic) data structure that provides all the necessarily synchronization for the corresponding 
operations that needs to be performed on a shared state. 
In the case of a simple counter we can use `AtomicInteger` class which has atomic `incrementAndGet` operations:

```kotlin
var counter = AtomicInteger()

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-03.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This is the fastest solution for this particular problem. It works for plain counters, collections, queues and other
standard data structures and basic operations on them. However, it does not easily scale to complex
state or to complex operations that do not have ready-to-use thread-safe implementations. 

### Thread confinement fine-grained

_Thread confinement_ is an approach to the problem of shared mutable state where all access to the particular shared
state is confined to a single thread. It is typically used in UI applications, where all UI state is confined to 
the single event-dispatch/application thread. It is easy to apply with coroutines by using a  
single-threaded context:

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) { // run each coroutine in CommonPool
        run(counterContext) { // but confine each increment to the single-threaded context
            counter++
        }
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-04.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This code works very slowly, because it does _fine-grained_ thread-confinement. Each individual increment switches 
from multi-threaded `CommonPool` context to the single-threaded context using [run] block. 

### Thread confinement coarse-grained

In practice, thread confinement is performed in large chunks, e.g. big pieces of state-updating business logic
are confined to the single thread. The following example does it like that, running each coroutine in 
the single-threaded context to start with.

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(counterContext) { // run each coroutine in the single-threaded context
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-05.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This now works much faster and produces correct result.

### Mutual exclusion

Mutual exclusion solution to the problem is to protect all modifications of the shared state with a _critical section_
that is never executed concurrently. In a blocking world you'd typically use `synchronized` or `ReentrantLock` for that.
Coroutine's alternative is called [Mutex]. It has [lock][Mutex.lock] and [unlock][Mutex.unlock] functions to 
delimit a critical section. The key difference is that `Mutex.lock` is a suspending function. It does not block a thread.

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-06.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

The locking in this example is fine-grained, so it pays the price. However, it is a good choice for some situations
where you absolutely must modify some shared state periodically, but there is no natural thread that this state
is confined to.

### Actors

An actor is a combination of a coroutine, the state that is confined and is encapsulated into this coroutine,
and a channel to communicate with other coroutines. A simple actor can be written as a function, 
but an actor with a complex state is better suited for a class. 

There is an [actor] coroutine builder that conveniently combines actor's mailbox channel into its 
scope to receive messages from and combines the send channel into the resulting job object, so that a 
single reference to the actor can be carried around as its handle.

```kotlin
// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: SendChannel<Int>) : CounterMsg() // a request with reply

// This function launches a new counter actor
fun counterActor() = actor<CounterMsg>(CommonPool) {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.send(counter)
        }
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val counter = counterActor() // create the actor
    massiveRun(CommonPool) {
        counter.send(IncCounter)
    }
    val response = Channel<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.receive()}")
    counter.close() // shutdown the actor
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-07.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

It does not matter (for correctness) what context the actor itself is executed in. An actor is
a coroutine and a coroutine is executed sequentially, so confinement of the state to the specific coroutine
works as a solution to the problem of shared mutable state.

Actor is more efficient than locking under load, because in this case it always has work to do and it does not 
have to switch to a different context at all.

> Note, that an [actor] coroutine builder is a dual of [produce] coroutine builder. An actor is associated 
  with the channel that it receives messages from, while a producer is associated with the channel that it 
  sends elements to.

## Select expression

Select expression makes it possible to await multiple suspending functions simultaneously and _select_
the first one that becomes available.

<!--- INCLUDE .*/example-select-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.selects.*
-->

### Selecting from channels

Let us have two producers of strings: `fizz` and `buzz`. The `fizz` produces "Fizz" string every 300 ms:

<!--- INCLUDE .*/example-select-01.kt
import kotlin.coroutines.experimental.CoroutineContext
-->

```kotlin
fun fizz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}
```

And the `buzz` produces "Buzz!" string every 500 ms:

```kotlin
fun buzz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}
```

Using [receive][ReceiveChannel.receive] suspending function we can receive _either_ from one channel or the
other. But [select] expression allows us to receive from _both_ simultaneously using its
[onReceive][SelectBuilder.onReceive] clauses:

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}
```

Let us run it all seven times:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val fizz = fizz(context)
    val buzz = buzz(context)
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-01.kt)

The result of this code is: 

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

### Selecting on close

The [onReceive][SelectBuilder.onReceive] clause in `select` fails when the channel is closed and the corresponding
`select` throws an exception. We can use [onReceiveOrNull][SelectBuilder.onReceiveOrNull] clause to perform a
specific action when the channel is closed. The following example also shows that `select` is an expression that returns 
the result of its selected clause:

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

Let's use it with channel `a` that produces "Hello" string four times and 
channel `b` that produces "World" four times:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // we are using the context of the main thread in this example for predictability ... 
    val a = produce<String>(context) { 
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String>(context) { 
        repeat(4) { send("World $it") }
    }
    repeat(8) { // print first eight results
        println(selectAorB(a, b))
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-02.kt)

The result of this code is quite interesting, so we'll analyze it in mode detail:

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

There are couple of observations to make out of it. 

First of all, `select` is _biased_ to the first clause. When several clauses are selectable at the same time, 
the first one among them gets selected. Here, both channels are constantly producing strings, so `a` channel,
being the first clause in select, wins. However, because we are using unbuffered channel, the `a` gets suspended from
time to time on its [send][SendChannel.send] invocation and gives a chance for `b` to send, too.

The second observation, is that [onReceiveOrNull][SelectBuilder.onReceiveOrNull] gets immediately selected when the 
channel is already closed.

### Selecting to send

Select expression has [onSend][SelectBuilder.onSend] clause that can be used for a great good in combination 
with a biased nature of selection.

Let us write an example of producer of integers that sends its values to a `side` channel when 
the consumers on its primary channel cannot keep up with it:

```kotlin
fun produceNumbers(side: SendChannel<Int>) = produce<Int>(CommonPool) {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel     
        }
    }
}
```

Consumer is going to be quite slow, taking 250 ms to process each number:
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val side = Channel<Int>() // allocate side channel
    launch(context) { // this is a very fast consumer for the side channel
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // let us digest the consumed number properly, do not hurry
    }
    println("Done consuming")
}
``` 
 
> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-03.kt)
  
So let us see what happens:
 
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

### Selecting deferred values

Deferred values can be selected using [onAwait][SelectBuilder.onAwait] clause. 
Let us start with an async function that returns a deferred string value after 
a random delay:

<!--- INCLUDE .*/example-select-04.kt
import java.util.*
-->

```kotlin
fun asyncString(time: Int) = async(CommonPool) {
    delay(time.toLong())
    "Waited for $time ms"
}
```

Let us start a dozen of them with a random delay.

```kotlin
fun asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

Now the main function awaits for the first of them to complete and counts the number of deferred values
that are still active. Note, that we've used here the fact that `select` expression is a Kotlin DSL, 
so we can provide clauses for it using an arbitrary code. In this case we iterate over a list
of deferred values to provide `onAwait` clause for each deferred value.

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

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-04.kt)

The output is:

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### Switch over a channel of deferred values

Let us write a channel producer function that consumes a channel of deferred string values, waits for each received
deferred value, but only until the next deferred value comes over or the channel is closed. This example puts together 
[onReceiveOrNull][SelectBuilder.onReceiveOrNull] and [onAwait][SelectBuilder.onAwait] clauses in the same `select`:

```kotlin
fun switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String>(CommonPool) {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->  
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}
```

To test it, we'll use a simple async function that resolves to a specified string after a specified time:

```kotlin
fun asyncString(str: String, time: Long) = async(CommonPool) {
    delay(time)
    str
}
```

The main function just launches a coroutine to print results of `switchMapDeferreds` and sends some test
data to it:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val chan = Channel<Deferred<String>>() // the channel for test
    launch(context) { // launch printing coroutine
        for (s in switchMapDeferreds(chan)) 
            println(s) // print each received string
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // enough time for "BEGIN" to be produced
    chan.send(asyncString("Slow", 500))
    delay(100) // not enough time to produce slow
    chan.send(asyncString("Replace", 100))
    delay(500) // give it time before the last one
    chan.send(asyncString("END", 500))
    delay(1000) // give it time to process
    chan.close() // close the channel ... 
    delay(500) // and wait some time to let it finish
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-05.kt)

The result of this code:

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

## Further reading

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
