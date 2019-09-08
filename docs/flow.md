<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*-##\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/FlowGuideTest.kt
// This file was automatically generated from flow.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class FlowGuideTest {
--> 

**目次**

<!--- TOC -->

* [非同期フロー](#非同期フロー)
  * [複数の値を表す](#複数の値を表す)
    * [シーケンス](#シーケンス)
    * [サスペンド関数](#サスペンド関数)
    * [フロー](#フロー)
  * [フローはコールド](#フローはコールド)
  * [フローのキャンセル](#フローのキャンセル)
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
    * [上流の例外のみ](#上流の例外のみ)
  * [命令型と宣言型](#命令型と宣言型)
  * [フローの起動](#フローの起動)

<!--- END_TOC -->

## 非同期フロー

サスペンド関数は非同期的に単一の値を返しますが、複数の非同期的に計算された値をどのように返すことができますか？ それがKotlin Flowsの目的です。

### 複数の値を表す

[collections]を使用して、Kotlinで複数の値を表すことができます。
例えば、3つの数字の[List]を返す関数 `foo()` があり、[forEach]を使用してすべてをプリントできます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun foo(): List<Int> = listOf(1, 2, 3)
 
fun main() {
    foo().forEach { value -> println(value) } 
}
```

</div>

> [ここ](../kotlinx-coroutines-core/jvm/test/guide/example-flow-01.kt)で完全なコードを取得できます。

このコードの出力は次のようになります。

```text
1
2
3
```

<!--- TEST -->

#### シーケンス

CPUを消費するブロッキングコードを使用して数値を計算する場合（各計算に100ミリ秒かかります）、[Sequence]を使用して数値を表すことができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun foo(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() {
    foo().forEach { value -> println(value) } 
}
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-02.kt).

このコードは同じ数字を出力しますが、それぞれを印刷する前に100ミリ秒待機します。

<!--- TEST 
1
2
3
-->

#### サスペンド関数

ただし、この計算はコードを実行しているメインスレッドをブロックします。
それらの値が非同期コードによって計算されるとき、関数 `foo` を `suspend` 修飾子でマークできるので、ブロックせずに作業を実行し、結果をリストとして返すことができます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*                 
                           
//sampleStart
suspend fun foo(): List<Int> {
    delay(1000) // pretend we are doing something asynchronous here
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    foo().forEach { value -> println(value) } 
}
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-03.kt).

このコードは、1秒待ってから数値を出力します。

<!--- TEST 
1
2
3
-->

#### フロー

`List<Int>` の結果タイプを使用すると、一度にすべての値のみを返すことができます。
非同期的に計算される値のストリームを表すために、同期的に計算される値の `Sequence<Int>` 型と同様に[`Flow<Int>`][Flow]型を使用できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart               
fun foo(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to see that the main thread is not blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    foo().collect { value -> println(value) } 
}
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-04.kt).

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

前の例の[Flow]のコードには以下の違いがあることに注意してください。

* [Flow]型のビルダー関数は[flow]です。
* `flow {...}` ビルダーブロック内のコードは中断できます。
* 関数 `foo()` は `suspend` 修飾子でマークされなくなりました。
* 値は、[emit][FlowCollector.emit]関数を使用してフローから_放出_されます。
* 値は[collect][collect]関数を使用してフローから_収集_されます。

> `foo` の `flow { ... }` 本文の [delay]を `Thread.sleep` に置き換えると、この場合メインスレッドがブロックされることがわかります。

### フローはコールド

フローは、シーケンスと同様に_コールド_ストリームです。[flow]ビルダー内のコードは、フローが収集されるまで実行されません。
これは、次の例で明らかになります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart      
fun foo(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling foo...")
    val flow = foo()
    println("Calling collect...")
    flow.collect { value -> println(value) } 
    println("Calling collect again...")
    flow.collect { value -> println(value) } 
}
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-05.kt).

次のようにプリントします。

```text
Calling foo...
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
 
これが `foo()` 関数（フローを返す）が `suspend` 修飾子でマークされていない主な理由です。
`foo()` はすぐに戻り、それ自体では何も待ちません。
フローは収集されるたびに開始されるため、 `collect` を再度呼び出すと、"Flow started" が再度プリントされることがわかります。

### フローのキャンセル

フローは、コルーチンの一般的な協調キャンセルに準拠しています。 ただし、フローインフラストラクチャでは、追加のキャンセルポイントは導入されません。
キャンセルに対して完全に透過的です。
通常、フロー収集はキャンセル可能なサスペンド関数（[delay]など）で中断された場合にキャンセルでき、それ以外の場合はキャンセルできません。

次の例は、[withTimeoutOrNull]ブロックが実行されているとき、タイムアウトでどのようにフローがキャンセルされ、コードの実行が停止するかを示しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart           
fun foo(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // Timeout after 250ms 
        foo().collect { value -> println(value) } 
    }
    println("Done")
}
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-06.kt).

`foo()` 関数のフローによって2つの数値のみが放出され、次の出力が生成されることに注目してください。

```text
Emitting 1
1
Emitting 2
2
Done
```

<!--- TEST -->

### フロービルダー

前の例の `flow { ... }` ビルダーは最も基本的なものです。
フローの便利な宣言のための他のビルダーがあります。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-07.kt).
 
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

例えば、リクエストの実行がサスペンド関数によって実装される長時間実行される操作であっても、[map]演算子を使用して到着するリクエストのフローを結果にマップできます。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-08.kt).

次の3行が生成され、各行が1秒後に表示されます。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-09.kt).

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-10.kt).

このコードの出力は、 `numbers()` 関数内の `flow { ... }` 本文の実行が2番目の数値を出力した後に停止したことを明確に示しています。

```text       
1
2
Finally in numbers
```

<!--- TEST -->

### 終端フロー演算子

フローのターミナルオペレーターは、フローの収集を開始する _サスペンド関数_ です。
[collect]演算子は最も基本的なものですが、利便性のために他の終端演算子もあります。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-11.kt).

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-12.kt).

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
例えば `foo` フローがある場合、その実装の詳細に関係なく、このコードの作成者が指定したコンテキストで次のコードが実行されます。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
withContext(context) {
    foo.collect { value ->
        println(value) // run in the specified context 
    }
}
``` 

</div>

<!--- CLEAR -->

フローのこのプロパティは、_コンテキストの維持_と呼ばれます。

そのため、デフォルトでは、 `flow { ... }` ビルダーのコードは、対応するフローのコレクターによって提供されるコンテキストで実行されます。
例えば、呼び出されるスレッドをプリントし、3つの数値を放出する `foo` の実装を考えます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
//sampleStart
fun foo(): Flow<Int> = flow {
    log("Started foo flow")
    for (i in 1..3) {
        emit(i)
    }
}  

fun main() = runBlocking<Unit> {
    foo().collect { value -> log("Collected $value") } 
}            
//sampleEnd
```                

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-13.kt).

このコードを実行すると、次のように生成されます。

```text  
[main @coroutine#1] Started foo flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

<!--- TEST FLEXIBLE_THREAD -->

`foo().collect` はメインスレッドから呼び出されるため、 `foo` のフローの本文もメインスレッドで呼び出されます。
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
fun foo(): Flow<Int> = flow {
    // WRONG way to change context for CPU-consuming code in flow builder
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // pretend we are computing it in CPU-consuming way
            emit(i) // emit next value
        }
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value -> println(value) } 
}            
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-14.kt).

このコードは次の例外を出します。

<!--- TEST EXCEPTION
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
		Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
		but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, DefaultDispatcher].
		Please refer to 'flow' documentation or use 'flowOn' instead
	at ...
-->
   
> この例外を示すために、この例では[kotlinx.coroutines.withContext][withContext]関数の完全修飾名を使用する必要があったことに注意してください。
`withContext` という短い名前は、この問題に遭遇するのを防ぐためにコンパイルエラーを生成する特別なスタブ関数に解決されるでしょう。

#### flowOnオペレーター
   
例外は、フロー放出のコンテキストを変更するために使用される[flowOn]関数を参照します。
フローのコンテキストを変更する正しい方法を以下の例に示します。この例では、対応するスレッドの名前もプリントして、すべての動作を示しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it in CPU-consuming way
        log("Emitting $i")
        emit(i) // emit next value
    }
}.flowOn(Dispatchers.Default) // RIGHT way to change context for CPU-consuming code in flow builder

fun main() = runBlocking<Unit> {
    foo().collect { value ->
        log("Collected $value") 
    } 
}            
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-15.kt).
  
バックグラウンドスレッドでの `flow { ... }` の動作に注意してください。コレクションはメインスレッドで行われます。
  
<!--- TEST FLEXIBLE_THREAD
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
-->

ここでのもう1つの所見は、[flowOn]演算子がフローのデフォルトのシーケンシャルな性質を変更したことです。
現在、収集は一方のコルーチン（"coroutine#1"）で発生し、放出はコルーチンの収集と並行して別のスレッドで実行されているもう一方のコルーチン（"coroutine#2"）で発生します。
[flowOn]演算子は、コンテキストで[CoroutineDispatcher]を変更する必要がある場合、アップストリームフローの別のコルーチンを作成します。

### バッファリング

異なるコルーチンでフローの異なる部分を実行すると、特に長時間実行される非同期操作が関係する場合に、フローの収集にかかる全体的な時間の観点から役立ちます。
例えば、 `foo()` フローによる放出が遅く、要素の生成に100ミリ秒かかり、また、コレクターも遅く、要素の処理に300ミリ秒かかる場合を考えます。 
このようなフローから3つの数字を収集するのにかかる時間を見てみましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        foo().collect { value -> 
            delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
    }   
    println("Collected in $time ms")
}
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-16.kt).

このようなものが生成され、コレクション全体で約1200ミリ秒（3つの数字それぞれで400ミリ秒）がかかります。

```text
1
2
3
Collected in 1220 ms
```

<!--- TEST ARBITRARY_TIME -->

シーケンシャルに実行するのとは反対に、フローで[buffer]演算子を使用してコードの収集と並行して `foo()` の放出コードを実行できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-17.kt).

処理パイプラインを効果的に作成し、最初の数値を100ミリ秒待つだけで、各数値の処理に300ミリ秒しかかからないため、同じ数値がより速く生成されます。 この方法では、実行に約1000ミリ秒かかります。

```text
1
2
3
Collected in 1071 ms
```                    

<!--- TEST ARBITRARY_TIME -->

> [flowOn]演算子は[CoroutineDispatcher]を変更する必要がある場合、同じバッファリングメカニズムを使用しますが、ここでは実行コンテキストを変更せずに明示的にバッファリングを要求します。

#### 合成

フローが一部の操作または操作ステータスの更新の部分的な結果を表す場合、各値を処理する必要はなく、最新の値のみを処理する必要があります。
この場合、[conflate]演算子を使用して、コレクターが処理するには遅すぎるときに中間値をスキップできます。
前の例に基づいて作成します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-18.kt).

最初の番号が処理されている間に、2番目と3番目の番号がすでに生成されているため、2番目の番号が合成され、最新の（3番目の）番号のみがコレクターに配信されました。

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
前の例を[conflate]から[collectLatest]に変更しましょう。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-19.kt).
 
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

複数のフローを構成するには、いくつかの方法があります。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-20.kt).

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

>この例では[onEach]中間演算子を使用して各要素を遅延させ、サンプルフローを生成するコードをより宣言的で短くします。
 
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-21.kt).

<!--- TEST ARBITRARY_TIME
1 -> one at 437 ms from start
2 -> two at 837 ms from start
3 -> three at 1243 ms from start
-->

一方、こちらは[zip]の代わりに[combine]演算子を使用しています。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms          
    val startTime = currentTimeMillis() // remember the start time 
    nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-22.kt).

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

すると、フローのフロー（ `Flow<Flow<String>>` ）になります。これは、さらなる処理のために単一のフローに_フラット化_する必要があります。
コレクションとシーケンスには、この目的のために[flatten][Sequence.flatten]および[flatMap][Sequence.flatMap]演算子があります。 ただし、フローの非同期的な性質により、さまざまなフラット化_モード_が必要になるため、フローにはフラット化演算子のファミリーがあります。

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
    val startTime = currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapConcat { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-23.kt).            

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
    val startTime = currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapMerge { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-24.kt).            

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

[最新の値を処理する](#最新の値を処理する)セクションで示された[collectLatest]演算子と同様の方法で、新しいフローが放出されるとすぐに前のフローの収集がキャンセルされる "Latest" フラット化モードがあります。
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
    val startTime = currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapLatest { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-25.kt).            

この例の出力は、[flatMapLatest]の動作のしかたを示しています。

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

エミッターまたはいずれかの演算子内のコードが例外をスローすると、フロー収集は例外で完了します。
これらの例外を処理する方法はいくつかあります。

#### コレクターのtryとcatch

コレクターはKotlinの[`try/catch`][exceptions]ブロックを使用して例外を処理できます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value ->         
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-26.kt).

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
fun foo(): Flow<String> = 
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
        foo().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}            
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-27.kt).

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

フローは_例外に対して透過的_でなければならず、 `try/catch` ブロック内からの `flow { ... }` ビルダーの[emit][FlowCollector.emit]値に対して例外透過性違反です。
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

fun foo(): Flow<String> = 
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
    foo()
        .catch { e -> emit("Caught $e") } // emit on exception
        .collect { value -> println(value) }
//sampleEnd
}            
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-28.kt). 
 
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
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}            
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-29.kt). 
 
`catch` 演算子があるにもかかわらず、"Caught ..." メッセージはプリントされません。

<!--- TEST EXCEPTION  
Emitting 1
1
Emitting 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
	at ...
-->

#### 宣言的にキャッチする

[catch]演算子の宣言的な性質と、[collect]演算子の本体を[onEach]に移動し `catch` 演算子の前に置くことですべての例外を処理したいという要求とを組み合わせることができます。
このフローの収集は、パラメーターなしで `collect()` を呼び出すことでトリガーする必要があります。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    foo()
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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-30.kt). 
 
これで、"Caught ..." メッセージが出力されることがわかります。したがって、 `try/catch` ブロックを明示的に使用せずにすべての例外をキャッチできます。

<!--- TEST EXCEPTION  
Emitting 1
1
Emitting 2
Caught java.lang.IllegalStateException: Collected 2
-->

### フローの完了

フローの収集が完了したときに（通常または例外的に）、何らかのアクションを実行する必要がある場合があります。
既にお気づきかもしれませんが、これは命令型と宣言型の2つの方法でも実行できます。

#### 命令的な最終ブロック

`try`/`catch` に加えて、コレクターは `finally` ブロックを使用して `collect` の完了時にアクションを実行することもできます。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}            
//sampleEnd
```  

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-31.kt). 

このコードは、 `foo()` フローによって生成された3つの数値に続いて文字列 "Done" を出力します。

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

fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
//sampleStart
    foo()
        .onCompletion { println("Done") }
        .collect { value -> println(value) }
//sampleEnd
}            
```
</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-32.kt). 

<!--- TEST 
1
2
3
Done
-->

[onCompletion]の主な利点は、フローの収集が正常に完了したか例外的に完了したかを判断するために使用できるラムダのnull許容の `Throwable` パラメーターです。
次の例では、 `foo()` フローは数値の1を出力した後に例外をスローします。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}            
//sampleEnd
```
</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-33.kt). 

As you may expect, it prints:

```text
1
Flow completed exceptionally
Caught exception
```

<!--- TEST -->

[onCompletion]演算子は、[catch]とは異なり、例外を処理しません。 上記のサンプルコードからわかるように、例外は引き続き下流に流れます。
これは、さらに `onCompletion` 演算子に配信され、 `catch` 演算子で処理できます。

#### 上流の例外のみ

[catch]演算子と同様に、[onCompletion]はアップストリームからの例外のみを認識し、ダウンストリームの例外は認識しません。
例えば、次のコードを実行します。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
//sampleEnd
```

</div>

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-34.kt). 

そして、完了原因がnullであることがわかりますが、収集は例外で失敗しました。

```text 
1
Flow completed with null
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

<!--- TEST EXCEPTION -->

### 命令型と宣言型

これで、フローを収集し、その完了と例外を命令的および宣言的な方法で処理する方法がわかりました。
ここでの自然な疑問は、どのアプローチが優先されるべきか、そしてその理由です。
ライブラリとして、特定のアプローチを推奨しておらず、両方のオプションが有効であり、独自の設定とコードスタイルに従って選択する必要があると考えています。

### フローの起動

何らかのソースからの非同期イベントを表すためにフローを使用すると便利です。
この場合、着信イベントへの反応を伴うコードの一部を登録し、さらなる作業を継続する `addEventListener` 関数の類似物が必要です。 [onEach]オペレーターはこの役割を果たします。
ただし、 `onEach` は中間演算子です。
また、フローを収集する終端演算子も必要です。
そうでなければ、 `onEach` を呼び出しても効果はありません。
 
`onEach` の後に[collect]終端演算子を使用する場合、後のコードはフローが収集されるまで待機します。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-35.kt). 
  
ご覧のようにプリントされます。

```text 
Event: 1
Event: 2
Event: 3
Done
```    

<!--- TEST -->
 
ここで、[launchIn]終端オペレーターが便利です。
`collect` を `launchIn` に置き換えると、別のコルーチンでフローの収集を起動できるため、さらなるコードの実行がすぐに継続します。

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

> You can get full code [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-36.kt). 
  
次のようにプリントされます。

```text          
Done
Event: 1
Event: 2
Event: 3
```    

<!--- TEST -->

`launchIn` の必須パラメーターは、フローを収集するコルーチンが起動される[CoroutineScope]を指定する必要があります。
上記の例では、このスコープは[runBlocking]コルーチンビルダーから取得されるため、フローの実行中にこの[runBlocking]スコープは子コルーチンの完了を待機し、メイン関数がこの例をリターンして終了しないようにします。

実際のアプリケーションでは、スコープは、存続期間が制限されているエンティティから取得されます。
このエンティティのライフタイムが終了するとすぐに、対応するスコープがキャンセルされ、対応するフローの収集がキャンセルされます。
このように、 `onEach { ... }.launchIn(scope)` のペアは `addEventListener` のように機能します。
ただし、キャンセルと構造化並行性がこの目的に役立つため、対応する `removeEventListener` 関数は必要ありません。

[launchIn]は[Job]を返すことに注意してください。[Job]は、スコープ全体をキャンセルせずに対応するフロー収集コルーチンのみを[cancel][Job.cancel]するか、[join][Job.join]のために使用できます。
 
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
<!--- INDEX kotlinx.coroutines.flow -->
[Flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html
[flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html
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
<!--- END -->
