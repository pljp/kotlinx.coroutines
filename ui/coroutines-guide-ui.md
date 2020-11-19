# コルーチンによるUIプログラミングガイド

このガイドは、[kotlinx.coroutinesのガイド](../docs/coroutines-guide.md)でカバーされている基本的なコルーチンの概念に精通していることを前提としており、UIアプリケーションでコルーチンを使用する方法の具体例を示しています。

すべてのUIアプリケーションライブラリに共通して1つのことがあります。 UIのすべての状態が拘束される単一のメインスレッドがあり、この特定のスレッドでUIに対するすべての更新が行われなければなりません。 コルーチンに関しては、コルーチンの実行をこのメインUIスレッドに制限する適切な _コルーチンディスパッチャーコンテキスト_ が必要であることを意味します。

具体的には、 `kotlinx.coroutines` は異なるUIアプリケーションライブラリにコルーチンのコンテキストを提供する3つのモジュールを持っています。

* [kotlinx-coroutines-android](kotlinx-coroutines-android) -- Androidアプリケーションのための `Dispatchers.Main` コンテキスト。
* [kotlinx-coroutines-javafx](kotlinx-coroutines-javafx) -- JavaFX UIアプリケーションのための `Dispatchers.JavaFx` コンテキスト。
* [kotlinx-coroutines-swing](kotlinx-coroutines-swing) -- Swing UI アプリケーションのための `Dispatchers.Swing` コンテキスト。

また、UIディスパッチャーは、 `kotlinx-coroutines-core` の `Dispatchers.Main` を介して利用でき、対応する実装（Android、JavaFx、またはSwing）は[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) APIによって検出されます。
たとえば、JavaFxアプリケーションを作成している場合、 `Dispatchers.Main` または `Dispachers.JavaFx` 拡張のいずれかを使用できますが、これは同じオブジェクトになります。

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
  * [ディスパッチせずにUIイベントハンドラーでコルーチンを開始する](#ディスパッチせずにuiイベントハンドラーでコルーチンを開始する)

<!--- END -->

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
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.4.0"
```

GitHubの[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)プロジェクトをワークステーションにクローンすることができます。 Android用のテンプレートプロジェクトは、[`ui/kotlinx-coroutines-android/example-app`](kotlinx-coroutines-android/example-app)ディレクトリにあります。
Android Studioで読み込んでAndroidのこのガイドを追試することができます。

## 基本的なUIコルーチン

このセクションでは、UIアプリケーションでのコルーチンの基本的な使い方を示します。

### UIコルーチンの起動

`kotlinx-coroutines-javafx` モジュールには、JavaFxアプリケーションスレッドにコルーチンの実行をディスパッチする[Dispatchers.JavaFx][kotlinx.coroutines.Dispatchers.JavaFx]ディスパッチャーが含まれています。
提示されたすべての例をAndroidに簡単に移植できるように、これを `Main` としてインポートします。

```kotlin
import kotlinx.coroutines.javafx.JavaFx as Main
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

アクターのコルーチンと通常のイベントハンドラーの統合の根底にある重要なアイデアは、[SendChannel]に待機しない[offer][SendChannel.offer]関数があることです。
可能ならばアクターにただちに要素を送信し、そうでない場合は要素を破棄します。
`offer` は実際にはここでは無視している `Boolean` の結果を返します。

このバージョンのコードで円を繰り返しクリックしてみてください。
カウントダウンアニメーションの実行中は、クリックは無視されます。 これは、アクターがアニメーションで忙しく、そのチャネルから受信しないために発生します。
デフォルトでは、アクターのメールボックスは `RendezvousChannel` によって支援されています。その `offer` オペレーションは、`receive` がアクティブな場合にのみ成功します。

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

[Channel.CONFLATED]の容量値を渡して、統合チャネルを使用するようにコードを変更しましょう。
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
アニメーションの実行中に繰り返しクリックされると、_合成_ して最新のイベントのみが処理されます。

これは、最近受信した更新に基づいてUIを更新することにより、受信する高頻度イベントストリームに対応する必要があるUIアプリケーションにとっても望ましい動作です。
`ConflatedChannel` を使用するコルーチンは通常、イベントのバッファリングによって発生する遅延を回避します。

上の行で `capacity` パラメーターを試して、コードの動作にどのように影響するかを調べることができます。
`capacity = Channel.UNLIMITED` を設定すると、すべてのイベントをバッファーする `LinkedListChannel` メールボックスを持つコルーチンが作成されます。 この場合、アニメーションは円がクリックされた回数だけ実行されます。

## ブロッキング操作

このセクションでは、スレッドをブロックする操作でUIコルーチンを使用する方法について説明します。

### UIフリーズ問題

すべてのAPIが実行スレッドを決してブロックしないサスペンド関数として記述されていれば素晴らしいことです。
しかし、ほとんどの場合そうではありません。 場合によっては、例えば呼び出し側スレッドをブロックするようなCPUを消費する計算を行ったり、ネットワークアクセス用のサードパーティAPIを呼び出すだけの場合もあります。
これは、メインUIスレッドをブロックしてUIがフリーズする原因になるため、メインUIスレッドやUI制約コルーチンから直接行うことはできません。

<!--- INCLUDE .*/example-ui-blocking-([0-9]+).kt
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) {
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
IOまたはバックグラウンド計算を実行する長時間実行コルーチンは、対応するUI要素への参照を必要以上に保持することができ、すでに破棄されて表示されなくなったUIオブジェクトのツリー全体のガベージコレクションを防ぎます。

この問題の自然な解決策は、ライフサイクルを持つ各UIオブジェクトに[CoroutineScope]オブジェクトを関連付け、このスコープのコンテキストですべてのコルーチンを作成することです。
簡単にするために、[MainScope()]ファクトリを使用できます。すべての子コルーチンに `Dispatchers.Main` と親ジョブを自動的に提供します。

例えば、Androidアプリケーションでは最初に `Activity` が _作成_ され、不要になったときやメモリを解放しなければならないときに _破棄_ されます。
自然な解決策は、 `Activity` のインスタンスに `CoroutineScope` のインスタンスを加えることです。

<!--- CLEAR -->

```kotlin
class MainActivity : AppCompatActivity() {
    private val scope = MainScope()

    override fun onDestroy() {
        super.onDestroy()
        scope.cancel()
    }

    fun asyncShowData() = scope.launch { // アクティビティのスコープを親としてUIコンテキストで呼び出されます
        // 実際の実装
    }

    suspend fun showIOData() {
        val data = withContext(Dispatchers.IO) {
            // バックグラウンドスレッドでデータを計算する
        }
        withContext(Dispatchers.Main) {
            // UIにデータを表示する
        }
    }
}
```

`MainActivity` の中から起動したコルーチンはすべてそのジョブを親として持ち、アクティビティが破棄されると直ちに取り消されます。

> Androidは、ライフサイクルを持つすべてのエンティティでコルーチンスコープをファーストパーティでサポートしていることに注意してください。
[対応するドキュメント](https://developer.android.com/topic/libraries/architecture/coroutines#lifecyclescope)を参照してください。

ジョブ間の親子関係は階層を形成します。
アクティビティに代わってバックグラウンドジョブを実行するコルーチンは、さらに子コルーチンを作成できます。
親ジョブがキャンセルされると、コルーチンのツリー全体がキャンセルされます。
その例は、コルーチンのガイドの[「コルーチンの子」](../docs/coroutine-context-and-dispatchers.md#コルーチンの子)セクションに示されています。

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

### ディスパッチせずにUIイベントハンドラーでコルーチンを開始する

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
`kotlinx.coroutines` のすべてのUIディスパッチャーはこのように実装されています。
なぜでしょうか？

基本的にここでの選択は、「JSスタイル」の非同期アプローチ（非同期アクションは常にイベントディスパッチスレッドで後から実行されるように延期されます）と「C#スタイル」アプローチ（非同期アクションは、最初の中断ポイントまで呼び出し元スレッドで実行されます）の間です。

C#のアプローチはより効率的であるように見えますが、「必要な場合は `yield` を使用する…」などの推奨事項があります。
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
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[MainScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-scope.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[CoroutineStart]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/index.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineStart.UNDISPATCHED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-u-n-d-i-s-p-a-t-c-h-e-d.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[SendChannel.offer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/offer.html
[SendChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/index.html
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[Channel.CONFLATED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/-c-o-n-f-l-a-t-e-d.html
<!--- MODULE kotlinx-coroutines-javafx -->
<!--- INDEX kotlinx.coroutines.javafx -->
[kotlinx.coroutines.Dispatchers.JavaFx]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-javafx/kotlinx.coroutines.javafx/kotlinx.coroutines.-dispatchers/-java-fx.html
<!--- MODULE kotlinx-coroutines-android -->
<!--- INDEX kotlinx.coroutines.android -->
<!--- END -->
