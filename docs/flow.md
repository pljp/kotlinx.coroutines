<!--- TEST_NAME FlowGuideTest -->

**目次**

<!--- TOC -->

* [非同期フロー](#非同期フロー)
  * [複数の値を表す](#複数の値を表す)
    * [シーケンス](#シーケンス)
    * [サスペンド関数](#サスペンド関数)
    * [フロー](#フロー)
  * [フローはコールド](#フローはコールド)
  * [フローキャンセルの基本](#フローキャンセルの基本)
  * [フロービルダー](#フロービルダー)
  * [中間フロー演算子](#中間フロー演算子)
    * [変換演算子](#変換演算子)
    * [サイズ制限演算子](#サイズ制限演算子)
  * [終端フロー演算子](#終端フロー演算子)
  * [フローはシーケンシャル](#フローはシーケンシャル)
  * [フローコンテキスト](#フローコンテキスト)
    * [withContextの不適切な放出](#withcontextの不適切な放出)
    * [flowOnオペレーター](#flowonオペレーター)
  * [バッファリング](#バッファリング)
    * [合成](#合成)
    * [最新の値を処理する](#最新の値を処理する)
  * [複数のフローを構成する](#複数のフローを構成する)
    * [Zip](#zip)
    * [Combine](#combine)
  * [フローを平坦化する](#フローを平坦化する)
    * [flatMapConcat](#flatmapconcat)
    * [flatMapMerge](#flatmapmerge)
    * [flatMapLatest](#flatmaplatest)
  * [フロー例外](#フロー例外)
    * [コレクターのtryとcatch](#コレクターのtryとcatch)
    * [すべてがキャッチされる](#すべてがキャッチされる)
  * [例外の透過性](#例外の透過性)
    * [透過的なキャッチ](#透過的なキャッチ)
    * [宣言的にキャッチする](#宣言的にキャッチする)
  * [フローの完了](#フローの完了)
    * [命令的な最終ブロック](#命令的な最終ブロック)
    * [宣言的な取り扱い](#宣言的な取り扱い)
    * [正常終了](#正常終了)
  * [命令型と宣言型](#命令型と宣言型)
  * [フローの起動](#フローの起動)
  * [フローキャンセルのチェック](#フローキャンセルのチェック)
    * [忙しいフローをキャンセル可能にする](#忙しいフローをキャンセル可能にする)
  * [フローとリアクティブストリーム](#フローとリアクティブストリーム)

<!--- END -->

## 非同期フロー

サスペンド関数は非同期で単一の値を返しますが、非同期で計算された複数の値を返すにはどのようにすればよいですか？ そこでKotlinフローの出番です。

### 複数の値を表す

[collections]を使用して、Kotlinで複数の値を表すことができます。
例えば、3つの数字の[List]を返す `simple` 関数があり、[forEach]を使用してすべてをプリントできます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun simple(): List<Int> = listOf(1, 2, 3)

fun main() {
    simple().forEach { value -> println(value) }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-01.kt)から完全なコードを取得できます。

このコードの出力は次のようになります。

```text
1
2
3
```

<!--- TEST -->

#### シーケンス

CPUを消費するブロッキングコードを使用して数値を計算している場合（各計算には100ミリ秒かかります）、[Sequence]を使用して数値を表すことができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun simple(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() {
    simple().forEach { value -> println(value) }
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-02.kt)から完全なコードを取得できます。

このコードは同じ数字を出力しますが、それぞれを印刷する前に100ミリ秒待機します。

<!--- TEST
1
2
3
-->

#### サスペンド関数

ただし、この計算はコードを実行しているメインスレッドをブロックします。
それらの値が非同期コードによって計算されるとき、 `simple` 関数 を `suspend` 修飾子でマークできるので、ブロックせずに作業を実行し、結果をリストとして返すことができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

//sampleStart
suspend fun simple(): List<Int> {
    delay(1000) // pretend we are doing something asynchronous here
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    simple().forEach { value -> println(value) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-03.kt)から完全なコードを取得できます。

このコードは、1秒待ってから数値を出力します。

<!--- TEST
1
2
3
-->

#### フロー

`List<Int>` の結果型を使用すると、一度にすべての値を返すことしかできないことを意味します。
非同期で計算される値のストリームを表すために、同期的に計算される値に `Sequence<Int>` 型を使用するのと同じように、[`Flow<Int>`][Flow]タイプを使用できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to check if the main thread is blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    simple().collect { value -> println(value) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-04.kt)から完全なコードを取得できます。

このコードは各数値をプリントする前に100ミリ秒待機しますが、メインスレッドをブロックしません。
これは、メインスレッドで実行されている別のコルーチンから100ミリ秒ごとに "I'm not blocked" と出力することで確認されます。

```text
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

<!--- TEST -->

前の例とは[Flow]のコードに以下の違いがあることに注意してください。

* [Flow]型のビルダー関数は[flow]です。
* `flow {...}` ビルダーブロック内のコードは中断できます。
* `simple` 関数は `suspend` 修飾子でマークされなくなりました。
* 値は、[emit][FlowCollector.emit]関数を使用してフローから _放出_ されます。
* 値は[collect][collect]関数を使用してフローから _収集_ されます。

> `simple` の `flow { ... }` 本文の[delay]を `Thread.sleep` に置き換えると、この場合メインスレッドがブロックされることがわかります。

### フローはコールド

フローは、シーケンスと同様に _コールド_ ストリームです。[flow][_flow]ビルダー内のコードは、フローが収集されるまで実行されません。
これは、次の例で明らかになります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling simple function...")
    val flow = simple()
    println("Calling collect...")
    flow.collect { value -> println(value) }
    println("Calling collect again...")
    flow.collect { value -> println(value) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-05.kt)から完全なコードを取得できます。

次のようにプリントします。

```text
Calling simple function...
Calling collect...
Flow started
1
2
3
Calling collect again...
Flow started
1
2
3
```

<!--- TEST -->

これが `simple` 関数（フローを返す）が `suspend` 修飾子でマークされていない主な理由です。
`simple` 呼び出しはすぐにリターンし、それ自体では何も待つことはありません。
フローは収集されるたびに開始さるため、 `collect` を再度呼び出すと「Flow started」とプリントされます。

### フローキャンセルの基本

フローは、コルーチンの一般的な協調キャンセルに準拠しています。
通常、フローがキャンセル可能なサスペンド関数（[delay]など）で一時停止されると、フロー収集をキャンセルできます。

次の例は、[withTimeoutOrNull]ブロックが実行されているとき、タイムアウトでどのようにフローがキャンセルされ、コードの実行が停止するかを示しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // Timeout after 250ms
        simple().collect { value -> println(value) }
    }
    println("Done")
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-06.kt)から完全なコードを取得できます。

`simple` 関数のフローによって2つの数値のみが放出され、次の出力が生成されることに注目してください。

```text
Emitting 1
1
Emitting 2
2
Done
```

<!--- TEST -->

詳細については、[フローキャンセルのチェック](#フローキャンセルのチェック)セクションを参照してください。

### フロービルダー

前の例の `flow { ... }` ビルダーは最も基本的なものです。
フローの宣言を簡単にするための他のビルダーがあります。

* 値の固定セットを放出するフローを定義する[flowOf]ビルダー。
* さまざまなコレクションとシーケンスは、 `.asFlow()` 拡張関数を使用してフローに変換できます。

したがって、フローから1～3の数字をプリントする例は、次のように記述できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    // Convert an integer range to a flow
    (1..3).asFlow().collect { value -> println(value) }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-07.kt)から完全なコードを取得できます。

<!--- TEST
1
2
3
-->

### 中間フロー演算子

フローは、コレクションやシーケンスと同様に演算子で変換できます。
中間演算子はアップストリームフローに適用され、ダウンストリームフローを返します。
これらの演算子は、フローがそうであるようにコールドです。 そのような演算子への呼び出しは、それ自体はサスペンド関数ではありません。
すぐに機能し、新しい変換されたフローの定義を返します。

基本的な演算子には、[map]や[filter]などのよく知られた名前があります。
シーケンスとの重要な違いは、これらの演算子内のコードブロックがサスペンド関数を呼び出すことができることです。

例えば、リクエストの実行が、サスペンド関数によって実装される長時間実行される操作であっても、[map]演算子を使用して、リクエストを受けるフローを結果にマップできます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-08.kt)から完全なコードを取得できます。

次の3行が生成され、各行が1秒ごとに表示されます。

```text
response 1
response 2
response 3
```

<!--- TEST -->

#### 変換演算子

フロー変換演算子の中で、最も一般的なものは[transform]と呼ばれます。 [map]や[filter]などの単純な変換を模倣したり、より複雑な変換を実装したりするために使用できます。
`transform` 演算子を使用すると、任意の値を任意の回数だけ[emit][FlowCollector.emit]できます。

例えば、 `transform` を使用すると、長時間実行される非同期リクエストを実行する前に文字列を出力し、それに応答を続けることができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
//sampleStart
    (1..3).asFlow() // a flow of requests
        .transform { request ->
            emit("Making request $request")
            emit(performRequest(request))
        }
        .collect { response -> println(response) }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-09.kt)から完全なコードを取得できます。

このコードの出力は次のようになります。

```text
Making request 1
response 1
Making request 2
response 2
Making request 3
response 3
```

<!--- TEST -->

#### サイズ制限演算子

[take]のようなサイズを制限する中間演算子は、対応する制限に達するとフローの実行をキャンセルします。
コルーチンのキャンセルは常に例外をスローすることにより実行され、すべてのリソース管理機能（ `try { ... } finally { ... }` ブロックなど）がキャンセルの場合に正常に動作します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun numbers(): Flow<Int> = flow {
    try {
        emit(1)
        emit(2)
        println("This line will not execute")
        emit(3)
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers()
        .take(2) // take only the first two
        .collect { value -> println(value) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-10.kt)から完全なコードを取得できます。

このコードの出力は、 `numbers()` 関数内の `flow { ... }` 本文の実行が2番目の数値を出力した後に停止したことを明確に示しています。

```text
1
2
Finally in numbers
```

<!--- TEST -->

### 終端フロー演算子

フローのターミナルオペレーターは、フローの収集を開始する _サスペンド関数_ です。
[collect]演算子は最も基本的な演算子ですが、他にも簡単にするための終端演算子があります。

* [toList]や[toSet]などのさまざまなコレクションへの変換。
* 最初の値を取得する[first]演算子、フローが単一の値を発行することを保証する[single]演算子。
* [reduce]と[fold]を使用してフローを一つの値にまとめます。

例:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    val sum = (1..5).asFlow()
        .map { it * it } // squares of numbers from 1 to 5
        .reduce { a, b -> a + b } // sum them (terminal operator)
    println(sum)
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-11.kt)から完全なコードを取得できます。

単一の数字をプリントします。

```text
55
```

<!--- TEST -->

### フローはシーケンシャル

複数のフローを操作する特別な演算子が使用されない限り、フローの個々の収集は順番に実行されます。
収集は、終端演算子を呼び出すコルーチンで直接動作します。
デフォルトでは、新しいコルーチンは起動されません。
放出された各値は、アップストリームからダウンストリームまでのすべての中間演算子によって処理され、その後、終端演算子に配信されます。

偶数の整数をフィルタリングして文字列にマッピングする次の例を参照してください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    (1..5).asFlow()
        .filter {
            println("Filter $it")
            it % 2 == 0
        }
        .map {
            println("Map $it")
            "string $it"
        }.collect {
            println("Collect $it")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-12.kt)から完全なコードを取得できます。

次のように出力されます。

```text
Filter 1
Filter 2
Map 2
Collect string 2
Filter 3
Filter 4
Map 4
Collect string 4
Filter 5
```

<!--- TEST -->

### フローコンテキスト

フローの収集は、常に呼び出し元のコルーチンのコンテキストで行われます。
例えば `simple` フローがある場合、次のコードは `simple` フローの実装の詳細に関係なく、このコードの作成者によって指定されたコンテキストで実行されます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
withContext(context) {
    simple().collect { value ->
        println(value) // run in the specified context
    }
}
```

</div>

<!--- CLEAR -->

フローのこのプロパティは、 _コンテキストの維持_ と呼ばれます。

そのため、デフォルトでは、 `flow { ... }` ビルダーのコードは、対応するフローのコレクターによって提供されるコンテキストで実行されます。
例えば、呼び出されるスレッドをプリントし、3つの数値を放出する `simple` 関数の実装を考えます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

//sampleStart
fun simple(): Flow<Int> = flow {
    log("Started simple flow")
    for (i in 1..3) {
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple().collect { value -> log("Collected $value") }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-13.kt)から完全なコードを取得できます。

このコードを実行すると、次のように生成されます。

```text
[main @coroutine#1] Started simple flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

<!--- TEST FLEXIBLE_THREAD -->

`simple().collect` はメインスレッドから呼び出されるため、 `simple` のフローの本文もメインスレッドで呼び出されます。
これは、実行コンテキストを気にせず、呼び出し元をブロックしない高速実行または非同期コードの完全なデフォルトです。

#### withContextの不適切な放出

ただし、長時間実行されるCPU消費コードは[Dispatchers.Default]のコンテキストで実行する必要があり、UI更新コードは[Dispatchers.Main]のコンテキストで実行する必要があります。
通常、[withContext]はKotlinコルーチンを使用するコードのコンテキストを変更するために使用されますが、 `flow { ... }` ビルダーのコードはコンテキスト維持プロパティを尊重する必要があり、異なるコンテキストからの[emit][FlowCollector.emit]を許可しません。

次のコードを実行してみてください。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    // The WRONG way to change context for CPU-consuming code in flow builder
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // pretend we are computing it in CPU-consuming way
            emit(i) // emit next value
        }
    }
}

fun main() = runBlocking<Unit> {
    simple().collect { value -> println(value) }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-14.kt)から完全なコードを取得できます。

このコードは次の例外を出します。

```text
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
		Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
		but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, Dispatchers.Default].
		Please refer to 'flow' documentation or use 'flowOn' instead
	at ...
```

<!--- TEST EXCEPTION -->

#### flowOnオペレーター

例外は、フロー放出のコンテキストを変更するために使用される[flowOn]関数を参照します。
フローのコンテキストを変更する正しい方法を以下の例に示します。この例では、対応するスレッドの名前もプリントして、すべてがどのように機能するかを示しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

//sampleStart
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it in CPU-consuming way
        log("Emitting $i")
        emit(i) // emit next value
    }
}.flowOn(Dispatchers.Default) // RIGHT way to change context for CPU-consuming code in flow builder

fun main() = runBlocking<Unit> {
    simple().collect { value ->
        log("Collected $value")
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-15.kt)から完全なコードを取得できます。

バックグラウンドスレッドでの `flow { ... }` の動作に注意してください。収集はメインスレッドで行われます。

<!--- TEST FLEXIBLE_THREAD
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
-->

ここで注意すべきもう1つの点は、[flowOn]演算子がフローのデフォルトのシーケンシャルな性質を変更したことです。
現在、収集は一方のコルーチン（"coroutine#1"）で発生し、放出はコルーチンの収集と並行して別のスレッドで実行されているもう一方のコルーチン（"coroutine#2"）で発生します。
[flowOn]演算子は、コンテキストで[CoroutineDispatcher]を変更する必要がある場合、アップストリームフローの別のコルーチンを作成します。

### バッファリング

異なるコルーチンでフローの異なる部分を実行すると、特に長時間実行される非同期操作が関係する場合に、フローの収集にかかる全体的な時間の観点から役立ちます。
例えば、 `simple()` フローによる放出が遅く、要素の生成に100ミリ秒かかり、また、コレクターも遅く、要素の処理に300ミリ秒かかる場合を考えます。
このような3つの数値を持つフローを収集するのにかかる時間を見てみましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

//sampleStart
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple().collect { value ->
            delay(300) // pretend we are processing it for 300 ms
            println(value)
        }
    }
    println("Collected in $time ms")
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-16.kt)から完全なコードを取得できます。

これは次のようなものを生成し、全体を収集するのに約1200ミリ秒（3つの数字それぞれで400ミリ秒）かかります。

```text
1
2
3
Collected in 1220 ms
```

<!--- TEST ARBITRARY_TIME -->

シーケンシャルに実行するのとは反対に、フローで[buffer]演算子を使用してコードの収集と並行して `simple()` フローの放出コードを実行できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        simple()
            .buffer() // buffer emissions, don't wait
            .collect { value ->
                delay(300) // pretend we are processing it for 300 ms
                println(value)
            }
    }
    println("Collected in $time ms")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-17.kt)から完全なコードを取得できます。

処理パイプラインを効果的に作成したため、同じ数値がより速く生成されます。最初の数値を100ミリ秒待つだけで、各数値の処理に300ミリ秒しかかかりません。
この方法では、実行に約1000ミリ秒かかります。

```text
1
2
3
Collected in 1071 ms
```

<!--- TEST ARBITRARY_TIME -->

> [flowOn]演算子は[CoroutineDispatcher]を変更する必要がある場合、同じバッファリングメカニズムを使用しますが、ここでは実行コンテキストを変更せずに明示的にバッファリングを要求します。

#### 合成

フローが操作の部分的な結果や操作状態の更新を表す場合、各値を処理する必要はなく、代わりに最新の値のみを処理する必要がある場合があります。
この場合、[conflate]演算子を使用して、コレクターが処理するには遅すぎるときに中間値をスキップできます。
前の例に基づいて作成します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        simple()
            .conflate() // conflate emissions, don't process each one
            .collect { value ->
                delay(300) // pretend we are processing it for 300 ms
                println(value)
            }
    }
    println("Collected in $time ms")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-18.kt)から完全なコードを取得できます。

最初の番号がまだ処理されている間に、2番目と3番目の番号がすでに生成されているため、2番目の番号が合成され、最新の（3番目の）番号のみがコレクターに配信されました。

```text
1
3
Collected in 758 ms
```

<!--- TEST ARBITRARY_TIME -->

#### 最新の値を処理する

エミッターとコレクターの両方が遅い場合、合成は処理を高速化する1つの方法です。
放出された値をドロップすることでそれを行います。
もう1つの方法は、遅いコレクターをキャンセルし、新しい値が放出されるたびに再始動することです。
`xxx` 演算子と同じ基本的なロジックを実行し、新しい値でブロック内のコードをキャンセルする `xxxLatest` 演算子のファミリーがあります。
前の例の[conflate]を[collectLatest]に変更してみましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // cancel & restart on the latest value
                println("Collecting $value")
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value")
            }
    }
    println("Collected in $time ms")
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-19.kt)から完全なコードを取得できます。

[collectLatest]の本体には300ミリ秒かかりますが、新しい値は100ミリ秒ごとに放出されるため、ブロックはすべての値で実行されますが、最後の値でのみ完了することがわかります。

```text
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
```

<!--- TEST ARBITRARY_TIME -->

### 複数のフローを構成する

複数のフローを構成する方法はたくさんあります。

#### Zip

Kotlin標準ライブラリの[Sequence.zip]拡張関数と同様に、フローには2つのフローの対応する値を結合する[zip]演算子があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    val nums = (1..3).asFlow() // numbers 1..3
    val strs = flowOf("one", "two", "three") // strings
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string
        .collect { println(it) } // collect and print
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-20.kt)から完全なコードを取得できます。

この例は次のようにプリントします。

```text
1 -> one
2 -> two
3 -> three
```

<!--- TEST -->

#### Combine

フローが変数または操作（[合成](#合成)の関連セクションも参照）の最新の値を表す場合、対応するフローの最新の値に依存する計算を実行し、アップストリームフローのいずれかが値を放出するたび再計算する必要がある場合があります。
対応する演算子のファミリーは[combine]と呼ばれます。

例えば、前の例の数値は300ミリ秒ごとに更新され、文字列は400ミリ秒ごとに更新される場合、[zip]演算子を使用してそれらを圧縮しても、結果は400ミリ秒ごとに印刷されますが、同じ結果が生成されます。

> この例では、[onEach]中間演算子を使用して各要素を遅延させ、サンプルフローを放出するコードをより宣言的かつ短くします。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
    val startTime = System.currentTimeMillis() // remember the start time
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string with "zip"
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-21.kt)から完全なコードを取得できます。

<!--- TEST ARBITRARY_TIME
1 -> one at 437 ms from start
2 -> two at 837 ms from start
3 -> three at 1243 ms from start
-->

一方、こちらは[zip]の代わりに[combine]演算子を使用する場合です。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
    val startTime = System.currentTimeMillis() // remember the start time
    nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-22.kt)から完全なコードを取得できます。

`nums` フローまたは `strs` フローからの各放出のたびにラインが印刷される、まったく異なる出力が得られます。

```text
1 -> one at 452 ms from start
2 -> one at 651 ms from start
2 -> two at 854 ms from start
3 -> two at 952 ms from start
3 -> three at 1256 ms from start
```

<!--- TEST ARBITRARY_TIME -->

### フローを平坦化する

フローは非同期に受信した値のシーケンスを表すため、各値が別の値のシーケンスの要求をトリガーする状況に陥るのは非常に簡単です。
例えば、500ミリ秒離れた2つの文字列のフローを返す次の関数を使用できます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}
```

</div>

<!--- CLEAR -->

ここで、3つの整数のフローがあり、次のようにそれぞれに対して `requestFlow`を呼び出すとき、

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
(1..3).asFlow().map { requestFlow(it) }
```

</div>

<!--- CLEAR -->

すると、フローのフロー（ `Flow<Flow<String>>` ）になります。これは、さらなる処理のために単一のフローに _フラット化_ する必要があります。
コレクションとシーケンスには、このための[flatten][Sequence.flatten]および[flatMap][Sequence.flatMap]演算子があります。ただし、フローの非同期性により、フラット化のさまざまな _モード_ が必要になるため、フローにはフラット化演算子のファミリーがあります。

#### flatMapConcat

連結モードは、[flatMapConcat]および[flattenConcat]演算子によって実装されます。それらは、対応するシーケンス演算子の最も直接的な類似物です。
次の例に示すように、内部フローが完了するのを待ってから、次のフローの収集を開始します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun main() = runBlocking<Unit> {
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapConcat { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-23.kt)から完全なコードを取得できます。

[flatMapConcat]のシーケンシャルな性質は、出力に明確に見られます。

```text
1: First at 121 ms from start
1: Second at 622 ms from start
2: First at 727 ms from start
2: Second at 1227 ms from start
3: First at 1328 ms from start
3: Second at 1829 ms from start
```

<!--- TEST ARBITRARY_TIME -->

#### flatMapMerge

別のフラット化モードは、すべての着信フローを並行して収集し、それらの値を単一のフローにマージして、値ができるだけ早く放出されるようにします。
[flatMapMerge]および[flattenMerge]演算子によって実装されます。
両方とも、同時に収集される並行フローの数を制限するオプションの `concurrency` パラメーターを受け入れます（デフォルトでは[DEFAULT_CONCURRENCY]と同じです）。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun main() = runBlocking<Unit> {
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapMerge { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-24.kt)から完全なコードを取得できます。

[flatMapMerge]の並行性は明らかです。

```text
1: First at 136 ms from start
2: First at 231 ms from start
3: First at 333 ms from start
1: Second at 639 ms from start
2: Second at 732 ms from start
3: Second at 833 ms from start
```

<!--- TEST ARBITRARY_TIME -->

> [flatMapMerge]はそのコードブロック（この例では `{ requestFlow(it) }` ）を順次呼び出しますが、結果のフローを並行して収集するため、最初にシーケンシャルな `map { requestFlow(it) }` を実行し、その結果に対して[flattenMerge]を呼び出すのと同じです。

#### flatMapLatest

[「最新の値を処理する」](#最新の値を処理する)セクションで示した[collectLatest]演算子と同様に、対応する「Latest」フラット化モードがあり、新しいフローが発行されるとすぐに前のフローの収集がキャンセルされます。
これは[flatMapLatest]演算子によって実装されます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun main() = runBlocking<Unit> {
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapLatest { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-25.kt)から完全なコードを取得できます。

この例の出力は、[flatMapLatest]がどのように動作するかを示す良い例です。

```text
1: First at 142 ms from start
2: First at 322 ms from start
3: First at 425 ms from start
3: Second at 931 ms from start
```

<!--- TEST ARBITRARY_TIME -->

> [flatMapLatest]は、新しい値のブロック（この例では `{ requestFlow(it) }` ）内のすべてのコードをキャンセルすることに注意してください。
この例では違いはありません。 `requestFlow` への呼び出し自体は高速であり、中断されず、キャンセルできないためです。
ただし、そこで `delay` のようなサスペンド関数を使用する場合にあらわになります。

### フロー例外

エミッターまたは演算子内のコードが例外をスローすると、フロー収集は例外で完了します。
これらの例外を処理する方法はいくつかあります。

#### コレクターのtryとcatch

コレクターはKotlinの[`try/catch`][exceptions]ブロックを使用して例外を処理できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value ->
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-26.kt)から完全なコードを取得できます。

このコードは、[collect]終端演算子で例外を正常にキャッチし、見ての通りそれ以降の値は放出されません。

```text
Emitting 1
1
Emitting 2
2
Caught java.lang.IllegalStateException: Collected 2
```

<!--- TEST -->

#### すべてがキャッチされる

前の例は、実際にはエミッターまたは中間/終端演算子で発生する例外をキャッチします。
例えば、放出された値が文字列に[mapped][map]されるようにコードを変更しても、対応するコードは例外を生成します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-27.kt)から完全なコードを取得できます。

それでもこの例外はキャッチされ、収集は停止します。

```text
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
```

<!--- TEST -->

### 例外の透過性

しかし、エミッターのコードはどのように例外処理動作をカプセル化できますか？

フローは _例外に対して透過的_ でなければならず、 `try/catch` ブロック内からの `flow { ... }` ビルダーの[emit][FlowCollector.emit]値に対して例外透過性違反です。
これにより、前の例のように例外をスローするコレクターは常に `try/catch` を使用して例外をキャッチできることが保証されます。

エミッターは、この例外の透過性を保持し、例外処理のカプセル化を許可する[catch]演算子を使用できます。
`catch` 演算子の本体は例外を分析し、どの例外がキャッチされたかに応じてさまざまな方法で例外に対応できます。

* 例外は、 `throw` を使用して再スローできます。
* 例外は、[catch]の本体から[emit][FlowCollector.emit]を使用して値の放出に変換できます。
* 例外は、無視、記録、または他のコードによって処理できます。

例えば、例外をキャッチしてテキストを放出してみましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simple(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }
        "string $value"
    }

fun main() = runBlocking<Unit> {
//sampleStart
    simple()
        .catch { e -> emit("Caught $e") } // emit on exception
        .collect { value -> println(value) }
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-28.kt)から完全なコードを取得できます。

コードの周りに `try/catch` がなくなっても、例の出力は同じです。

<!--- TEST
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
-->

#### 透過的なキャッチ

[catch]中間演算子は、例外の透過性を尊重して上流の例外のみをキャッチします（これは、 `catch` より下ではなく、それより上のすべての演算子からの例外です）。
`collect { ... }`（ `catch` の下に配置）のブロックが例外をスローした場合、それはエスケープします。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-29.kt)から完全なコードを取得できます。

`catch` 演算子があるにもかかわらず、"Caught ..." メッセージはプリントされません。

```text
Emitting 1
1
Emitting 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
	at ...
```

<!--- TEST EXCEPTION -->

#### 宣言的にキャッチする

[collect]演算子の本体を[onEach]に移動し、それを `catch` 演算子の前に置くことで、[catch]演算子の宣言的な性質とすべての例外を処理したいという要求を組み合わせることができます。
このフローの収集は、パラメーターなしで `collect()` を呼び出すことでトリガーする必要があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    simple()
        .onEach { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
        .catch { e -> println("Caught $e") }
        .collect()
//sampleEnd
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-30.kt)から完全なコードを取得できます。

これで、「Caught ...」メッセージが出力され、 `try/catch` ブロックを明示的に使用せずにすべての例外をキャッチできることがわかります。

```text
Emitting 1
1
Emitting 2
Caught java.lang.IllegalStateException: Collected 2
```

<!--- TEST EXCEPTION -->

### フローの完了

フロー収集が完了したときに（通常または例外的に）、アクションを実行する必要がある場合があります。
既にお気づきかもしれませんが、これは命令型または宣言型の2つの方法で実行できます。

#### 命令的な最終ブロック

`try`/`catch` に加えて、コレクターは `finally` ブロックを使用して `collect` の完了時にアクションを実行することもできます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-31.kt)から完全なコードを取得できます。

このコードは、 `simple()` フローによって生成された3つの数値に続いて文字列 "Done" を出力します。

```text
1
2
3
Done
```

<!--- TEST  -->

#### 宣言的な取り扱い

宣言的アプローチの場合、フローには[onCompletion]中間演算子があり、フローが完全に収集されたときに呼び出されます。

前の例は、[onCompletion]演算子を使用して書き換えることができ、同じ出力を生成します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
//sampleStart
    simple()
        .onCompletion { println("Done") }
        .collect { value -> println(value) }
//sampleEnd
}
```
</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-32.kt)から完全なコードを取得できます。

<!--- TEST
1
2
3
Done
-->

[onCompletion]の主な利点は、フローの収集が正常に完了したか例外的に完了したかを判断するために使用できるラムダのnull許容の `Throwable` パラメーターです。
次の例では、 `simple()` フローは数値の1を出力した後に例外をスローします。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}
//sampleEnd
```
</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-33.kt)から完全なコードを取得できます。

予想通り、次のようにプリントされます。

```text
1
Flow completed exceptionally
Caught exception
```

<!--- TEST -->

[onCompletion]演算子は、[catch]とは異なり、例外を処理しません。上記のサンプルコードからわかるように、例外は引き続き下流に流れます。
これは、さらに `onCompletion` 演算子に配信され、 `catch` 演算子で処理できます。

#### 正常終了

[catch]演算子とのもう1つの違いは、[onCompletion]はすべての例外を確認し、アップストリームフローが正常に完了した場合にのみ（キャンセルや失敗なしで）`null` 例外を受け取ることです。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-34.kt)から完全なコードを取得できます。

完了の原因がnullではないことがわかります。これはダウンストリーム例外のためにフローが中止されたためです。

```text
1
Flow completed with java.lang.IllegalStateException: Collected 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

<!--- TEST EXCEPTION -->

### 命令型と宣言型

これで、フローを収集し、その完了と例外を命令的および宣言的に処理する方法がわかりました。
ここでの自然な疑問は、どのアプローチが好ましいのか、そしてその理由は何でしょうか？
ライブラリとして、特定のアプローチを推奨しておらず、両方のオプションが有効であり、独自の設定とコードスタイルに従って選択する必要があると考えています。

### フローの起動

フローを使用して、あるソースからの非同期イベントを表すのは簡単です。
この場合、着信イベントへの反応でコードの一部を登録し、さらなる作業を継続する `addEventListener` 関数の類似物が必要です。 [onEach]演算子はこの役割を果たすことができます。
ただし、 `onEach` は中間演算子です。
また、フローを収集する終端演算子も必要です。
そうでなければ、 `onEach` を呼び出しても効果はありません。

`onEach` の後に[collect]終端演算子を使用すると、その後のコードはフローが収集されるまで待機します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .collect() // <--- Collecting the flow waits
    println("Done")
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-35.kt)から完全なコードを取得できます。

ご覧のようにプリントされます。

```text
Event: 1
Event: 2
Event: 3
Done
```

<!--- TEST -->

ここで[launchIn]終端演算子が役に立ちます。 `collect` を `launchIn` に置き換えることで、フローの収集を別のコルーチンで起動できるため、次のコードの実行がすぐに続行されます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

//sampleStart
fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- Launching the flow in a separate coroutine
    println("Done")
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-36.kt)から完全なコードを取得できます。

次のようにプリントされます。

```text
Done
Event: 1
Event: 2
Event: 3
```

<!--- TEST -->

`launchIn` の必須パラメーターは、フローを収集するコルーチンが起動される[CoroutineScope]を指定する必要があります。
上記の例では、このスコープは[runBlocking]コルーチンビルダーからのものであるため、フローの実行中、この[runBlocking]スコープは子コルーチンの完了を待機し、メイン関数がこの例をリターンしたり終了したりしないようにします。

実際のアプリケーションでは、スコープは有効期間が制限されたエンティティから取得されます。
このエンティティのライフタイムが終了するとすぐに、対応するスコープがキャンセルされ、対応するフローの収集がキャンセルされます。
このように、 `onEach { ... }.launchIn(scope)` のペアは `addEventListener` のように機能します。
ただし、キャンセルと構造化並行性がこの目的に役立つため、対応する `removeEventListener` 関数は必要ありません。

[launchIn]が返す[Job]は、スコープ全体をキャンセルまたは[Job.join]するのではなく、このJobに対応するフロー収集コルーチンだけを[cancel][Job.cancel]するために使用できることに注意してください。

### フローキャンセルのチェック

便宜上、[flow][_flow]ビルダーは、発行された値ごとにキャンセルの追加の[ensureActive]チェックを実行します。
これは、 `flow { ... }` からの放出のビジーループがキャンセル可能であることを意味します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..5) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-37.kt)から完全なコードを取得できます。

4を放出しようとすると、3までの数字と[CancellationException]しか得られません。

```text
Emitting 1
1
Emitting 2
2
Emitting 3
3
Emitting 4
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@6d7b4f4c
```

<!--- TEST EXCEPTION -->

ただし、他のほとんどのフロー演算子は、パフォーマンス上の理由から、独自に追加のキャンセルチェックを実行しません。
たとえば、[IntRange.asFlow]拡張関数を使用して同じビジーループを記述し、どこも中断しない場合、キャンセルのチェックはありません。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun main() = runBlocking<Unit> {
    (1..5).asFlow().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-38.kt)から完全なコードを取得できます。

1から5までのすべての番号が収集され、キャンセルは `runBlocking` からリターンする前にのみ検出されます。

```text
1
2
3
4
5
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@3327bd23
```

<!--- TEST EXCEPTION -->

#### 忙しいフローをキャンセル可能にする

コルーチンでビジーループがある場合は、キャンセルを明示的に確認する必要があります。
`.onEach { currentCoroutineContext().ensureActive() }` を追加できますが、それを行うためにすぐに使用できる[cancellable]演算子が用意されています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun main() = runBlocking<Unit> {
    (1..5).asFlow().cancellable().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
//sampleEnd
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-39.kt)から完全なコードを取得できます。

`cancellable` 演算子を使用すると、1から3までの数字のみが収集されます。

```text
1
2
3
Exception in thread "main" kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#1":BlockingCoroutine{Cancelled}@5ec0a365
```

<!--- TEST EXCEPTION -->

### フローとリアクティブストリーム

[リアクティブストリーム](https://www.reactive-streams.org/)またはRxJavaやReactorプロジェクトなどのリアクティブフレームワークに精通している人にとって、フローの設計は非常に馴染みがあるように見えるかもしれません。

実際、その設計はReactiveStreamsとそのさまざまな実装に触発されました。
ただし、Flowの主な目標は、可能な限りシンプルな設計にし、Kotlinとサスペンションに対応し、構造化された並行性を尊重することです。
この目標を達成することは、リアクティブの先駆者と彼らの多大な努力なしには不可能です。
ストーリー全体は、[Reactive Streams and Kotlin Flows](https://medium.com/@elizarov/reactive-streams-and-kotlin-flows-bfd12772cda4)の記事で読むことができます。

概念的には異なりますが、フローはリアクティブストリームで*あり*、リアクティブの（仕様およびTCK準拠）Publisherに変換したり、その逆を行うことができます。
このようなコンバーターは、すぐに使用できる `kotlinx.coroutines` によって提供され、対応するリアクティブモジュール（ReactiveStreamsの場合は `kotlinx-coroutines-reactive` 、Reactorプロジェクトの場合は `kotlinx-coroutines-reactor` 、RxJava2/RxJava3の場合は `kotlinx-coroutines-rx2` / ` kotlinx-coroutines-rx3` ）にあります。
統合モジュールには、 `Flow` との間の変換、Reactorの `Context` との統合、さまざまなリアクティブエンティティを操作するためのサスペンションに適した方法が含まれます。

<!-- stdlib references -->

[collections]: https://kotlinlang.org/docs/reference/collections-overview.html
[List]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-list/index.html
[forEach]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/for-each.html
[Sequence]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/index.html
[Sequence.zip]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/zip.html
[Sequence.flatten]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flatten.html
[Sequence.flatMap]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flat-map.html
[exceptions]: https://kotlinlang.org/docs/reference/exceptions.html

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[Dispatchers.Main]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[ensureActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/ensure-active.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
<!--- INDEX kotlinx.coroutines.flow -->
[Flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html
[_flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html
[FlowCollector.emit]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html
[collect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html
[flowOf]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-of.html
[map]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html
[filter]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html
[transform]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/transform.html
[take]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/take.html
[toList]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html
[toSet]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html
[first]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/first.html
[single]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html
[reduce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html
[fold]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/fold.html
[flowOn]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html
[buffer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html
[conflate]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html
[collectLatest]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html
[zip]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html
[combine]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html
[onEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html
[flatMapConcat]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-concat.html
[flattenConcat]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-concat.html
[flatMapMerge]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html
[flattenMerge]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-merge.html
[DEFAULT_CONCURRENCY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-d-e-f-a-u-l-t_-c-o-n-c-u-r-r-e-n-c-y.html
[flatMapLatest]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html
[catch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html
[onCompletion]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html
[launchIn]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/launch-in.html
[IntRange.asFlow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/kotlin.ranges.-int-range/as-flow.html
[cancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/cancellable.html
<!--- END -->
