<!--- TEST_NAME BasicsGuideTest -->

**目次**

<!--- TOC -->

* [コルーチンの基礎](#コルーチンの基礎)
  * [初めてのコルーチン](#初めてのコルーチン)
  * [ブロッキングとノンブロッキングの世界の橋渡し](#ブロッキングとノンブロッキングの世界の橋渡し)
  * [ジョブを待つ](#ジョブを待つ)
  * [構造化並行性](#構造化並行性)
  * [スコープビルダー](#スコープビルダー)
  * [関数抽出リファクタリング](#関数抽出リファクタリング)
  * [コルーチンは軽量](#コルーチンは軽量)
  * [グローバルコルーチンはデーモンスレッドに似ている](#グローバルコルーチンはデーモンスレッドに似ている)

<!--- END -->

## コルーチンの基礎

このセクションでは、基本的なコルーチンの概念について説明します。

### 初めてのコルーチン

次のコードを実行します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // バックグラウンドで新しいコルーチンを起動し、続行する
        delay(1000L) // 1秒間ノンブロッキング遅延 (デフォルトの時間単位はms)
        println("World!") // delayのあとでプリント
    }
    println("Hello,") // コルーチンが遅延している間、メインスレッドは継続する
    Thread.sleep(2000L) // メインスレッドを2秒間ブロックしてJVMを存続させます
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt)で完全なコードを取得できます。

次の結果が表示されます。

```text
Hello,
World!
```

<!--- TEST -->

基本的に、コルーチンは軽量スレッドです。
それらは[CoroutineScope]のコンテキストで[launch] _コルーチンビルダー_ で起動されます。
ここでは、[GlobalScope]で新しいコルーチンを起動しています。つまり、新しいコルーチンの存続期間は、アプリケーション全体の存続期間によってのみ制限されます。

`GlobalScope.launch { ... }` を `thread { ... }` に、 `delay(...)` を `Thread.sleep(...)` に置き換えても同じ結果が得られます。試してみてください（`kotlin.concurrent.thread`をインポートすることを忘れないでください）。

`GlobalScope.launch` を `thread` に置き換えることから始めると、コンパイラは次のエラーを生成します。

```
Error: Kotlin: サスペンド関数は、コルーチンまたは他のサスペンド関数からのみ呼び出すことができます
```

これは、[delay]がスレッドをブロックせずにコルーチンを _中断_ する特別な _サスペンド関数_ であり、コルーチンからのみ使用できるためです。

### ブロッキングとノンブロッキングの世界の橋渡し

最初の例では、 同じコードに _ノンブロッキング_ `delay(...)` と _ブロッキング_ `Thread.sleep(...)` を混在させています。
これではどちらがブロックされていて、どちらがブロックされていないかを簡単に見失います。
[runBlocking]コルーチンビルダーを使用して、ブロッキングを明確にしましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
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

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-02.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は同じですが、このコードではノンブロッキング[delay]のみを使用しています。
`runBlocking` を呼び出すメインスレッドは、`runBlocking` 内のコルーチンが完了するまで _ブロック_ します。

この例は、 `runBlocking` を使ってmain関数の実行をラップすることにより、より慣用的な方法で書き直すこともできます：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> { // メインコルーチンを開始
    GlobalScope.launch { // バックグラウンドで新しいコルーチンを起動し、続行する
        delay(1000L)
        println("World!")
    }
    println("Hello,") // メインコルーチンはすぐにここに続く
    delay(2000L)      // JVMを維持するために2秒間遅延する
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-03.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

ここで `runBlocking<Unit> { ... }` は、トップレベルのメインコルーチンを起動するためのアダプタとして機能します。
Kotlinの適格な `main` 関数は `Unit` を返さなければならないので、リターンタイプ `Unit` を明示的に指定します。

これはまた、サスペンド関数の単体テストを書く方法でもあります。

<!--- INCLUDE
import kotlinx.coroutines.*
-->

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // ここでは、好きなアサーションスタイルを使ってサスペンド関数を使うことができます
    }
}
```

</div>

<!--- CLEAR -->

### ジョブを待つ

別のコルーチンが動作している間遅延させるのは良い方法ではありません。
立ち上げたバックグラウンド[Job]が完了するまで明示的に（ノンブロッキングの方法で）待ちましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    val job = GlobalScope.launch { // 新しいコルーチンを起動し、そのJobへの参照を保持する
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 子コルーチンが完了するまで待つ
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-04.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

結果は変わりませんが、メインコルーチンのコードはバックグラウンドジョブの継続時間に結びついていません。ずっと良いです。

### 構造化並行性

コルーチンを実際に使用するには、まだ何かが必要です。
`GlobalScope.launch` を使うと、トップレベルのコルーチンが作成されます。 軽量であるにもかかわらず、実行中にいくらかのメモリリソースを消費します。 新しく起動したコルーチンの参照を保持するのを忘れた場合でも実行されます。 もしコルーチンのコードがハングして（例えば、長すぎる誤った遅延）、多くのコルーチンを起動しすぎてメモリが足りなくなった場合はどうなりますか？
起動されたすべてのコルーチンへの参照を手動で保持し、[Join][Job.join]する必要があるとエラーが発生しやすくなります。

より良い解決策があります。私たちのコードでは、構造化並行性を使用できます。
[GlobalScope]でコルーチンを起動するのではなく、通常はスレッド（スレッドは常にグローバル）と同様に、実行中の操作の特定の範囲でコルーチンを起動することができます。

この例では、[runBlocking]コルーチンビルダーを使用してコルーチンに変換される `main` 関数があります。
`runBlocking` を含むすべてのコルーチンビルダーは、そのコードブロックのスコープに[CoroutineScope]のインスタンスを追加します。
スコープで起動されたコルーチンがすべて完了するまで外部コルーチン（この例では `runBlocking`）が完了しないため、明示的に `join` せずにこのスコープでコルーチンを起動することができます。
したがって、例を簡単にすることができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> { // this: CoroutineScope
    launch { // runBlockingのスコープで新しいコルーチンを起動
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-05.kt)で完全なコードを取得できます。

<!--- TEST
Hello,
World!
-->

### スコープビルダー

さまざまなビルダーによって提供されるコルーチンスコープに加えて、[coroutineScope][_ coroutineScope]ビルダーを使用して独自のスコープを宣言することができます。
これはコルーチンスコープを作成し、起動されたすべての子が完了するまで完了しません。

[runBlocking]と[coroutineScope][_coroutineScope]は、どちらもその本体とすべての子が完了するのを待つため、似ているように見える場合があります。
主な違いは、[runBlocking]メソッドが現在のスレッドを待機するために _ブロック_ するのに対し、[coroutineScope][_coroutineScope]は一時停止するだけで、基になるスレッドを他の用途に解放することです。
その違いのため、[runBlocking]は通常の関数であり、[coroutineScope][_coroutineScope]はサスペンド関数です。

これは、次の例で示すことができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    coroutineScope { // コルーチンスコープを作る
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

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-06.kt)で完全なコードを取得できます。

<!--- TEST
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
-->

[coroutineScope][_coroutineScope]がまだ完了していない場合でも、「Task from coroutine scope」メッセージの直後（ネストされた起動を待機中）に「Task from runBlocking」が実行されて出力されることに注意してください。

### 関数抽出リファクタリング

`launch { ... }` 内のコードブロックを別の関数に抽出してみましょう。
このコードで「Extract function」リファクタリングを実行すると、 `suspend` 修飾子付きの新しい関数が得られます。
これがあなたの最初の _サスペンド関数_ です。
サスペンド関数は、通常の関数と同じようにコルーチン内で使用できますが、追加の機能として、他のサスペンド関数（この例では 'delay' など）を使用してコルーチンの実行を _一時停止_ することができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// これはあなたの最初のサスペンド関数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-07.kt)で完全なコードを取得できます

<!--- TEST
Hello,
World!
-->


しかし、抽出された関数に現在のスコープで呼び出されるコルーチンビルダーが含まれているとしたらどうでしょうか？
この場合、抽出された関数の `suspend` 修飾子では不十分です。
`doWorld` を `CoroutineScope` の拡張メソッドにすることは解決策の1つですが、APIが明確にならないので、常に適用できるとは限りません。
慣用的な解決策は、ターゲット関数を含むクラスのフィールドとして明示的な `CoroutineScope` を使用するか、外部クラスが `CoroutineScope` を実装するときに暗黙的なフィールドを使用することです。
最後の手段として、[CoroutineScope(coroutineContext)][CoroutineScope()]を使用できますが、このメソッドの実行範囲を制御できないため、このようなアプローチは構造的に安全ではありません。
このビルダーを使用できるのはプライベートAPIのみです。

### コルーチンは軽量

次のコードを実行します。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // たくさんのコルーチンを起動する
        launch {
            delay(5000L)
            print(".")
        }
    }
    jobs.forEach { it.join() } // すべてのジョブが完了するのを待つ
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-08.kt)で完全なコードを取得できます

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

10万個のコルーチンを起動し、5秒後に各コルーチンがドットをプリントします。

スレッドを使って試したらどうなるでしょうか？ （ほとんどの場合、あなたのコードはメモリ不足エラーを引き起こすでしょう）

### グローバルコルーチンはデーモンスレッドに似ている

次のコードでは、「I'm sleeping」というメッセージを毎秒2回プリントする長時間実行するコルーチンを[GlobalScope]で起動し、ある程度遅れてmain関数からリターンします。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 遅れて終了する
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-basic-09.kt)で完全なコードを取得できます

実行すると、3行を出力して終了することがわかります。

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

[GlobalScope]で起動されたアクティブなコルーチンはプロセスを生かし続けるわけではありません。それらはデーモンスレッドのようなものです。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
<!--- END -->


