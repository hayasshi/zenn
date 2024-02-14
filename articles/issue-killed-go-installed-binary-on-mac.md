---
title: "`go install` したバイナリが起動時に KILL される"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "mac"]
published: true
---

## 問題

下記の環境で起きた問題です。

Apple M2 Pro
macOS Ventura
```
$ sw_vers
ProductName:		macOS
ProductVersion:		13.6.4
BuildVersion:		22G513
```

macOS 上に Golang の環境を整えた際、`go install` で取得したいくつかのバイナリが、起動時に KILL される現象に遭遇しました。

```
$ gopls version
[1]    32528 killed     gopls version
```

## 調査

`log` コマンドで OS ログを確認したところ、下記のようなログが出ていました。

```
$ log show --pager  # log 量が多いので `--start` オプションで量を絞った方が良い
...
// gopls で検索
kernel: (AppleMobileFileIntegrity) AMFI: '/gopath/bin/gopls' has no CMS blob?
kernel: (AppleMobileFileIntegrity) AMFI: '/gopath/bin/gopls': Unrecoverable CT signature issue, bailing out.
kernel: (AppleMobileFileIntegrity) AMFI: code signature validation failed.
syspolicyd: (Security) [com.apple.securityd:security_exception] MacOS error: -67062
...
gopls: (libsystem_info.dylib) Retrieve User by ID
kernel: CODE SIGNING: cs_invalid_page(0x104b00000): p=32528[gopls] final status 0x23020200, denying page sending SIGKILL
kernel: CODE SIGNING: process 32528[gopls]: rejecting invalid page at address 0x104b00000 from offset 0x4000 in file "/gopath/bin/gopls" ...
kernel: CODE SIGNING: process 32528[gopls]: rejecting invalid page at address 0x104b00000 from offset 0x4000 in file "/gopath/bin/gopls" ...
kernel: Text page corruption detected for pid 32528
kernel: Text page corruption detected in dying process 32528
...
```

調べてみたところ、コード署名に問題があり、macOS のシステム整合性保護(SIP)機能によって検出され、SIGKILL されたようです。\
Golang のビルド時にコード署名の問題がある(あった？)といった GitHub issue を見つけました。

@[card](https://github.com/golang/go/issues/42684)

ざっと読んだ感じ、Apple のコード署名まわりの変更により、Golang のクロスコンパイル時のコード署名に設定が必要になり、\
いくつかのバイナリは対応できていないのではと推察しています。


## Workaround

上記の GitHub issue に記載されている通り、該当のバイナリを `codesign` で自己署名すると動作するようになりました。

```
$ codesign -s - /gopath/bin/gopls


$ gopls version
golang.org/x/tools/gopls v0.14.2
    golang.org/x/tools/gopls@v0.14.2 h1:sIw6vjZiuQ9S7s0auUUkHlWgsCkKZFWDHmrge8LYsnc=
```

根本的には、バイナリ提供者が対応したビルドをして貰う必要があるかと思われます。
