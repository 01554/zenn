---
title: "残差と MLP：集めた情報を作り直す"
---

![](/images/scratch-microgpt-llm/tobira_07.png)

![](/images/scratch-microgpt-llm/manga_07_p1.png)

![](/images/scratch-microgpt-llm/manga_07_p2.png)


## アテンションの次は、その場で練り直す

アテンションは、他の文字から情報を集める工程でした。集めただけでは材料が並んでいるだけです。次は、その材料をその位置だけで練り上げます。これが MLP（多層パーセプトロン、Multi-Layer Perceptron）で、フィードフォワード層とも呼びます。

アテンションが横のつながり（文字どうしの情報交換）だとすれば、MLP は縦の深掘りです。1文字ぶんのベクトルを、その場で非線形に変形します。Transformer ブロックは、この2つを交互に行います。

MLP は3手でできています。

1. 16個の数値を64個に広げる（一時的に情報を展開する）
2. ReLU で「マイナスを0にする」非線形化
3. 64個を16個に戻す

## 広げて、曲げて、戻す

$$
\boldsymbol{h} = \mathrm{ReLU}(W_1 \boldsymbol{x} + \boldsymbol{b}_1), \qquad
\boldsymbol{m} = W_2 \boldsymbol{h} + \boldsymbol{b}_2
$$

- $W_1$：16→64 に広げる重み。$\boldsymbol{h}$ は64次元。
- $\mathrm{ReLU}(u) = \max(0, u)$：負を0にする関数。$\max$ は「大きいほうを選ぶ」記号なので、$0$ と $u$ をくらべて、$u$ がマイナスなら $0$、プラスならそのまま。
- $W_2$：64→16 に戻す重み。$\boldsymbol{m}$ は16次元。

なぜわざわざ64に広げて戻すのでしょう。次元を増やすと、たくさんの特徴の検出器を並べられます。ReLU がそれらを選択的にオン/オフし、戻す段で組み合わせ直す。この「広げて・曲げて・戻す」があるおかげで、モデルは直線では表せない複雑な関係を学べます。ReLU の非線形性がなければ、いくら層を重ねても結局ただの1回の線形変換に潰れてしまいます。

## microgpt.py ではこう書く

この本のモデルは、Karpathy の microgpt.py（純粋な Python だけで書かれた小さな GPT）を Scratch に移したものです。MLP の部分は、microgpt.py ではたった数行です。

```python
# 2) MLP block
x_residual = x
x = linear(x, state_dict['mlp_fc1'])        # 16 → 64 に広げる（W1）
x = [xi.relu() for xi in x]                 # ReLU（マイナスを0に）
x = linear(x, state_dict['mlp_fc2'])        # 64 → 16 に戻す（W2）
x = [a + b for a, b in zip(x, x_residual)]  # 残差：元に足し戻す
```

`mlp_fc1` が数式の $W_1$、`mlp_fc2` が $W_2$ です。`linear` は「掛けて足す」だけの関数（第13章）、`relu` はマイナスを0にする関数。最後の1行が残差の足し戻し（$\boldsymbol{x} \leftarrow \boldsymbol{x} + \mathrm{MLP}(\boldsymbol{x})$）です。Scratch では、この `linear` を「ベクトルと重みのループ」、`relu` を「ベクトルReLU」ブロックとして自分で書いています。

Scratch の forward ブロックの中では、この MLP はこう並びます。

![forward の中の MLP：linear で64に広げ→ベクトルReLU→linear で16に戻し→残差](/images/microgpt_on_scratch/fn_mlp.png)

上から順に、正規化（rmsnorm）→「linear(全結合層) …16 64」で16個を64個に広げ（$W_1$）→「ベクトルReLU」でマイナスを0にし→「linear(全結合層) …64 16」で64個を16個に戻し（$W_2$）→「ベクトルたし算」で元のベクトルに残差として足し戻す、という並びです。microgpt.py の4行と、行の順番がそのまま対応しています（途中の「h\_hidden／h\_mlp を送って待つ」は、アニメを一時停止して64個の広がりや足し戻しを見せるための合図で、計算そのものには関係しません）。

## アニメで見る MLP

アニメモードでは、まず16個の数が64個に広がる様子を見せます。

![MLP：64個に広げて考える](/images/scratch-microgpt-llm/step_mlp_expand.png)

キャプションは「MLP：64個に広げて考える（灰色=ReLUで0になった）」。その下の「↓いまのベクトル16個 → MLPが64個に広げた（↑緑の列）」の矢印が指す上の緑の列が、64個に広げた値です。ReLU で0に潰された成分は灰色で表示されます（この場面ではたまたま64個すべてが生き残って緑になっています。文脈によっては灰色が混じります）。生き残った正の成分だけが、次に情報を伝えます。

64個に戻したあとは、それを残差として元のベクトルに足します。

![MLPの結果（64個→16個に戻す）をベクトルに足した（上下をくらべて）](/images/scratch-microgpt-llm/step_mlp_add.png)

キャプションは「MLPの結果（64個→16個に戻す）をベクトルに足した」。その下は「↑上=書き込む前 → ↓下=MLPの結果を足した後」。アテンションのときと同じ残差接続です。

$$
\boldsymbol{x}_i \leftarrow \boldsymbol{x}_i + \mathrm{MLP}(\boldsymbol{x}_i)
$$

上の段（MLP 前）と下の段（MLP 後）を見比べると、いくつかの丸の色や濃さが変わっています。MLP が計算した結果が、元のベクトルに上乗せされたのです。

## ブロックの完成

ここまでで、Transformer の1ブロックが1周しました。

$$
\boldsymbol{x} \;\leftarrow\; \boldsymbol{x} + \mathrm{Attention}(\boldsymbol{x}) \quad\text{（横：文脈を集める）}
$$
$$
\boldsymbol{x} \;\leftarrow\; \boldsymbol{x} + \mathrm{MLP}(\boldsymbol{x}) \quad\text{（縦：その場で練る）}
$$

集める、練る。これを1セットにしたのが Transformer ブロックです。本物の GPT はこのブロックを何十段も積み、そのたびにベクトルはより豊かな意味を帯びていきます。このモデルは1段ですが、やっていることは同じです。

ブロックを抜けたベクトル $\boldsymbol{x}$ には、「ふるいけや」の文脈をすべて踏まえた、次に何が来るべきかの情報が詰まっています。このベクトルから次の文字の確率を取り出すのが第8章です。
