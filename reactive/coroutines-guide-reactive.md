<!--- INCLUDE .*/example-reactive-([a-z]+)-([0-9]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide-reactive.md by Knit tool. Do not edit.
package kotlinx.coroutines.rx2.guide.$$1$$2

-->
<!--- KNIT     kotlinx-coroutines-rx2/test/guide/.*\.kt -->
<!--- TEST_OUT kotlinx-coroutines-rx2/test/guide/test/GuideReactiveTest.kt
// This file was automatically generated from coroutines-guide-reactive.md by Knit tool. Do not edit.
package kotlinx.coroutines.rx2.guide.test

import kotlinx.coroutines.guide.test.*
import org.junit.Test

class GuideReactiveTest : ReactiveTestBase() {
-->

# コルーチンによるリアクティブストリームのガイド

このガイドでは、Kotlinコルーチンとリアクティブストリームの重要な違いについて説明し、これらを一緒に使用してより良い結果を得る方法を示します。
[kotlinx.coroutinesのガイド](../docs/coroutines-guide.md)でカバーされている基本的なコルーチンの概念に慣れていなくても大丈夫です。 リアクティブストリームに精通している場合は、このガイドがコルーチンの世界へのより良い導入になるかもしれません。

`kotlinx.coroutines` プロジェクトには、リアクティブストリームに関連するいくつかのモジュールがあります。

* [kotlinx-coroutines-reactive](kotlinx-coroutines-reactive) -- [リアクティブストリーム](https://www.reactive-streams.org)のユーティリティ
* [kotlinx-coroutines-reactor](kotlinx-coroutines-reactor) -- [Reactor](https://projectreactor.io)のユーティリティ
* [kotlinx-coroutines-rx2](kotlinx-coroutines-rx2) -- [RxJava 2.x](https://github.com/ReactiveX/RxJava)のユーティリティ

このガイドは主に[リアクティブストリーム](https://www.reactive-streams.org)仕様に基づいており、[RxJava 2.x](https://github.com/ReactiveX/RxJava) に基づくいくつかの例とともに `Publisher` インターフェイスを使用しています。これはリアクティブストリーム仕様を実装しています。

提示されたすべての例を実行するために[`kotlinx.coroutines` プロジェクト](https://github.com/Kotlin/kotlinx.coroutines)をGitHubからあなたのワークステーションにクローンすることを歓迎します。
これらはプロジェクトの[reactive/kotlinx-coroutines-rx2/test/guide](kotlinx-coroutines-rx2/test/guide)ディレクトリに含まれています。
 
## 目次

<!--- TOC -->

* [リアクティブストリームとチャネルの違い](#リアクティブストリームとチャネルの違い)
  * [反復の基礎](#反復の基礎)
  * [サブスクリプションとキャンセル](#サブスクリプションとキャンセル)
  * [バックプレッシャー](#バックプレッシャー)
  * [Rx Subject と BroadcastChannel](#rx-subject-と-broadcastchannel)
* [演算子](#演算子)
  * [Range](#range)
  * [filterとmapの融合](#filterとmapの融合)
  * [Take until](#take-until)
  * [Merge](#merge)
* [コルーチンコンテキスト](#コルーチンコンテキスト)
  * [Rxのスレッド](#rxのスレッド)
  * [コルーチンのスレッド](#コルーチンのスレッド)
  * [Rx observeOn](#rx-observeon)
  * [すべてを制御するコルーチンコンテキスト](#すべてを制御するコルーチンコンテキスト)
  * [Unconfinedコンテキスト](#unconfinedコンテキスト)

<!--- END_TOC -->

## リアクティブストリームとチャネルの違い

このセクションでは、リアクティブストリームとコルーチンを基礎とするチャネルの主な違いの概要を説明します。

### 反復の基礎

[チャネル][Channel]は、以下のリアクティブストリームクラスと幾分似ています。

* リアクティブストリームの[Publisher](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/api/src/main/java/org/reactivestreams/Publisher.java)、
* Rx Java 1.x の[Observable](https://reactivex.io/RxJava/javadoc/rx/Observable.html)、
* `Publisher` を実装した Rx Java 2.x の[Flowable](https://reactivex.io/RxJava/2.x/javadoc/)。

これらはすべて無限または有限の要素（Rxで言うアイテム）の非同期ストリームを記述し、それらのすべてがバックプレッシャーをサポートします。

ただし、 `Channel` は常にRx用語で言うところのアイテムの _ホット_ ストリームを表します。
プロデューサーコルーチンによって要素がチャネルに送信され、コンシューマーコルーチンによって受信されます。
すべての[receive][ReceiveChannel.receive]の呼び出しは、チャネルから要素を消費します。
次の例でそれを説明しましょう。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    // 200ミリ秒間隔で遅延する1～3の数値を生成するチャンネルを作成する
    val source = produce<Int> {
        println("Begin") // このコルーチンの開始を出力する
        for (x in 1..3) {
            delay(200) // 200ミリ秒待つ
            send(x) // チャネルに数値 x を送る
        }
    }
    // ソースからの要素をプリントする
    println("Elements:")
    source.consumeEach { // 要素を消費する
        println(it)
    }
    // ソースから再び要素をプリントする
    println("Again:")
    source.consumeEach { // 要素を消費する
        println(it)
    }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-01.kt)で完全なコードを取得できます。

このコードは、次の出力を生成します。

```text
Elements:
Begin
1
2
3
Again:
```

<!--- TEST -->

[produce] _コルーチンビルダー_ が実行されると、コルーチンを1つ起動して要素のストリームを生成するため、"Begin" 行が1回だけプリントされたことに注目してください。
生成された要素はすべて[ReceiveChannel.consumeEach][consumeEach]拡張関数で消費されます。 このチャンネルから要素をもう一度受け取る方法はありません。
チャネルはプロデューサーコルーチンが終了したときに閉じられ、もう一度受信しようとして何も受信できません。

`kotlinx-coroutines-core` モジュールの[produce]の代わりに `kotlinx-coroutines-reactive` モジュールの[publish]コルーチンビルダーを使ってこのコードを書き直しましょう。

コードは同じままですが、以前は `source` が[ReceiveChannel]型だったところが、リアクティブストリームの[Publisher](https://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html)型になり、チャンネルの要素を _消費_ するために[consumeEach]が使用されていたところは、パブリッシャーから要素を _収集_ するために[collect][org.reactivestreams.Publisher.collect]が使用されています。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    // 200ミリ秒間隔で遅延する1～3の数値を生成するパブリッシャーを作成する
    val source = publish<Int> {
    //           ^^^^^^^  <---  以前の例との違いはここ
        println("Begin") // このコルーチンの開始を出力する
        for (x in 1..3) {
            delay(200) // 200ミリ秒待つ
            send(x) // チャネルに数値 x を送る
        }
    }
    // ソースからの要素をプリントする
    println("Elements:")
    source.collect { // 要素を収集する
        println(it)
    }
    // ソースから再び要素をプリントする
    println("Again:")
    source.collect { // 要素を収集する
        println(it)
    }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-02.kt)で完全なコードを取得できます。

今度はこのコードの出力が次のように変わります。

```text
Elements:
Begin
1
2
3
Again:
Begin
1
2
3
```

<!--- TEST -->

この例では、リアクティブストリームとチャネルの主な違いを強調しています。リアクティブストリームは高次の機能の概念です。
チャネルは要素のストリームですが、リアクティブストリームは要素のストリームの生成方法に関するレシピを定義します。
_収集_ されると、要素の実際のストリームになります。
各コレクターは、対応する `Publisher` の実装の動作に応じて、同じまたは異なる要素のストリームを受け取る場合があります。

上記の例で使用されている[publish]コルーチンビルダーはコルーチンを起動しませんが、すべての[collect] [org.reactivestreams.Publisher.collect]呼び出しでは起動します。
ここには2つあるので"Begin"が2回表示されます。

Rxの専門用語では、この種のパブリッシャーは_コールド_と呼ばれます。
多くの標準Rxオペレーターもコールドストリームを生成します。
コルーチンからそれらを収集することができ、すべてのコレクターは同じ要素のストリームを取得します。

> Rx [publish](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#publish())演算子と[connect](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/flowables/ConnectableFlowable.html#connect())メソッドを使って、チャンネルで見たのと同じ振る舞いを再現できます。

### サブスクリプションとキャンセル

前のセクションの2番目の例では、すべての要素を収集するために `source.collect { ... }` が使用されました。
代わりに、[openSubscription][org.reactivestreams.Publisher.openSubscription]を使用してチャネルを開き、それを反復処理できます。
このようにして、以下に示すように反復をより細かく制御できます（たとえば、 `break` を使用）。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.reactive.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // 5つの数値のレンジ
        .doOnSubscribe { println("OnSubscribe") } // 洞察を提供する
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... 何が起きているか
    var cnt = 0 
    source.openSubscription().consume { // ソースのチャネルを開く
        for (x in this) { // 反復してチャネルから要素を受け取る
            println(x)
            if (++cnt >= 3) break // 3つの要素をプリントしたら中止する
        }
        // このコードブロックが完了すると、 `consume`がチャンネルをキャンセルする
    }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-03.kt)で完全なコードを取得できます。

次の出力が生成されます。

```text
OnSubscribe
1
2
3
Finally
```

<!--- TEST -->
 
明示的に `openSubscription` を使用すると、ソースから登録解除するために対応するサブスクリプションを[キャンセル] [ReceiveChannel.cancel]する必要がありますが、[consume]が内部でそれを行うため明示的に `cancel` を呼び出す必要はありません。

インストールされた[doFinally](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#doFinally(io.reactivex.functions.Action))リスナーは、サブスクリプションが実際に閉じられていることを確認するために "Finally" をプリントします。
"OnComplete" は、すべての要素を消費しなかったので決してプリントされません。

すべての要素を `collect` する場合も、明示的な `cancel` を使用する必要はありません。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // 5つの数値のレンジ
        .doOnSubscribe { println("OnSubscribe") } // 洞察を提供する
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... 何が起きているか
    // ソースを完全に収集する
    source.collect { println(it) }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-04.kt)で完全なコードを取得できます。

次の出力が得られます。

```text
OnSubscribe
1
2
3
OnComplete
Finally
4
5
```

<!--- TEST -->

最後の要素 "4" と "5" の前に "OnComplete" と "Finally" がどのように出力されるかに注意してください。
これは、この例の `main` 関数が[runBlocking]コルーチンビルダーで始まるコルーチンであるために起こります。
メインコルーチンは、 `source.collect { ... }` 式を使用してflowableで受け取ります。
メインコルーチンは、ソースがアイテムを出力するのを待つ間 _中断_ されます。
最後のアイテムが `Flowable.range(1、5)` によって放出されると、メインコルーチンが_再開_され、メインスレッドにディスパッチされて後の時点でこの最後の要素がプリントされ、ソースが完了して "Finally" がプリントされます。

### バックプレッシャー

バックプレッシャーは、リアクティブストリームの中で最も興味深く複雑な特徴の1つです。
コルーチンは _中断_ することができ、バックプレッシャーを処理するための自然な答えを提供します。

Rx Java 2.xでは、バックプレッシャー対応クラスは[Flowable](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html)と呼ばれます。
次の例では、 `kotlinx-coroutines-rx2` モジュールの[rxFlowable]コルーチンビルダーを使用して、1から3までの3つの整数を送信するflowableを定義します。
[send][SendChannel.send]サスペンド関数を呼び出す前に出力にメッセージをプリントし、その操作方法を調べることができます。

整数はメインスレッドのコンテキストで生成されますが、サブスクリプションはサイズ1のバッファでRx [observeOn](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler,%20boolean,%20int))演算子を使用して別のスレッドに移されます。
サブスクライバーは遅いです。 `Thread.sleep` を使ってシミュレートされ各アイテムを処理するのに500ミリ秒かかります。

<!--- INCLUDE
import io.reactivex.schedulers.*
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> { 
    // コルーチン - メインスレッドコンテキストにおける要素の高速プロデューサー
    val source = rxFlowable {
        for (x in 1..3) {
            send(x) // これはサスペンド関数
            println("Sent $x") // アイテムが正常に送信された後にプリントする
        }
    }
    // Rxを使って別のスレッドの低速のサブスクライバーで購読する
    source
        .observeOn(Schedulers.io(), false, 1) // 1アイテム分のバッファーサイズを指定
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(500) // 各アイテムを処理するのに500ミリ秒
            println("Processed $x")
        }
    delay(2000) // 数秒間メインスレッドを中断
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-05.kt)で完全なコードを取得できます。

このコードの出力は、コルーチンでバックプレッシャーがどのように機能するかをうまく示しています。

```text
Sent 1
Processed 1
Sent 2
Processed 2
Sent 3
Processed 3
Complete
```

<!--- TEST -->

ここでは、プロデューサーのコルーチンが最初の要素をバッファに入れ、別の要素を送信しようとしている間中断されていることがわかります。
コンシューマーが最初のアイテムを処理した後でのみ、プロデューサーは2番目のアイテムを送信して再開します。

### Rx Subject と BroadcastChannel

RxJavaには、すべてのサブスクライバーに要素を効果的にブロードキャストするオブジェクトである[Subject](https://github.com/ReactiveX/RxJava/wiki/Subject)という概念があります。
コルーチンの世界で一致する概念は[BroadcastChannel]と呼ばれます。
Rxには、[BehaviorSubject](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/BehaviorSubject.html)が状態を管理するために使用されるものであるように、様々なサブジェクトがあります。

<!--- INCLUDE
import io.reactivex.subjects.BehaviorSubject
-->

```kotlin
fun main() {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two") // BehaviorSubjectの状態を更新し、"1"の値が失われる
    // ここでこのサブジェクトを購読してすべてをプリントする
    subject.subscribe(System.out::println)
    subject.onNext("three")
    subject.onNext("four")
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-06.kt)で完全なコードを取得できます。

このコードは、サブスクリプションとそのすべての更なるアップデートでサブジェクトの現在の状態をプリントします。

```text
two
three
four
```

<!--- TEST -->

他のリアクティブストリームと同様に、コルーチンからサブジェクトを購読することができます。
   
<!--- INCLUDE 
import io.reactivex.subjects.BehaviorSubject
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.collect
-->   
   
```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // すべてを印刷するコルーチンを起動する
    GlobalScope.launch(Dispatchers.Unconfined) { // 制約のないコンテキストでコルーチンを起動する
        subject.collect { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
}
```   

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-07.kt)で完全なコードを取得できます。

結果は同じです。

```text
two
three
four
```

<!--- TEST -->

ここでは、[Dispatchers.Unconfined]コルーチンコンテキストを使用して、Rxのサブスクリプションと同じ動作をする消費コルーチンを起動します。
基本的には、起動したコルーチンは要素が出力されるのと同じスレッドで直ちに実行されることを意味します。
コンテキストの詳細については、[別のセクション](#コルーチンコンテキスト)を参照してください。

コルーチンの利点は、シングルスレッドのUI更新の集約動作を簡単に得られることです。
典型的なUIアプリケーションは、すべての状態変更に反応する必要はありません。最新の状態のみが関係します。
UIスレッドが解放されるとすぐに、アプリケーションの状態に対する一連の連続した更新を1度だけUIに反映させる必要があります。
次の例では、メインスレッドのコンテキストで消費コルーチンを起動し、[yield]関数を使用して一連の更新の中断をシミュレートしメインスレッドを解放することで、これをシミュレートします。

<!--- INCLUDE
import io.reactivex.subjects.*
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // 最新の更新をプリントするコルーチンを起動
    launch { // コルーチンにメインスレッドのコンテキストを使用する
        subject.collect { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
    yield() // 起動したコルーチンへメインスレッドを譲る <--- ここ
    subject.onComplete() // サブジェクトのシーケンスを完了してコンシューマーもキャンセルする
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)で完全なコードを取得できます。

コルーチンは最新のアップデートのみを処理（プリント）します。

```text
four
```

<!--- TEST -->

純粋なコルーチンの世界でこれに相当する挙動は、[ConflatedBroadcastChannel]によって実装されています。これはブリッジを経由してリアクティブストリームに行くことなく、コルーチンチャネルの上に同じロジックを直接提供します。

<!--- INCLUDE
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val broadcast = ConflatedBroadcastChannel<String>()
    broadcast.offer("one")
    broadcast.offer("two")
    // 最新の更新をプリントするコルーチンを起動
    launch { // コルーチンのメインスレッドのコンテキストを使用する
        broadcast.consumeEach { println(it) }
    }
    broadcast.offer("three")
    broadcast.offer("four")
    yield() // 起動したコルーチンへメインスレッドを譲る
    broadcast.close() // ブロードキャストチャネルを閉じてコンシューマーもキャンセルする
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-09.kt)で完全なコードを取得できます。

`BehaviorSubject` に基づく前の例と同じ出力を生成します。

```text
four
```

<!--- TEST -->

[BroadcastChannel]の別の実装は、指定された `capacity` の配列ベースのバッファーを持つ `ArrayBroadcastChannel` です。
`BroadcastChannel(capacity)` で作成できます。
対応するサブスクリプションが開かれるとすぐに、すべてのイベントをすべてのサブスクライバーに配信します。
Rxの[PublishSubject](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/PublishSubject.html)に相当します。
`ArrayBroadcastChannel` のコンストラクタ内のバッファの容量は、受信側が要素を受け取るのを送信側が待たずに送信できる要素の数を制御します。

## 演算子

Rxのようなフル機能のリアクティブストリームライブラリには、ストリームを作成、変換、結合、およびその他の方法で処理するための[非常に多くの演算子](https://reactivex.io/documentation/operators.html)が付属しています。 バックプレッシャーをサポートする独自の演算子を作成することは[難しい](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0)ことで[有名](https://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)です。

コルーチンとチャンネルは逆の経験を提供するように設計されています。
組み込みの演算子はありませんが、エレメントのストリームを処理するのは非常に簡単で、バックプレッシャーは明示的に考える必要なく自動的にサポートされます。

このセクションでは、いくつかのリアクティブストリーム演算子のコルーチンベースの実装を示します。

### Range

リアクティブストリームの `Publisher` インターフェイスのために、[range](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int))演算子の独自の実装を展開しましょう。
リアクティブストリーム用のこの演算子の非同期のきれいな実装については、[このブログ記事](https://akarnokd.blogspot.ru/2017/03/java-9-flow-api-asynchronous-integer.html)で説明しています。
それは多くのコードを必要とします。
コルーチンを使ったコードを以下に示します。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun CoroutineScope.range(context: CoroutineContext, start: Int, count: Int) = publish<Int>(context) {
    for (x in start until start + count) send(x)
}
```

ここでは、 `Executor` の代わりに `CoroutineScope` と `context` が使用され、すべてのバックプレッシャーの側面はコルーチンの機械によって処理されます。
この実装は、 `Publisher` インターフェイスとその仲間を定義する小さなリアクティブストリームライブラリのみに依存することに注意してください。

コルーチンからそれを使用するのは簡単です。

```kotlin
fun main() = runBlocking<Unit> {
    // RangeはrunBlockingから親ジョブを継承しますが、Dispatchers.Defaultでディスパッチャーをオーバーライドします。
        range(Dispatchers.Default, 1, 5).collect { println(it) }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-01.kt)で完全なコードを取得できます。

このコードの結果は極めて当然です。
   
```text
1
2
3
4
5
```

<!--- TEST -->

### filterとmapの融合

[filter](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#filter(io.reactivex.functions.Predicate))や[map](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#map(io.reactivex.functions.Function))のようなリアクティブ演算子は、コルーチンで実装するのは簡単です。
ちょっとしたチャレンジとショーケースのために、それらを単一の `fusedFilterMap` 演算子に組み合わせましょう。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T, R> Publisher<T>.fusedFilterMap(
    context: CoroutineContext,   // このコルーチンを実行するためのコンテキスト
    predicate: (T) -> Boolean,   // フィルター述語
    mapper: (T) -> R             // mapper関数
) = publish<R>(context) {
    collect {                    // ソースストリームを収集する
        if (predicate(it))       // フィルター部
            send(mapper(it))     // マップ部
    }        
}
```

前の例の `range` を使うと、偶数にフィルタリングして文字列にマッピングする `fusedFilterMap` のテストができます。

<!--- INCLUDE

fun CoroutineScope.range(start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) send(x)
}
-->

```kotlin
fun main() = runBlocking<Unit> {
   range(1, 5)
       .fusedFilterMap(Dispatchers.Unconfined, { it % 2 == 0}, { "$it is even" })
       .collect { println(it) } // 結果の文字列をすべてプリントする
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-02.kt)で完全なコードを取得できます。

結果が次のようになることを確認するのは難しくありません。

```text
2 is even
4 is even
```

<!--- TEST -->

### Take until

独自のバージョンの[takeUntil](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#takeUntil(org.reactivestreams.Publisher))演算子を実装しましょう。
2つのストリームのサブスクリプションを追跡および管理する必要があるため、非常に[扱いにくい](https://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)です。
他のストリームが完了するか何かを送出するまで、ソースストリームからすべての要素をリレーする必要があります。
しかし、コルーチンの実装を助ける[select]式があります。

<!--- INCLUDE
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlinx.coroutines.selects.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T, U> Publisher<T>.takeUntil(context: CoroutineContext, other: Publisher<U>) = publish<T>(context) {
    this@takeUntil.openSubscription().consume { // Publisher<T>のチャネルを明示的に開く
        val current = this
        other.openSubscription().consume { // Publisher<U>のチャネルを明示的に開く
            val other = this
            whileSelect {
                other.onReceive { false }            // `other` から何か要素を受け取ったら脱出する
                current.onReceive { send(it); true } // thisChannelの要素を再送して続行する
            }
        }
    }
}
```

このコードは、[whileSelect]を `while(select {...}){}` ループへのショートカットとして使用し、Kotlinの[consume]式を使用して、対応するパブリッシャーからサブスクライブを解除するチャネルを終了します。

以下の決め打ちの[range](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int))と[interval](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#interval(long,%20java.util.concurrent.TimeUnit,%20io.reactivex.Scheduler))の組み合わせがテストに使用されます。
`publish` コルーチンビルダーを使用してコード化されています（純粋なRxの実装は後のセクションで説明しています）。

```kotlin
fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // 各数値を送る前に待つ
        send(x)
    }
}
```

次のコードは `takeUntil` の働きを示しています。

```kotlin
fun main() = runBlocking<Unit> {
    val slowNums = rangeWithInterval(200, 1, 10)         // 200ミリ秒間隔の数列
    val stop = rangeWithInterval(500, 1, 10)             // 最初のものは500ミリ秒後
    slowNums.takeUntil(Dispatchers.Unconfined, stop).collect { println(it) } // テストしてみる
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-03.kt)で完全なコードを取得できます。

出力は、

```text
1
2
```

<!--- TEST -->

### Merge

コルーチンで複数のデータストリームを処理するには、常に少なくとも2つの方法があります。
前の例では、[select]を伴う1つの方法が示されていました。
もう1つの方法は、複数のコルーチンを起動することです。 後者の手法を使って[merge](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#merge(org.reactivestreams.Publisher))演算子を実装しましょう。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T> Publisher<Publisher<T>>.merge(context: CoroutineContext) = publish<T>(context) {
  collect { pub -> // 収集されたパブリッシャーごと
      launch {  // 子コルーチンを起動
          pub.collect { send(it) } // このパブリッシャーからすべての要素を再送する
      }
  }
}
```

ここで起動されているすべてのコルーチンは `publish` コルーチンの子であり、 `publish` コルーチンがキャンセルされるか完了しないとキャンセルされます。
さらに、親コルーチンはすべての子が完了するまで待機するので、この実装はすべての受信ストリームを完全にマージします。

テストのために、前の例の `rangeWithInterval` 関数と、いくらか遅れてその結果を2回送るプロデューサーを書くことから始めましょう。

<!--- INCLUDE

fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // 各数値を送信する前に待つ
        send(x)
    }
}
-->

```kotlin
fun CoroutineScope.testPub() = publish<Publisher<Int>> {
    send(rangeWithInterval(250, 1, 4)) // 数値 1 は 250ms, 2 は 500ms, 3 は 750ms, 4 は 1000ms 
    delay(100) // 100ms待つ
    send(rangeWithInterval(500, 11, 3)) // 数値 11 は 600ms, 12 は 1100ms, 13 は 1600ms
    delay(1100) // 1.1秒待つ - 開始から1.2秒後に完了
}
```

次のテストコードは、 `testPub` に `merge` を使い、結果を表示します。

```kotlin
fun main() = runBlocking<Unit> {
    testPub().merge(Dispatchers.Unconfined).collect { println(it) } // ストリーム全体をプリントする
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-04.kt)で完全なコードを取得できます。

結果は次のようになります。

```text
1
2
11
3
4
12
13
```

<!--- TEST -->

## コルーチンコンテキスト

前のセクションで示したすべての演算子の例には明示的な[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/-coroutine-context/)パラメーターがあります。
Rxの世界では、それはおおよそ[Scheduler](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Scheduler.html)に相当します。

### Rxのスレッド

次の例は、Rxでのスレッドコンテキスト管理の基本を示しています。
ここで `rangeWithIntervalRx` はRx `zip`、 `range` 、 `interval` 演算子を使った `rangeWithInterval` 関数の実装です。

<!--- INCLUDE
import io.reactivex.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> = 
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-context-01.kt)で完全なコードを取得できます。

`rangeWithIntervalRx` 演算子に明示的に[Schedulers.computation()](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#computation())スケジューラーを渡しています。これはRx計算スレッドプールで実行されます。
出力は次のようになります。

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### コルーチンのスレッド

コルーチンの世界では、 `Schedulers.computation()` は[Dispatchers.Default]にほぼ対応しているので、前の例は次の例に似ています。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // 各数値を送信する前に待つ
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-context-02.kt)で完全なコードを取得できます。

生成される出力は次のようになります。

```text
1 on thread ForkJoinPool.commonPool-worker-1
2 on thread ForkJoinPool.commonPool-worker-1
3 on thread ForkJoinPool.commonPool-worker-1
```

<!--- TEST LINES_START -->

ここでは、独自のスケジューラーを持たず、パブリッシャーと同じスレッドで動作するRx [subscribe](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#subscribe(io.reactivex.functions.Consumer))演算子を使用しました。この例では デフォルトの共有スレッドプールを使用しています。

### Rx observeOn

Rxでは特別な演算子を使用してチェーン内の操作のスレッドコンテキストを変更します。
まだ慣れていない場合、それらについての[良いガイド](https://tomstechnicalblog.blogspot.ru/2016/02/rxjava-understanding-observeon-and.html)を見つけることができます。

例えば、[observeOn](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler))演算子があります。
`Schedulers.computation()` を使ってオブザーブするために前の例を修正しましょう。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.schedulers.Schedulers
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // 各数値を送信する前に待つ
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .observeOn(Schedulers.computation())                           // <-- この行が追加された
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-context-03.kt)で完全なコードを取得できます。

出力の違いは次のとおりです。"RxComputationThreadPool" に注目してください。

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### すべてを制御するコルーチンコンテキスト

コルーチンは、常に何らかのコンテキストで動作しています。 例えば、[runBlocking]を使用してメインスレッドでコルーチンを開始し、Rxバージョンの `rangeWithIntervalRx` 演算子の結果をRx `subscribe` 演算子を使用するのではなく反復処理しましょう。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .collect { println("$it on thread ${Thread.currentThread().name}") }
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-context-04.kt)で完全なコードを取得できます。

結果として得られるメッセージはメインスレッドでプリントされます。

```text
1 on thread main
2 on thread main
3 on thread main
```

<!--- TEST LINES_START -->

### Unconfinedコンテキスト

ほとんどのRxオペレーターには特定のスレッド（スケジューラー）が関連付けられておらず、呼び出されたスレッドで動作しています。
[Rxのスレッド](#rxのスレッド)セクションの `subscribe` 演算子の例で見てきました。

コルーチンの世界では、[Dispatchers.Unconfined]コンテキストも同様の役割を果たします。
前の例を変更してみましょう。メインスレッドに限定された `runBlocking` コルーチンからソース `Flowable` を反復するのではなく、 `Dispatchers.Unconfined` コンテキストで新しいコルーチンを起動します。メインコルーチンは[Job.join]を使用してただ完了を待つだけです。

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    val job = launch(Dispatchers.Unconfined) { // Unconfinedコンテキストで新しいコルーチンを起動する（独自のスレッドプールなし）
        rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
            .collect { println("$it on thread ${Thread.currentThread().name}") }
    }
    job.join() // コルーチンが完了するのを待つ
}
```

> [ここ](kotlinx-coroutines-rx2/test/guide/example-reactive-context-05.kt)で完全なコードを取得できます。

ここで出力は、Rx `subscribe` 演算子を使用した初期の例のように、コルーチンのコードがRx計算スレッドプールで実行されていることを示しています。

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST LINES_START -->

[Dispatchers.Unconfined]コンテキストは注意して使用する必要があります。
操作のスタックローカリティ性の向上とスケジューリングのオーバーヘッドのために、特定のテストの全体的なパフォーマンスが向上する場合がありますが、スタックが深くなり、使用しているコードの非同期性を推測するのが難しくなります。

コルーチンがチャネルに要素を送ると、[send][SendChannel.send]を呼び出したスレッドは[Dispatchers.Unconfined]ディスパッチャーでコルーチンのコードの実行を開始することがあります。
`send` を呼び出す元のプロデューサーコルーチンは、unconfinedのコンシューマーコルーチンが次の中断ポイントに達するまで一時停止されます。
これは、スレッドシフト演算子がない場合のRx世界でのロックステップのシングルスレッドの `onNext` の実行と非常によく似ています。
オペレータは通常、非常に小さなチャンクで作業をしており、複雑な処理のために多くの演算子を結合する必要があるため、これはRxの通常のデフォルトです。
しかし、コルーチンではこれが異常です。コルーチンでは、任意の複雑な処理を行うことができます。
通常、複数のワーカーコルーチン間でファンインとファンアウトを持つ複雑なパイプラインのストリーム処理コルーチンをチェーンするだけで済みます。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[ReceiveChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/index.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[consume]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[BroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-broadcast-channel/index.html
[ConflatedBroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-conflated-broadcast-channel/index.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
[whileSelect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/while-select.html
<!--- MODULE kotlinx-coroutines-reactive -->
<!--- INDEX kotlinx.coroutines.reactive -->
[publish]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/publish.html
[org.reactivestreams.Publisher.collect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/collect.html
[org.reactivestreams.Publisher.openSubscription]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/open-subscription.html
<!--- MODULE kotlinx-coroutines-rx2 -->
<!--- INDEX kotlinx.coroutines.rx2 -->
[rxFlowable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-rx2/kotlinx.coroutines.rx2/rx-flowable.html
<!--- END -->


