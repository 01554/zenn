---
title: "ラズパイにNemoClawを入れてLAN越しのMacのQwen3.5を使う"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [RaspberryPi,NemoClaw,Qwen,Ubuntu,LLM]
published: false
---

お疲れ様です、OpenClaw気にはなっていたけど怖くて手をだせていなかったチキン野郎の波浪です
そんなチキンのためにNVIDIAからザリガニ水槽が出たので、遅ればせながら参加してみました。

大抵の家に転がってるであろうラズパイ、僕の家にもやっぱり転がっていたのでこれにインストールして、NVIDIAのLLM使えって言われたけどそれは拒否してMac上に立てたQwen3.5を使わせてみました。



## この記事のポイント

- Raspberry Pi5(8GB)にUbuntu 24.04を入れてNVIDIA NemoClawをセットアップ
- MacでホストしているQwen3.5-397B(llama.cpp)にLAN経由で接続
- NemoClawの推論先を、ローカルLLMに変更



## 構成

```
[Raspberry Pi 5]                    [Mac]
Ubuntu 24.04 LTS (aarch64)         macOS
NemoClaw (Agent Runtime)     ──→    llama.cpp (port 8016)
                              LAN   Qwen3.5-397B-A17B
192.168.0.206                       192.168.0.77
```


## 手順

### 1. ラズパイを完全初期化してUbuntu 24.04 desktopを入れる

Raspberry Pi ImagerでUbuntu 24.04.4 LTS desktopを焼いて起動。
ServerじゃなくてDesktopにしたのは、知り合いがChromeのヘッド有りが使いたいのに使えないと不満を述べていたので
画面側もいじれるところがザリガニはいいんか？いいんか？と思いDesktopにしました




### 2. Mac 上のQwen3.5への疎通確認

```bash
# モデル一覧
curl http//192.168.0.778016/v1/models

# チャット
curl -X POST http//192.168.0.778016/v1/chat/completions \
  -H "Content-Type application/json" \
  -d '{"model""Qwen3.5-397B-A17B-UD-Q4_K_XL-00001-of-00006.gguf","messages"[{"role""user","content""Hello"}],"max_tokens"50}'
```

### 3. NemoClawのインストール

当然ながらUbuntu 24.04の初期状態では必要なパッケージが入っていない。NemoClawのセットアップ前にまとめて入れておきます。
あと途中でNVIDAのAPI_KEY入れろとか言われます、入れないと進まないのでNVIDIAのアカウントを作るだけ作りましょう。


```bash
sudo apt install -y curl git docker.io python3-pip
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```
dockerグループの反映のため、一度ログアウト・再ログイン。


その後NemoClawのインストール
```bash
curl -fsSL https//nvidia.com/nemoclaw.sh | bash
```

Node.js v24.14.0 + nemoclaw CLIがインストールされる。

### 4. cgroup v2の設定

NemoClawのOpenShellゲートウェイはk3sをDocker内で動かすため、cgroup v2環境では追加設定が必要

```bash
nemoclaw setup-spark
```

これで `/etc/docker/daemon.json` に `"default-cgroupns-mode" "host"` が追加されてDockerが再起動される。

`setup-spark` はDGX Spark用のスクリプトなので、vLLMもインストールしようとする。しかしラズパイ環境ではシステムの `jsonschema` パッケージとpipが競合して失敗する

```
Cannot uninstall jsonschema 4.10.3, RECORD file not found
```

LAN越しにLLMを使う場合はvLLMは不要なので、`setup-spark` はここらでSkip `nemoclaw onboard` に進んで問題ない。cgroupの設定さえ済んでいればOK。

### 5. nemoclaw onboard

```bash
nemoclaw onboard
```

約10分で全ステップ完了

```
[1/7] Preflight checks        ✓ Docker, openshell, cgroup OK
[2/7] Starting OpenShell gateway  ✓ https//127.0.0.18080
[3/7] Creating sandbox         ✓ "my-assistant" (Dockerイメージビルド含む)
[4/7] Configuring inference    → nvidia/nemotron-3-super-120b-a12b (NVIDIAクラウド)
[5/7] Setting up inference provider  ✓ nvidia-nim プロバイダー作成
[6/7] Setting up OpenClaw      ✓ サンドボックス内でgateway起動
[7/7] Policy presets           pypi, npm を適用
```

サンドボックスUI `http//127.0.0.118789/`

### 6. サンドボックスに接続

```bash
nvm use 24 && nemoclaw my-assistant connect
```

onboard中にNode v22がインストールされてデフォルトが変わることがある。`nemoclaw` コマンドが見つからない場合は `nvm use 24` でNode v24に切り替える。

### 7. 推論先をMac のQwen3.5に切り替える

onboard完了時点での推論設定
- **プロバイダー** `nvidia-nim`
- **モデル** `nvidia/nemotron-3-super-120b-a12b`
- **ルート** `inference.local` → NVIDIAクラウド

サンドボックスはネットワーク名前空間で完全隔離されており、LAN(`192.168.0.x`)への直接通信はできない。ゲートウェイ側でinference providerの向き先を変える

```
サンドボックス → inference.local443 (TLS)
    → ゲートウェイプロキシ (10.200.0.1)
    → ホスト側のプロバイダー
    → http//192.168.0.778016 (Macllama.cpp)
```

ゲートウェイはラズパイのホスト上で動いているのでLANに到達できる。

```bash
# サンドボックスの外（ラズパイ側）で実行

# 現在のプロバイダー確認
openshell provider get nvidia-nim
# → Config keys OPENAI_BASE_URL

# MacのURLに書き換え
openshell provider update nvidia-nim --config OPENAI_BASE_URL=http//192.168.0.778016/v1
# → ✓ Updated provider nvidia-nim
```

加えて、サンドボックス内の `~/.openclaw/openclaw.json` の `baseUrl` も `inference.local` 経由に設定する

```bash
# サンドボックス内で実行
python3 -c "
import json
with open('/sandbox/.openclaw/openclaw.json') as f
    c = json.load(f)
c['models']['providers']['nvidia']['baseUrl'] = 'https//inference.local/v1'
c['models']['providers']['nvidia']['apiKey'] = 'none'
c['models']['providers']['nvidia']['models'][0]['id'] = 'Qwen3.5-397B-A17B-UD-Q4_K_XL-00001-of-00006.gguf'
c['models']['providers']['nvidia']['models'][0]['name'] = 'Qwen3.5 397B A17B'
c['agents']['defaults']['model']['primary'] = 'nvidia/Qwen3.5-397B-A17B-UD-Q4_K_XL-00001-of-00006.gguf'
with open('/sandbox/.openclaw/openclaw.json', 'w') as f
    json.dump(c, f, indent=2)
print('done')
"
```
別にvimで書き換えてもいいです。

`baseUrl` は `http//192.168.0.778016/v1`（LAN直接）ではなく `https//inference.local/v1`（ゲートウェイ経由）にするのがポイント。サンドボックスからLANには直接到達できません。

### 8. 疎通確認

サンドボックス内から `inference.local` 経由でQwen3.5にアクセスできることを確認

```bash
curl https//inference.local/v1/models
# → server llama.cpp / Qwen3.5-397B のモデル情報が返る

curl -X POST https//inference.local/v1/chat/completions \
  -H "Content-Type application/json" \
  -d '{"model""Qwen3.5-397B-A17B-UD-Q4_K_XL-00001-of-00006.gguf","messages"[{"role""user","content""Hello"}],"max_tokens"10}'
# → "Hi there! 👋 How can I help you"
```

### 9. OpenClaw TUIでエージェント起動

```bash
openclaw tui
```

```
> Hello, what can you do?

Hey! 👋 I'm your personal AI assistant. I can
- Read/write files
- Search the web
- Run shell commands
- Send messages (Discord, Telegram, WhatsApp, etc.)
- Schedule reminders
- Spawn sub-agents
...
```

Qwen3.5-397BがNemoClawのエージェントとして動作することを確認。ラズパイからLAN越しにMac の推論を使い、サンドボックス内でファイル操作・Web検索・メッセージ送信などができる状態になった。
ファイル書き出しとかもやってくれました、通信はなんかBraveのAPI_KEYよこせとかいわれるのでまた今度で！


## ハマりポイント

| 問題 | 解決 |
|------|------|
| cgroup v2 問題 | `nemoclaw setup-spark` で設定 |
| nemoclaw command not found | `nvm use 24` でNode v24に切り替え |
| サンドボックスからLAN不通 | `openshell provider update` でゲートウェイ側を書き換え |
| openclaw.jsonのbaseUrl | `inference.local` 経由にする（LAN直接はNG） |

## 今日のまとめ

- NemoClawのサンドボックスはネットワーク名前空間で完全隔離。直接LAN通信は不可
- LLMの向き先を変えるなら `openshell provider update` が正解
- サンドボックス内の `openclaw.json` の `baseUrl` は `https//inference.local/v1` にする
- ゲートウェイが `inference.local` へのリクエストをプロバイダー設定に基づいてプロキシしてます


以上、よろしくお願いします。

## 参考

- [NVIDIA/NemoClaw](https//github.com/NVIDIA/NemoClaw)
- [Qwen3.5](https//unsloth.ai/docs/jp/moderu/qwen3.5)
