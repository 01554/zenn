---
title: "3層ニューラルネットワークでMNIST on Scratch 3.0 day2 重みの移植"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [scratch,MNIST,ML]
published: false
---

この記事は[株式会社ガラパゴス（有志） Advent Calendar 2025
](https://qiita.com/advent-calendar/2025/galapagos)の10日目です


お疲れ様です波浪です。

前回はScratch3.0側のコードができたので今度は重みをPythonで作って移植します

前回の記事はこれ
https://zenn.dev/galapagos/articles/mnist_predictor_on_scratch


# Pythonコード

https://colab.research.google.com/drive/1sRpvkPIC63ZNmgjg7ydhvIFXezyiI6s8?usp=sharing

Pythonコードはこれです。
注意点がいくつかあるので書いてきます。

## 学習画像の調整
まず学習画像ですが、MNISTは黒背景に白い文字でfloatになっています。

前回作った入力は白背景に黒文字、かつ 0/1の2値
よって元画像を修正して、白背景、0/1 の2値にします

また スクラッチ側は14x14の入力なので、学習画像28x28を縮小します。

この時単純に1/2にしようと思ったんですが、入力画像を見ると

![](/images/mnist_predictor_on_scratch/step18.png)

周辺の余白が多いので、上下左右を削ってから縮小することにしました。


というわけで上記の処理をしているのがこれ

```python
(x_train, y_train), (x_test, y_test) = mnist.load_data()

def preprocess_binary(images, threshold=0.5):
    images = images.astype('float32') / 255.0

    # (N, 28, 28) → (N, 28, 28, 1)
    images_tf = tf.expand_dims(images, -1)

    # Crop: 上下左右の余白を削る
    images_cropped = tf.image.crop_to_bounding_box(
        images_tf,
        offset_height=4,  # 上から4px
        offset_width=4,   # 左から4px
        target_height=20,
        target_width=20
    )

    # 14x14 にリサイズ
    images_resized = tf.image.resize(
        images_cropped,
        (14, 14),
        method='bilinear'
    ).numpy()

    # 次元調整 (N, 14, 14)
    images_resized = np.squeeze(images_resized)

    #  二値化
    images_binary = (images_resized > threshold).astype(np.float32)
    images_binary = 1.0 - images_binary # 白黒反転

    return images_binary


x_train_bin = preprocess_binary(x_train)
x_test_bin  = preprocess_binary(x_test)

#  ラベルを One-Hot Encoding に
y_train_one_hot = tf.keras.utils.to_categorical(y_train, num_classes=10)
y_test_one_hot  = tf.keras.utils.to_categorical(y_test, num_classes=10)
```

ただ、この処理をした後の画像も
![](/images/mnist_predictor_on_scratch/step19.png)

こんな感じでスクラッチの入力欄で書く数字とは太さとか形がかなり違う感じなのであまり精度が上がらないと思います。



## モデル定義
コード的には前後しますが モデル定義は以下の通り
```python
# --- モデル定義 ---
model = Sequential([
    Flatten(input_shape=(14,14)),    # 14x14 → 196
    Dense(8, activation='relu', name='layer_1'),
    Dense(4, activation='relu', name='layer_2'),
    Dense(10, activation='softmax', name='output_layer')
])

model.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

```
196(14x14) →　8 →　4 →　10

スクラッチで実装するにあたって表現を簡易にするため極めて小さなモデルにしています。

## 重みの出力
```python
# 重みの抽出
weights = {}

# 第1層: 196 -> 8
W1, b1 = model.get_layer('layer_1').get_weights()
weights['W1'] = W1
weights['b1'] = b1

# 第2層: 8 -> 4
W2, b2 = model.get_layer('layer_2').get_weights()
weights['W2'] = W2
weights['b2'] = b2

# 第3層 (出力層): 4 -> 10
W3, b3 = model.get_layer('output_layer').get_weights()
weights['W3'] = W3
weights['b3'] = b3
    
# Google drive に重みを保存するため マウント
from google.colab import drive
drive.mount('/content/drive')

def export_weights_flat(weights_dict, output_file='/content/drive/MyDrive/scratch_weights_flat.csv'):
    """
    重みとバイアスをすべて直列化して 1 ファイルにまとめて出力。
    Scratch 側では index を使ってスライスして各レイヤーに割り当て可能。
    """
    print("直列化して CSV 出力を開始します...")

    all_data = []

    for name, array in weights_dict.items():
        if name.startswith('W'):
            # W は転置して列ごとに連結
            W_T = array.T
            for i in range(W_T.shape[0]):
                all_data.extend(W_T[i].tolist())  # list に変換して追加
            print(f"✅ {name}: {W_T.shape[0]}列を直列化")
        elif name.startswith('b'):
            all_data.extend(array.tolist())
            print(f"✅ {name}: バイアス {array.shape[0]} 個を追加")

    # CSV 出力（1列として改行区切り、指数表記を避ける）
    os.makedirs(os.path.dirname(output_file), exist_ok=True)
    np.savetxt(output_file, all_data, delimiter='\n', fmt='%.8f')

    print(f"\n✅ 全データを直列化して '{output_file}' に保存しました。")
    print("Scratch 側ではリストに読み込んでリストの初期化をしてください")


export_weights_flat(weights)

```
こちらのコードで 重みを一次元配列にしてGoogleドライブに出力しています。
`W1 → B1 →　W2 →　B2 →　W3 →B3`
の順で直列に出力します。

`/content/drive/MyDrive/scratch_weights_flat.csv`
に出力してあるので、これを DLして移植します。

![](/images/mnist_predictor_on_scratch/step20.png)



# Scratch 側で 重みの初期化
スクラッチ側にリストを作って 名前を weight にします。

画面上に リストが表示されるので右クリックから「読み込み」 を選択することで csvや改行区切りのテキストをリストに読み込めます。
![](/images/mnist_predictor_on_scratch/step21.png)

## 重みの切り出し

自分は W1_col_1 W1_col_2...

と列ごとにリストを作ってしまっているため、スライスしていきます。
この処理があるからモデルを小さくしたんですが、バックプロパゲーションまで考えるならweightのまま扱えるように毎回スライスして処理するように組み直す方が良いです。


とりあえず推論する事が目的だし40個程度なので今回はこのままでいきます。

順番は 先ほど書いた通り
`W1 → B1 →　W2 →　B2 →　W3 → B3`
の順です

![](/images/mnist_predictor_on_scratch/step22.png)
![](/images/mnist_predictor_on_scratch/step23.png)

リストはグローバル変数なので

すいろん スプライトにこの初期化処理を紐づける理由がありません、なので自分は別スプライトにしました。

ちなみに関数定義は直接クリックすると動作します、weightリストに重みをtxtから読み込むために直接マウス操作が必要なので ここは完全手動で 重みの初期化 定義もマウスクリックで動かしておきます。


# 完成

はい、というわけで完成です。

https://scratch.mit.edu/projects/1252363057/


Python上でのテスト精度は80％超えていますが、学習画像とは実際に入力できる数字の形や形式がかなり違うので

スクラッチ上の体感は40％くらいですね、学習画像の質をあげるか、入力欄をfloat入力できるようにするともっと精度が上がりそうな気がします。

マウスがスプライトに触れてる時の距離や時間をとれればそれをfloatにできるかな？


以上、3層ニューラルネットワークをScratch3.0で組んでみた でした。

