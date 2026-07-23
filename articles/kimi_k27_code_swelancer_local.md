---
title: "2bitに量子化したKimi K2.7 CodeにMac Studio 1台で$69,875を稼いでもらった"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "llamacpp", "Kimi", "SWELancer", "量子化"]
published: true
---

お疲れ様です波浪です。

Kimi K3も出てるので、出すタイミングを失ってしまったんですが、いまさらK2.7の話です。
これ以上伸ばしてK3のオープンウェイトでちゃったらまじで出すタイミング失うなと思って、恥を忍んで提出します。

遡ること先月の事、2026年6月17日から6月30日まで、Mac Studio(Apple M3 Ultra、512GB)1台で、Moonshot AI の Kimi K2.7 Code に SWE-Lancer の IC SWE(Diamond)198タスクを解かせました。

かかった時間は308時間34分、13日弱。93タスク正解の47.0%、獲得報酬は $69,875 / $189,300(36.9%)でした。

かなりの長時間かかったので、成果をとっとと出したかったんですが、Fableが、へへへ、使い切りたいという欲で、へへへ、他のことしてました。

と、いうわけで、周回遅れの話で申し訳ないですが Kimi K2.7の話になります
動かしたのは Unsloth の GGUF、2bit 量子化版(UD-Q2_K_XL、約341GB)を llama-server で動かしてます。

SWE-Lancer はデフォルトのままだと動かなかったので。ベンチマーク付属のソルバーを私が Kimi 向けに書き直しています、具体的なコードは

https://github.com/01554/frontier-evals/blob/kimi-k2.7-code-tool-aware-solver/project/swelancer/swelancer/solvers/swelancer_agent/solver.py

ここに置いておきます。


## 環境

Mac Studio(Apple M3 Ultra、512GB)
モデルは Kimi K2.7 Code の Unsloth GGUF、UD-Q2_K_XL(2bit、約341GB)
llama-server にコンテキスト長 262,144 トークンで載せました。

ベンチマーク側は、OpenAI が公開している SWE-Lancer の評価コードと、それに付属するエージェントソルバーをKimi K2.7にあわせて修正。
13日間で送った入力は累計2億7,133万トークン、出力は267万トークン。1タスクの平均は約1.6時間でした。

## ソルバーを直すまでは 3.85% だった

テストで回した1回目は、52タスクを採点した時点で正解2つ、3.85%。そこで止めました。

ログを見ると、Kimi が問題を解けていないのではなく、答えを出せていなかった。SWE-Lancer 付属の標準ソルバー(SimpleAgentSolver)は、モデルに ```python … ``` のコードブロックを本文へ書かせ、それを取り出して実行します。

ところが Kimi K2.7 Code はツール呼び出しの作法を学習で強く身につけていて、そのコードブロックを書かず、存在しない `functions.python` のようなツール呼び出しを延々と出していることが確認できました。

そして一度もコードを動かせないまま100ターンを使い切る、そういうタスクばかりだったので、先ほども書いた通りSolverを書き直すことにしました。

まず、OpenAI 互換の関数ツール `python`(引数は `code`)を定義して、モデルにそれを呼ばせる新しいソルバー(ToolAwareAgentSolver)を用意。
コードブロックを書かせるのをやめ、Kimi が訓練で慣れている「ツールを呼ぶ」形をそのまま使わせました。

最終ランの198タスクすべてで、Kimi はこのツール経由でコードを実行。それでも Kimi がツール呼び出しの生の記法(`<|tool_call_argument_begin|>` など)や JSON の `code` 欄でコードを出してくることがあったので、そこから Python を取り出して実行する処理も足しました。

この拾い上げは198タスク中21タスクで働いており入れないとベンチ結果はもっと落ちたはずでうs。

にしてもプロンプトにも「`functions.*` というツールは無い、実行できるのは `python` だけ」と書いているんですが、やっぱPromptによる禁止は効果が薄いですね。

あと実行基盤を テストの時はLM Studioでしたが、長丁場になりそうだったのっで llama.cpp に替え、コンテキストも 262,144 トークンに広げ、
なんかDocker のコンテナ削除がぶつかって(409 Conflict)タスクがエラー扱いになる不具合と戦ったりしてました。（愚痴）


## 成果

| 報酬額帯 | タスク数 | 正解 | 正解率 | 報酬総額 | 獲得額 |
|---|---|---|---|---|---|
| $500未満 | 87 | 38 | 44% | $21,300 | $9,375 |
| $500〜999 | 62 | 33 | 53% | $31,000 | $16,500 |
| $1,000〜1,999 | 27 | 14 | 52% | $27,000 | $14,000 |
| $2,000〜3,999 | 9 | 3 | 33% | $18,000 | $6,000 |
| $4,000以上 | 13 | 5 | 38% | $92,000 | $24,000 |

高い案件ほど取りこぼしてますね。
$2,000 を超えると正解率が落ちる。$4,000 以上の13タスクは報酬の総額が $92,000 とセットの半分近くを占めるのに、取れたのは $24,000。
いちばん高い $32,000 の案件も落とした。
正解率は47.0%なのに獲得額の率が36.9%にとどまっているのはちょっと悲しい。

SWE-Lancer の論文が出た2025年2月の時点で、いちばん強かったのは Anthropic の Claude 3.5 Sonnet で、IC SWE(Diamond)は26.2%(出典: SWE-Lancer 論文)。その1年4か月後に、2bit まで削ったオープンウェイトのモデルが、店で買えるデスクトップ1台で47.0%を出したと考えれば上々ではありますね。


## 実装のせいで数字が実際の落ちている件

実は本ベンチで作ったSolverは応答の打ち切り部分が入っています。
今回の採点環境には、モデルの1回の応答が900秒を超えたらそこで止めて、途中の状態のまま採点する、という上限が入っており。
元の SWE-Lancer ソルバーには無い制限で、私が足したものです。この900秒の打ち切りは198タスク中109タスク、半分以上で少なくとも一度起きていて、そのうち75タスクは0点でした。

最後まで待っていれば通っていたものもあったはずで、47.0%は半分以上のタスクを途中で切った上での数字になります。
いやだってね、これいれないと、さらに時間がかかったわけで！！

いやまあ、Fableがこんなに長引いて結局、ずっとつかえるよーってなるなら、こんな制限入れずに流してもよかったかなーなんて思いますが！？それは結果論でして
Fable前までに終わらせたーいっていう見積もりからするとこうするのがあの時は妥当だったんですよ！！！（言い訳）


思考(reasoning)は有効にして走らせてます。llama-server 側で on にしています。

## GLM-5.2 も試したが、61日かかる計算で諦めた

同じ Mac Studio、同じ採点環境で、GLM-5.2(754B、Unsloth GGUF の UD-Q2_K_XL、約236GB)にも同じ198タスクをやらせようとしたんですが
1タスクに8時間前後かかり、198タスクだと約61日になる計算なので諦めました。
ターンが積もって会話が伸びると毎秒2.2トークンまで落ちてて。数タスク回したところで止めました。

## まとめ

MacStudio一台で、SWE-Lancer の Diamond(IC SWE)は完走できる。ただし直列で13日！
でもでも、2bit の Kimi K2.7 Code で47.0%、$69,875 分を稼げました！やったね！！！二週間弱で $7万なら、大金持ちじゃね？

## 次回に向けて
ClaudeCodeをハーネスにした方が動きが良さそうな気がするけど、さらに重くなるからうーん？ 
あと llama serverのキャッシュが効いてない気がするのでそこも次やるなら解決したい。


以上、最後まで読んでくださりありがとうございました。


## 文献ノート

- SWE-Lancer 論文: "SWE-Lancer: Can Frontier LLMs Earn $1 Million from Real-World Freelance Software Engineering?"(OpenAI、2025年2月、arXiv:2502.12115、査読前プレプリント)。Claude 3.5 Sonnet の IC SWE(Diamond)26.2%はこの論文の値。
- Moonshot AI 公式リソースページ: https://www.kimi.com/resources/kimi-k2-7-code (2026年7月22日閲覧)。評価条件(Kimi Code CLI、thinking 有効、temperature 1.0、top-p 0.95)の記載あり。SWE-Lancer のスコア掲載なし。
- モデル: unsloth/Kimi-K2.7-Code-GGUF の UD-Q2_K_XL(Hugging Face)。
- ソルバーの改修: `swelancer/solvers/swelancer_agent/solver.py`。ネイティブ `python` ツールを与える `ToolAwareAgentSolver` の追加、崩れたツール呼び出し出力からの Python 抽出、ターン上限・応答タイムアウト・コンテキスト上限の設定化を含む。改修版のコードは OpenAI 版 SWE-Lancer(`openai/frontier-evals`)を fork した次のブランチに置いた: https://github.com/01554/frontier-evals/tree/kimi-k2.7-code-tool-aware-solver 。ソルバー本体は [`project/swelancer/swelancer/solvers/swelancer_agent/solver.py`](https://github.com/01554/frontier-evals/blob/kimi-k2.7-code-tool-aware-solver/project/swelancer/swelancer/solvers/swelancer_agent/solver.py)、再現用スクリプトは同ブランチの `project/swelancer/scripts/` にある。
- 本記事の一次データ: タスク別スコア・トークン数・獲得額をまとめた [results.csv](https://github.com/01554/frontier-evals/blob/kimi-k2.7-code-tool-aware-solver/project/swelancer/runs/2026-06-17T09-21-13-UTC_run-group_tool-aware-solver/results.csv)(198タスク分、上記 fork ブランチに収録)。正解数・獲得額・トークン数・価格帯別内訳はすべてこの CSV から集計した。ソルバー経路の内訳(全198タスクでネイティブツール実行、うち21タスクでフォールバック抽出が作動)は同じ実行の run.log から数えた(run.log 自体はローカルのみ)。
