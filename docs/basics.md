<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/BasicsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class BasicsGuideTest {
--> 

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-02.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-03.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-03s.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-04.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-05.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-05s.kt)で完全なコードを取得できます。

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-06.kt)で完全なコードを取得できます

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

> [ここ](../core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)で完全なコードを取得できます

実行すると、3行を出力して終了することがわかります。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

[GlobalScope]で起動されたアクティブなコルーチンはプロセスを生かし続けるわけではありません。それらはデーモンスレッドのようなものです。
