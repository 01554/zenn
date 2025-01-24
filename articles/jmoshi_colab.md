---
title: "J-moshiをGoogleColab(L4)で動かす"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [colab,jmoshi,j_moshi,j-moshi]
published: false
---

お疲れ様です、波浪です。

https://x.com/atsumoto_ohashi/status/1882633871176630595

なめらかに会話できるAIとして社内でちょっと話題になっていたj-moshiですが
おもろそうじゃんって思ったんで試してみました


リポジトリ類はここですね
https://github.com/nu-dialogue/j-moshi

https://huggingface.co/nu-dialogue/j-moshi-ext

親切に日本語解説が書いてあるので
ざっと見てみるとめっちゃ簡単に実行できそう

```
pip install moshi
python -m moshi.server --hf-repo nu-dialogue/j-moshi-ext

```
のたった2Stepみたいです
ただ
```
実行には，24GB以上のVRAMを搭載したLinux GPUマシンが必要です．MacOSには対応していません．
```

とのことだったので家のGPUマシン使おうとしたら嫁さんが三國無双オリジンしてたんで
Colabで実行してみました。

T4だとVRAM足りなそうなのでL4使います（L4は有料プランでしか使えませんので注意）

あとColabだとGradioかなんかに繋がないといけないんですが

```
実装の詳細は，オリジナルMoshiのリポジトリ kyutai-labs/moshi を参照してください．
```
ここで指示されているmoshiみたら、そのものズバリ

https://github.com/kyutai-labs/moshi?tab=readme-ov-file#python-pytorch:~:text=Start%20the%20server%20with%3A

```
python -m moshi.server [--gradio-tunnel] [--hf-repo kyutai/moshika-pytorch-bf16]
```
って書いてありました、たすかるぅ〜

実行するとモデルDLに少々待たされたあとに
gradioのURLが発行されます
![](/images/jmoshi_colab/gradio.png)

実行した動画をつけたいところなんですが
zennに動画アップする方法がわからんのでgifだけはっときますね

。。。と思ったら上限の3MB超えてたんで すいませんがスクショだけ貼ります
![](/images/jmoshi_colab/microphone.png)
![](/images/jmoshi_colab/run.png)

まあ、動画なんてみなくても、やればめちゃくちゃ簡単に試せるので！！！
みんなもLet's j-moshi ！！！！！

