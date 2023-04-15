---
title: "Java の Virtual thread は `java.io` パッケージをノンブロッキングにしてくれるのか"
emoji: "☕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java"]
published: false
---
## TL;DR;

WIP です。特に調査内容を整理します。

JDBC など、従来 OS スレッドをブロックしていたネットワークソケットを扱う処理が、Virtual thread をつかえば、OS スレッドをブロックしなくなります。

## 背景

Java 19 から、Virtual thread (軽量スレッド)が preview 機能として追加されました。
Virtual thread の導入によって、Java にも N:M モデルの非同期処理ランタイムがはいることになります。

ただ結局、システムコールレベルでブロッキング I/O な関数が実行されている場合、どちらにしても OS スレッドをブロックしてしまうのでスループットがでなくなってしまうのではと思っていました。

しかし、JEP を読んでいたところ、気になる記述がありました。

https://openjdk.org/jeps/444

> When code running in a virtual thread calls a blocking I/O operation in the java.* API, the runtime performs a non-blocking OS call and automatically suspends the virtual thread until it can be resumed later.

> 仮想スレッドで動作するコードがjava.* APIでブロッキングI/Oオペレーションを呼び出すと、ランタイムはノンブロッキングOSコールを実行し、後で再開できるまで仮想スレッドを自動的にサスペンドさせます。

Virtual thread にすることで、どうしてシステムコールレベルでブロッキング I/O をあつかっていても、OS スレッドをブロックしなくなるの？と疑問が生じ、調査しました。


## 用語

//TODO 下記あたりをまとめます。

### Java の Thread

OSスレッド、プラットフォームスレッド、java.lang.Thread

### フロッキングI/O, ノンブロッキングI/O

open, read, write
select, poll, epoll
aio_read, aio_write


## 調査

//TODO 調べた箇所、気になったポイントを纏めます。
// LTS である JDK17 と、最新のリリースである JDK20 のベースで比較します。

https://github.com/openjdk/jdk20u
https://github.com/openjdk/jdk17u
https://github.com/openjdk/loom

Virtual thread の実装
InputStream の実装 (Socket, File)


## 結果

実は、とくに Legacy Socket API (`java.net.Socket`, `java.net.ServerSocket` など)の実装は、下記の JEP で再実装されていて、Java 13 よりシステムコールレベルでもブロッキング I/O ではなくなっていました。
呼び出し箇所で、Java 側で待機するような形に変わっていました。

https://openjdk.org/jeps/353

これにより、JDBC 等の TCP 接続で用いられている Socket からの InputStream, OutputStream を Virtual thread で扱うと、OS スレッドをブロックせずに扱えるようになります。
またこれらは、Java をアップデート(して、Socket を扱う実行スレッドを Virtual thread に)するだけでその恩恵を受けれるようになります。

GA となる Java 21 (しかも LTS!) がすごく楽しみですね。
