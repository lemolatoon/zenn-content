---
title: "GitでEmpty Commitを最初に作り忘れたときに見る記事"
emoji: "️🍃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---
## 勢いでプロジェクトを開始！してコミットしてしまった！
適当に`git init`してプロジェクトをガッと進めて`commit`をしたときに、最初に空コミット（`Empty Commit`）し忘れたことに気づく......
そんなこと、よくありますよね。
```bash:気づいたらEmpty Commitなしで３コミットもしてしまっている様子
$ git log
commit f8f7f49095630ea0ab67df06019b4de0cec63c53 (HEAD -> master, origin/master)
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:37:40 2024 +0900

    Add README

commit d75c6119c285c2483e0a57702836557371e770b8
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:37:29 2024 +0900

    Add LICENSE

commit 1f593873e6dbeb7dfd1ea85c337cb920cc86d8a0
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:24:09 2024 +0900

    Add BasicDiskManager
```
## 解決法
```
$ git commit --allow-empty -m "Init project"
$ git rebase -i --root
```
するとエディタが立ち上がります。
```git
pick 1f59387 Add BasicDiskManager
pick d75c611 Add LICENSE
pick f8f7f49 Add README
pick 3f19685 Init project # empty
```
先ほど追加した`Empty Commit`を上にもっていきます。
```git
pick 3f19685 Init project # empty
pick 1f59387 Add BasicDiskManager
pick d75c611 Add LICENSE
pick f8f7f49 Add README
```
この状態で保存して閉じましょう。（Vimなら`:wq`）
晴れてコミットの最初に`Empty Commit`を追加できました。
```bash
$ git log
commit 9e90d149863d15f110122242e96c7ee4701bb0ac (HEAD -> master, origin/master)
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:37:40 2024 +0900

    Add README

commit a8ad0fac0830c9329bf6179ca79f9572d979cc04
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:37:29 2024 +0900

    Add LICENSE

commit 45017721c93e7160feb291739577b972a866423b
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:24:09 2024 +0900

    Add BasicDiskManager

commit 3f196851458624e97b8f70936aaf6ce84316d98f
Author: lemolatoon <63438515+lemolatoon@users.noreply.github.com>
Date:   Thu Jun 20 00:46:21 2024 +0900

    Init project
```
