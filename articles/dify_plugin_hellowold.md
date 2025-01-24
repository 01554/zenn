---
title: "Difyプラグインの作り方：HelloWorldするよ"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dify,plguin,v1.0.0]
published: true
---

お疲れ様です波浪です。
今日はDifyPlugin開発の話です。

知らん人はまだ知らないと思いますが
Difyは今までのtoolがver1.0.0からプラグインって形になります
プラグインになるとDifyにPRださないでも新しいノードを追加できるようになるので
今のうちにちょっとさわっておくかーってことで今回触ってみます

といっても公式に沿って手順通りやるだけですけどね
https://docs.dify.ai/plugins/quick-start/develop-plugins/tool-plugin


##  Difyプラグイン スキャフォールディングツールのインストール
https://github.com/langgenius/dify-plugin-daemon/releases
僕はMacbookM1なので`dify-plugin-darwin-arm64`ですね

そのままだと実行できないので
https://support.apple.com/ja-jp/guide/mac-help/mh40616/mac
セキュリティを弱めます(この時点で正直やめたくなってる)

またダウンロードしたファイルは`/usr/local/bin`にコピーしたほうが便利らしいのでいれときます
```
~/app ❯ sudo cp ./dify-plugin-darwin-arm64 /usr/local/bin/dify    
Password:

~/app ❯ dify version    
v0.0.1
```

##  Difyプラグインツール起動

`dify plugin init`
を打ち込むと、カレントディレクトリ以下にプラグインの骨子が作られます
とりあえず名前何？とか何したいの？みたいな事聞かれるので適当に答えていきます

![](/images/dify_plugin_hellowold/step1.png)


最後のStepではプラグインで何をやるのかを設定します
ここでチェックをつけたものにしたがって作成されるファイルや用意しないといけないものが変わります。


```
プラグインのパーミッションを設定し、上下でナビゲートし、タブで選択し、選択後、Enterを押して終了します。
後方呼び出し：
ツール：
  → 有効： 有効: [✘] Dify内部でツールを呼び出すことができます。
モデル
    有効： 有効: [✘] Dify内でモデルを呼び出すことができます。
    LLM: [✘] LLMが有効であれば、Difyの中でLLMモデルを呼び出すことができます。
    テキストの埋め込み テキスト埋め込み: [✘] Difyでテキスト埋め込みが有効になっていれば、そのモデルを呼び出すことができる
    Rerank: [✘] リランクモデルが有効であれば、Difyの中で呼び出すことができます。
    TTS: [✘] Dify内部でTTSモデルを呼び出すことができます。
    Speech2Text： Speech2Text: [✘] DifyでSpeech2Textモデルを呼び出すことができます。
    モデレーション (✘) モデレーションが有効な場合、Dify内部でモデレーションモデルを呼び出すことができます。
アプリ
    有効です： [BasicChat/ChatFlow/Agent/Workflow などのアプリを呼び出すことができます。
リソース
ストレージ
    有効： [✘] プラグインの永続ストレージ
    サイズ： N/A ストレージの最大サイズ
エンドポイント：
    有効： [エンドポイントを登録する機能

※　DeepL.com（無料版）で翻訳しました。
```
プラグインの永続ストレージのなんてあるんですね。
LLMとか有効化するとそこらへんの設定をかかなきゃプラグインとして使えないので
HelloWorldしたいだけならやめときましょう。
今回はtool、アプリ、ストレージ、エンドポイントを有効化しておきます。


## 作成されるファイル群
以下のファイルが作成されます。
```
$ tree

app/dify-plugin/helloworld
|--.difyignore
|--.env.example
|--.gitignore
|--GUIDE.md
|--PRIVACY.md
|--README.md
|--_assets
| |--icon.svg
|--main.py
|--manifest.yaml
|--provider
| |--helloworld.py
| |--helloworld.yaml
|--requirements.txt
|--tools
| |--helloworld.py
| |--helloworld.yaml
```

`*.yaml`は
https://github.com/langgenius/dify/blob/da67916843249390ee75438207042b215e752183/api/core/tools/provider/builtin/arxiv/tools/arxiv_search.yaml#L7

以前はこんな感じで配置してたやつですね。
説明を追加するなら `ja_JP:` に続けて書いておくと日本語メニューでも表示されるのかな？

まあ今回はほっときましょう。


## コード部
```
from collections.abc import Generator
from typing import Any

from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage

class HelloworldTool(Tool):
    def _invoke(self, tool_parameters: dict[str, Any]) -> Generator[ToolInvokeMessage]:
        yield self.create_json_message({
            "result": "Hello, world!"
        })
```
デフォルトで上記の形になっていたのでこのままいきます。
何がきても、HelloWorldっていうだけですね。

## プラグインのインストール
さて、早速Difyにプラグインとして入れたいと思います。

Dify 1.0.0beta をDocker-compose で起動しまして
画面右上の
![](/images/dify_plugin_hellowold/debugging.png)

この虫マークを押して出てくるKeyを`.env`に設定します

``` .env
INSTALL_METHOD=remote
REMOTE_INSTALL_HOST=localhost
REMOTE_INSTALL_PORT=5003
REMOTE_INSTALL_KEY=********-****-****-****-************

```

設定ファイルができたら、main.py を叩くと勝手にインストールされます。


```
$ python3.12 -m venv .venv
$ source .venv/bin/activate   
$ pip install -r requirements.txt
$ python main.py
```
![](/images/dify_plugin_hellowold/install.gif)

インストール中はこんな感じでプラグインアイコンがうにょうにょします

しばらく待つと
![](/images/dify_plugin_hellowold/plugindebug.png)

入りましたね

## 使ってみる

![](/images/dify_plugin_hellowold/use.png)

![](/images/dify_plugin_hellowold/trace.png)


helloworldノードの出力にさっきの
```
{
    "result": "Hello, world!"
}
```
が入っていることが確認できました！！！やったぜ！！！！


### 実際のコード
https://github.com/01554/dify-plugin-helloworld