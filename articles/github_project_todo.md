---
title: "github ToDo"
emoji: "⛳"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [github,ToDo,Workflow]
published: false
---

お疲れ様です、波浪です
さて2025年が始まりましたね
新しい年といえば新しいカレンダー、新しいToDo管理です。

波浪が思う最強のToDo管理は飽きたら次のToDo管理をするってことですね。
なんだっけ？仕事は楽しいかね？でしたっけ？どんだけ優れた方法でも人間はどうせ飽きてやらなくなるが
逆に新しいことをやり続ける限り生産性はあがり続けるって話が書いてあったの

まあそういうわけで最強のToDo管理は飽きる前に次のToDo管理に切り替えるってことだと思ってます。

そんなわけで去年まではクアデルノで管理してましたが
今年はGitHubでToDo管理してみることにしました

いわゆるProject看板方式によるToDo管理ですね。

ちなみに自分はまあまあの期間エンジニアしてるのでCSVもSubVersionもVisualSourceSafeもなんならカゲマイだって使ったことあるけどGitHubではissueやProjectを使ったことがありません。
なのでこの機会にもうちょっとくらいGitHubになれておきたいなっていう小賢しさも発揮しています。

手順は以下の通り

1. 雑にリポジトリを作る
2. Projectを作る
3. Projectを編集して使いやすくする
4. issue追加と同時にProjectに入るようWorkflow設定
5. 実際にやってみる

## 1. 雑にリポジトリを作る

https://github.com/new

リポジトリ名はToDoでもTaskでもSampleでも好きにつけてください。

![](/images/github_project_todo/create_new_repository.png)

## 2. Projectを作る
リポジトリのProjectから作成します。

![](/images/github_project_todo/create_project.png)

プロジェクト名も適当につけておkです。

## 3. Projectを編集して使いやすくする

Setting -> Status

![](/images/github_project_todo/setting.png)
![](/images/github_project_todo/status.png)

TODO管理に
BacklogとかReviewはないので編集しておきます。

![](/images/github_project_todo/edit.png)
![](/images/github_project_todo/modify.png)

## 4. issue追加と同時にProjectに入るようWorkflow設定

Project画面の右上からWorkflowを選択します
![](/images/github_project_todo/workflow.png)


使うのはナビゲーションメニューの
"Item added to project"
と
"Auto-add to project"
の二つです。

まずはAuto-add to Projectから

![](/images/github_project_todo/autoadd-to-project.png)

右上のEditボタンを押下して
When the filter matches a new or updated item のFilterに
1で雑に作ったリポジトリと
`is:issue is:open`
を設定してSave and turn workflow

次に "Item added to project"でEditを選んで
画像のように"issue"が入ったら"やること"に入るよう設定して右上のSave and turn workflow ボタンを押下
![](/images/github_project_todo/edit_workflow.png)


## 5. 実際にやってみる

1.で雑に作ったリポジトリのissueを適当に追加します

![](/images/github_project_todo/new_issue.png)

はい、Projectの"やること"に入りましたね。

![](/images/github_project_todo/complete.png)

以上、よろしくお願いいたします。




### 参考

https://www.amazon.co.jp/dp/4877710787/

https://zenn.dev/unsoluble_sugar/articles/75e65267907956


