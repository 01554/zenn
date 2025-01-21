---
title: "guardrails-aiのgibberish_textでHelloがガードされる"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [guardrails-ai,gibberish_text]
published: true
---
お疲れ様です、波浪です。

今日は
https://generative-agents.connpass.com/event/342683/
この勉強会に参加してたんですが

勉強会の中で
https://github.com/guardrails-ai/gibberish_text

gibberish_textを実行したところ
「Hello」という文言が引っかかってしまう

という状況になっていたので何が起きていたのか確認してみました。

## そもそもgibberish_text って何？
>この検証ツールは、言語モデルによって生成されたテキストの「クリーンさ」を検証します。事前トレーニング済みのモデルを使用して、テキストが一貫性があり、意味不明なものではないかどうかを判断します。検証ツールを使用すると、一貫性のないテキストや意味をなさないテキストを除外できます。

とのことで、意味不明な文言と認識されるらしいんですよねHelloが。
えー？って感じですね。


## コード
というわけで実際のコードを確認すると
https://github.com/guardrails-ai/gibberish_text/blob/ef04ad1529b657edf41fe23ddb45482bb1df1085/validator/main.py#L62

このモデルを利用しています。
これは

https://huggingface.co/madhurjindal/autonlp-Gibberish-Detector-492513457
つまりこいつなんですが。

APIを直接実行できるみたいなんでHelloを入れて、いったいどのラベルに放り込まれたのかみてみたいと思います

## 実行
```
curl -X POST -H "Authorization: Bearer {HF_API_KEY}" -H "Content-Type: application/json" -d '{"inputs": "I love Machine Learning!"}' https://api-inference.huggingface.co/models/madhurjindal/autonlp-Gibberish-Detector-492513457
{"error":"Model madhurjindal/autonlp-Gibberish-Detector-492513457 is currently loading","estimated_time":20.0}%
```
初回実行はちょっとロードするから待てよって言われるので待ちましょう

```
curl -X POST -H "Authorization: Bearer {HF_API_KEY}" -H "Content-Type: application/json" -d '{"inputs": "Hello"}' https://api-inference.huggingface.co/models/madhurjindal/autonlp-Gibberish-Detector-492513457
[[{"label":"word salad","score":0.29563990235328674},{"label":"mild gibberish","score":0.2810679078102112},{"label":"clean","score":0.26269251108169556},{"label":"noise","score":0.16059967875480652}]]%
```

はい、出ました
`"label":"word salad"`

どうやら「Hello」だけだと「word salad」に分類されたみたいですね。

これスルーされるメッセージだと
```
[[{"label":"clean","score":0.7139410972595215},{"label":"mild gibberish","score":0.17421475052833557},{"label":"word salad","score":0.08730166405439377},{"label":"noise","score":0.02454245649278164}]]%

```
`"label":"clean"`
こんな感じで「Clean」に含まれます

https://github.com/guardrails-ai/gibberish_text/blob/ef04ad1529b657edf41fe23ddb45482bb1df1085/validator/main.py#L83
このコードはラベル:cleanじゃないと失敗するようになっているので、helloだと死んじゃうんですね。

うーむ？ ま、日本語には使えなさそうなモデルなので使うことはないかな...


以上、よろしくお願いします。


