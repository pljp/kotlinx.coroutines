<!--- INCLUDE .*/example-ui-([a-z]+)-([0-9]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide-ui.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.javafx.guide.$$1$$2

import kotlinx.coroutines.experimental.*
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.javafx.JavaFx as Main
import javafx.application.Application
import javafx.event.EventHandler
import javafx.geometry.*
import javafx.scene.*
import javafx.scene.input.MouseEvent
import javafx.scene.layout.StackPane
import javafx.scene.paint.Color
import javafx.scene.shape.Circle
import javafx.scene.text.Text
import javafx.stage.Stage

fun main(args: Array<String>) {
    Application.launch(ExampleApp::class.java, *args)
}

class ExampleApp : Application() {
    val hello = Text("Hello World!").apply {
        fill = Color.valueOf("#C0C0C0")
    }

    val fab = Circle(20.0, Color.valueOf("#FF4081"))

    val root = StackPane().apply {
        children += hello
        children += fab
        StackPane.setAlignment(hello, Pos.CENTER)
        StackPane.setAlignment(fab, Pos.BOTTOM_RIGHT)
        StackPane.setMargin(fab, Insets(15.0))
    }

    val scene = Scene(root, 240.0, 380.0).apply {
        fill = Color.valueOf("#303030")
    }

    override fun start(stage: Stage) {
        stage.title = "Example"
        stage.scene = scene
        stage.show()
        setup(hello, fab)
    }
}
-->
<!--- KNIT     kotlinx-coroutines-javafx/test/guide/.*\.kt -->

# コルーチンによるUIプログラミングガイド

このガイドは、[kotlinx.coroutinesのガイド](../docs/coroutines-guide.md)でカバーされている基本的なコルーチンの概念に精通していることを前提としており、UIアプリケーションでコルーチンを使用する方法の具体例を示しています。

すべてのUIアプリケーションライブラリに共通して1つのことがあります。 UIのすべての状態が拘束される単一のメインスレッドがあり、この特定のスレッドでUIに対するすべての更新が行われなければなりません。 コルーチンに関しては、コルーチンの実行をこのメインUIスレッドに制限する適切な _コルーチンディスパッチャーコンテキスト_ が必要であることを意味します。

具体的には、 `kotlinx.coroutines` は異なるUIアプリケーションライブラリにコルーチンのコンテキストを提供する3つのモジュールを持っています。
 
* [kotlinx-coroutines-android](kotlinx-coroutines-android) -- Androidアプリケーションのための `Dispatchers.Main` コンテキスト。
* [kotlinx-coroutines-javafx](kotlinx-coroutines-javafx) -- JavaFX UIアプリケーションのための `Dispatchers.JavaFx` コンテキスト。
* [kotlinx-coroutines-swing](kotlinx-coroutines-swing) -- Swing UI アプリケーションのための `Dispatchers.Swing` コンテキスト。

このガイドでは、すべてのUIライブラリを同時に扱います。なぜなら、これらのモジュールのそれぞれは、数ページの長さのただ1つだけのオブジェクト定義から成っているからです。 ここには含まれていなくても、これらのどれかを例として使って、好みのUIライブラリ用の対応するコンテキストオブジェクトを書くことができます。

## 目次

<!--- TOC -->

* [セットアップ](#セットアップ)
  * [JavaFx](#javafx)
  * [Android](#android)
* [基本的なUIコルーチン](#基本的なuiコルーチン)
  * [UIコルーチンの起動](#uiコルーチンの起動)
  * [UIコルーチンのキャンセル](#uiコルーチンのキャンセル)
* [UIコンテキスト内でのアクターの使用](#uiコンテキスト内でのアクターの使用)
  * [コルーチンのための拡張](#コルーチンのための拡張)
  * [最大で1つの同時ジョブ](#最大で1つの同時ジョブ)
  * [イベントの合流](#イベントの合流)
* [ブロッキング操作](#ブロッキング操作)
  * [UIフリーズ問題](#uiフリーズ問題)
  * [構造化並列処理、ライフサイクルおよびコルーチンの親子階層](#構造化並列処理ライフサイクルおよびコルーチンの親子階層)
  * [ブロッキング操作](#ブロッキング操作)
* [高度なトピック](#高度なトピック)
  * [ディスパッチせずにUIイベントハンドラでコルーチンを開始する](#ディスパッチせずにuiイベントハンドラでコルーチンを開始する)

<!--- END_TOC -->

## セットアップ

このガイドの実行可能な例は、JavaFx用に提供されています。 利点は、エミュレータなどを必要とせずにすべてのサンプルを任意のOSで直接起動でき、完全に自己完結していることです（各例は1つのファイルにあります）。
Androidでそれらを再現するために必要な変更点（もしあれば）について別途メモがあります。

### JavaFx

JavaFxの基本的なアプリケーションの例は、最初に文字列 "Hello World！" を含む `hello` という名前のテキストラベルと、右下に `fab` （フローティングアクションボタン）という名前のピンクの円を持つウィンドウで構成されています。

![UI example for JavaFx](ui-example-javafx.png)

JavaFXアプリケーションの `start` 関数は `hello` と `fab` ノードへの参照を渡す `setup` 関数を呼び出します。
ここには、このガイドの残りの部分でさまざまなコードを記述しています。

```kotlin
fun setup(hello: Text, fab: Circle) {
    // placeholder
}
```

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-basic-01.kt)で完全なコードを取得できます

GitHubの[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)プロジェクトをワークステーションにクローンし、IDEでプロジェクトを開くことができます。 このガイドのすべての例は、[`ui/kotlinx-coroutines-javafx`](kotlinx-coroutines-javafx)モジュールのテストフォルダにあります。
これにより、各サンプルがどのように動作するかを確認し、変更して実験することができます。

### Android

Android StudioでKotlinプロジェクトを作成するには、[Getting Started With Android and Kotlin](https://kotlinlang.org/docs/tutorials/kotlin-android.html)のガイドに従ってください。 アプリケーションに[Kotlin Android Extensions](https://kotlinlang.org/docs/tutorials/android-plugin.html)を追加することもお勧めします。

Android Studio 2.3では、次のようなアプリケーションが表示されます。

![UI example for Android](ui-example-android.png)

アプリケーションの `context_main.xml` に移動し、文字列 "Hello World！"を持ったテキストビューに "hello" というIDを割り当てます。 これは、あなたのアプリケーションで、Kotlin Android拡張機能から `hello` として利用できるようにするためです。
ピンク色のフローティングアクションボタンは、作成されたプロジェクトテンプレート内で既に `fab` という名前が付いています。

アプリケーションの `MainActivity.kt` では `fab.setOnClickListener {...}` ブロックを削除し、代わりに `onCreate` 関数の最後の行として `setup(hello、fab)` 呼び出しを追加します。
ファイルの最後にプレースホルダーの `setup` 関数を作成します。
ここには、このガイドの以降の部分でさまざまなコードを記述しています。

```kotlin
fun setup(hello: TextView, fab: FloatingActionButton) {
    // placeholder
}
```

<!--- CLEAR -->

`kotlinx-coroutines-android` モジュールへの依存関係を `app/build.gradle` ファイルの `dependencies { ... }` セクションに追加してください。

```groovy
compile "org.jetbrains.kotlinx:kotlinx-coroutines-android:0.26.1"
```

コルーチンはKotlinの実験的な機能です。
`gradle.properties` ファイルに次の行を追加することで、Kotlinコンパイラでコルーチンを有効にする必要があります。 
```properties
kotlin.coroutines=enable
```

GitHubの[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)プロジェクトをワークステーションにクローンすることができます。 Android用のテンプレートプロジェクトは、[`ui/kotlinx-coroutines-android/example-app`](kotlinx-coroutines-android/example-app)ディレクトリにあります。
Android Studioで読み込んでAndroidのこのガイドを追試することができます。

## 基本的なUIコルーチン

このセクションでは、UIアプリケーションでのコルーチンの基本的な使い方を示します。

### UIコルーチンの起動

`kotlinx-coroutines-javafx` モジュールには、JavaFxアプリケーションスレッドにコルーチンの実行をディスパッチする[Dispatchers.JavaFx][kotlinx.coroutines.experimental.Dispatchers.JavaFx]ディスパッチャーが含まれています。
提示されたすべての例をAndroidに簡単に移植できるように、これを `Main` としてインポートします。
 
```kotlin
import kotlinx.coroutines.experimental.javafx.JavaFx as Main
```
 
<!--- CLEAR -->

メインUIスレッドに限定されたコルーチンは、メインスレッドをブロックすることなく、UI内の何かを自由に更新して中断することができます。
例えば、アニメーションを命令的なスタイルでコーディングして実行することができます。 次のコードは、[launch] コルーチンビルダーを使用して、1秒に2回、10から1へのカウントダウンでテキストを更新します。

```kotlin
fun setup(hello: Text, fab: Circle) {
    GlobalScope.launch(Dispatchers.Main) { // メインコンテキストでコルーチンを起動
        for (i in 10 downTo 1) { // 10から1までカウントダウン
            hello.text = "Countdown $i ..." // テキストを更新
            delay(500) // 0.5秒待つ
        }
        hello.text = "Done!"
    }
}
```

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-basic-02.kt)で完全なコードを取得できます

さて、何が起こりましたか？
メインUIコンテキストでコルーチンを起動しているので、このコルーチン内からUIを自由に更新し、それと同時に[delay]などのサスペンド関数を呼び出すことができます。
`delay` が待っている間UIスレッドはブロックされないのでUIはフリーズしません。ただ単にコルーチンを中断します。

> Androidアプリケーション用のコードも同じです。
`setup` 関数の本体をAndroidプロジェクトの対応する関数にコピーするだけです。

### UIコルーチンのキャンセル

`launch` 関数が返す[Job]オブジェクトへの参照を保持し、コルーチンをキャンセルするために使うことができます。
ピンクの円がクリックされたときにコルーチンをキャンセルしましょう。

```kotlin
fun setup(hello: Text, fab: Circle) {
    val job = GlobalScope.launch(Dispatchers.Main) { // メインスレッドでコルーチンを起動
        for (i in 10 downTo 1) { // 10から1までカウントダウン
            hello.text = "Countdown $i ..." // テキストを更新
            delay(500) // 0.5秒待つ
        }
        hello.text = "Done!"
    }
    fab.onMouseClicked = EventHandler { job.cancel() } // クリックされたらコルーチンをキャンセル
}
```

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-basic-03.kt)で完全なコードを取得できます

カウントダウンが実行されている間に円がクリックされると、カウントダウンは停止します。
[Job.cancel]は完全にスレッドセーフでノンブロッキングです。
実際に終了するのを待つことなく、コルーチンがそのジョブをキャンセルするように通知するだけです。 どこからでも呼び出すことができます。
すでに取り消されている、または完了しているコルーチン上でそれを呼び出しても何もしません。

> Androidの対応する行は次の通りです。

```kotlin
fab.setOnClickListener { job.cancel() }  // クリックされたらコルーチンをキャンセル
```

<!--- CLEAR -->

## UIコンテキスト内でのアクターの使用

このセクションではUIアプリケーションがUIコンテキスト内でアクターをどのように使用できるかを示し、コルーチンの起動数が無限に増加していないことを確認します。

### コルーチンのための拡張

私たちの目標は、この簡単なコードで円がクリックされるたびにカウントダウンアニメーションを実行できるように、 `onClick`という拡張コルーチンビルダー関数を書くことです。

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onClick { // 円がクリックされたらコルーチンを開始する
        for (i in 10 downTo 1) { // 10から1までカウントダウン
            hello.text = "Countdown $i ..." // テキストを更新
            delay(500) // 0.5秒待つ
        }
        hello.text = "Done!"
    }
}
```

<!--- INCLUDE .*/example-ui-actor-([0-9]+).kt -->

`onClick` の最初の実装では、各マウスイベントで新しいコルーチンを起動し、対応するマウスイベントを指定されたアクションに渡します（必要な場合のみ）。

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    onMouseClicked = EventHandler { event ->
        GlobalScope.launch(Dispatchers.Main) { 
            action(event)
        }
    }
}
```  

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-actor-01.kt)で完全なコードを取得できます

円がクリックされるたびに新しいコルーチンが始まり、すべてが競合してテキストを更新することに注意してください。
試してみてください。とても良いとは思えません。後で修正します。

> Androidでは、対応する拡張を `View` クラス用に書くことができるので、上に示した `setup` 関数のコードを変更せずに使うことができます。AndroidのOnClickListenerには `MouseEvent` は使われていないので省略されています。

```kotlin
fun View.onClick(action: suspend () -> Unit) {
    setOnClickListener { 
        GlobalScope.launch(Dispatchers.Main) {
            action()
        }
    }
}
```

<!--- CLEAR -->

### 最大で1つの同時ジョブ

新しいジョブを開始する前にアクティブなジョブをキャンセルすることで、多くとも1つのコルーチンだけがカウントダウンを動かしていることを確実にできます。 
しかし、それは一般的に最良のアイデアではありません。
[cancel][Job.cancel]関数は、コルーチンを中止するための信号としてのみ機能します。 キャンセルは協調的であり、コルーチンは現時点でキャンセル不可能な何かをしているか、そうでなければキャンセル信号を無視しているかもしれません。 より良い解決策は、同時に実行すべきでないタスクに対して[actor]を使用することです。
`onClick` 拡張の実装を変更しましょう。
  
```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // このノード上のすべてのイベントを処理する1つのアクターを起動する
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main) {
        for (event in channel) action(event) // アクションにイベントを渡す
    }
    // このアクターにイベントを提供するリスナーをインストールする
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-actor-02.kt)で完全なコードを取得できます
  
アクターのコルーチンと通常のイベントハンドラの統合の根底にある重要なアイデアは、[SendChannel]に待機しない[offer][SendChannel.offer]関数があることです。
可能ならばアクターにただちに要素を送信し、そうでない場合は要素を破棄します。
`offer` は実際にはここでは無視している `Boolean` の結果を返します。

このバージョンのコードで円を繰り返しクリックしてみてください。
カウントダウンアニメーションの実行中は、クリックは無視されます。 これは、アクターがアニメーションで忙しく、そのチャネルから受信しないために発生します。
デフォルトでは、アクターのメールボックスは[RendezvousChannel]によって支援されています。その `offer` オペレーションは、`receive` がアクティブな場合にのみ成功します。

> Androidでは、 `MouseEvent` はありませんので、シグナルとしてアクターに `Unit` を送ります。
`View` クラスの対応する拡張は次のようになります。
> Androidでは、OnClickListenerに `View` が送られているので、 `View` をシグナルとしてアクターに送ります。
`View` クラスの対応する拡張は次のようになります。

```kotlin
fun View.onClick(action: suspend (View) -> Unit) {
    // 1つのアクターを起動する
    val eventActor = GlobalScope.actor<View>(Dispatchers.Main) {
        for (event in channel) action(event)
    }
    // このアクターをアクティブにするリスナーをインストールする
    setOnClickListener { 
        eventActor.offer(it)
    }
}
```

<!--- CLEAR -->


### イベントの合流
 
以前のイベントを処理している間にイベントを無視するのではなく、最新のイベントを処理する方が適切な場合もあります。
[actor]コルーチンビルダーは、このアクターがメールボックスに使用しているチャネルの実装を制御する、オプションの `capacity` パラメーターを受け取ります。
利用可能なすべての選択肢の説明は、[`Channel()`][Channel]ファクトリ関数のドキュメントに記載されています。

[Channel.CONFLATED]の容量値を渡すことで[ConflatedChannel]を使用するコードを変更しましょう。
この変更は、アクターを作成する行にのみ適用されます。

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // このノード上のすべてのイベントを処理する1つのアクターを起動する
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) { // <--- ここを変更
        for (event in channel) action(event) // イベントをアクションに渡す
    }
    // このアクターにイベントを提供するリスナーをインストールする
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-actor-03.kt)でJavaFxの完全なコードを取得できます。
Androidでは、前の例の `val eventActor = ...` 行を更新する必要があります。

アニメーションの実行中にサークルをクリックすると、アニメーションの終了後にアニメーションが一度だけ再び起動されます。
アニメーションの実行中に繰り返しクリックされると、_合流_ して最新のイベントのみが処理されます。

これはまた、直前に受信した更新に基づいてUIを更新することによって、入ってくる高頻度のイベントストリームに反応しなければならないUIアプリケーションにとって望ましい動作です。
[ConflatedChannel]を使用しているコルーチンは、イベントのバッファリングによって大抵発生する遅延を防ぎます。

上の行で `capacity` パラメーターを試して、コードの動作にどのように影響するかを調べることができます。
`capacity = Channel.UNLIMITED` を設定すると、すべてのイベントをバッファーする[LinkedListChannel]メールボックスを持つコルーチンが作成されます。 この場合、アニメーションは円がクリックされた回数だけ実行されます。

## ブロッキング操作

このセクションでは、スレッドをブロックする操作でUIコルーチンを使用する方法について説明します。

### UIフリーズ問題

すべてのAPIが実行スレッドを決してブロックしないサスペンド関数として記述されていれば素晴らしいことです。
しかし、ほとんどの場合そうではありません。 場合によっては、例えば呼び出し側スレッドをブロックするようなCPUを消費する計算を行ったり、ネットワークアクセス用のサードパーティAPIを呼び出すだけの場合もあります。
これは、メインUIスレッドをブロックしてUIがフリーズする原因になるため、メインUIスレッドやUI制約コルーチンから直接行うことはできません。

<!--- INCLUDE .*/example-ui-blocking-([0-9]+).kt

fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    val eventActor = actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) {
        for (event in channel) action(event) // アクションにイベントを渡す
    }
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
-->

次の例は、この問題を示しています。 メインUIスレッドで最後のクリックを処理するために、前のセクションのUI制約のイベント合流アクターで `onClick` 拡張を使用します。
この例では、[フィボナッチ数](https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A3%E3%83%9C%E3%83%8A%E3%83%83%E3%83%81%E6%95%B0)の素朴な計算を実行します。
 
```kotlin
fun fib(x: Int): Int =
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
``` 
 
円がクリックされるたびに、より大きなフィボナッチ数を計算します。
UIのフリーズをより明確にするために、常に実行中の高速カウントアニメーションもあり、メインUIディスパッチャーのテキストを常に更新しています。

```kotlin
fun setup(hello: Text, fab: Circle) {
    var result = "none" // 直前の結果
    // カウントアニメーション
    GlobalScope.launch(Dispatchers.Main) {
        var counter = 0
        while (true) {
            hello.text = "${++counter}: $result"
            delay(100) // 100ミリ秒ごとにテキストを更新する
        }
    }
    // クリックするたびに次のフィボナッチ数を計算する
    var x = 1
    fab.onClick {
        result = "fib($x) = ${fib(x)}"
        x++
    }
}
```
 
> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-01.kt)で完全なJavaFxコードを取得できます。
`fib` 関数と `setup` 関数の本体をあなたのAndroidプロジェクトにコピーすることができます。

この例の円をクリックしてみてください。
およそ30～40回のクリック後、素朴な計算はかなり遅くなり、UIがフリーズしている間アニメーションが停止するのでメインUIスレッドがフリーズする様子がすぐにわかります。

### 構造化並列処理、ライフサイクルおよびコルーチンの親子階層

典型的なUIアプリケーションには、ライフサイクルの要素がいくつかあります。
ウィンドウ、UIコントロール、アクティビティ、ビュー、フラグメント、その他の視覚的要素が作成され、破棄されます。
IOまたはバックグラウンド計算を実行する長期実行コルーチンは、必要以上に長くUI要素への参照を保持し、すでに破棄されて表示されないUIオブジェクトのツリー全体のガベージコレクションを阻みます。

この問題の自然な解決策は、ライフサイクルを持つ各UIオブジェクトに[Job]オブジェクトを関連付けて、このジョブのコンテキストですべてのコルーチンを作成することです。
しかし、関連するジョブオブジェクトをすべてのコルーチンのビルダーに渡すのは忘れやすく、エラーが発生しやすくなります。
この目的のためには、[CoroutineScope]インターフェイスをUI所有者が実装することで、[CoroutineScope]の拡張として定義されているすべてのコルーチンビルダーは、明示的に言及せずにUIジョブを継承します。

例えば、Androidアプリケーションでは最初に `Activity` が _作成_ され、不要になったときやメモリを解放しなければならないときに _破棄_ されます。
自然な解決策は、 `Activity` のインスタンスに `Job` のインスタンスを付随することです。

<!--- CLEAR -->

```kotlin
abstract class ScopedAppActivity: AppCompatActivity(), CoroutineScope {
    protected lateinit var job: Job
    override val coroutineContext: CoroutineContext 
        get() = job + Dispatchers.Main
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        job = Job()
    }
        
    override fun onDestroy() {
        super.onDestroy()
        job.cancel()
    } 
}
```

ここで、ジョブに関連するアクティビティは、ScopedAppActivityを拡張する必要があります。

```kotlin
class MainActivity : ScopedAppActivity() {

    fun asyncShowData() = launch { // アクティビティのジョブを親とするUIコンテキストで呼び出される
        // 実際の実装
    }
    
    suspend fun showIOData() {
        val deferred = async(Dispatchers.IO) {
            // 実装
        }
        withContext(Dispatchers.Main) {
          val data = deferred.await()
          // UIにデータを表示する
        }
    }
}
```

`MainActivity` の中から起動したコルーチンはすべてそのジョブを親として持ち、アクティビティが破棄されると直ちに取り消されます。

アクティビティスコープをそのビューとプレゼンターに伝播するには、現在のスコープをキャプチャする[currentScope]ビルダーを使用できます。
別のソリューションは、デリゲーションを使用して子要素に[CoroutineScope]を実装することです。例：

```kotlin
class ActivityWithPresenters: ScopedAppActivity() {
    fun init() {
        val presenter = Presenter()
        val presenter2 = NonSuspendingPresenter(this)
    }
}

class Presenter {
    suspend fun loadData() = currentScope {
        // ここは ActivityWithPresenters のスコープの中
    }
}

class NonSuspendingPresenter(scope: CoroutineScope): CoroutineScope by scope {
    fun loadData() = launch { // ActivityWithPresentersのスコープの拡張
        // 実装
    }
}
``` 

ジョブ間の親子関係は階層を形成します。
ビューの代わりにバックグラウンドジョブを実行するコルーチンは、そのコンテキストでさらなる子コルーチンを作り出すことができます。
親ジョブがキャンセルされると、コルーチンのツリー全体がキャンセルされます。
その例は、コルーチンのガイドの[「コルーチンの子」](../docs/coroutines-guide.md#コルーチンの子)セクションに示されています。
<!--- CLEAR -->

### ブロッキング操作

メインUIスレッドでのブロッキング操作の修正は、コルーチンでは非常に簡単です。
「ブロックキング」 `fib` 関数をノンブロッキングサスペンド関数に変換します。このサスペンド関数は、[withContext]関数を使用して実行コンテキストをバックグラウンドのスレッドプールによって支えられた[Dispatchers.Default]に変更して計算を実行します。
`fib` 関数は `suspend` 修飾子でマークされていることに注意してください。
これは、呼び出されたコルーチンをブロックしませんが、バックグラウンドスレッドの計算が動作しているときにその実行を中断します。

<!--- INCLUDE .*/example-ui-blocking-0[23].kt

fun setup(hello: Text, fab: Circle) {
    var result = "none" // 直前の結果
    // カウントアニメーション
    GlobalScope.launch(Dispatchers.Main) {
        var counter = 0
        while (true) {
            hello.text = "${++counter}: $result"
            delay(100) // 100ミリ秒ごとにテキストを更新する
        }
    }
    // クリックするたびに次のフィボナッチ数を計算する
    var x = 1
    fab.onClick {
        result = "fib($x) = ${fib(x)}"
        x++
    }
}
-->

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
}
```

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-02.kt)で完全なコードを取得できます。

このコードを実行して、大きなフィボナッチ数が計算されている間、UIがフリーズしていないことを確認できます。
しかし、このコードは `fib` をいくらか遅く計算します。なぜなら、 `fib` へのすべての再帰呼び出しは `withContext` を経由するからです。
これは、実際には大きな問題ではありません。なぜなら、 `withContext` はコルーチンがすでに必要なコンテキストで実行されているかどうかを確認するのに十分スマートで、別のスレッドにコルーチンを再ディスパッチするオーバーヘッドを避けるからです。
それにもかかわらず、 `withContext` の呼び出しの間に整数を追加する他に何もしないこのプリミティブコードでは明らかにオーバーヘッドがあります。
より実用的なコードでは、余分な `withContext` 呼び出しのオーバーヘッドは重要ではありません。

それでも、元の `fib` 関数の名前を `fibBlocking` に変更し、 `fib` を `fibBlocking` の上に `withContext` ラッパーを被せて定義してバックグラウンドスレッドで動かせば、この `fib` の実装は以前のように高速で実行できます。

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    fibBlocking(x)
}

fun fibBlocking(x: Int): Int = 
    if (x <= 1) x else fibBlocking(x - 1) + fibBlocking(x - 2)
```

> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-03.kt)で完全なコードを取得できます。

メインUIスレッドをブロックせずに、フルスピードのフィボナッチ計算を楽しむことができます。
必要なのは、 `withContext(Dispatchers.Default)` だけです。

`fib` 関数はコード内の単一のアクターから呼び出されるので、与えられた時間に最大で同時に1つの計算をすることに注意してください。したがって、このコードはリソース使用率に自然な制限があります。
最大で1つのCPUコアを飽和させることができます。
  
## 高度なトピック

このセクションでは、さまざまな高度なトピックについて説明します。

### ディスパッチせずにUIイベントハンドラでコルーチンを開始する

UIスレッドからコルーチンが起動したときの実行順序を視覚化するために `setup` に次のコードを書きましょう。

<!--- CLEAR -->

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main) {
            println("Inside coroutine")
            delay(100)
            println("After delay")
        } 
        println("After launch")
    }
}
```
 
> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-advanced-01.kt)で完全なJavaFxコードを取得できます。

このコードを開始しピンクの円をクリックすると、次のメッセージがコンソールに出力されます。
 
```text
Before launch
After launch
Inside coroutine
After delay
```

ご覧のように、[launch]の後すぐに実行が継続され、後で実行するためにコルーチンがメインUIスレッドにポストされます。
`kotlinx.coroutines` のすべてのUIディスパッチャはこのように実装されています。
なぜでしょうか？

基本的にここでの選択は、「JSスタイル」の非同期アプローチ（非同期アクションは常に後で同一のディスパッチスレッドで実行されるように延期されます）と「C#スタイル」アプローチ（非同期アクションは、最初の中断ポイントまで呼び出し元スレッドで実行されます）の間です。

一方、C#のアプローチはより効率的だと思われますが、「必要な場合には`yield`を使う…」のような推奨事項で終わります。
これはエラーが起こりやすいです。
JSスタイルのアプローチはより一貫性があり、プログラマーはyieldする必要があるかどうかについて考える必要はありません。

しかし、この特定のケースでは、コルーチンがイベントハンドラーから開始されそのまわりに他のコードがない場合、この追加のディスパッチは実際には付加価値を持たず余分なオーバーヘッドを追加します。
この場合、[launch]、[async]および[actor]コルーチンビルダーに対するオプションの[CoroutineStart]パラメーターを使用して、パフォーマンスを最適化することができます。
これを[CoroutineStart.UNDISPATCHED]の値に設定すると、次の例に示すように最初の中断ポイントまですぐにコルーチンを実行し始める効果があります。

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main, CoroutineStart.UNDISPATCHED) { // <--- この変更に注意
            println("Inside coroutine")
            delay(100)                            // <--- そしてここがコルーチンが中断する場所
            println("After delay")
        }
        println("After launch")
    }
}
```
 
> [ここ](kotlinx-coroutines-javafx/test/guide/example-ui-advanced-02.kt)で完全なJavaFxコードを取得できます。

クリックすると次のメッセージをプリントします。コルーチンのコードの実行がすぐに開始されることを確認してください。

```text
Before launch
Inside coroutine
After launch
After delay
```
  
<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/cancel.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html
[currentScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/current-scope.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-context.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-dispatchers/-default.html
[CoroutineStart]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/index.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html
[CoroutineStart.UNDISPATCHED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/-u-n-d-i-s-p-a-t-c-h-e-d.html
<!--- INDEX kotlinx.coroutines.experimental.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/actor.html
[SendChannel.offer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/offer.html
[SendChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/index.html
[RendezvousChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-rendezvous-channel/index.html
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html
[ConflatedChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-conflated-channel/index.html
[Channel.CONFLATED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/-c-o-n-f-l-a-t-e-d.html
[LinkedListChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-linked-list-channel/index.html
<!--- MODULE kotlinx-coroutines-javafx -->
<!--- INDEX kotlinx.coroutines.experimental.javafx -->
[kotlinx.coroutines.experimental.Dispatchers.JavaFx]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-javafx/kotlinx.coroutines.experimental.javafx/kotlinx.coroutines.experimental.-dispatchers/-java-fx.html
<!--- MODULE kotlinx-coroutines-android -->
<!--- INDEX kotlinx.coroutines.experimental.android -->
<!--- END -->
