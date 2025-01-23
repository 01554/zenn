---
title: "DeepSeekはすごいけどoutput_tokenが少ない、出力量を増やすにはどうするか"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [DeepSeekR1,gpt]
published: true
---

お疲れ様です波浪です。
DeepSeek大流行りですね！

早速僕も使ってみたんですが、職場の人からは
「o1と違って出力が少なすぎる、4o-miniのほうがマシ」
って言われちゃったんで、outputを増やすにはどうしたらいいかなぁ？と考えてみました

で、こいつ、リファレンス見ると推論結果が取れるんですね
https://api-docs.deepseek.com/guides/reasoning_model

これをoutput_tokenは多いけどあんまり頭良くない子にいれたら
もしかして良い感じになるのでは？

と思ったんで、それの検証です。

コードは以下の通りです
https://colab.research.google.com/drive/1UYs9mA7sYjk00xY9GapKEdxqgEvRVyUX?usp=sharing


では解説。

の前に注意ですが、実行時はAPI_KEYを自力で用意してください
![](/images/deepseek_resoning_in_gpt/apikey.png)

Colabのシークレット領域にAPI_KEYを登録してください


## 解説
```
from openai import OpenAI
deepseek_client = OpenAI(api_key=userdata.get("DEEPSEEK_API_KEY")
, base_url="https://api.deepseek.com")

# Round 1
messages = [{"role": "user", "content": "How many Rs are there in the word 'strawberry'"}]
response = deepseek_client.chat.completions.create(
    model="deepseek-reasoner",
    messages=messages
)

reasoning_content = response.choices[0].message.reasoning_content
```

といっても、解説するほどのものはないんですけどね。
この`reasoning_content`が推論中の内容です。
これを別のGPTに食わせたらちゃんと頭の良いGPT3.5ができるかどうかって事です
つまり

```
messages = [
    {"role": "user", "content": "How many Rs are there in the word 'strawberry'"},
    {"role": "assistant", "content": reasoning_content}
  ]
response = openai_client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=messages
)

content = response.choices[0].message.content
```
この3行目のところみたいに放り込んでみたよってことです。

gpt3.5はこのstrawberryに「R」が何個入っているかって問題を実は解けなくて
何も入れないと自信満々に

![](/images/deepseek_resoning_in_gpt/round1.png)
Rは2個です！！！って言い切ります。

が、deepseek-reasonerの推論部分だけ抜いて渡すと

![](/images/deepseek_resoning_in_gpt/round2.png)

GPT3.5を高知能化することはできることは確認できました。

o1とかGemini thinkingは推論をoutput_tokenに含むのでその分値段が高くなりますが
推論部分をDeepSeekでやって、推論部分をInputに入れる場合はその分安くあがることになりますね。

ジャストアイデアですがo1のreasoning_effortをlowにして
推論はDeepSeekRにやってもらったやつをInputするのが
実はo1を使う上でも最安になるのかも？

コスト削減案やoutput_tokenを増やす方法は色々思いつきそうですね
以上、よろしくお願いします。




