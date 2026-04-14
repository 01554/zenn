---
title: "Android termux上でllama.cpp Vulkanが動かなかったのでMNN+OpenCLでスマホGPU推論した話"
emoji: "📝"
type: "tech"
topics: [llamacpp,MNN,Qwen,Android,GPU]
published: true
---

『スマホでOpenClawが動き出す』の著者 hellohazime です。

https://hellohazime.booth.pm/items/8168199

本の中ではスマホ上での2Bモデルの動作は難しいと書きましたが、アプリ経由でMNNを使えばGPU推論が可能だったので、その手順をnoteに書きました。

https://note.com/hellohazime/n/nec2bf64ce8e5

## サークル hellohazime の本

https://hellohazime.booth.pm/

- [実用AGI Long Horizon Agent](https://hellohazime.booth.pm/items/8164176)：米投資会社セコイアが2026年1月に宣言した「AGI論争もう嫌だ、実務的に長時間駆動AgentならAGIでいいじゃん！」からはじまる、長時間駆動Agentを設計するための本
- [実用 AGI - パーソナル AGI の設計](https://hellohazime.booth.pm/items/8168179)：Long Horizon Agentが実用AGIならOpenClawは実用AGIじゃん、を起点に長時間駆動Agentを設計運用するためのAI筋を鍛え、後半はOpenClawをはじめとする実用AGIがチャットUIの世界を飛び出し新たな生態系を築いている事を実例から説明する本
- [スマホで OpenClaw が動き出す](https://hellohazime.booth.pm/items/8168199)：OpenClawにスマホのセンサー使わせたら面白いんちゃう？って思ったからスマホのTermuxにOpenClawを入れてみた本
- [俺たちの検閲破壊ローカルモデル](https://hellohazime.booth.pm/items/8168207)：OpenClawの脳みそにガードレール除去したローカルAI使ったら面白そうだけど品質どうなんだろうと疑問に思ったんで検証してみた本
