<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT core/kotlinx-coroutines-core/test/guide/test/GuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class GuideTest {
--> 

# 実例によるkotlinx.coroutinesの手引き

これは一連の例による `kotlinx.coroutines` の中核的機能についてのガイドです。

## 導入とセットアップ

言語としてのKotlinは、他の様々なライブラリがコルーチンを利用できるようにするために標準ライブラリに最小限の低レベルAPIしか提供していません。 同様の機能を持つ他の多くの言語とは異なり、 `async` と `await` はKotlinのキーワードではなく、標準ライブラリの一部でもありません。
さらに、Kotlinの _サスペンド関数_ の概念は、フューチャーやプロミスよりも非同期操作のための、より安全で誤りの少ない抽象化を提供します。

`kotlinx.coroutines` はJetBrainsによって開発されたコルーチン用の豊富なライブラリです。
これには、 `launch` 、 `async` などを含む、このガイドで扱う高水準のコルーチンを可能にするプリミティブが含まれています。
あなたのプロジェクトでこのガイドのプリミティブを使用するには、[ここ](README.md#using-in-your-projects)で説明する `kotlinx-coroutines-core` モジュールに依存関係を追加する必要があります。

## 目次

<!--- TOC -->

* [コルーチンの基礎](#コルーチンの基礎)
  * [初めてのコルーチン](#初めてのコルーチン)
  * [ブロッキングとノンブロッキングの世界の橋渡し](#ブロッキングとノンブロッキングの世界の橋渡し)
  * [ジョブを待つ](#ジョブを待つ)
  * [構造化同時実行性](#構造化同時実行性)
  * [スコープビルダー](#スコープビルダー)
  * [関数抽出リファクタリング](#関数抽出リファクタリング)
  * [コルーチンは軽量](#コルーチンは軽量)
  * [グローバルコルーチンはデーモンスレッドに似ている](#グローバルコルーチンはデーモンスレッドに似ている)
* [キャンセルとタイムアウト](#キャンセルとタイムアウト)
  * [コルーチンの実行をキャンセルする](#コルーチンの実行をキャンセルする)
  * [キャンセルは協調的](#キャンセルは協調的)
  * [計算コードをキャンセル可能にする](#計算コードをキャンセル可能にする)
  * [finallyでリソースを閉じる](#finallyでリソースを閉じる)
  * [キャンセル不可ブロックの実行](#キャンセル不可ブロックの実行)
  * [タイムアウト](#タイムアウト)
* [サスペンド関数の作成](#サスペンド関数の作成)
  * [デフォルトではシーケンシャル](#デフォルトではシーケンシャル)
  * [asyncを使用した並列処理](#asyncを使用した並列処理)
  * [遅延して開始されるasync](#遅延して開始されるasync)
  * [Asyncスタイル関数](#asyncスタイル関数)
  * [asyncでの構造化された並列処理](#asyncでの構造化された並列処理)
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
* [例外処理](#例外処理)
  * [例外の伝播](#例外の伝播)
  * [CoroutineExceptionHandler](#coroutineexceptionhandler)
  * [キャンセルと例外](#キャンセルと例外)
  * [例外の集約](#例外の集約)
* [チャネル（実験的）](#チャネル（実験的）)
  * [チャネルの基礎](#チャネルの基礎)
  * [チャネルのクローズと反復](#チャネルのクローズと反復)
  * [チャネルプロデューサーの作成](#チャネルプロデューサーの作成)
  * [パイプライン](#パイプライン)
  * [パイプラインによる素数](#パイプラインによる素数)
  * [ファンアウト](#ファンアウト)
  * [ファンイン](#ファンイン)
  * [バッファーされたチャネル](#バッファーされたチャネル)
  * [チャネルは公正](#チャネルは公正)
  * [ティッカーチャネル](#ティッカーチャネル)
* [共有ミュータブルステートと並列処理](#共有ミュータブルステートと並列処理)
  * [問題](#問題)
  * [Volatileは助けにならない](#volatileは助けにならない)
  * [スレッドセーフなデータ構造](#スレッドセーフなデータ構造)
  * [細粒度のスレッド制約](#細粒度のスレッド制約)
  * [粗粒度のスレッド制約](#粗粒度のスレッド制約)
  * [排他制御](#排他制御)
  * [アクター](#アクター)
* [セレクト式（実験的）](#セレクト式セレクト式（実験的）)
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
    GlobalScope.launch { // バックグラウンドで新しいコルーチンを起動し、続行する
        delay(1000L) // 1秒間ノンブロッキング遅延 (デフォルトの時間単位はms)
        println("World!") // delayのあとでプリント
    }
    println("Hello,") // コルーチンが遅延している間、メインスレッドは継続する
    Thread.sleep(2000L) // メインスレッドを2秒間ブロックしてJVMを存続させます
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)で完全なコードを取得できます。

このコードを実行します。

```text
Hello,
World!
```

<!--- TEST -->

基本的に、コルーチンは軽量スレッドです。
それらは[CoroutineScope]のコンテキストで[launch] _コルーチンビルダー_ で起動されます。
ここでは、[GlobalScope]で新しいコルーチンを起動しています。つまり、新しいコルーチンの存続期間は、アプリケーション全体の存続期間によってのみ制限されます。

`GlobalScope.launch { ... }` を `thread { ... }` に、 `delay(...)` を `Thread.sleep(...)` に置き換えても同じ結果が得られます。試してみてください。

`GlobalScope.launch` を `thread` に置き換えて起動すると、コンパイラは次のエラーを生成します。

```
Error: Kotlin: サスペンド関数は、コルーチンまたは他のサスペンド関数からのみ呼び出すことができます
```

これは、[delay]がスレッドをブロックせず、コルーチンを _中断_ しコルーチンからのみ使用できる特別な _サスペンド関数_ であるためです。

### ブロッキングとノンブロッキングの世界の橋渡し

最初の例では、 同じコードに _ノンブロッキング_ `delay(...)` と _ブロッキング_ `Thread.sleep(...)` を混在させています。
ある方はブロックされていて、もう一方はブロックしていませんので迷子になるのは簡単です。
[runBlocking]コルーチンビルダーを使用して、ブロッキングを明確にしましょう。

```kotlin
fun main(args: Array<String>) { 
    GlobalScope.launch { // バックグラウンドで新しいコルーチンを起動し、続行する
        delay(1000L)
        println("World!")
    }
    println("Hello,") // メインスレッドはすぐにここに続く
    runBlocking {     // しかし、この式はメインスレッドをブロックします
        delay(2000L)  // ... 2秒間遅延してJVMを存続させる
    } 
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-02.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は同じですが、このコードではノンブロッキング[delay]のみを使用しています。
`runBlocking` を呼び出すメインスレッドは、`runBlocking` 内部のコルーチンが完了するまで _ブロック_ します。

この例は、 `runBlocking` を使ってmain関数の実行をラップすることにより、より慣用的な方法で書き直すこともできます：

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> { // メインコルーチンを開始
    GlobalScope.launch { // バックグラウンドで新しいコルーチンを起動し、続行する
        delay(1000L)
        println("World!")
    }
    println("Hello,") // メインコルーチンはすぐにここに続く
    delay(2000L)      // JVMを維持するために2秒間遅延する
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

ここで `runBlocking<Unit> { ... }` は、トップレベルのメインコルーチンを起動するためのアダプタとして機能します。
Kotlinの適格な `main` 関数は `Unit` を返さなければならないので、リターンタイプ `Unit` を明示的に指定します。

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
    val job = GlobalScope.launch { // 新しいコルーチンを起動し、そのJobへの参照を保持する
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 子コルーチンが完了するまで待つ
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-03.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は変わりませんが、メインコルーチンのコードはバックグラウンドジョブの継続時間に結びついていません。ずっと良いです。

### 構造化同時実行性

コルーチンの実用化には依然として何かが必要です。
`GlobalScope.launch` を使うと、トップレベルのコルーチンが作成されます。 軽量であるにもかかわらず、実行中にいくつかのメモリリソースを消費します。 新しく起動したコルーチンへの参照の保持を忘れたとしても、まだ実行されています。 もしコルーチンのコードがハングして（例えば、長すぎる誤った遅延）、多くのコルーチンを起動しすぎてメモリが足りなくなった場合はどうなりますか？
起動されたすべてのコルーチンへの参照を手動で保持し、[Join][Job.join]するとエラーが発生しやすくなります。

より良い解決策があります。私たちのコードでは、構造化された並列処理を使用できます。
[GlobalScope]でコルーチンを起動するのではなく、通常はスレッド（スレッドは常にグローバル）と同様に、実行中の操作の特定の範囲でコルーチンを起動することができます。

この例では、[runBlocking]コルーチンビルダーを使用してコルーチンに変換される `main` 関数があります。
`runBlocking` を含むすべてのコルーチンビルダーは、そのコードブロックのスコープに[CoroutineScope]のインスタンスを追加します。
スコープで起動されたコルーチンがすべて完了するまで外部コルーチン（この例では `runBlocking`）が完了しないため、明示的に `join` せずにこのスコープでコルーチンを起動することができます。
したがって、例を簡単にすることができます：

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> { // this: CoroutineScope
    launch { // runBlockingのスコープで新しいコルーチンを起動
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-03s.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

### スコープビルダー
異なるビルダーが提供するコルーチンのスコープに加えて、[coroutineScope]ビルダーを使用して独自のスコープを宣言することも可能です。
これは新しいコルーチン範囲を作成し、起動したすべての子が完了するまで完了しません。
[runBlocking]と[coroutineScope]の主な違いは、後者はすべての子が完了するのを待つ間、現在のスレッドをブロックしないことです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 新しいコルーチンスコープを作る
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // この行はネストしたlaunchより前にプリントする
    }
    
    println("Coroutine scope is over") // この行はネストしたlaunchが完了するまでプリントしない
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-04.kt)で完全なコードを取得できます。

<!--- TEST
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
-->

### 関数抽出リファクタリング

`launch { ... }` の中のコードブロックを別の関数に抽出しましょう。
このコードで 「Extract function」リファクタリングを実行すると、 `suspend` 修飾子付きの新しい関数が得られます。
それがあなたの最初の _サスペンド関数_ です。
サスペンド関数は、通常の関数と同様にコルーチン内で使用できますが、追加機能として、この例では `delay`のような他のサスペンド関数を使用してコルーチンの実行を _中断_ することができます。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch { doWorld() }
    println("Hello,")
}

// これはあなたの最初のサスペンド関数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-05.kt)で完全なコードを取得できます

<!--- TEST
Hello,
World!
-->

しかし、抽出された関数に現在のスコープで呼び出されるコルーチンビルダーが含まれているとしたらどうでしょうか？
この場合、抽出された関数の `suspend` 修飾子では不十分です。
`CoroutineScope` 上で `doWorld` 拡張メソッドを作ることは解決策の1つですが、APIをより明確にしないので、必ずしも適用可能とは限りません。
[currentScope]ビルダーが助けになります。それが呼び出されるコルーチンのコンテキストから現在の[CoroutineScope]を継承します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launchDoWorld()
    println("Hello,")
}

// これはあなたの最初のサスペンド関数
suspend fun launchDoWorld() = currentScope {
        launch {
        println("World!")
    }
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-05s.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

### コルーチンは軽量

次のコードを実行します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    repeat(100_000) { // たくさんのコルーチンを起動する
        launch {
            delay(1000L)
            print(".")
        }
    }
    jobs.forEach { it.join() } // すべてのジョブが完了するのを待つ
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-06.kt)で完全なコードを取得できます

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

10万個のコルーチンを起動し、1秒後に各コルーチンがドットをプリントします。
スレッドを使って試したらどうなるでしょうか？ （ほとんどの場合、あなたのコードはメモリ不足エラーを引き起こすでしょう）

### グローバルコルーチンはデーモンスレッドに似ている

次のコードでは、「I'm sleeping」というメッセージを毎秒2回プリントする長時間実行するコルーチンを[GlobalScope]で起動し、ある程度遅れてmain関数からリターンします。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 遅れて終了する
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)で完全なコードを取得できます

実行すると、3行を出力して終了することがわかります。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

[GlobalScope]で起動されたアクティブなコルーチンはプロセスを生かし続けるわけではありません。それらはデーモンスレッドのようなものです。

## キャンセルとタイムアウト

このセクションでは、コルーチンのキャンセルとタイムアウトについて説明します。

### コルーチンの実行をキャンセルする

長時間実行しているアプリケーションでは、バックグラウンドコルーチンをきめ細かく制御する必要があります。
例えば、あるユーザがコルーチンを起動したページを閉じて、その結果が不要になり、その作業を取り消されるかもしれません。
[launch]関数は、実行中のコルーチンを取り消すために使用できる[job]を返します。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancel() // ジョブをキャンセル
    job.join() // ジョブの完了を待つ
    println("main: Now I can quit.")
}
``` 

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-01.kt)で完全なコードを取得できます

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
[cancel][Job.cancel]と[join][Job.join]の呼び出しを組み合わせた[Job]の拡張関数[cancelAndJoin]もあります。

### キャンセルは協調的

コルーチンのキャンセルは _協調的_ です。コルーチンコードは取り消し可能にするために協調しなければなりません。
`kotlinx.coroutines` のサスペンド関数はすべて _キャンセル可能_ です。
これらはコルーチンのキャンセルをチェックし、キャンセルすると[CancellationException]をスローします。
ただし、コルーチンが計算で作業していて取り消しをチェックしていない場合、次の例のように取り消すことはできません。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) { // CPUを浪費するだけの計算ループ
            // 1秒間に2回メッセージをプリントする
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了するのを待つ
    println("main: Now I can quit.")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-02.kt)で完全なコードを取得できます

実行して、キャンセル後も "I'm sleeping" とプリントし続けることを確認します。

<!--- TEST 
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm sleeping 3 ...
I'm sleeping 4 ...
main: Now I can quit.
-->

### 計算コードをキャンセル可能にする

計算コードをキャンセル可能にするには2つの方法があります。
1つは、キャンセルをチェックするサスペンド関数を定期的に呼び出すことです。
その目的のために良い選択肢である[yield]関数があります。
もう1つは、キャンセルステータスを明示的にチェックすることです。後者の方法を試してみましょう。

前の例の `while (i < 5)` を `while (isActive)` に置き換えて再実行してください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // キャンセル可能な計算ループ
            // 1秒間に2回メッセージをプリントする
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 少し遅らせる
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了するのを待つ
    println("main: Now I can quit.")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-03.kt)で完全なコードを取得できます

ご覧のとおり、このループはキャンセルされました。 [isActive]は、[CoroutineScope]オブジェクトを介してコルーチンのコード内で使用できる拡張プロパティです。

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
-->

### finallyでリソースを閉じる

キャンセル可能なサスペンド関数は、キャンセル時に通常の方法で処理できる[CancellationException]をスローします。
例えば、 `try {...} finally {...}` 式とKotlin `use` 関数は、コルーチンがキャンセルされたときに、通常の終了処理を実行します。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch {
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
    job.cancelAndJoin() // ジョブをキャンセルして完了を待つ
    println("main: Now I can quit.")
}
``` 

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-04.kt)で完全なコードを取得できます

[join][Job.join]も[cancelAndJoin]も、すべてのファイナライズ・アクションが完了するまで待機するため、上記の例では次の出力が生成されます。

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
ただし、キャンセルされたコルーチンで中断する必要があるまれなケースでは、次の例のように [withContext]関数と [NonCancellable]コンテキストを使用して `withContext(NonCancellable) {...}` に対応するコードをラップすることができます。
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // ジョブをキャンセルして完了を待つ
    println("main: Now I can quit.")
}
``` 

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-05.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-06.kt)で完全なコードを取得できます

これは次の出力を生成します。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.experimental.TimeoutCancellationException: Timed out waiting for 1300 MILLISECONDS
```

<!--- TEST STARTS_WITH -->

[withTimeout]によってスローされる `TimeoutCancellationException` は、[CancellationException]のサブクラスです。
先のコンソールにはスタックトレースが表示されていませんでした。
キャンセルされたコルーチン内の `CancellationException` がコルーチン完了の通常の理由であると考えられるためです。
しかし、この例では `main` 関数の中で `withTimeout` を使用しています。

キャンセルは単なる例外なので、すべてのリソースは通常の方法で閉じられます。
任意の種類のタイムアウトに特別なアクションを追加する必要がある場合は、try {...} catch（e：TimeoutCancellationException）{...} `ブロックでタイムアウトを指定してコードをラップすることができます。また、[withTimeout]と似た[withTimeoutOrNull]関数を使用します。これは例外をスローするのではなくタイムアウトで `null` を返します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // この結果が出る前にキャンセルされます
    }
    println("Result is $result")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-cancel-07.kt)で完全なコードを取得できます

このコードを実行しても例外は出ません。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

<!--- TEST -->

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-01.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-02.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-03.kt)で完全なコードを取得できます

次のようなものが生成されます。

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

ここでは2つのコルーチンが定義されていますが、前の例のようには実行されません。[start][Job.start]を呼び出す実行開始の制御はプログラマに与えられます。
最初に `one` を開始し、次に `two` を開始し、個々のコルーチンが終了するのを待ちます。

`println` で個々のコルーチンの[await][Deferred.await]を呼び出し、[start][Job.start]を省略した場合、シーケンシャルに[await][Deferred.await]コルーチンの実行を開始し、実行が終了するのを待ちます。これは、遅延の意図された使用例ではありません。
`async(start = CoroutineStart.LAZY)' のユースケースは、値の計算にサスペンド関数が含まれる場合に標準 `lazy` 関数の代わりになります。

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-04.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-05.kt)で完全なコードを取得できます。

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-compose-06.kt)で完全なコードを取得できます。

最初の `async` と待っている親が1つの子の失敗でどのように取り消されるかに注目してください。
```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

<!--- TEST -->


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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-01.kt)で完全なコードを取得できます

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

[GlobalScope]でコルーチンを起動するときに使用されるデフォルトのディスパッチャは、[Dispatchers.Default]で表され、共有バックグラウンド スレッドプールを使用するため、 `launch（Dispatchers.Default）{...}`は`GlobalScope.launch {...} 'と同じディスパッチャを使用します。
  
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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-02.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-03.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-04.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-05.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-06.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-07.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-08.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-09.kt)で完全なコードを取得できます。

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-10.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-context-11.kt)で完全なコードを取得できます。

この例では、[Dispatchers.Default]のスレッドプールでバックグラウンドの新しいコルーチンを起動するのでコルーチンはスレッドプールの異なるスレッドで動作しますが、コルーチンがどのスレッドで実行されても `threadLocal.asContextElement(value = "launch")` で指定したスレッドローカル変数の値は保持しています。
したがって、出力（[debug](#コルーチンとスレッドのデバッグ)を用いる）は次のようになります。

```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[CommonPool-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[CommonPool-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

<!--- TEST FLEXIBLE_THREAD -->

`ThreadLocal` はファーストクラスのサポートを持ち、`kotlinx.corotuines` が提供するプリミティブと一緒に使うことができます。
これには1つの重要な制限があります。スレッドローカルが変更された場合、新しい値はコルーチンの呼び出し元に伝播されず（コンテキスト要素はすべての `ThreadLocal` オブジェクトへのアクセスを追跡できないため）、更新された値は次の一時停止時に失われます。
コルーチンのスレッドローカルの値を更新するには[withContext]を使用してください。詳細は[asContextElement]を参照してください。

あるいは、 `class Counter(var i: Int)` のような変更可能なボックスに値を格納することもできます。これはスレッドローカル変数に格納されます。
ただし、このケースでは、この変更可能なボックス内の変数に潜在的に同時に発生する変更を同期する責任があります。

高度な使い方、例えば、ロギングMDC、トランザクションコンテキスト、またはデータを渡すためにスレッドローカルを内部的に使用する他のライブラリとの統合などについては、実装する必要がある[ThreadContextElement]インターフェイスのドキュメントを参照してください。

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-01.kt)で完全なコードを取得できます。

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-02.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-03.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-04.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-05.kt)で完全なコードを取得できます

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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-exceptions-06.kt)で完全なコードを取得できます

このコードの出力は以下の通り。

```text
Rethrowing JobCancellationException with original cause
Caught original java.io.IOException
```
<!--- TEST-->

## チャネル（実験的）

遅延値は、コルーチン間で単一の値を転送する便利な方法を提供します。
チャネルは、ストリーム値を転送する方法を提供します。

> チャンネルは `kotlinx.coroutines` の実験的な機能です。これらのAPIは、 `kotlinx.coroutines` ライブラリの今後のアップデートで、潜在的に大きな変化を伴って進化することが期待されています。

<!--- INCLUDE .*/example-channel-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
-->

### チャネルの基礎

[Channel]は概念的には `BlockingQueue` に非常によく似ています。主な違いの1つは、ブロックする `put` 操作の代わりにサスペンド[send][SendChannel.send]、ブロックする `take` 操作の代わりにサスペンド[receive][ReceiveChannel.receive]を持っていることです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch {
        // これはCPU使用量が多い計算や非同期ロジックかもしれないが、ここではただ5つの平方を送るだけ
        for (x in 1..5) channel.send(x * x)
    }
    // ここで受け取った5つの整数をプリントする
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-01.kt)で完全なコードを取得できます

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
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // 送信完了
    }
    // ここでは `for` ループを使って受け取った値をプリントします（チャネルが閉じられるまで）
    for (y in channel) println(y)
    println("Done!")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-02.kt)で完全なコードを取得できます

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
そのようなプロデューサーを、パラメーターとしてchannelをとる関数に抽象化することはできますが、結果は関数から返さなければならないという常識とは逆になります。

プロデューサー側で簡単に行うことを容易にする[produce]という便利なコルーチンビルダーと、コンシューマー側の `for` ループを置き換える拡張関数[consumeEach]があります。

```kotlin
fun CoroutineScope.produceSquares() = produce<Int> {
    for (x in 1..5) send(x * x)
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-03.kt)で完全なコードを取得できます

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
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // 1から始まる整数の無限ストリーム
}
```

そして、別のコルーチンはそのストリームを消費し、いくつかの処理を行い、他の結果を生成しています。
以下の例では、数値は単に二乗されます。

```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>) = produce<Int> {
    for (x in numbers) send(x * x)
}
```

メインコードはパイプライン全体を開始し接続します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val numbers = produceNumbers() // 1から始まる整数を生成する
    val squares = square(numbers) // 整数を平方にする
    for (i in 1..5) println(squares.receive()) // 最初の5つをプリントする
    println("Done!") // 完了
    coroutineContext.cancelChildren() // 子コルーチンをキャンセルする
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-04.kt)で完全なコードを取得できます

<!--- TEST 
1
4
9
16
25
Done!
-->

> コルーチンを作成するすべての関数は、[CoroutineScope]の拡張として定義されているため、アプリケーションでグローバルなコルーチンが残っていないことを確認するために[構造化同時実行性](#構造化同時実行性)に頼ることができます。

### パイプラインによる素数

コルーチンのパイプラインを使って素数を生成する例で、パイプラインを徹底的に使ってみましょう。 無限の数列から始めます。
 
<!--- INCLUDE  
import kotlin.coroutines.experimental.*
-->
 
```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // startからの無限の整数ストリーム
}
```

次のパイプラインステージでは、入力数列をフィルタリングして、指定された素数で割り切れるすべての数値を削除します。

```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

2からの数列を開始し、現在のチャネルから素数を取り出し、見つかった各素数に対して新しいパイプラインステージを開始することでパイプラインを構築します。
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
``` 
 
次の例では最初の10個の素数をプリントし、パイプライン全体をメインスレッドのコンテキストで実行します。
すべてのコルーチンはメインの[runBlocking]コルーチンの範囲で起動されるため、開始したすべてのコルーチンの明示的なリストを保持する必要はありません。
最初の10個の素数をプリントした後、[cancelChildren][kotlin.coroutines.experimental.CoroutineContext.cancelChildren]拡張関数を使用してすべての子コルーチンを取り消します。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    var cur = numbersFrom(2)
    for (i in 1..10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // すべての子をキャンセルしてメインを終わる
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-05.kt)で完全なコードを取得できます

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

標準ライブラリの [`buildIterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/build-iterator.html) コルーチンビルダーを使って同じパイプラインを構築できることに留意してください。
`produce` を `buildIterator`、 `send` を `yield`、 `receive` を `next`、 `ReceiveChannel` を `Iterator` で置き換え、コルーチンスコープを取り除きます。 `runBlocking` も必要ありません。
ただし、上記のようなチャネルを使用するパイプラインの利点は、[Dispatchers.Default]コンテキストで実行すると実際に複数のCPUコアを使用できることです。

どのみち、これは素数を見つけるには非常に非実用的な方法です。 
実際にはパイプラインは（リモートサービスへの非同期呼び出しのような）いくつかの他のサスペンド呼び出しを必要とします。これらのパイプラインは `buildSequence`/`buildIterator` を使用して構築することはできません 。なぜなら完全に非同期の `produce` とは異なり任意の中断を許さないためです。
 
### ファンアウト

複数のコルーチンが同じチャネルから受信し、それらの間で作業を分散することがあります。
定期的に整数（毎秒10個の数値）を生成するプロデューサーコルーチンから始めましょう。

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // 1から始める
    while (true) {
        send(x++) // 次を生成
        delay(100) // 0.1秒待つ
    }
}
```

いくつかのプロセッサコルーチンを持つことができます。この例では、IDと受け取った数値をプリントします。

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

今度は5つのプロセッサを起動して、それらを約1秒間動作させましょう。何が起こるか確かめてください。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // プロデューサーのコルーチンを取り消し、すべてを殺す
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-06.kt)で完全なコードを取得できます

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

プロデューサーのコルーチンをキャンセルするとそのチャネルが閉じられ、最終的にプロセッサのコルーチンが行っているチャネルでの繰り返しが終了することに注意してください。

また、 `launchProcessor` コードでファンアウトを実行するために `for` ループを使って明示的にチャネルを反復する方法にも注意してください。
`consumeEach` とは異なり、この `for` ループパターンは、複数のコルーチンから完全に安全に使用できます。
プロセッサコルーチンの1つが失敗した場合、他のものは依然としてチャネルを処理しているのに対して、 `consumeEach` によって書かれたプロセッサは、その正常または異常終了時に常に下位のチャネルを消費（キャンセル）します。

### ファンイン

複数のコルーチンが同じチャネルに送信することがあります。
例えば、文字列のチャネルと、指定された文字列を指定された遅延でこのチャネルに繰り返し送信するサスペンド関数を持っています。

<!--- INCLUDE  
import kotlin.coroutines.experimental.*
-->

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

さて、文字列を送信するコルーチンをいくつか起動するとどうなるか見てみましょう（この例ではメインスレッドのコンテキストでメインコルーチンの子として起動します）。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // 最初の6個を受け取る
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // すべての子をキャンセルしてメインを終わる
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-07.kt)で完全なコードを取得できます

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

[Channel()]ファクトリ関数と[produce]ビルダーは、_バッファーサイズ_ を指定するためのオプションの `capacity` パラメーターをとります。 バッファーは、指定された容量を持つ `BlockingQueue` と同様に、送信側が中断する前に複数の要素を送信できるようにします。これはバッファーがいっぱいになるとブロックします。

次のコードの動作を見てみましょう。

<!--- INCLUDE  
import kotlin.coroutines.experimental.*
-->

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>(4) // バッファーされたチャネルを作る
    val sender = launch { // 送信側コルーチンを起動
        repeat(10) {
            println("Sending $it") // 各要素を送信する前にプリント
            channel.send(it) // バッファーがいっぱいになったら中断する
        }
    }
    // 何も受け取らずに待つ...
    delay(1000)
    sender.cancel() // 送信者コルーチンをキャンセルする
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-08.kt)で完全なコードを取得できます

これは容量 _4_ のバッファーされたチャネルを使って _5_ 回 "sending" を表示します。

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
次の例では、2つのコルーチン「ping」と「pong」が共有「table」チャネルから「ball」オブジェクトを受け取ります。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
data class Ball(var hits: Int)

fun main(args: Array<String>) = runBlocking<Unit> {
    val table = Channel<Ball>() // 共有テーブル
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // ボールを供給する
    delay(1000) // 1秒遅らせる
    coroutineContext.cancelChildren() // ゲームオーバー。これらをキャンセルする
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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-09.kt)で完全なコードを取得できます

「ping」コルーチンが最初に開始されるので、ボールを受け取るのは最初のコルーチンです。 「ping」コルーチンは、ボールをテーブルに戻した後すぐに再びボールを受け取るようになっていますが、ボールは既に受信を待っていた「pong」コルーチンによって受信さます。

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

<!--- TEST -->

チャネルによっては、使用されているエグゼキューターの性質上不公平に見える実行が生成されることがあります。
詳細については、[この問題](https://github.com/Kotlin/kotlinx.coroutines/issues/111)を参照してください。

### ティッカーチャネル

ティッカーチャネルは、このチャネルからの最後の消費から遅延が与えられるたびに `Unit` を生成する特別なランデブーチャネルです。
単独では役に立たないように見えるかもしれませんが、ウィンドウ処理やその他時間に依存した処理を行う複雑な時間ベースの[produce]パイプラインと演算子を作成するのに便利な構成要素です。
ティッカーチャンネルは[select]で 「on tick」アクションを実行するために使用できます。

このようなチャンネルを作成するには、ファクトリメソッド[ticker]を使用します。
これ以上要素が必要ないことを示すには、[ReceiveChannel.cancel]メソッドを使用します。

実際にどのように動作するかを見てみましょう。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val tickerChannel = ticker(delay = 100, initialDelay = 0) // ティッカーチャネルを作る
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // 初期の遅延時間はまだ経過していない

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() } // すべての後続要素は100ms遅延する
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    // 大きな消費の遅延をエミュレートする
    println("Consumer pauses for 150ms")
    delay(150)
    // 次の要素はすぐに利用可能
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // `receive` 呼び出しの間の休止が考慮され、次の要素がより速く到着することに注意
    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() } 
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel() // これ以上要素が必要でないことを示す
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-channel-10.kt)で完全なコードを取得できます

これは次の以下のようにプリントします。

```text
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```

<!--- TEST -->

[ticker]はコンシューマーの一時停止を認識しており、一時停止が発生した場合は生成される要素の固定レートを維持しようとし、デフォルトでは次回に生成される要素の遅延を調整します。
 
オプションとして、 `mode` パラメーターに[TickerMode.FIXED_DELAY]を指定して、要素間の遅延時間を固定することができます。

## 共有ミュータブルステートと並列処理

コルーチンは、[Dispatchers.Default]のようなマルチスレッドディスパッチャを使用して同時に実行できます。
これは、すべての通常の並列処理の問題を提起します。
主な問題は、**共有ミュータブルステート**へのアクセスの同期です。
コルーチンの世界でのこの問題に対するいくつかの解決策は、マルチスレッドの世界の解決策と似ていますが、他は独自のものです。

### 問題

1000のコルーチンを同じように100回実行してみましょう。
さらなる比較のために完了時間も測定します。

<!--- INCLUDE .*/example-sync-03.kt
import java.util.concurrent.atomic.*
-->

<!--- INCLUDE .*/example-sync-06.kt
import kotlinx.coroutines.experimental.sync.*
-->

<!--- INCLUDE .*/example-sync-07.kt
import kotlinx.coroutines.experimental.channels.*
-->

<!--- INCLUDE .*/example-sync-([0-9a-z]+).kt
import kotlin.system.*
import kotlin.coroutines.experimental.*
-->

```kotlin
suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 起動するコルーチンの数
    val k = 1000 // 各コルーチンによってアクションが繰り返される回数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

<!--- INCLUDE .*/example-sync-([0-9a-z]+).kt -->

まず、マルチスレッド化された[CommonPool]コンテキストを使用して、共有ミュータブル変数をインクリメントする非常に単純なアクションから始めます。
まず、[GlobalScope]で使用されるマルチスレッドの[Dispatchers.Default]を使用して、共有ミュータブル変数をインクリメントする非常に単純なアクションから始めます。

```kotlin
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-01.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

最後に何がプリントされますか？ 1000個のコルーチンが同期なしで複数のスレッドから同時に `counter` をインクリメントするため、"Counter = 100000" をプリントすることはほとんどありません。

> 注：2つ以下のCPUを持つ古いシステムを使用している場合、スレッドプールはこの場合は1つのスレッドでのみ実行されているため、一貫して100000と表示されます。 問題を再現するには、以下の変更を行う必要があります。

```kotlin
val mtContext = newFixedThreadPoolContext(2, "mtPool") // 明示的に2つのスレッドのコンテキストを定義する
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    CoroutineScope(mtContext).massiveRun { // このサンプル以降Dispatchers.Defaultの代わりに使用します
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-01b.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

### Volatileは助けにならない

変数を `volatile` にすると並列処理の問題が解決されるという誤解が一般的です。 それを試してみましょう。

```kotlin
@Volatile // Kotlinの `volatile` はアノテーション
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-02.kt)で完全なコードを取得できます

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

このコードはより遅く動作しますが、volatile変数は対応する変数の線形（専門用語で「アトミック」）読み書きを保証するものの、より大きなアクション（この場合はインクリメント）のアトミック性を提供しないため、最後に「Counter = 100000」を得られません。 

### スレッドセーフなデータ構造

スレッドとコルーチンの両方で機能する一般的な解決策は、共有状態で実行する必要があるすべての操作で必ず同期を提供するスレッドセーフ（別名、同期、線形化、またはアトミック）データ構造を使用することです。
単純なカウンタの場合、アトミックな `incrementAndGet` 操作を持つ `AtomicInteger` クラスを使うことができます。

```kotlin
var counter = AtomicInteger()

fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-03.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

これは、この特定の問題に対する最速の解決策です。 単純なカウンター、コレクション、キュー、その他の標準的なデータ構造とそれらの基本的な操作では機能します。 ただし、複雑な状態やすぐに使用できるスレッドセーフな実装を持たない複雑な操作には、容易に拡張できません。

### 細粒度のスレッド制約

_スレッド制約_ は、特定の共有状態へのすべてのアクセスが1つのスレッドに限定されている、共有ミュータブルステートの問題への提案です。
これは通常、すべてのUI状態が単一のイベントディスパッチ/アプリケーションスレッドに限定されるUIアプリケーションで使用されます。 単一スレッドのコンテキストを使用してコルーチンで簡単に適用できます。

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.massiveRun { // 各コルーチンをDefaultDispathcerで実行する
        withContext(counterContext) { // それぞれのインクリメントを単一スレッドのコンテキストに限定する
            counter++
        }
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-04.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

このコードは _細粒度_ のスレッド制約を行うため、非常にゆっくりと動作します。
個々のインクリメントは[withContext]ブロックを使用してマルチスレッドの[Dispatchers.Default]コンテキストからシングルスレッドのコンテキストに切り替わります。

### 粗粒度のスレッド制約

現実にはスレッド制約は大きなチャンクで行われます。例えば、状態を更新するビジネスロジックの大きな部分は単一のスレッドに限定されます。
次の例では、そのようにしてシングルスレッドコンテキストで各コルーチンを起動して実行します。
ここでは、[CoroutineScope()]関数を使用して、コルーチンのコンテキスト参照を[CoroutineScope]に変換します。

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    CoroutineScope(counterContext).massiveRun { // 各コルーチンをシングルスレッドコンテキストで実行する
        counter++
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-05.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

これで、はるかに高速に動作し正しい結果が得られます。

### 排他制御

この問題に対する排他制御の解決策は、決して同時に実行されない _クリティカルセクション_ で共有状態のすべての変更を保護することです。
ブロックする世界では通常 `synchronized` または `ReentrantLock` を使用します。
コルーチンの代案は[Mutex]と呼ばれています。
それはクリティカルセクションを区切る[lock][Mutex.lock]と[unlock][Mutex.unlock]関数を持っています。
主な違いは、 `Mutex.lock()` はサスペンド関数であることです。
これはスレッドをブロックしません。

`mutex.lock(); try { ... } finally { mutex.unlock() }` パターンを表す便利な[withLock]拡張関数もあります。

```kotlin
val mutex = Mutex()
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    GlobalScope.massiveRun {
        mutex.withLock {
            counter++        
        }
    }
    println("Counter = $counter")
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-06.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

この例でのロックは細粒度なので、代償を払っています。
しかし、共有状態を定期的に変更しなければならない状況には適していますが、この状態が限定された自然なスレッドはありません。

### アクター

[actor](https://en.wikipedia.org/wiki/Actor_model)は、コルーチン、このコルーチンに閉じ込められカプセル化された状態、および他のコルーチンと通信するためのチャンネルの組み合わせで構成されるエンティティです 。
単純なアクターは関数として記述できますが、複雑な状態のアクターはクラスに適しています。

[actor]コルーチンビルダーがアクターのメールボックスチャネルをメッセージを受信するスコープに結合し、
結果のジョブオブジェクトに送信チャネルを結合するので、アクターへの単一の参照をそのハンドルとして持ち運ぶことができます。

アクターを使用する最初のステップは、アクターが処理するメッセージのクラスを定義することです。
Kotlinの[シールドクラス](https://kotlinlang.org/docs/reference/sealed-classes.html)はその目的に適しています。
カウンタをインクリメントする `IncCounter` メッセージと、その値を取得する `GetCounter` メッセージを持つ `CounterMsg` シールドクラスを定義します。
後で応答を送信する必要があります。 将来知られる（通信される）単一の値を表す[CompletableDeferred]通信プリミティブは、その目的のためにここで使用されます。

```kotlin
// counterActorのメッセージ型
sealed class CounterMsg
object IncCounter : CounterMsg() // カウンターをインクリメントする一方向のメッセージ
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 返信を持ったリクエスト
```

次に、[actor]コルーチンビルダーを使用してアクターを起動する関数を定義します。

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

メインコードは簡単です。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val counter = counterActor() // アクターを作る
    GlobalScope.massiveRun {
        counter.send(IncCounter)
    }
    // アクターからカウンター値を得るためのメッセージを送る
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // アクターを終了する
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-sync-07.kt)で完全なコードを取得できます

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

アクター自体がどのようなコンテキストで実行されるかは（正確さにおいて）問題ではありません。
アクターはコルーチンでありコルーチンはシーケンシャルに実行されるので、特定のコルーチンへの状態の閉じ込めは共有ミュータブルステートの問題に対する解決策として機能します。
実際、アクターは自分のプライベートな状態を変更することができますが、メッセージを通じてのみ互いに影響を与えることができます（ロックを必要としません）。

この場合、常に実行する作業があり別のコンテキストに切り替える必要がないため、負荷の下ではロックよりもアクターのほうが効率的です。

> [actor]コルーチンビルダーは二重の[produce]コルーチンビルダーであることに注意してください。
  アクターはメッセージを受信するチャネルに関連付けられ、プロデューサーは要素を送信するチャネルに関連付けられます。

## セレクト式（実験的）

セレクト式を使用すると複数のサスペンド関数を同時に待つことができ、利用可能になった最初のものを _選択_ することができます。

> セレクト式は `kotlinx.coroutines` の実験的な機能です。 これらのAPIは、 `kotlinx.coroutines` ライブラリの今後のアップデートで、潜在的に大きな変化を伴って進化することが期待されています。

<!--- INCLUDE .*/example-select-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.selects.*
-->

### チャネルからの選択

2つの文字列のプロデューサー、 `fizz` と `buzz` があります。 `fizz` は300ミリ秒ごとに"Fizz"文字列を生成します。

<!--- INCLUDE
import kotlinx.coroutines.experimental.*
import kotlin.coroutines.experimental.*
-->

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // 300ミリ秒ごとに "Fizz" を送る
        delay(300)
        send("Fizz")
    }
}
```

`buzz` は500ミリ秒ごとに "Buzz!" 文字列を生成します。

```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // 500ミリ秒ごとに "Buzz!" を送る
        delay(500)
        send("Buzz!")
    }
}
```

[receive][ReceiveChannel.receive]サスペンド関数を使用すると、一方のチャネルから _または_ 他方のチャネルから受信することができます。
しかし、[select]式は、[onReceive][ReceiveChannel.onReceive]節を使って _両方_ から同時に受け取ることができます。

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
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren() // cancel fizz & buzz coroutines    
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-select-01.kt)で完全なコードを取得できます

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

チャネルが閉じられると `select` の[onReceive][ReceiveChannel.onReceive]節が失敗し、対応する `select` が例外をスローします。
[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]節を使用して、チャンネルが閉じられたときに特定のアクションを実行できます。
次の例は、 `select` が選択された節の結果を返す式であることも示しています。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

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
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    repeat(8) { // 最初の8個の結果をプリントする
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()    
}
```

> [ここ](core/kotlinx-coroutines-core/test/guide/example-select-02.kt)で完全なコードを取得できます

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

2番目の所見は、[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]は、チャネルが既に閉じられているときに直ちに選択されることです。

### 送信の選択

Select式には、[onSend][SendChannel.onSend]節があり、選択のバイアスされた性質と組み合わせてとても有効に使用できます。

プライマリチャネルのコンシューマーが送信に追いつかないときに、その値を `side` チャネルに送る整数のプロデューサーの例を書きましょう。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
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
    launch { // これはサイドチャネルの非常に高速なコンシューマー
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // 急がずに、消費した値をきっちり消化する
    }
    println("Done consuming")
    coroutineContext.cancelChildren()    
}
``` 
 
> [ここ](core/kotlinx-coroutines-core/test/guide/example-select-03.kt)で完全なコードを取得できます
  
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

遅延値は、[onAwait][Deferred.onAwait]節を使用して選択できます。
ランダム遅延の後に遅延文字列値を返す非同期関数から始めましょう。

<!--- INCLUDE .*/example-select-04.kt
import java.util.*
-->

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```

ランダムな遅延でこれを1ダース開始してみましょう。

```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-select-04.kt)で完全なコードを取得できます

出力は、

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 延期された値のチャネルの切り替え

次の遅延値が来るかチャネルが閉じられるまで、遅延ストリング値のチャネルを消費し、受信した遅延値を待つチャネルプロデューサー関数を書きましょう。
この例では、同じ `select` に[onReceiveOrNull][ReceiveChannel.onReceiveOrNull]節と[onAwait][Deferred.onAwait]節を入れています。

<!--- INCLUDE
import kotlin.coroutines.experimental.*
-->

```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
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
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```

メイン関数は、単に `switchMapDeferreds` の結果をプリントするコルーチンを起動し、いくつかのテストデータを送信するだけです。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val chan = Channel<Deferred<String>>() // テスト用のチャネル
    launch { // プリント用のコルーチンを起動する
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

> [ここ](core/kotlinx-coroutines-core/test/guide/example-select-05.kt)で完全なコードを取得できます

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

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/join.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/coroutine-scope.html
[currentScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/current-scope.html
[cancelAndJoin]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/cancel-and-join.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/cancel.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception/index.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/yield.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/is-active.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-context.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-non-cancellable/index.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout-or-null.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/start.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-dispatcher/index.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-dispatchers/-unconfined.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-dispatchers/-default.html
[newSingleThreadContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-single-thread-context.html
[ThreadPoolDispatcher.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-thread-pool-dispatcher/close.html
[newCoroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-coroutine-context.html
[CoroutineScope.coroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/coroutine-context.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-name/index.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job.html
[asContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/java.lang.-thread-local/as-context-element.html
[ThreadContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-thread-context-element/index.html
[CoroutineExceptionHandler]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-exception-handler/index.html
[kotlin.coroutines.experimental.CoroutineContext.cancelChildren]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/kotlin.coroutines.experimental.-coroutine-context/cancel-children.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-completable-deferred/index.html
[Deferred.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/on-await.html
<!--- INDEX kotlinx.coroutines.experimental.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.experimental.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/receive.html
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/send.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/close.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/consume-each.html
[Channel()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel.html
[ticker]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/ticker.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/cancel.html
[TickerMode.FIXED_DELAY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-ticker-mode/-f-i-x-e-d_-d-e-l-a-y.html
[ReceiveChannel.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/on-receive.html
[ReceiveChannel.onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/on-receive-or-null.html
[SendChannel.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/on-send.html
<!--- INDEX kotlinx.coroutines.experimental.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/select.html
<!--- END -->
