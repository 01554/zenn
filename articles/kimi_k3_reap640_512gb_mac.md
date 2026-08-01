---
title: "594GBのKimi K3を441GBに枝刈りして、Mac Studio 1台でエージェントとして動かした"
emoji: "✂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "llamacpp", "Kimi", "SWELancer", "量子化"]
published: false
---

お疲れ様です波浪です。

[前回](https://zenn.dev/hellohazime/articles/kimi_k27_code_swelancer_local)、2bitのKimi K2.7 CodeにMac Studio 1台で$69,875稼がせた話を書きました。今回はその続編、K3です。

結論から。**Kimi K3(2.78T)がMac Studio(Apple M3 Ultra、512GB)1台で、エージェントとして動きました。** Moonshot純正のKimi Code CLIを繋いで、SWE-Lancerの実タスク3/3正解、$2,000/$2,000。モデルはHugging Faceに置いてあります。

https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF

K3はいちばん小さいGGUF(Unslothの1bit版)でも594GBあって、512GBのMacには載りません。なので**英語とコードにほぼ使われないexpertを896個中256個削って、441GBにしました**。名前のREAP640は「各層に640個残した」の意味です。

ここに至るまでに丸3日、Fableと一緒に失敗の山を築いてます。今回はその失敗の話からします。失敗の方が役に立つので。

## まず素直な方法は全部だめだった

最初に試したのはMLX版のREAP枝刈りビルドでした。expertを896個から242個まで削って、4bit(mxfp4)で451GBに収めた公開モデルがあったんですね。mlx-lmにK3対応(チャットテンプレート、ツール呼び出しパーサ、thinking分離)を自分で書き足して、繋がるところまでは行きました。丸一日かけて。

で、Kimi Code CLIの実プロンプト(ツール24個、約24kトークン)を入れた瞬間、こうなります。

```
The folder is a folder. The folder is a folder. The folder is a folder. ...
```

無限ループです。しかも確率的じゃなくて、このプロンプトだと毎回壊れます。短いプロンプトだと完璧に動くのがまた質が悪い。

原因の切り分けにさらに丸一日使いました。疑って潰した仮説は5つ。

| 仮説 | 結果 |
|---|---|
| 温度0の貪欲デコードが悪い | 無関係でした |
| repetition_penaltyで直る | **逆に悪化**。K3は`<|open|>`とか構造トークンを延々繰り返す形式なので、反復を罰すると構造ごと壊れます |
| プロンプトキャッシュがKDAの再帰状態を汚してる | キャッシュ切っても同じ |
| expert数が足りないのでは | expert多め(326個)のgradedビルドで試したら**もっと悪い** |
| thinkingの扱いが悪い | 有効/無効/effort低、全部無関係 |

全部外れ。5連敗です。手詰まりになってUnslothのDeepSeek-R1の記事を読み直しました。

## ヒントはUnslothのブログにあった

https://unsloth.ai/blog/deepseekr1-dynamic

R1を一律1.58bitにすると「infinite repetitions」になる、と書いてあります。まさに見てる症状じゃん、となりまして。彼らの対策は「重要な1〜2%(routerとlayernorm)を高精度で守って、残りを深く削る」でした。

ここで気づきます。MLX版はexpertを**4bit**で持つ代わりに**242個まで削って**いて、routerも8bitに量子化されています。UnslothのGGUFは全896個を**約1.6bit**まで削る代わりに、**routerはF32のまま**。真逆の配分なんですね。

どっちが正しいのか。594GBの未プルーニング1bit版を、ディスクオフロードで無理やり動かして(3時間かかった)、MLX版を毎回壊してたのと同じリクエストを投げました。返ってきたのは:

```
tool_call: Bash("grep -ri \"choose file\" /app/expensify/src ...")
```

**完璧に正常。** 課題(ボタンの"Choose File"の大文字問題)を正しく理解して、正しい最初の一手を打ってきました。

つまり「4bit×242個」より「1.6bit×896個」の方がエージェントとして壊れない。**ビット精度よりexpertの数**でした。少なくともこのモデルとこのワークロードでは。

## なら、1bit版を「少しだけ」削ればいい

896個で594GBが入らないなら、入るところまで削ればいい。計算すると640個で441GB、512GBに収まります。削減率29%。MLX版の73%削減に比べればかなり穏当です。

削るexpertの選定はREAP(Cerebrasの手法)です。校正コーパスは**英語とコードだけ**にしました。私の用途がそれだけなので。1.56TBの元モデルを層ごとにストリームして顕著性を測ると(79分で完走)、640個で**顕著性の93.5%を保持**できるとわかりました。逆に言うと中国語や日本語はこの選定で犠牲になってます。意図的なトレードオフです。

枝刈り自体は、GGUFのexpertテンソルをexpert軸でスライスするだけです。GGUFはexpert軸がいちばん外側で、量子化ブロックがexpert境界を跨がないので、**バイト列をそのままコピーできます。再量子化ゼロ**。

ひとつだけ罠があって、routerの行列と補正バイアスをkeep順に並べ替える処理を間違えると、「流暢にしゃべりながら全トークンを間違ったexpertに送るモデル」が出来上がります。形は合ってるのでエラーは出ません。怖いですね。ここは恒等プルーンがバイト一致することをテストで固定しました。

コードはここに置いてあります。

https://github.com/hellohazime/kimi-k3-gguf-prune

余談ですが、441GBをHugging Faceにアップロードしたとき、最後の45GBのシャードで実際に転送された新規データは**815MBだけ**でした。HFのXet重複排除が、Unslothの元リポジトリとバイト一致するチャンクを勝手に見つけてくれたんですね。「再量子化してない」という主張をアップロード基盤が証明してくれた形で、ちょっと感動しました。

## 動かす

llama.cppのK3対応はまだ本家にマージされていないので、Unslothのフォークを使います。

```bash
git clone https://github.com/unslothai/llama.cpp
cd llama.cpp && git fetch origin pull/48/head:kimi-k3 && git checkout kimi-k3
cmake -B build -DGGML_METAL=ON && cmake --build build -j --target llama-server

llama-server -m Kimi-K3-REAP640-IQ1_S-00001-of-00010.gguf \
    -ngl 99 -c 131072 --jinja --cache-reuse 0 --temp 1.0 --top-p 0.95
```

`--cache-reuse 0`は必須です。K3のKDA(線形アテンション)は再帰状態を持つので、プレフィックスキャッシュの部分再利用で状態が壊れます(PR #26185で報告されてる既知問題)。前回「llama serverのキャッシュが効いてない気がする」と書きましたが、K3では効かせちゃいけないやつでした。

Kimi Code CLIはOpenAI互換設定でそのまま繋がります。前回あんなに苦労してソルバーを自作したのに、今回はllama.cppフォークがK3のツール呼び出しもthinking分離もネイティブでやってくれます。時代の進歩。

## 結果

SWE-Lancer IC SWE(Diamond)から、K2.7が正解した最軽量3タスクで動作確認しました。

| タスク | 結果 | 報酬 | 参考: MLX版K3 |
|---|---|---|---|
| 28096_836 | 正解 | $500 | 0点 |
| 18827_741 | 正解 | $1,000 | 0点 |
| 29618_781 | 正解 | $500 | 0点 |

3タスクで3.4時間(68分/タスク)。実行はコンテナ内のKimi Code CLIが全部やって、採点は元のSWE-Lancerのまま一切触っていません。

<!-- TBD: K2.7失敗5タスク(14294, 24508_791, 15815_1, 27353_776, 15925)の結果をここに -->

速度はdecode約3.0トークン/秒、prefill約47トークン/秒。前回のK2.7より遅いので、フル198タスクは3〜5週間コースです。回すかどうかは電気代と相談します。

## 正直な注意書き

前回と同じく、測ってないことは測ってないと書いておきます。

- **フル198タスクは走らせていません。** n=3です。「たまたまこの3タスクで動いた」可能性は残ります
- perplexityも標準ベンチも測ってません。エージェントとして動くかだけを見ました
- 30kトークンを超える長文脈の品質は未検証
- 中国語・日本語は設計上壊れてます(校正から意図的に外したので)
- visionは未検証です。mmprojも同梱してません

このへんはモデルカードにも「Verified / Not verified」の表で明記してあります。誇張したいところですが、前回900秒打ち切りの件で反省したので正直にいきます。

## まとめ

594GBで載らなかったK3が、256個のexpertと引き換えに512GBのMac Studioで動くようになりました！Moonshot純正エージェント込みで！

効いたのは「量子化を深くしてexpertを残す」方向でした。expertを7割残せば1.6bitでもエージェントは壊れない。逆に4bitでもexpertを73%削ると壊れる。routerとlayernormを守るUnslothの設計を、バイトコピーの枝刈りならそのまま継承できるのも大きかった。

失敗5連発の切り分けも含めて、手順は全部公開してあります。512GBのMacをお持ちの方はぜひ。

以上、最後まで読んでくださりありがとうございました。

## 文献ノート

- モデル本体: [hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF](https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF)(441.4GB、10シャード)。枝刈りコードとテスト: [hellohazime/kimi-k3-gguf-prune](https://github.com/hellohazime/kimi-k3-gguf-prune)。
- ベースモデル: [unsloth/Kimi-K3-GGUF](https://huggingface.co/unsloth/Kimi-K3-GGUF) の UD-IQ1_S(594GB)。dynamic量子化の設計(重要層を8/16bit、routerとlayernormを高精度で保護)は[Unslothのドキュメント](https://unsloth.ai/docs/models/kimi-k3)と[DeepSeek-R1 1.58bitの記事](https://unsloth.ai/blog/deepseekr1-dynamic)に記載。「一律低bit化はinfinite repetitionsを起こす」という記述も後者より。
- REAP: [CerebrasResearch/reap](https://github.com/CerebrasResearch/reap)。expertの顕著性をルーターゲート×出力ノルムで測る。
- llama.cppのK3対応: [PR #26185](https://github.com/ggml-org/llama.cpp/pull/26185)(未マージ)。KDA+MLAハイブリッド、XTMLツール呼び出しパーサ、`--cache-reuse 0`が必要な理由(KDA再帰状態の破損)もこのPRの議論に記載。ビルドはUnslothフォークのPR 48ブランチを使用。
- MLX版REAPビルド: pipenetworkのKimi-K3-REAP73/REAPgraded(kimi-k3-mlx)。本記事の校正手法はこのリポジトリの実装(reap_calibrate.py)に依拠。「校正コーパスに含まれないものは黙って刈り取られる」「en+code校正で中国語がtotal collapse」という実測もこのリポジトリのREADMEより。
- 検証環境: SWE-Lancer(OpenAI、arXiv:2502.12115)のIC SWE Diamond。エージェントは[MoonshotAI/kimi-code](https://github.com/MoonshotAI/kimi-code) v0.30.0をタスクコンテナ内で実行(`kimi -p`、`--auto`や`-y`は`-p`と併用不可)。採点は無改変。
- 前回: [2bitに量子化したKimi K2.7 CodeにMac Studio 1台で$69,875を稼いでもらった](https://zenn.dev/hellohazime/articles/kimi_k27_code_swelancer_local)
