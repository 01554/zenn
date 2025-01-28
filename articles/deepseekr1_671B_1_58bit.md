---
title: "Not蒸留物、本物のR1を1.58bit量子化したモデルを動かす（1500円/時)"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [deepseekR1,huggigface]
published: false
---

# 皆に本物のDeepseek R1を見せてあげますよ

お疲れ様です、波浪です。

どうも世間には
deepseek R1をローカルPCで動かしました！ドヤっっってしてる記事がたくさんありますがその人たちが動かしているブツの大半はR1の蒸留物でサイズが13Bとかせいぜい70Bくらいなんですよね。

そんな中
ガチのDeepseek R1(model size 671B)の1.58bit量子化版がHFに登録されました。
https://unsloth.ai/blog/deepseekr1-dynamic


といっても、さすがに動かすためには最低でも24GBのVRAMと64GのRAMが必要です。

ま、逆に言えば、それが家にある人はこれをローカルPCで動かせちゃいます！
なお波浪の家にはRTX3090(24GB)はありますが、RAMが32しかのっていないのでこいつをすぐには試せません！！！！！

というわけでColabにのらないか、GCPでどうにかするか？をなやんでいたら
こちらのllama.cpp作った人のツイートが流れてきたんですな
https://x.com/ggerganov/status/1883961201371042120


このツイートのリンク先は以下なんですが
https://endpoints.huggingface.co/new?repository=unsloth%2FDeepSeek-R1-GGUF&vendor=aws&region=us-east-1&accelerator=gpu&instance_id=aws-us-east-1-nvidia-l40s-x4&task=text-generation&no_suggested_compute=true&env_LLAMA_ARG_CACHE_TYPE_K=q8_0&env_LLAMA_ARG_UBATCH=64


これはつまり、HugginfFaceにクレジットカード登録しておけば
ボタン一発でR1(1.58Bit)を試させてやるよと
なお金額は $ 8.3 /h だ

と、まあ、そういうわけですね。

ぶっちゃけhuggigfaceにクレジットカードちゃんと登録してあとはボタン押すだけなんで、なんも説明することはないんですが

実際に動かすとこれくらいの速度でtokenがでたんで実用可能レベルですね。
![](/images/r1_158/480run.gif)

なお精度に関しては

https://unsloth.ai/blog/deepseekr1-dynamic#:~:text=DeepSeek%20Original-,1.58%2Dbit%20Version,-We%20see%20surprisingly

元のBLOGにありますが、多少落ちる程度です。

日本語に関しては今からやりますが
取り急ぎ、驚き仕草しとこうと思ったんで記事をしたためた次第
はー、RAM買ってこよ。

以上、よろしくお願いします。