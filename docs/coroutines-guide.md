言語としてのKotlinは、他の様々なライブラリがコルーチンを利用できるようにするために標準ライブラリに最小限の低レベルAPIしか提供していません。 同様の機能を持つ他の多くの言語とは異なり、 `async` と `await` はKotlinのキーワードではなく、標準ライブラリの一部でもありません。
さらに、Kotlinの _サスペンド関数_ の概念は、フューチャーやプロミスよりも非同期操作のための、より安全で誤りの少ない抽象化を提供します。

`kotlinx.coroutines` はJetBrainsによって開発されたコルーチン用の豊富なライブラリです。
これには、 `launch` 、 `async` などを含む、このガイドで扱う高水準のコルーチンを可能にするプリミティブが含まれています。

これは、 `kotlinx.coroutines` のコア機能に関するガイドであり、さまざまなトピックに分かれた一連の例が含まれています。

このガイドの例だけでなくコルーチンを使用するには、[プロジェクトのREADME](../README.md#using-in-your-projects)で説明されているように、 `kotlinx-coroutines-core` モジュールの依存関係を追加する必要があります。

## 目次

* [コルーチンの基礎](basics.md)
* [キャンセルとタイムアウト](cancellation-and-timeouts.md)
* [サスペンド関数の作成](composing-suspending-functions.md)
* [コルーチンコンテキストとディスパッチャー](coroutine-context-and-dispatchers.md)
* [例外処理](exception-handling.md)
* [チャネル（実験的）](channels.md)
* [共有ミュータブルステートと並列処理](shared-mutable-state-and-concurrency.md)
* [セレクト式（実験的）](select-expression.md)

## Additional references

* [コルーチンによるUIプログラミングガイド](../ui/coroutines-guide-ui.md)
* [コルーチンによるリアクティブストリームのガイド](../reactive/coroutines-guide-reactive.md)
* [コルーチン設計文書 (KEEP)](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
* [完全なkotlinx.coroutines APIリファレンス](http://kotlin.github.io/kotlinx.coroutines)
