---
title: "Java の VirtualThread は `java.net.Socket` をノンブロッキングにしてくれるのか"
emoji: "☕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java"]
published: true
---
## TL;DR;

ランタイムを Java 21 にアップデートし、処理を VirtualThread 上でおこなうようにするだけで、\
JDBC など、従来 OS スレッドをブロックしていたネットワークソケットを扱う I/O 処理が、ノンブロッキングに扱えるようになります。

## 背景

Java 19 から、VirtualThread (軽量スレッド)が preview 機能として追加され、
Java 21 より正式に GA となりました。
VirtualThread の導入によって、Java にも N:M モデルの非同期処理ランタイムがはいることになります。

ただ結局、システムコールレベルでブロッキング I/O な関数が実行されている場合、どちらにしても OS スレッドをブロックしてしまうのでスループットがでなくなってしまうのではと思っていました。

しかし、JEP を読んでいたところ、気になる記述がありました。

https://openjdk.org/jeps/444

> When code running in a virtual thread calls a blocking I/O operation in the java.* API, the runtime performs a non-blocking OS call and automatically suspends the virtual thread until it can be resumed later.

> 仮想スレッドで動作するコードがjava.* APIでブロッキングI/Oオペレーションを呼び出すと、ランタイムはノンブロッキングOSコールを実行し、後で再開できるまで仮想スレッドを自動的にサスペンドさせます。

VirtualThread にすることで、どうしてシステムコールレベルでブロッキング I/O をあつかっていても、OS スレッドをブロックしなくなるの？と疑問が生じ、調査しました。


## 用語

### Java の Thread

従来の `java.lang.Thread` は、OS のネイティブスレッドと 1:1 で紐づく。

そのスレッド内で待機処理がはいると、ネイティブスレッドごと待機してしまう。

VirtualThread によって、ユーザーランドの待機処理が入った場合、その処理を中断し、ネイティブスレッドに別の処理を割り当てて実行することができる。\
(実スレッドの処理(タスク)の割り当て制御は、ForkJoin のスレッドプールでおこなわれているとのこと)

### ブロッキングI/O, ノンブロッキングI/O

ここでは、`O_NONBLOCK` が設定されたファイル記述子(file descriptor)への、read, write による入出力(I/O)を**ノンブロッキング I/O**、
設定されていない状態での入出力を**ブロッキング I/O** と呼ぶ。

とくに、レガシーな API だが JDBC 等でもよく使われる、`java.net.Socket` による入出力時の挙動について確認する。

`mysql-connector-j` で使われている様子
- https://github.com/mysql/mysql-connector-j/blob/8.1.0/src/main/protocol-impl/java/com/mysql/cj/protocol/a/NativeSocketConnection.java#L62-L63
- https://github.com/mysql/mysql-connector-j/blob/8.1.0/src/main/core-api/java/com/mysql/cj/conf/PropertyDefinitions.java#L294-L295
- https://github.com/mysql/mysql-connector-j/blob/8.1.0/src/main/core-impl/java/com/mysql/cj/protocol/StandardSocketFactory.java#L77-L78


## 調査

下記のバージョンのコードを確認しました。

- jdk17u 17.0.8.1
  - https://github.com/openjdk/jdk17u/tree/jdk17.0.8.1
- jdk21u jdk-21+35
  - https://github.com/openjdk/jdk21u/tree/jdk-21%2B35

結論から言うと、とくに Legacy Socket API (`java.net.Socket`, `java.net.ServerSocket` など)の実装は、下記の JEP で再実装されていて、Java 13 よりシステムコールレベルでもブロッキング I/O ではなくなっていました。

https://openjdk.org/jeps/353

`java.net.Socket` の実体(実際の入出力)は、`java.net.SocketImpl` に実装されており、`SocketImpl.createPlatformSocketImpl` でインスタンスを取得しています。

Java 17 では、下記のようにパラメータによって実体が異なっており、デフォルトでは `PlainSocketImpl` が利用されるようです。(Java 13 から)

:::details jdk17u コードトレース
- https://github.com/openjdk/jdk17u/blob/jdk17.0.8.1/src/java.base/share/classes/java/net/SocketImpl.java#L79-L84
- https://docs.oracle.com/javase/jp/17/docs/api/java.base/java/net/SocketImpl.html
:::

`PlainSocketImpl` を追っていくと、例えば read では最終的に JNI を通してネイティブコードが実行されますが、そのコード内で待機するようになっています。\
これによって、実行時のスレッドで待機する形になり、結果としてスレッドをブロックしそうです。

:::details jdk17u コードトレース
- https://github.com/openjdk/jdk17u/blob/jdk17.0.8.1/src/java.base/share/classes/java/net/AbstractPlainSocketImpl.java#L596
- https://github.com/openjdk/jdk17u/blob/jdk17.0.8.1/src/java.base/share/classes/java/net/SocketInputStream.java#L165
- https://github.com/openjdk/jdk17u/blob/jdk17.0.8.1/src/java.base/share/classes/java/net/SocketInputStream.java#L90
- https://github.com/openjdk/jdk17u/blob/jdk17.0.8.1/src/java.base/unix/native/libnet/SocketInputStream.c#L127-L135
:::

Java 21 では、`sun.nio.ch.NioSocketImpl` のみがつかわれるようになり、ソケットの fd (ファイル記述子、file descriptor)を non-blocking モードに設定し、read, write を取り扱っています。\
read では最終的に、fd を epoll (デフォルト)に登録し、read ready となるまで VirtualThread.park を使い、実スレッドに別のタスクを割り当てれるように待機宣言しています。

:::details jdk21u コードトレース
- https://github.com/openjdk/jdk21u/blob/768592a52fb36e66da77e7642f2a3c3cb2849e2d/src/java.base/share/classes/java/net/SocketImpl.java#L52
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/sun/nio/ch/NioSocketImpl.java#L787
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/sun/nio/ch/NioSocketImpl.java#L274-L281
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/sun/nio/ch/NioSocketImpl.java#L172-L175
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/sun/nio/ch/Poller.java#L82
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/sun/nio/ch/Poller.java#L136-L139
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/java/util/concurrent/locks/LockSupport.java#L266-L267
- https://github.com/openjdk/jdk21u/blob/jdk-21%2B35/src/java.base/share/classes/jdk/internal/misc/VirtualThreads.java#L66-L67
:::

これによって、VirtualThread 上では、`java.net.Socket` から得られる入出力ストリームの read, write operation をおこなってもノンブロッキングに扱われ、ネイティブスレッドをブロックせず他のタスクに処理を割り当てることが可能になっているようです。

`java.net.Socket` のインターフェースは全く変わっていないため、Java を 21 以上にアップデートし、Socket から取得できる入出力ストリームを扱うスレッドを VirtualThread にするだけで、これまでブロッキング I/O だった処理をノンブロッキング I/O として処理できるようになっていることがわかりました。


## まとめ

調査にあった通り、Java の Project Loom を通して、`java.io` や `java.net` などの古い API も多くの改善が入り、これまでカーネルでの入出力待機があったところも、ユーザーランドでの待機処理に変更されていました。

そこに VirtualThread が入ることにより、そこで扱われていた待機タスクをネイティブスレッドから切り離し、別のタスクを割り当てることで、ネイティブスレッドをブロックせず、少ないスレッドで多くのスループットを出すことが可能になりました。

多くの JDBC 実装など、これまで同期的なインターフェースだったためにブロッキング I/O となっていた処理も、Java のアップデートをおこなうだけでノンブロッキングに扱うことができるようになり、多くのアプリケーションに恩恵があると考えられます。

### 発展

- 実際に検証してこれらを確かめられていない。検証する
- VirtualThread と実スレッドのタスクの切り替え回りの処理を確認できていない。確認する
