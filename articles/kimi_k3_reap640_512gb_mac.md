---
title: "Kimi K3を441GBに枝刈りして、Mac Studio 1台で動かした"
emoji: "✂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "llamacpp", "Kimi",  "K3","SWELancer"]
published: true
---

お疲れ様です波浪です。

今回は、最近流行りのKimi K3です。前回の失敗を活かして自分の中でも熱があるうちにBLOGにしました。

結論。枝刈りしたKimi K3がMac Studio(Apple M3 Ultra、512GB)1台で動きました。ハーネスとしてKimi Code CLIを繋いで、SWE-Lancerの実タスク8本中5本正解、$3,500。そのうち2本は、前回2bitのK2.7が解けなかった問題でした。

モデルはHugging Faceに置いてあります。

https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF

K3はUnslothの1bit版でも594GBあって、512GBのMac一台には載りません。多言語（日本語含む）やコーディングに関係ないexpertを削って、441GBにしました。名前のREAP640は各層に640個残したという意味です。

以上報告終わり！ お疲れ様でした。


---

ここから先は、履歴という名の愚痴と自分のための記録です。

## MLX版のREAP枝刈りビルド

まず初めに見つけたのは

https://x.com/hellohazime/status/2082319922076729380?s=20
これでした。7/29の事です。

expertを896個から242個まで削って、4bit(mxfp4)を451GBに削ったモデルですね。
この時点で、
mlx-lmにK3対応(チャットテンプレート、ツール呼び出しパーサ、thinking分離)を自分で書き足して、繋がるところまでは行きました。

でも、Kimi Code CLIの実プロンプト(ツール24個、約24kトークン)を入れた瞬間、こうなります。

```
The folder is a folder. The folder is a folder. The folder is a folder. ...
```

無限ループです。しかも確率的じゃなくて、Kimi Code CLIの実プロンプは毎回壊れます。短いプロンプトだと完璧に動くのもたちが悪い。


![](/images/2026/k3/table.png)
まあ、よくみればgithubのREADMEにも書いてあるんですけどね。

## 1bit量子化モデル

さて、同じ7/29に 1bit量子化したK3も出ていました
https://x.com/UnslothAI/status/2082463988953367031

記事によると594GBだから、MacStudio+RAM128Gで動くぞ、とのことで、DGX Sparkを組み合わせれば動くんやな、とは思っていたんですが、DGX Sparkは他の学習に使っていたので後回しにしていました。
とりあえず確かめるために、1bit版を、SSDオフロードで無理やり動かして(3時間かかった)、MLX版を毎回壊してたのと同じリクエストを投げました。返ってきたのは:

```
tool_call: Bash("grep -ri \"choose file\" /app/expensify/src ...")
```

さっきと違い正常っぽい。SWELancerの課題(ボタンのChoose Fileの大文字問題)を正しく理解してそう。

## 1bit量子化モデルから枝刈りというアイデア

896個で594GBが入らないなら、入るところまで削ればいい。計算すると640個で441GB、512GBに収まります。削減率29%。MLX版の73%削減に比べればかなり穏当です。

削るexpertの選定は元にしているMLX版
[kimi-k3-mlx](https://github.com/PipeNetwork/kimi-k3-mlx)リポジトリのスクリプト(reap_calibrate.py / reap_plan.py)をそのまま使わせてもらいました。
校正コーパスは英語とコードだけにしました。SWE Lancerではその二つだけあれば十分だろうということで。

1.56TBの元モデルを層ごとにストリームして顕著性（どこが一番使われているか）を測ると、640個で英語+コードの顕著性の93.5%を保持できるとわかりました。

枝刈り自体は、GGUFのexpertテンソルをexpert軸でスライスするだけです。GGUFはexpert軸がいちばん外側で、量子化ブロックがexpert境界を跨がないので、バイト列をそのままコピーできます。って、まあFableが教えてくれました、初めて知ったよ、ありがとうFable5

ひとつだけ罠があって、routerの行列と補正バイアスをkeep順に並べ替える処理を間違えると、流暢にしゃべりながら全トークンを間違ったexpertに送るモデルが出来上がります。形は合ってるのでエラーは出ません。
これはこれで面白いですが。ベンチでは困るので、出力が元とバイト一致することをテストで確認してから本番を流しました。

本枝刈りでは、選定結果をGGUFに適用する(expert軸でスライスして詰め直す)200行くらいのスクリプトだけがオリジナル部分ですが。それはここに置いておきます。

https://github.com/01554/kimi-k3-gguf-prune

余談ですが、441GBをHugging Faceにアップロードしたとき、最後の45GBのシャードで実際に転送された新規データは815MBだけでした。HuggingFaceの仕様をよく理解していないのでなんでぇぇえぇ！？！？って割と長時間悩みました。
Unslothの元リポジトリとバイト一致するチャンクを検出して転送をスキップするんですね、まあある意味、思わぬところで量子化ブロックがexpert境界を跨がないので、バイト列をそのままコピーしているだけの確認テストができたってことですね。


## 校正コーパスとexpert選定

あまり興味持つ人は少ないと思うけど、僕的には面白かったので選定について
もう少し書きます。

MLX版のコードを見る限り、REAPがやってることは単純で
[https://github.com/PipeNetwork/kimi-k3-mlx/blob/main/scripts/make_calib.py
](https://github.com/PipeNetwork/kimi-k3-mlx/blob/20a4fb101ce81380ab8af0036743d49e7256c521/scripts/make_calib.py#L58)
このスクリプトで校正コーパスを用意、実際にモデルに流してみることで、expertごとにどこが使われているか確認してるだけです。

「実際にモデルに流す」のために1.5TBのメモリ必要なのでは？となるんですが

1.全体ではなく、第1層の重みデータだけをSSDからメモリにロードする。
2.用意したテキスト（calib.txt）をその層に流し込み、「どのExpertがどれくらい反応したか（Saliency）」をスコアとして記録する。
3.次の層に渡すための中間データだけを保持し、第1層の重みデータはメモリから破棄（解放）する。
4.続いて第2層だけをSSDからロードし、同じように計算する。
5.これを最後の層まで繰り返す。
これでMacStudio一台でも処理できるってことになってました。Out-of-core実行とか呼ぶらしいですね、SSDオフロードしてるだけといえばそうです。

expertの担当がどれくらいドメインで分かれているかは、kimi-k3-mlxに実測があります。校正をソース別に集計して、ソースごとの上位expert集合の重なりを見たものが

| ペア | 重なり | 偶然比 |
|---|---|---|
| コード(Python) ↔ コード(多言語) | 57.2% | 2.1倍 |
| ドイツ語 ↔ スペイン語 | 59.3% | 2.2倍 |
| 中国語 ↔ 日本語 | 42.8% | 1.6倍 |
| 中国語 ↔ コード(Python) | 17.8% | 0.66倍(偶然以下) |

コード系は固まっていて、欧州言語も固まっていて、CJKもやっぱり固まっている。担当がほぼ分かれています。だから英語+コードで校正すると中国語がぶっ壊れるし、逆に中国語だけで校正するとコードがぶっ壊れる(どちらもkimi-k3-mlxの実測)。

## 日本語+業界特化版

たとえば日本語+マーケティング業界みたいな特化版も、作れるはずです。パイプラインは全く同じで、変えるのは校正コーパスだけ。

1. 日本語Webテキスト+マーケティング文書(広告コピー、LP、業界レポート、メルマガ等)を集める
2. reap_calibrateに流して顕著性を測る
3. keep 640でプランを作ってGGUFをスライス

ただ、実際にやるなら注意点が3つあります。

(1) 配合はバイト比じゃなくトークン比で。CJKはUTF-8で3バイト/文字なのにBPEでは密に詰まるので、バイトで混ぜると比率が狂います。kimi-k3-mlxの実測では、バイト比15%の中国語がトークン比では32.6%になっていました。日本語でも同じことが起きます。

(2) 量を確保する。今回は合計26万トークン流しました。狙いのドメインには最低でも数万トークンは欲しいところです。

(3) マーケティングがexpertとして分離しているかは、測るまでわからない。上の表で綺麗に分かれているのは言語とコードという大きな軸です。同じ日本語の中のマーケティング文体が専用expertを持っているかは正直怪しくて、たぶん日本語一般のexpertとかなり重なります。その場合マーケ文書を混ぜる効果は、日本語expert内の優先順位付け程度に薄まる。本気でやるなら、先にソース別集計で重なり率を測ってから判断するのがおすすめです。

それと当然ですが、削った能力は本当に消えます。日本語+マーケ版に、あとからちょっとSQLを書かせたくなっても、もう書けません。削る前に、エージェントにやらせる仕事を全部書き出して、その全部をコーパスに入れてください。

転じて言えば、モデルを複数用意することも理論上は可能です。
コーディング用モデル、業界特化モデル、のように削ったモデルを用意しておき
SakanaAIのFuguのように、その時その時で一番妥当なモデルを使わせるって方法とか

いやまてよ？ なんなら1つのモデルだけど、毎回使うexpertを選定すればいいのか。
Fuguのコアになっている小型モデルの部分をKimiK3専用モデルとして用意するみたいな手も考えられますね。
MoEはTokenごとにexpart変えますが、prompt全体を見て、expertを変えるって考え方を確かFuguはとっていたはずなので、（https://arxiv.org/html/2606.21228v1） 同じ発想で選定できるかも？ まあ、気が向いたら考えてみましょう。



## 動かす

llama.cppのK3対応はまだ本家にマージされていないので、Unslothのフォークを使います。

```bash
git clone https://github.com/unslothai/llama.cpp
cd llama.cpp && git fetch origin pull/48/head:kimi-k3 && git checkout kimi-k3
cmake -B build -DGGML_METAL=ON && cmake --build build -j --target llama-server

llama-server -m Kimi-K3-REAP640-IQ1_S-00001-of-00010.gguf \
    -ngl 99 -c 131072 --jinja --cache-reuse 0 --temp 1.0 --top-p 0.95
```

`--cache-reuse 0`は必須です。K3のKDA(線形アテンション)は再帰状態を持つので、プレフィックスキャッシュの部分再利用で状態が壊れます(PR #26185で報告されてる既知問題)。前回、llama serverのキャッシュが効いてない気がすると書きましたが、K3では効かせちゃいけないやつでした。

Kimi Code CLIはOpenAI互換設定でそのまま繋がります。前回あんなに苦労してソルバーを自作したのに、今回はllama.cppフォークがK3のツール呼び出しもthinking分離もネイティブでやってくれます。ありがたし🙏。

## 結果

SWE-Lancer IC SWE(Diamond)から、K2.7が正解した3タスクで動作確認しました。

| タスク | 結果 | 報酬 | 参考: MLX版K3 |
|---|---|---|---|
| 28096_836 | 正解 | $500 | 0点 |
| 18827_741 | 正解 | $1,000 | 0点 |
| 29618_781 | 正解 | $500 | 0点 |

3タスクで3.4時間(68分/タスク)。実行はコンテナ内のKimi Code CLIが全部やって、採点は元のSWE-Lancerのまま一切触っていません。

追試もやりました。前回K2.7が正解できなかった105タスクから5つ選んで、実行。

| タスク | K2.7(2bit、341GB) | K3 REAP640(1bit、441GB) | 報酬 |
|---|---|---|---|
| 14294 | 不正解 | 不正解 | - |
| 24508_791 | 不正解 | 正解 | $1,000 |
| 15815_1 | 不正解 | 不正解 | - |
| 27353_776 | 不正解 | 正解 | $500 |
| 15925 | 不正解 | 不正解 | - |

2/5。K2.7に解けなかった問題を、1bitまで削ってexpertを256個間引いたK3が2つ通しました。n=5なので強くは言いませんが、こんだけ削ってもK2.7より高性能の可能性が高そう。実際に198タスク動かせばベンチも取れますが、decode約3.0トークン/秒、prefill約47トークン/秒。前回のK2.7より遅いので、ベンチが回り切るまで60日コース。

前回のK2.7（13日）の4倍以上、さすがに。うーん？厳しいですね、やってる間にK3が時代遅れになってお蔵入りしちゃいそうだわ...


## 注意書き

- フル198タスクは走らせていません。動作確認3+追試5のn=8です
- perplexityも標準ベンチも測ってません。エージェントとして動くかだけを見ました
- 中国語・日本語は設計上壊れてます(校正から意図的に外したので)
- visionは未検証です。mmprojも同梱してません

## まとめ

1bitでも594GBで、MacStudioに載りきらないK3でも、256個のexpertと引き換えに512GBのMac Studioで動く事が確認できました！！！！！！！ MoEだからできる芸だぜやっほぅ！！！！！

以上、最後まで読んでくださり、ありがとうございました。

## 文献ノート
- 恒等ミキプルーン: 最後の最後まで入れるか悩んだ一発ギャグ、もう伝わる人もいなそうなので削除した。
- モデル本体: [hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF](https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF)(441.4GB、10シャード)。GGUFスライスのコードとテスト: [01554/kimi-k3-gguf-prune](https://github.com/01554/kimi-k3-gguf-prune)(MIT)。
- ベースモデル: [unsloth/Kimi-K3-GGUF](https://huggingface.co/unsloth/Kimi-K3-GGUF) の UD-IQ1_S(594GB)。dynamic量子化の設計(重要層を8/16bit、routerとlayernormを高精度で保護)は[Unslothのドキュメント](https://unsloth.ai/docs/models/kimi-k3)と[DeepSeek-R1 1.58bitの記事](https://unsloth.ai/blog/deepseekr1-dynamic)に記載。一律低bit化はinfinite repetitionsを起こすという記述も後者より。
- REAP: [CerebrasResearch/reap](https://github.com/CerebrasResearch/reap)。expertの顕著性をルーターゲート×出力ノルムで測る。
- llama.cppのK3対応: [PR #26185](https://github.com/ggml-org/llama.cpp/pull/26185)(未マージ)。KDA+MLAハイブリッド、XTMLツール呼び出しパーサ、`--cache-reuse 0`が必要な理由(KDA再帰状態の破損)もこのPRの議論に記載。ビルドはUnslothフォークのPR 48ブランチを使用。
- MLX版REAPビルド: pipenetworkの[kimi-k3-mlx](https://github.com/PipeNetwork/kimi-k3-mlx)(Kimi-K3-REAP73/REAPgraded)。本記事の顕著性計測と選定はこのリポジトリのスクリプト(reap_calibrate.py / reap_plan.py)をそのまま使用。校正コーパスに含まれないものは黙って刈り取られる、en+code校正で中国語がtotal collapseする、という実測もこのリポジトリのREADMEより。本文でだめだったと書いたMLXビルドの作者に、枝刈りの道具一式で助けられているという構図です。
- 検証環境: SWE-Lancer(OpenAI、arXiv:2502.12115)のIC SWE Diamond。エージェントは[MoonshotAI/kimi-code](https://github.com/MoonshotAI/kimi-code) v0.30.0をタスクコンテナ内で実行(`kimi -p`、`--auto`や`-y`は`-p`と併用不可)。採点は無改変。
- 前回: [2bitに量子化したKimi K2.7 CodeにMac Studio 1台で$69,875を稼いでもらった](https://zenn.dev/hellohazime/articles/kimi_k27_code_swelancer_local)
