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

ここから先は、動作させるまでの、自分のための記録です。

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


## English version
 
*Machine-assisted translation of the Japanese text above.*
 
---
 
# I pruned Kimi K3 down to 441GB and ran it on a single Mac Studio
 
This time it's Kimi K3, the model everyone's talking about. I learned from last
time and got the blog post out while I was still fired up about it.
 
Bottom line: **a pruned Kimi K3 runs on one Mac Studio (Apple M3 Ultra, 512GB).**
I hooked up Kimi Code CLI as the harness and ran real SWE-Lancer tasks — **5 out
of 8 solved, $3,500.** Two of those were problems that last time's 2-bit K2.7
could not solve.
 
The model is on Hugging Face:
 
https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF
 
Even Unsloth's 1-bit K3 is 594GB, which won't fit on a single 512GB Mac. I cut
the experts that handle multilingual work (including Japanese) and anything
unrelated to coding, bringing it down to 441GB. The REAP640 in the name means
640 experts kept per layer.
 
That's the report! Thanks for reading.
 
---
 
Everything past this point is grumbling disguised as a changelog, written mostly
for my own records.
 
## The MLX REAP-pruned build
 
The first thing I found was this:
 
https://x.com/hellohazime/status/2082319922076729380?s=20
 
That was July 29.
 
It cuts experts from 896 down to 242, taking 4-bit (mxfp4) to 451GB. At that
point I'd written K3 support into mlx-lm myself — chat template, tool-call
parser, thinking separation — and gotten as far as a working connection.
 
But the moment I fed it Kimi Code CLI's actual prompt (24 tools, roughly 24k
tokens), this happened:
 
```
The folder is a folder. The folder is a folder. The folder is a folder. ...
```
 
Infinite loop. And not probabilistically — Kimi Code CLI's real prompt broke it
*every single time*. What made it especially nasty is that short prompts worked
perfectly.
 
![](/images/2026/k3/table.png)
 
Which, well, is written right there in the GitHub README if you actually look.
 
## The 1-bit quantized model
 
Meanwhile, on that same July 29, a 1-bit quantized K3 also dropped:
 
https://x.com/UnslothAI/status/2082463988953367031
 
The post said 594GB, so it'd run on a Mac Studio plus 128GB of RAM. I figured I
could combine it with my DGX Spark and make it work, but the Spark was busy with
other training runs, so I put it off.
 
Just to check, I forced the 1-bit version to run with SSD offloading (took three
hours) and threw at it the exact same request that kept breaking the MLX
version. What came back:
 
```
tool_call: Bash("grep -ri \"choose file\" /app/expensify/src ...")
```
 
Unlike before, this looks sane. It seems to have correctly understood the
SWE-Lancer task — the capitalization issue on the "Choose File" button.
 
## The idea: prune the 1-bit model
 
If 896 experts at 594GB won't fit, cut until it does. Running the numbers: 640
experts is 441GB, which fits in 512GB. A 29% reduction — pretty modest compared
to the MLX version's 73%.
 
For selecting which experts to cut, I used the scripts from the MLX build's
repo, [kimi-k3-mlx](https://github.com/PipeNetwork/kimi-k3-mlx)
(`reap_calibrate.py` / `reap_plan.py`), exactly as-is.
 
I limited the calibration corpus to English and code only, figuring those two
are all SWE-Lancer needs.
 
Streaming the 1.56TB source model layer by layer and measuring saliency (which
parts get used most), it turned out that 640 experts retains **93.5%** of the
English+code saliency.
 
The pruning itself is just slicing the GGUF expert tensors along the expert
axis. In GGUF the expert axis is outermost, and quantization blocks don't
straddle expert boundaries, so you can copy the byte sequences straight across.
…Which Fable taught me, by the way. First I'd heard of it. Thanks, Fable 5.
 
There's exactly one trap. If you botch the reordering of the router matrix and
correction bias into keep-order, you end up with a model that speaks fluently
while routing every single token to the wrong expert. The shapes are valid, so
nothing errors out.
 
Which is entertaining in its own right. But inconvenient for benchmarking, so I
verified with a test that the output was byte-identical to the original before
running anything real.
 
The only original part of this whole thing is the ~200-line script that applies
the selection to the GGUF (slicing along the expert axis and repacking). Here it
is:
 
https://github.com/01554/kimi-k3-gguf-prune
 
Side note: when I uploaded 441GB to Hugging Face, the final 45GB shard only
actually transferred 815MB of new data. I don't understand HF's internals well,
so I spent a genuinely long time going *whaaaaat?!* It turns out it detects
chunks that are byte-identical to Unsloth's source repo and skips transferring
them. So in a sense I got an unexpected bonus confirmation test that
quantization blocks don't straddle expert boundaries and I really was just
copying bytes.
 
## Calibration corpus and expert selection
 
Probably not many people care about this part, but I found it interesting, so
here's a bit more on the selection.
 
From reading the MLX code, what REAP does is simple:
 
[make_calib.py](https://github.com/PipeNetwork/kimi-k3-mlx/blob/20a4fb101ce81380ab8af0036743d49e7256c521/scripts/make_calib.py#L58)
 
This script prepares the calibration corpus, and then you actually run it
through the model to see which parts get used per expert. That's it.
 
You'd think "actually running it through the model" requires 1.5TB of memory.
But:
 
1. Load only layer 1's weights from SSD into memory — not the whole model.
2. Push the prepared text (calib.txt) through that layer, recording how strongly
   each expert responded (saliency) as a score.
3. Keep only the intermediate data needed for the next layer, and free layer 1's
   weights from memory.
4. Load only layer 2 from SSD and repeat.
5. Continue to the final layer.
That's how it fits on one Mac Studio. Apparently this is called out-of-core
execution. Though you could also just call it SSD offloading.
 
As for how cleanly experts divide up by domain, kimi-k3-mlx has actual
measurements. Aggregating calibration by source and looking at the overlap
between each source's top-expert set:
 
| Pair | Overlap | vs. chance |
|---|---|---|
| Code (Python) ↔ Code (multi-lang) | 57.2% | 2.1× |
| German ↔ Spanish | 59.3% | 2.2× |
| Chinese ↔ Japanese | 42.8% | 1.6× |
| Chinese ↔ Code (Python) | 17.8% | 0.66× (below chance) |
 
Code clusters together. European languages cluster together. CJK also clusters
together. The assignments are almost entirely separate. Which is why calibrating
on English+code destroys Chinese, and conversely calibrating on Chinese alone
destroys code (both measured in kimi-k3-mlx).
 
## A Japanese + industry-specific variant
 
You could build something like a Japanese + marketing-industry variant. The
pipeline is identical; the only thing you change is the calibration corpus.
 
1. Collect Japanese web text plus marketing documents (ad copy, landing pages,
   industry reports, newsletters, etc.)
2. Run it through `reap_calibrate` to measure saliency
3. Build a keep-640 plan and slice the GGUF
Three caveats if you actually do this.
 
**(1) Mix by token ratio, not byte ratio.** CJK is 3 bytes/char in UTF-8 but
packs densely under BPE, so mixing by bytes throws your ratios off. In
kimi-k3-mlx's measurements, Chinese at 15% by bytes came out to 32.6% by tokens.
The same thing happens with Japanese.
 
**(2) Get enough volume.** I pushed 260k tokens total. You want at least tens of
thousands of tokens for your target domain.
 
**(3) You can't know whether "marketing" separates as an expert until you
measure it.** What separates cleanly in the table above are the big axes —
language and code. Whether marketing register *within* Japanese has dedicated
experts is honestly doubtful; it probably overlaps heavily with general Japanese
experts. In that case, mixing in marketing documents dilutes down to something
like reprioritization within the Japanese experts. If you're serious, I'd
recommend measuring the overlap rates by source first, then deciding.
 
And obviously, the capabilities you cut are genuinely gone. If you later want
your Japanese+marketing build to write a little SQL, it can't. Before you cut,
write down every job you want the agent to do, and put all of them in the
corpus.
 
Flipping that around, having multiple models is theoretically possible too. Keep
a coding model, an industry-specific model, and so on, and pick whichever is
most appropriate at the time — like SakanaAI's Fugu.
 
…Wait, actually? Why not one model where you select the experts each time?
You could prepare a K3-specific version of the small model at Fugu's core. MoE
switches experts per token, but as I recall Fugu's approach was to look at the
whole prompt and switch experts on that basis
(https://arxiv.org/html/2606.21228v1), so maybe you could select on the same
idea? Well, something to think about when the mood strikes.
 
## Running it
 
K3 support isn't merged upstream in llama.cpp yet, so use the Unsloth fork.
 
```bash
git clone https://github.com/unslothai/llama.cpp
cd llama.cpp && git fetch origin pull/48/head:kimi-k3 && git checkout kimi-k3
cmake -B build -DGGML_METAL=ON && cmake --build build -j --target llama-server
 
llama-server -m Kimi-K3-REAP640-IQ1_S-00001-of-00010.gguf \
    -ngl 99 -c 131072 --jinja --cache-reuse 0 --temp 1.0 --top-p 0.95
```
 
**`--cache-reuse 0` is mandatory.** K3's KDA (linear attention) carries
recurrent state, so partial reuse of the prefix cache corrupts it (a known issue
reported in PR #26185). Last time I wrote that llama-server's cache didn't seem
to be doing anything — with K3, it's a cache you must *not* let work.
 
Kimi Code CLI connects directly with OpenAI-compatible settings. After all the
pain of writing my own solver last time, this round the llama.cpp fork handles
K3's tool calling and thinking separation natively. Much appreciated 🙏
 
## Results
 
I sanity-checked with 3 tasks from SWE-Lancer IC SWE (Diamond) that K2.7 had
solved.
 
| Task | Result | Payout | Ref: MLX K3 |
|---|---|---|---|
| 28096_836 | pass | $500 | 0 |
| 18827_741 | pass | $1,000 | 0 |
| 29618_781 | pass | $500 | 0 |
 
3.4 hours for 3 tasks (68 min/task). Kimi Code CLI inside the container did all
the execution, and grading is stock SWE-Lancer, entirely untouched.
 
I also did a retry round: 5 tasks picked from the 105 that K2.7 failed last
time.
 
| Task | K2.7 (2-bit, 341GB) | K3 REAP640 (1-bit, 441GB) | Payout |
|---|---|---|---|
| 14294 | fail | fail | - |
| 24508_791 | fail | pass | $1,000 |
| 15815_1 | fail | fail | - |
| 27353_776 | fail | pass | $500 |
| 15925 | fail | fail | - |
 
2 out of 5. Problems K2.7 couldn't solve, cleared by a K3 cut down to 1 bit with
256 experts thinned out. n=5, so I won't push this hard, but it looks quite
likely that even after all that cutting it outperforms K2.7.
 
I could get an actual benchmark by running all 198 tasks, but decode is ~3.0
tok/s and prefill ~47 tok/s. It's slower than last time's K2.7, so a full run is
a 60-day affair.
 
More than four times last time's K2.7 (13 days). Hmm. That's rough — K3 would
probably go obsolete mid-run and the whole thing would end up shelved…
 
## Caveats
 
- I did not run the full 198 tasks. 3 sanity checks + 5 retries = n=8
- No perplexity, no standard benchmarks. I only looked at whether it works as an
  agent
- Chinese and Japanese are broken by design (deliberately excluded from
  calibration)
- Vision is unverified. mmproj is not included
## Summary
 
Even K3, which doesn't fit on a Mac Studio at 594GB in 1-bit, confirmed running
on a 512GB Mac Studio in exchange for 256 experts!!!!!!! This trick is only
possible because it's MoE, woohoo!!!!!
 
Thanks for reading all the way to the end.
 
## Notes and references
 
- **Model**: [hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF](https://huggingface.co/hellohazime/Kimi-K3-REAP640-IQ1_S-GGUF)
  (441.4GB, 10 shards). GGUF slicing code and tests:
  [01554/kimi-k3-gguf-prune](https://github.com/01554/kimi-k3-gguf-prune) (MIT).
- **Base model**: [unsloth/Kimi-K3-GGUF](https://huggingface.co/unsloth/Kimi-K3-GGUF),
  UD-IQ1_S (594GB). The dynamic quantization design (critical layers at 8/16
  bit, router and layernorm protected at high precision) is documented in
  [Unsloth's docs](https://unsloth.ai/docs/models/kimi-k3) and the
  [DeepSeek-R1 1.58bit post](https://unsloth.ai/blog/deepseekr1-dynamic). The
  note that uniform low-bit quantization causes infinite repetitions is also
  from the latter.
- **REAP**: [CerebrasResearch/reap](https://github.com/CerebrasResearch/reap).
  Measures expert saliency as router gate × output norm.
- **K3 support in llama.cpp**:
  [PR #26185](https://github.com/ggml-org/llama.cpp/pull/26185) (unmerged). The
  KDA+MLA hybrid, the XTML tool-call parser, and the reason `--cache-reuse 0` is
  required (KDA recurrent state corruption) are all covered in that PR's
  discussion. Built from the Unsloth fork's PR 48 branch.
- **The MLX REAP build**: pipenetwork's
  [kimi-k3-mlx](https://github.com/PipeNetwork/kimi-k3-mlx)
  (Kimi-K3-REAP73/REAPgraded). The saliency measurement and selection in this
  post use that repo's scripts (`reap_calibrate.py` / `reap_plan.py`) as-is. The
  findings that anything absent from the calibration corpus gets silently
  reaped, and that en+code calibration causes total collapse in Chinese, are
  also measurements from that repo's README. So: the author of the MLX build I
  described as not working for me is the same person whose entire pruning
  toolkit I'm relying on.
- **Evaluation environment**: SWE-Lancer (OpenAI, arXiv:2502.12115), IC SWE
  Diamond. Agent is [MoonshotAI/kimi-code](https://github.com/MoonshotAI/kimi-code)
  v0.30.0 run inside the task container (`kimi -p`; note that `--auto` and `-y`
  can't be combined with `-p`). Grading unmodified.
- **Previous post**: [I had a 2-bit Kimi K2.7 Code earn $69,875 on a single Mac Studio](https://zenn.dev/hellohazime/articles/kimi_k27_code_swelancer_local)
