---
title: "Akka で Raft を実装してみた話"
emoji: "🪵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raft", "Scala", "Akka", "Pekko"]
published: true
---

これは Scala Advent Calendar 2023 の 10 日目の記事です。(空いていたので埋めました)

@[card](https://qiita.com/advent-calendar/2023/scala)

最近の分散データベースでよく聞く Raft について、その論文を読んでみたところ基本的な部分はシンプルで理解しやすく、メッセージを送りあい、ログをつかって自分の状態を変更していくステートマシンモデルであったため、「Akka でいい感じに実装できそうだな」と思い、やってみました。

## Raft とは

Raft は、分散システム上のプロセス間でログを複製し、それをつかった状態変更(遷移)の合意(コンセンサス)を保証してくれます。
同じ分散合意アルゴリズムである Paxos より理解しやすいように設計されており、コンセンサスのための要素を分けて定義することで、それぞれが比較的シンプルなものになっています。
理解しやすく、シンプルにすることで、実用的なシステムを構築し、特に運用していくためのより良い選択肢になることを目的としているようです。

詳しくは論文や、Raft の Web サイトを閲覧してください。

- [Raft paper](https://raft.github.io/raft.pdf)
- [The Raft Consensus Algorithm](https://raft.github.io/)

論文を読んだり実装するにあたり、下記の邦訳や考察を参考にさせてもらいました。

- [論文翻訳: In Search of an Understandable Consensus Algorithm (Extended Version)](https://hazm.at/mox/distributed-system/algorithm/transaction/raft/index.html)
- [分散合意アルゴリズム Raft を理解する](https://qiita.com/torao@github/items/5e2c0b7b0ea59b475cce)
- [NewSQLのコンポーネント詳解](https://qiita.com/tzkoba/items/3e875e5a6ccd99af332f)

最近の分散データベース、データストアや NewSQL と言われるようなデータベースでつかわれています。

- etcd
- Kafka(KRaft)
- TiDB, CockroachDB, YugabyteDB

## なぜ Akka で実装したのか

冒頭に記載した通り、メッセージを送り合い、そのステートを変更していくステートマシンの記述は、アクターモデルそのものなイメージがあり、どのように実装できるか試してみたかったことが大きいです。
Raft は、Leader, Follower, Candidate の三つの役割が定義されており、RequestVote, AppendEntires という RPC を介してステートの変更や状態遷移をおこないます。
アクターモデルはまさにそれらをメッセージとしてやり取りし、その内容に基づいて振る舞いを変更できるステートマシンの記述になるため、自然に実装できるのではと考えました。

またAkka には AkkaCluster ですでに、ゴシッププロトコルベースのクラスタリング機能(モジュール)がありますが、別の分散処理の仕組みがあっても良さそうだとも思いました。

実装時期の都合上、[Apache Pekko](https://pekko.apache.org/) は使っていませんが、Akka 2.6 (ApacheLicense) を使っているので、import や config 周りを調整するだけで比較的簡単に移行できるのでは？と思っています。 

## 実装してみた

Raft は複数の要素からなりたっており、今回はその根幹となる、下記二つを実装してみました。
- Leader Election
- Log Replication

Log Compaction や Membership Changes も、実用上では必須になりますが、Raft の仕組みを知る上ではとりあえず上記だけでよさそうです。

実装したコードはこちらです。

@[card](https://github.com/hayasshi/akka-raft)

README がなかったり、バージョンが諸々古くなってしまっていますがご容赦ください🙏
また、個人的な習熟度の関係で、Classic Actor で実装しています。

今回の実装は、プロセス内でのエミュレーションまでとなっており、`RaftTest` というテストを実行すると Raft ノードに相当するアクターが作成され、それらがやり取りをおこないます。
やり取りの様子は標準出力にロギングされます。

以下、実際の動作ログの一部を記載します。
※該当ログのみ抜粋
※見やすいように少し改変

:::details 最初の Leader Election のログ
```
...
[INFO] [12/25/2023 20:44:12.044] [akka://RaftTest/user/member4] Start raft member with Set(Actor[akka://RaftTest/user/member1#1214893844], Actor[akka://RaftTest/user/member2#882887653], Actor[akka://RaftTest/user/member3#2088407234], Actor[akka://RaftTest/user/member5#659633385])
[INFO] [12/25/2023 20:44:15.708] [akka://RaftTest/user/member4] Follower election timeout
[INFO] [12/25/2023 20:44:15.735] [akka://RaftTest/user/member1] Follower RequestVoteRPC receive RequestVote(1,Actor[akka://RaftTest/user/member4#915776548],0,0)
...
[INFO] [12/25/2023 20:44:15.736] [akka://RaftTest/user/member4] Candidate start election: currentTerm=1, requestedMembers=HashMap(Actor[akka://RaftTest/user/member4#915776548] -> 1, Actor[akka://RaftTest/user/member5#659633385] -> 0, Actor[akka://RaftTest/user/member3#2088407234] -> 0, Actor[akka://RaftTest/user/member2#882887653] -> 0, Actor[akka://RaftTest/user/member1#1214893844] -> 0)
[INFO] [12/25/2023 20:44:15.743] [akka://RaftTest/user/member1] Follower RequestVoteRPC return RequestVoteResult(1,true)
...
[INFO] [12/25/2023 20:44:15.745] [akka://RaftTest/user/member4] Candidate receive voted result: term=1
[INFO] [12/25/2023 20:44:15.748] [akka://RaftTest/user/member4] Candidate receive voted result: term=1
[INFO] [12/25/2023 20:44:15.767] [akka://RaftTest/user/member4] Candidate become to leader: nextIndex=Map(Actor[akka://RaftTest/user/member1#1214893844] -> 1, Actor[akka://RaftTest/user/member2#882887653] -> 1, Actor[akka://RaftTest/user/member3#2088407234] -> 1, Actor[akka://RaftTest/user/member5#659633385] -> 1), matchIndex=Map(Actor[akka://RaftTest/user/member1#1214893844] -> 0, Actor[akka://RaftTest/user/member2#882887653] -> 0, Actor[akka://RaftTest/user/member3#2088407234] -> 0, Actor[akka://RaftTest/user/member5#659633385] -> 0)
...
```
:::

:::details heartbeat ~ Log Replication ~ heatbeat(apply state) のログ
```
// heartbeat (他の member へも同様のログ)
[INFO] [12/25/2023 20:44:21.854] [akka://RaftTest/user/member4] Leader send heartbeat to Actor[akka://RaftTest/user/member5#659633385]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(),0)
[INFO] [12/25/2023 20:44:21.855] [akka://RaftTest/user/member5] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(),0)
[INFO] [12/25/2023 20:44:21.855] [akka://RaftTest/user/member5] Follower AppendEntriesRPC return true: commitIndex=0, lastApplied=0, state=0
[INFO] [12/25/2023 20:44:21.855] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(),0))
...
// Log Replication
[INFO] [12/25/2023 20:44:22.026] [akka://RaftTest/user/member3] Follower redirect command: Command(2), leader=Some(Actor[akka://RaftTest/user/member4#915776548])
[INFO] [12/25/2023 20:44:22.041] [akka://RaftTest/user/member4] Leader receive command from client: Log(1,1,2)

[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member4] Leader send AppendEntries log to Actor[akka://RaftTest/user/member1#1214893844]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member1] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member1] Follower AppendEntriesRPC return true: commitIndex=0, lastApplied=0, state=0

[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member4] Leader send AppendEntries log to Actor[akka://RaftTest/user/member2#882887653]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member2] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member2] Follower AppendEntriesRPC return true: commitIndex=0, lastApplied=0, state=0

[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member4] Leader send AppendEntries log to Actor[akka://RaftTest/user/member3#2088407234]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member3] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member3] Follower AppendEntriesRPC return true: commitIndex=0, lastApplied=0, state=0

[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member4] Leader send AppendEntries log to Actor[akka://RaftTest/user/member5#659633385]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member5] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0)
[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member5] Follower AppendEntriesRPC return true: commitIndex=0, lastApplied=0, state=0

[INFO] [12/25/2023 20:44:22.042] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0))
[INFO] [12/25/2023 20:44:22.043] [akka.actor.ActorSystemImpl(RaftTest)] Test receive Accepted(Command(2))
[INFO] [12/25/2023 20:44:22.044] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0))
[INFO] [12/25/2023 20:44:22.044] [akka://RaftTest/user/member4] Leader commit logs: commitIndex=1, lastApplied=1, state=2
[INFO] [12/25/2023 20:44:22.044] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0))
[INFO] [12/25/2023 20:44:22.044] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],0,0,Vector(Log(1,1,2)),0))
...
// heartbeat (他の member へも同様のログ)
[INFO] [12/25/2023 20:44:23.553] [akka://RaftTest/user/member4] Leader send heartbeat to Actor[akka://RaftTest/user/member1#1214893844]: AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],1,1,Vector(),1)
[INFO] [12/25/2023 20:44:23.553] [akka://RaftTest/user/member1] Follower AppendEntriesRPC receive AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],1,1,Vector(),1)
[INFO] [12/25/2023 20:44:23.554] [akka://RaftTest/user/member1] Follower AppendEntriesRPC return true: commitIndex=1, lastApplied=1, state=2
[INFO] [12/25/2023 20:44:23.554] [akka://RaftTest/user/member4] Leader receive AppendEntriesResult: AppendEntriesResult(1,true,AppendEntries(1,Actor[akka://RaftTest/user/member4#915776548],1,1,Vector(),1))
...
```
:::

:::details Leader stop ~ Leader Election(再) のログ
```
[INFO] [12/25/2023 20:44:24.029] [akka://RaftTest/user/member1] Follower redirect command: Command(0), leader=Some(Actor[akka://RaftTest/user/member4#915776548])
[INFO] [12/25/2023 20:44:24.030] [akka.actor.ActorSystemImpl(RaftTest)] Stop leader: Actor[akka://RaftTest/user/member4#915776548]
[INFO] [12/25/2023 20:44:26.773] [akka://RaftTest/user/member3] Follower election timeout
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member3] Candidate start election: currentTerm=2, requestedMembers=HashMap(Actor[akka://RaftTest/user/member4#915776548] -> 0, Actor[akka://RaftTest/user/member5#659633385] -> 0, Actor[akka://RaftTest/user/member3#2088407234] -> 1, Actor[akka://RaftTest/user/member2#882887653] -> 0, Actor[akka://RaftTest/user/member1#1214893844] -> 0)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member5] Follower RequestVoteRPC receive RequestVote(2,Actor[akka://RaftTest/user/member3#2088407234],1,1)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member5] Follower RequestVoteRPC return RequestVoteResult(2,true)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member2] Follower RequestVoteRPC receive RequestVote(2,Actor[akka://RaftTest/user/member3#2088407234],1,1)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member2] Follower RequestVoteRPC return RequestVoteResult(2,true)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member1] Follower RequestVoteRPC receive RequestVote(2,Actor[akka://RaftTest/user/member3#2088407234],1,1)
[INFO] [12/25/2023 20:44:26.775] [akka://RaftTest/user/member1] Follower RequestVoteRPC return RequestVoteResult(2,true)
[INFO] [12/25/2023 20:44:26.776] [akka://RaftTest/user/member3] Candidate receive voted result: term=2
[INFO] [12/25/2023 20:44:26.776] [akka://RaftTest/user/member3] Candidate receive voted result: term=2
[INFO] [12/25/2023 20:44:26.776] [akka://RaftTest/user/member3] Candidate become to leader: nextIndex=Map(Actor[akka://RaftTest/user/member1#1214893844] -> 2, Actor[akka://RaftTest/user/member2#882887653] -> 2, Actor[akka://RaftTest/user/member4#915776548] -> 2, Actor[akka://RaftTest/user/member5#659633385] -> 2), matchIndex=Map(Actor[akka://RaftTest/user/member1#1214893844] -> 1, Actor[akka://RaftTest/user/member2#882887653] -> 1, Actor[akka://RaftTest/user/member4#915776548] -> 1, Actor[akka://RaftTest/user/member5#659633385] -> 1)
[INFO] [12/25/2023 20:44:26.777] [akka://RaftTest/user/member3] Leader send heartbeat to Actor[akka://RaftTest/user/member1#1214893844]: AppendEntries(2,Actor[akka://RaftTest/user/member3#2088407234],1,1,Vector(),1)
[INFO] [12/25/2023 20:44:26.777] [akka://RaftTest/user/member3] Leader send heartbeat to Actor[akka://RaftTest/user/member2#882887653]: AppendEntries(2,Actor[akka://RaftTest/user/member3#2088407234],1,1,Vector(),1)
...
```
:::

## 感想

### やはりステートマシンの実装は Akka はやりやすい

そのためのアクターモデルです、感がありました。

Leader, Follower, Candidate それぞれの振る舞い定義を分けて記述し、順次処理のメッセージベースで状態や内部ステートを安全に変更しやすく、コードの見通しがよいように感じます。
Raft はタイムアウトを契機に状態変化をおこすことが多いのですが、それもスケジューラと自身へのメッセージングですんなり実装できました。

またこのまま AkkaRemote をつかい、プロセス間のやりとりもこのコードのまま(もしくはタイムアウト値の調整)で済みそうで、アクターの位置透過性の特性を活用できそうです。(デバッグがしやすい)

### State 更新のタイミング

実装を通して疑問を解消することもできました。

Log Replication において、Leader からの AppendEntries リクエストに対し、Follower は条件チェックの後にログを追加してレスポンスを返します。この時点でのログは、Leader によってコミットされない可能性もあるため、ステートに apply できません。
ではどのタイミングで apply するのかと思っていたら、ログ追加の AppendEntries リクエストの後におこなわれる、ハートビートまたは別のログ追加のための AppendEntries 時に apply されることが、今回の実装を通して理解できました。

Leader は、各 Follower がどこまでログを持っているかを管理しており、AppendEntries リクエスト時に最新までの差分ログの情報と、leader はここまでコミットしているという情報も送っています。
Follower は非同期に、その Leader のコミットログ番号までを State に apply する形になります。これによって、あるログを追加した後のリクエストで Leader のコミット番号が進んでいれば適用することが理解できました。

ただしこれは、**Leader が Client からのリクエストに応える**という論文の前提においてになるので、Follower からもデータを参照するような要求がある場合は、非同期に適用される部分に注意する必要がありそうです。

### Membership管理

今回は Leader Election と Log Replication の実装にとどまりましたが、ノードがクラスタを作り分散処理するためには **Membership 管理** が重要です(分散処理の核)。
論文には、Membership (クラスタ)変更によるログの移行についての記述はありますが、そもそもどのように Membership を管理するかについての言及はありませんでした。

最初に思いつくのが、Membership 管理自体も Raft でおこない、その Membership をつかって他の Raft クラスタをつくるというものです。
また Akka をつかっている場合、Membership 管理は AkkaCluster でおこない、その上で Raft をつかってソフトウェアを実装することも可能ではと思いました。
(TIS さんの [lerna-stack/akka-entity-replication](https://github.com/lerna-stack/akka-entity-replication) がそれに相当するのかも)

## まとめ

- やはり Akka/Pekko などアクターモデルと Raft のステートマシン記述は相性が良かった
- 実装してみることで、理解が曖昧な部分についてより明確にイメージをつけることができた
    - 「こういうことができるのではないか」の詳細度をより高められる

## 実装の参考にしたもの

今回実装するにあたり、下記の実装や情報も参考にしました。

@[card](https://github.com/hashicorp/raft)

@[card](https://github.com/ktoso/akka-raft)

@[card](https://chat.openai.com/)
