---
layout: post
title: "OSX 10.10 + El-get + Flycheck = Error!!"
date: 2015-03-27
categories:
- Emacs
---

最近Emacsの挙動が少し怪しくなってきていた。心当たりはすこしあってEmacsのバージョンを上げた際に面倒くさくて全てのプラグインをバイトコンパイルし直さなかった。一旦全てのパッケージを最新にアップデートしたかったこともあり、el-get-dir以下を全て削除してインストールし直した。

```sh
doc/flycheck.texi:6: 警告: unrecognized encoding name `UTF-8'.
doc/flycheck.texi:86: @pxref expected braces.
doc/flycheck.texi:86: ` {Installation}
and @ref{Quickstart} res...' is too long for expansion; not expanded.
doc/flycheck.texi:86: First argument to cross-reference may not be empty.
./doc/flycheck.texi:75: 次 reference to nonexistent node `Introduction' (perhaps incorrect sectioning?).
./doc/flycheck.texi:85: 相互 reference to nonexistent node `Introduction' (perhaps incorrect sectioning?).
makeinfo: エラーにより、出力ファイル `doc/flycheck.info' を削除します。
       -- 残したい場合には `--force' オプションを使ってください。
```
