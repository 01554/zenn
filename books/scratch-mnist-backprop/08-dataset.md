---
title: "学習データ：MNIST を Scratch に持ち込む"
---

## MNIST とは

MNIST は、手書き数字の画像を集めた有名なデータセットです。アメリカの国勢調査局の職員と高校生が書いた数字を集めたもので、学習用6万枚、テスト用1万枚。1枚は28×28ピクセルの濃淡画像で、正解の数字（ラベル）がついています。機械学習の入門で最初に使われる定番で、『ゼロから作る Deep Learning』でもこのデータを使います。

## Scratch 用に加工する

MNIST の画像はそのままでは使えません。3つ加工します。

1. **14×14 に縮小する。** 入力パネルが14×14だからです（第3章）。ただ半分にすると数字が小さくなりすぎるので、先に周囲の余白を4ピクセルずつ切り落として20×20にしてから、14×14へ縮めます。
2. **0 と 1 の2値にする。** 元は濃淡つきですが、パネルのマスは白か黒かの2択なので、しきい値より濃ければ 1、薄ければ 0 に割り切ります。
3. **白黒を反転する。** MNIST は黒地に白文字、この作品のパネルは白地に黒文字なので、合わせます。

![MNIST の元画像。余白が広い](/images/mnist_predictor_on_scratch/step18.png)

![加工後の画像。14×14、2値](/images/mnist_predictor_on_scratch/step19.png)

加工は Python でやります。

```python
import numpy as np
import tensorflow as tf
from tensorflow.keras.datasets import mnist

(x_train, y_train), (x_test, y_test) = mnist.load_data()

def preprocess_binary(images, threshold=0.5):
    images = images.astype('float32') / 255.0
    images_tf = tf.expand_dims(images, -1)
    # 周囲4ピクセルを切り落として 20×20 に
    images_cropped = tf.image.crop_to_bounding_box(
        images_tf, offset_height=4, offset_width=4,
        target_height=20, target_width=20)
    # 14×14 に縮小
    images_resized = tf.image.resize(images_cropped, (14, 14),
                                     method='bilinear').numpy()
    images_resized = np.squeeze(images_resized)
    # 2値化して白黒反転
    images_binary = (images_resized > threshold).astype(np.float32)
    return 1.0 - images_binary

x_train_bin = preprocess_binary(x_train)
x_test_bin  = preprocess_binary(x_test)
```

## テキストに書き出して、リストに読み込む

第5章の exp の表と同じ手を使います。画像を1ピクセル1行のテキストに書き出して、Scratch のリストに読み込みます。1枚が196行なので、600枚で117600行です。ラベルは1枚1行で、数字をそのまま書きます。

```python
with open("train_images.txt", "w") as f:
    for img in x_train_bin[:600]:
        for v in img.flatten():
            f.write(f"{int(v)}\n")

with open("train_labels.txt", "w") as f:
    for label in y_train[:600]:
        f.write(f"{label}\n")
```

テスト用も同じように、`x_test_bin` から100枚を書き出します。Scratch 側で `train_images`、`train_labels`、`test_images`、`test_labels` の4本のリストを作り、それぞれ右クリックの「読み込み」で流し込めば、教材の持ち込みは完了です。

![4本のリストに読み込んだところ。右クリックで読み込みメニューが出る](/images/backpropagation_on_scratch/init/images.png)

`train_labels` の先頭が 5, 0, 4, 1, 9 と並んでいます。MNIST の学習データの最初の5枚のラベルはこの並びで、世界中の機械学習の教科書に出てくる顔ぶれです。

## なぜ600枚しか使わないのか

最初は6万枚を全部入れました。読み込みはできたのですが、プロジェクトの保存が通りません。

![60000枚入れたらプロジェクトが保存できなくなった](/images/backpropagation_on_scratch/init/save_failed.png)

6万枚 × 196ピクセルは1176万行です。Scratch のプロジェクトはリストの中身ごと保存されるので、さすがに上限を超えました。そこで学習600枚・テスト100枚まで減らしてあります。学習データが100分の1なので精度もそれなりに下がりますが、学習の仕組みを見るには十分です。

## 枚数などの定数

枚数まわりの定数は「MNIST定義」ブロックにまとまっています。

![MNIST定義ブロック。枚数と入力サイズを決める](/images/backpropagation_on_scratch/init/MNIST.png)

## 画像1枚を取り出すブロック

学習のたびに、リストの中から1枚ぶんの196個を取り出して、`activations` の入力の場所（1〜196番目）へコピーします。何枚目が欲しいかを `p_sample_index` で渡すと、読み出し開始位置を `(枚数 − 1) × 196 + 1` で計算して、196回コピーします。ラベルも `p_target_label` に取り出します。

![u_load_train_sample の定義。画像1枚を activations にコピーする](/images/backpropagation_on_scratch/train/train_load_sample.png)

## 教材を目で確かめる

リストに入った117600個の 0/1 は、そのままでは何も見えません。「sample」スプライトを押すと、ランダムに1枚選んでパネルに表示し、ラベルをしゃべります（パネルの並べ方と、非表示ボタンの表示切り替えは第3章のとおり）。

![sample スプライトのコード。乱数で1枚選び、196個をパネルに送る](/images/backpropagation_on_scratch/init/sample1.png)

![表示された学習画像。ラベルは9](/images/backpropagation_on_scratch/init/sample4.png)

眺めていると、これで9なのか、と言いたくなる字も出てきます。14×14まで縮めたせいもありますが、もともと人間の手書きはこのくらい揺れています。この揺れごと覚えさせるのが、次の章から始まる学習です。
