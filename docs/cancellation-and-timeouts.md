<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/CancellationTimeOutsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class CancellationTimeOutsGuideTest {
--> 
## 目次

<!--- TOC -->

* [キャンセルとタイムアウト](#キャンセルとタイムアウト)
  * [コルーチンの実行をキャンセルする](#コルーチンの実行をキャンセルする)
  * [キャンセルは協調的](#キャンセルは協調的)
  * [計算コードをキャンセル可能にする](#計算コードをキャンセル可能にする)
  * [finallyでリソースを閉じる](#finallyでリソースを閉じる)
  * [キャンセル不可ブロックの実行](#キャンセル不可ブロックの実行)
  * [タイムアウト](#タイムアウト)

<!--- END_TOC -->

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-01.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-02.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-03.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-04.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-05.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-06.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-cancel-07.kt)で完全なコードを取得できます

このコードを実行しても例外は出ません。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

<!--- TEST -->
