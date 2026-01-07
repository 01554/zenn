---
title: "SDDで作ったタスク(48個)を何時間かかろうとClaudeSkills+Vibekanbanで一気通貫、最後まで実行させる"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

お疲れ様です波浪です。

みなさんVibeKanban+AIエージェントで開発してますか？
自分も去年は大変お世話になりました。

さて、年も明けて2026年になったので環境をVerUPしたいなと思いSkillを見ていたら
これは可能性の塊ですね。いままで自分がちまちま動かしていたKanbanのオーケストレーション作業をClaudeCodeに全振りできたので嬉しくなって書いちゃった記事になります。


## Taskを作成

今回は手近にあったAgentCore runtime のpayload上限が100Mだから
AgentCore runtimeとS3を繋げてファイルをやり取りする形のpayloadに治す機能追加をやりたいと思います

はい開始
![](/images/2026/sdd_kanban_claudecode/init.png)

sddカスタムコマンドとしては cc-sddをベースに色々と改造したやつを愛用しています、
今回はcc-sddの使い方とかは説明しません。

雑にいまある機能の状態とやりたいことを書き連ねてタスク作成までいきます。
余談ですが、僕はこのcc-sddで言うところの init, requirements,validate-gap,design,validate-design 
この5つだけで一週間くらいかけることもありますが逆にこれより先の impl,validate-impl はほぼ脳死です。

ま、今回は記事書くために設計も脳死で進めてタスク作成までいきます。


## タスク作成完了

```
● タスク生成が完了しました。

  タスクサマリー

  | 項目             | 数量                                |
  |------------------|-------------------------------------|
  | メジャータスク   | 8                                   |
  | サブタスク       | 15                                  |
  | 要件カバレッジ   | 全 7 要件（28 acceptance criteria） |
  | 平均タスクサイズ | 1-3 時間/サブタスク                 |

  並列実行可能タスク

  (P) マーカー付きタスクは独立して並列実行可能：
  - 2.1, 2.2: S3MediaUploader（クライアント側）
  - 5.1, 5.2: S3MediaDownloader（サーバー側）
  - 7.1: インフラスクリプト

  品質検証

  - ✅ 全 7 要件がタスクにマッピング済み
  - ✅ タスク依存関係が適切（スキーマ → 機能 → 統合）
  - ✅ テストタスクが各機能に含まれる
  - ✅ E2E テストは slow マーカーでオプション（*）

  次のステップ

  タスクを確認し、実装を開始する場合：

  /kiro:spec-impl s3-file-transfer 1.1
```

## VibeKanbanへのタスク登録

さて、ではVibeKanbanに実装タスクを登録していきましょう
去年までは下の画像みたいな感じでスラッシュコマンド＋VibeKanban MCPで登録していたんですが

![](/images/2026/sdd_kanban_claudecode/タスク登録.png)
![](/images/2026/sdd_kanban_claudecode/mcp.png)

VibeKanbanのMCPって8個しか実装されておらず貧弱なんですよね
![](/images/2026/sdd_kanban_claudecode/mcp_tools.png)
初めはMCPを拡張した独自VibeKanbanを作ろうと考えていたんですがよく考えたらVibeKanbanもAIエージェントも全部同じDocker内で動かしている。

だったら、バックエンドのAPI直接叩いた方が早くね？と気がついたのでClaudeSkillsを使って VibeKanbanのバックエンドAPIを叩いていきます。

## VibeKanban API
まずはVibeKanbanのAPI群をコードから作成します。

今使うのはTaskだけなのでTaskだけ載せときます。

### タスク (Tasks)

  | メソッド   | エンドポイント                     | 説明                |
  |--------|-----------------------------|-------------------|
  | GET    | /api/tasks                  | タスク一覧取得           |
  | POST   | /api/tasks                  | タスク作成             |
  | GET    | /api/tasks/stream/ws        | タスク一覧 (WebSocket) |
  | POST   | /api/tasks/create-and-start | タスク作成 + Attempt開始 |
  | GET    | /api/tasks/{id}             | タスク詳細取得           |
  | PUT    | /api/tasks/{id}             | タスク更新             |
  | DELETE | /api/tasks/{id}             | タスク削除             |
  | POST   | /api/tasks/{id}/share       | タスク共有             |

<details>
  <summary>VibeKanbanで使えるその他API群</summary>

###  Task Attempts (Workspace)

  | メソッド | エンドポイント                                    | 説明                |
  |------|--------------------------------------------|-------------------|
  | GET  | /api/task-attempts                         | Attempt一覧取得       |
  | POST | /api/task-attempts                         | Attempt作成         |
  | GET  | /api/task-attempts/{id}                    | Attempt詳細取得       |
  | POST | /api/task-attempts/{id}/stop               | 実行停止              |
  | GET  | /api/task-attempts/{id}/branch-status      | ブランチ状態取得          |
  | GET  | /api/task-attempts/{id}/diff/ws            | Diff (WebSocket)  |
  | GET  | /api/task-attempts/{id}/repos              | リポジトリ一覧           |
  | GET  | /api/task-attempts/{id}/children           | 子タスク取得            |
  | POST | /api/task-attempts/{id}/run-agent-setup    | エージェントセットアップ実行    |
  | POST | /api/task-attempts/{id}/gh-cli-setup       | GitHub CLI セットアップ |
  | POST | /api/task-attempts/{id}/start-dev-server   | 開発サーバー起動          |
  | POST | /api/task-attempts/{id}/run-setup-script   | セットアップスクリプト実行     |
  | POST | /api/task-attempts/{id}/run-cleanup-script | クリーンアップスクリプト実行    |
  | POST | /api/task-attempts/{id}/open-editor        | エディタで開く           |

###  Git 操作

  | メソッド | エンドポイント                                      | 説明          |
  |------|----------------------------------------------|-------------|
  | POST | /api/task-attempts/{id}/merge                | マージ         |
  | POST | /api/task-attempts/{id}/push                 | プッシュ        |
  | POST | /api/task-attempts/{id}/push/force           | 強制プッシュ      |
  | POST | /api/task-attempts/{id}/rebase               | リベース        |
  | POST | /api/task-attempts/{id}/conflicts/abort      | コンフリクト中止    |
  | POST | /api/task-attempts/{id}/change-target-branch | ターゲットブランチ変更 |
  | POST | /api/task-attempts/{id}/rename-branch        | ブランチ名変更     |

###  PR 操作

  | メソッド | エンドポイント                             | 説明       |
  |------|-------------------------------------|----------|
  | POST | /api/task-attempts/{id}/pr          | PR作成     |
  | POST | /api/task-attempts/{id}/pr/attach   | 既存PR紐付け  |
  | GET  | /api/task-attempts/{id}/pr/comments | PRコメント取得 |

###  セッション (Sessions)

  | メソッド   | エンドポイント                      | 説明             |
  |--------|------------------------------|----------------|
  | GET    | /api/sessions                | セッション一覧取得      |
  | POST   | /api/sessions                | セッション作成        |
  | GET    | /api/sessions/{id}           | セッション詳細取得      |
  | POST   | /api/sessions/{id}/follow-up | 会話継続 (フォローアップ) |
  | POST   | /api/sessions/{id}/queue     | メッセージキュー追加     |
  | DELETE | /api/sessions/{id}/queue     | キューキャンセル       |

###  実行プロセス (Execution Processes)

  | メソッド | エンドポイント                                          | 説明                 |
  |------|--------------------------------------------------|--------------------|
  | GET  | /api/execution-processes/{id}                    | プロセス詳細取得           |
  | POST | /api/execution-processes/{id}/stop               | プロセス停止             |
  | GET  | /api/execution-processes/{id}/repo-states        | リポジトリ状態取得          |
  | GET  | /api/execution-processes/{id}/raw-logs/ws        | 生ログ (WebSocket)    |
  | GET  | /api/execution-processes/{id}/normalized-logs/ws | 正規化ログ (WebSocket)  |
  | GET  | /api/execution-processes/stream/ws               | プロセス一覧 (WebSocket) |

###  プロジェクト (Projects)

  | メソッド   | エンドポイント                           | 説明                   |
  |--------|-----------------------------------|----------------------|
  | GET    | /api/projects                     | プロジェクト一覧取得           |
  | POST   | /api/projects                     | プロジェクト作成             |
  | GET    | /api/projects/stream/ws           | プロジェクト一覧 (WebSocket) |
  | GET    | /api/projects/{id}                | プロジェクト詳細取得           |
  | PUT    | /api/projects/{id}                | プロジェクト更新             |
  | DELETE | /api/projects/{id}                | プロジェクト削除             |
  | GET    | /api/projects/{id}/repositories   | リポジトリ一覧              |
  | GET    | /api/projects/{id}/remote/members | リモートメンバー取得           |
  | GET    | /api/projects/{id}/search         | ファイル検索               |
  | POST   | /api/projects/{id}/open-editor    | エディタで開く              |
  | POST   | /api/projects/{id}/link/create    | リモートプロジェクト作成・リンク     |

###  リポジトリ (Repos)

  | メソッド | エンドポイント                  | 説明       |
  |------|--------------------------|----------|
  | POST | /api/repos               | リポジトリ登録  |
  | POST | /api/repos/init          | リポジトリ初期化 |
  | GET  | /api/repos/{id}/branches | ブランチ一覧取得 |

###  画像 (Images)

  | メソッド   | エンドポイント                             | 説明         |
  |--------|-------------------------------------|------------|
  | POST   | /api/images                         | 画像アップロード   |
  | GET    | /api/images/{id}/file               | 画像ファイル取得   |
  | DELETE | /api/images/{id}                    | 画像削除       |
  | GET    | /api/images/task/{task_id}          | タスクの画像一覧   |
  | GET    | /api/images/task/{task_id}/metadata | タスク画像メタデータ |

###  タグ (Tags)

  | メソッド   | エンドポイント        | 説明     |
  |--------|----------------|--------|
  | GET    | /api/tags      | タグ一覧取得 |
  | POST   | /api/tags      | タグ作成   |
  | PUT    | /api/tags/{id} | タグ更新   |
  | DELETE | /api/tags/{id} | タグ削除   |

###  承認 (Approvals)

  | メソッド | エンドポイント                     | 説明   |
  |------|-----------------------------|------|
  | POST | /api/approvals/{id}/respond | 承認応答 |

###  設定 (Config)

  | メソッド | エンドポイント                        | 説明              |
  |------|--------------------------------|-----------------|
  | GET  | /api/info                      | システム情報取得        |
  | PUT  | /api/config                    | 設定更新            |
  | GET  | /api/sounds/{sound}            | サウンドファイル取得      |
  | GET  | /api/mcp-config                | MCP設定取得         |
  | POST | /api/mcp-config                | MCP設定更新         |
  | GET  | /api/profiles                  | プロファイル取得        |
  | PUT  | /api/profiles                  | プロファイル更新        |
  | GET  | /api/agents/check-availability | エージェント利用可能性チェック |

###  認証 (OAuth)

  | メソッド | エンドポイント                    | 説明        |
  |------|----------------------------|-----------|
  | POST | /api/auth/handoff/init     | 認証開始      |
  | GET  | /api/auth/handoff/complete | 認証完了      |
  | POST | /api/auth/logout           | ログアウト     |
  | GET  | /api/auth/status           | 認証状態取得    |
  | GET  | /api/auth/token            | トークン取得    |
  | GET  | /api/auth/user             | 現在のユーザー取得 |

###  ファイルシステム (Filesystem)

  | メソッド | エンドポイント                   | 説明         |
  |------|---------------------------|------------|
  | GET  | /api/filesystem/directory | ディレクトリ一覧   |
  | GET  | /api/filesystem/git-repos | Gitリポジトリ一覧 |

###  コンテナ (Containers)

  | メソッド | エンドポイント                         | 説明              |
  |------|---------------------------------|-----------------|
  | GET  | /api/containers/attempt-context | Attemptコンテキスト取得 |

###  共有タスク (Shared Tasks)

  | メソッド   | エンドポイント                | 説明       |
  |--------|------------------------|----------|
  | POST   | /api/shared-tasks/link | 共有タスクリンク |
  | DELETE | /api/shared-tasks/{id} | 共有タスク削除  |

###  組織 (Organizations)

  | メソッド   | エンドポイント                                   | 説明     |
  |--------|-------------------------------------------|--------|
  | GET    | /api/organizations                        | 組織一覧取得 |
  | POST   | /api/organizations                        | 組織作成   |
  | GET    | /api/organizations/{id}                   | 組織詳細取得 |
  | PATCH  | /api/organizations/{id}                   | 組織更新   |
  | DELETE | /api/organizations/{id}                   | 組織削除   |
  | GET    | /api/organizations/{id}/invitations       | 招待一覧   |
  | POST   | /api/organizations/{id}/invitations       | 招待作成   |
  | GET    | /api/organizations/{id}/members           | メンバー一覧 |
  | DELETE | /api/organizations/{id}/members/{user_id} | メンバー削除 |
  | GET    | /api/invitations/{token}                  | 招待詳細取得 |
  | POST   | /api/invitations/{token}/accept           | 招待承諾   |

###  その他

  | メソッド | エンドポイント      | 説明        |
  |------|--------------|-----------|
  | GET  | /api/health  | ヘルスチェック   |
  | GET  | /api/scratch | スクラッチ一覧   |
  | GET  | /api/events  | イベントストリーム |

</details>

## Skill化
めんどうな人は全部一気にSkillsにしてもいいと思いますが
僕はチキンの似非Vibeエンジニャーなので少しずつ確認しながら進めます。

今回はTask登録に関係する以下のAPIを対象にします


  | メソッド   | エンドポイント                                | 説明                 |
  |--------|----------------------------------------|--------------------|
  | GET    | /api/projects                     | プロジェクト一覧取得           |
  | POST   | /api/tasks                             | タスク作成              |

### Skill-creator をインストール

ClaudeCode上からskill-creator をインストールします
![](/images/2026/sdd_kanban_claudecode/plugin.png)

/skills を使うにはVerが古かったので updateします

`npm i -g @anthropic-ai/claude-code`

![](/images/2026/sdd_kanban_claudecode/2.0.76.png)

### skill-creator で 作成
依頼メッセージ
```
 | メソッド   | エンドポイント                                | 説明                 |
    |--------|----------------------------------------|--------------------|
    | GET    | /api/projects                     | プロジェクト一覧取得           |
    | POST   | /api/tasks                             | タスク作成              |
  ---
  この二つのAPIを使うスキルを作成して。
  プロジェクト一覧を取得して作業中のフォルダ名と一致するプロジェクトに対して
  指定したタスクの一覧（まとめてファイルで指定されるか、個別にユーザーが指定するか）をVibeKa
  nbanに登録するスキルです
```

AIからの完了メッセージ
```
● スキル作成完了しました。

  /workspace/.claude/skills/vk-register-tasks/SKILL.md  ← スキル本体
  /workspace/vk-register-tasks.skill                    ← 配布用パッケージ

  スキルの内容:
  - ポート検出: /tmp/vibe-kanban/vibe-kanban.port から取得
  - プロジェクトマッチング: フォルダ名で自動判定
  - tasks.md パース: 親タスクのみ登録（サブタスクは Description に含める）
  - REST API: POST /api/tasks で登録
  - 降順登録: タスク8 → 7 → ... → 1

  トリガー例:
  - 「タスクを Vibe Kanban に登録して」
  - 「tasks.md を登録」
  - 「VibeKanban にタスク連携」

  他のプロジェクトで使う場合は .skill ファイルをインストールするか、.claude/skills/ にコピーしてください。
```

## Claude Skillインストール 
はい、指示にしたがって `.claude/skills/vk-register-tasks`を使いたいプロジェクトにコピーしてから Claudeを起動します

`$ claude --dangerously-skip-permissions`

![](/images/2026/sdd_kanban_claudecode/register-task起動.png)

![](/images/2026/sdd_kanban_claudecode/task登録1.png)

問題なく動いてますね。

<details>
<summary>出力されたvk-register-tasksのSKILL.md</summary>

```
---
name: vk-register-tasks
description: Kiro spec の tasks.md を Vibe Kanban に REST API で登録する。「タスクを Vibe Kanban に登録」「tasks.md を登録」「VibeKanban にタスク連携」などと言われた時に使用。Kiro spec-driven development のタスクを管理ツールに連携したい時にトリガー。
---
```

# Vibe Kanban タスク登録

Kiro spec の `.kiro/specs/{feature}/tasks.md` から実装タスクを抽出し、Vibe Kanban REST API で登録する。

## ワークフロー

### 1. ポート検出
```bash
VIBE_PORT=$(cat /tmp/vibe-kanban/vibe-kanban.port 2>/dev/null)
```
ファイルなし → Vibe Kanban 未起動。

### 2. プロジェクトマッチング
```bash
curl -s "http://localhost:$VIBE_PORT/api/projects"
```
レスポンス `data[]` から現在フォルダ名（`basename $(pwd)`）と `folder_name` または `name` が一致するプロジェクトを特定。不一致時は一覧表示して選択を促す。

### 3. feature-name 特定
引数で指定されていなければ `.kiro/specs/` 配下を一覧表示して選択を促す。

### 4. tasks.md パース
`.kiro/specs/{feature}/tasks.md` を読み込み。形式:
```
- [ ] 1. 親タスクタイトル
- [ ] 1.1 サブタスク詳細
  - 詳細項目
  - _Requirements: X.X_
```
親タスク（`1.`, `2.` 等）のみ登録対象。サブタスクは Description に含める。

### 5. タスク構築
**Title**: `{TaskNo} {英語タイトル} {feature-name}`
例: `1. Extend MediaItem schema s3-file-transfer`

**Description**:
```markdown
以下のタスクを `.claude/commands/kiro/spec-impl.md` の指示に従って実行する

feature-name: **{feature-name}**
task: **{TaskNo}**

## Task内容
{tasks.md から対象タスクとサブタスクを転記}

## REDテスト
- {完了確認のためのテスト内容}

## REFACTOR
- {見直しポイント}
```

### 6. 登録（降順）
タスク8 → 7 → ... → 1 の順で登録:
```bash
curl -s -X POST "http://localhost:$VIBE_PORT/api/tasks" \
  -H "Content-Type: application/json" \
  -d '{"project_id":"<UUID>","title":"<タイトル>","description":"<説明>","status":"todo"}'
```


</details>


## タスクの実行計画を修正
この方法でAPIは普通に叩けるなとわかったんで
実際の実行計画に合わせていきます。

自分はCodexとClaudeCodeで実装を行い、かつ検証をClaudeCodeとCodexで行い、両方でGOが出たらOK
どちらかでNGが出たら、実装を修正させて再検証という手順を踏んでいます。
どちらの実装もGOの場合はCodexの実装を優先して取り込みます。

つまりsddで作成したタスク1つにつき6つのVibeKanbanタスクが作られます。
```
taskX - Codex impl taskX  
        + validate Codex - Codex impl taskX
        + validate Claude - Codex impl taskX                    
      - ClaudeCode impl taskX
        + validate Codex - ClaudeCode impl taskX
        + validate Claude - ClaudeCode impl taskX                    
```
            
また、NGが出た場合は、修正タスクに対してNGを出したAIエージェントのinspectを増やします

```
taskX - Codex impl taskX  
        + validate Codex - Codex impl taskX　🆖
        + validate Claude - Codex impl taskX  🆗
```

であれば         

```
taskX - Codex impl taskX 追加実装  
        + validate Codex - Codex impl taskX　追加実装  
        + validate Codex - Codex impl taskX　追加実装  
        + validate Claude - Codex impl taskX  追加実装  
```
こうなるようにvalidateタスクをDuplicateしています。
あと 追加実装のvalidateは Rebaseが必要になるのでそこも手でぽちぽちしてました。
オーケストレーション俺！！！！


今回はタスクが8個できたので、VibeKanbanで実行するタスク数としては最低48個ですね。
リトライが入った場合はさらに増えます。


これを自分で管理しないで一気通貫にやってもらいます。
普通にやるとClaudeCodeがタイムアウトしちゃうんで、管理用のスクリプトを用意して
そこをうまいことやるぜ！ってのが今回の記事の趣旨ですね


## オーケストレーションskillを作成

まず検証タスクの検証結果を取るためにSQLiteへのアクセスが必要になります
また検証結果がNGだった場合の修正依頼をするために実装タスクへfollow-up apiを使います。
リトライ回数は3回に制限して、それでも完了しない場合ユーザーにSlackで通知させます

というわけで出来たSKILLが以下

<details>
<summary>オーケストレーションskill</summary>

```
---
name: vk-orchestrator
description: Vibe Kanban のタスク進行を自動管理する。in_review のタスク確認、検証タスク作成・実行、GO/NO-GO 判定、PR 作成、CI 監視、マージまでを自動化。監視スクリプトから「in_review にデータが入った」と通知された時、または手動で「タスクを進めて」と指示された時に使用。
---
```

# Vibe Kanban オーケストレーター

タスクの状態を確認し、次のアクションを自動実行する。

## ワークフロー概要

```
状況確認
  │
  ├─ in_progress=0, in_review=0 → 次のタスクを開始
  │
  └─ in_review にタスクあり
       ├─ 実装タスク（検証なし）→ 検証タスク4つ作成・実行
       ├─ 実装タスク（検証済）→ PR作成 → CI監視 → マージ
       └─ 検証タスク → GO/NO-GO 判定
```

## 状況確認

`scripts/get_tasks_status.py` でプロジェクトの状態を取得:

```bash
python3 scripts/get_tasks_status.py <folder_name>
```

## 判定ロジック

### 0. in_progress=0, in_review=0 → 次のタスクを開始

todo から次のタスクを取得し、**Claude と Codex の両方で並列実行**を開始する。

**手順:**

1. **dev ブランチを最新化**
   ```bash
   git fetch origin dev
   git checkout dev
   git pull origin dev
   ```

2. **todo の先頭タスクを取得**
   ```python
   status = get_project_status(project_id)
   next_task = status["todo"][0]  # 作成日順で最初のタスク
   ```

3. **タスクを複製（Claude 用 / Codex 用）→ 元タスク削除**

   元タスクを複製して、それぞれにマーカーを付ける:
   ```bash
   VIBE_PORT=$(cat /tmp/vibe-kanban/vibe-kanban.port)

   # Claude 用タスク作成
   curl -X POST "http://localhost:$VIBE_PORT/api/tasks" \
     -H "Content-Type: application/json" \
     -d '{
       "project_id": "<project_id>",
       "title": "<original_title> [Claude]",
       "description": "<original_description>"
     }'

   # Codex 用タスク作成
   curl -X POST "http://localhost:$VIBE_PORT/api/tasks" \
     -H "Content-Type: application/json" \
     -d '{
       "project_id": "<project_id>",
       "title": "<original_title> [Codex]",
       "description": "<original_description>"
     }'

   # 元タスクを削除（重複実行を防ぐ）
   curl -X DELETE "http://localhost:$VIBE_PORT/api/tasks/<original_task_id>"
   ```

   **重要:** 元タスクを削除しないと、次回のアイドル検知で再度選ばれてしまう。

4. **各タスクで attempt 作成・実行開始**

   **repo_id の取得:**
   ```bash
   python3 scripts/get_repo_id.py <project_id>
   # 例: python3 scripts/get_repo_id.py 0748EE6C619A4A9B9190F090AF3DEEB1
   # 出力: e2d7267e-4873-01a6-6238-de08b793fd0f (UUID形式)
   ```

   **Claude タスク開始:**
   ```bash
   curl -X POST "http://localhost:$VIBE_PORT/api/task-attempts" \
     -H "Content-Type: application/json" \
     -d '{
       "task_id": "<claude_task_id>",
       "executor_profile_id": {
         "executor": "CLAUDE_CODE",
         "variant": "DEFAULT"
       },
       "repos": [{
         "repo_id": "<repo_id>",
         "target_branch": "dev"
       }]
     }'
   ```

   **Codex タスク開始:**
   ```bash
   curl -X POST "http://localhost:$VIBE_PORT/api/task-attempts" \
     -H "Content-Type: application/json" \
     -d '{
       "task_id": "<codex_task_id>",
       "executor_profile_id": {
         "executor": "CODEX",
         "variant": "DEFAULT"
       },
       "repos": [{
         "repo_id": "<repo_id>",
         "target_branch": "dev"
       }]
     }'
   ```

5. **実行開始を確認**
   - Vibe Kanban UI で 2 タスクが in_progress になることを確認
   - または API で状態を再取得して確認

**注意:**
- 実装タスクのコードはまだ dev に入っていない
- Vibe Kanban は**ローカルの target_branch** をベースにする
- そのため事前に `git pull origin dev` で最新化が必須
- 並列実行可能 `(P)` マーカーのタスクは複数同時開始可

### 1. in_review に実装タスクがある場合

**子タスク（検証タスク）を確認:**

```python
# 実装タスクの最新 attempt から子タスクを取得
child_tasks = get_child_tasks(parent_workspace_id)
```

- 子タスクなし → 検証タスク3つを作成・実行
- 子タスクあり → 検証結果を確認

### 2. 検証タスク作成

**3つの検証タスクを作成:**

```
{タスク名} [Claude-1]  ← Claude Code で実行
{タスク名} [Codex-1]   ← Codex で実行
{タスク名} [Codex-2]   ← Codex で実行
```

**API:**
```bash
# ポート取得
VIBE_PORT=$(cat /tmp/vibe-kanban/vibe-kanban.port)

# サブタスク作成
curl -X POST "http://localhost:$VIBE_PORT/api/tasks" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "<project_id>",
    "title": "inspect X.X {feature} [Claude-1]",
    "description": "`.claude/commands/kiro/validate-impl.md` の指示に従って\nfeature-name: **{feature}**\ntask: **{task_no}**\nを検査する。",
    "parent_workspace_id": "<親の task_attempt_id>"
  }'
```

**タスク実行:**
```bash
# attempt 作成 + 実行開始
curl -X POST "http://localhost:$VIBE_PORT/api/task-attempts" \
  -H "Content-Type: application/json" \
  -d '{"task_id": "<task_id>", ...}'
```

**重複タスクのクリーンアップ:**

attempt 作成後、TODO に残っている `inspect` タスク（attempt なし）を削除する:

```python
# TODO の inspect タスクを取得
todo_tasks = get_tasks_by_status(project_id, "todo")
inspect_tasks = [t for t in todo_tasks if t["title"].startswith("inspect ")]

# 削除
for task in inspect_tasks:
    task_uuid = format_uuid(task["id"])  # UUID形式に変換
    curl -X DELETE "http://localhost:$VIBE_PORT/api/tasks/$task_uuid"
```

**注意:** これは attempt 作成時のレスポンス確認漏れで重複タスクが作られた場合の保険処理。

### 3. 検証結果確認

`scripts/get_verdict.py` で GO/NO-GO を取得:

```bash
python3 scripts/get_verdict.py <task_attempt_id>
# {"verdict": "GO", "summary": "..."}
```

**判定ルール:**
- 1つでも NO-GO → 全検証タスク停止・削除、実装タスクに follow-up
- 3つ全部 GO → PR 作成へ

### 4. NO-GO 時の処理

```bash
# 実行中タスクを停止
curl -X POST "http://localhost:$VIBE_PORT/api/execution-processes/{id}/stop"

# タスク削除
curl -X DELETE "http://localhost:$VIBE_PORT/api/tasks/{id}"

# 実装タスクに修正依頼 (follow-up)
curl -X POST "http://localhost:$VIBE_PORT/api/sessions/{session_id}/follow-up" \
  -H "Content-Type: application/json" \
  -d '{"message": "検証結果: NO-GO\n理由: {summary}\n修正してください"}'
```

**リトライカウント:**
- タイトルに `[R1]` → `[R2]` → `[R3]` を追加
- 3回失敗 → タスク削除・新規作成 `[V2]`
- V2 で3回失敗 → エスカレーション `[ESCALATED]`

### 5. 全 GO → PR 作成

**Codex vs Claude:**
- 両方 GO → Codex 優先、Claude 削除
- 片方だけ GO → 先に完了した方を採用、もう片方は停止・削除

**既存 PR チェック:**

PR 作成前に、実装タスクのブランチで既に PR が存在するか確認する:

```bash
# 実装タスクの attempt から branch 名を取得
# ブランチ名は通常 vibe-kanban/{task_attempt_id} 形式

# 既存 PR を確認
gh pr list --head <branch_name> --repo <owner/repo> --json number,state,mergedAt

# 結果の判定:
# - PR なし → 新規 PR 作成へ
# - PR あり & state=OPEN → CI 監視へ（セクション 6）
# - PR あり & state=MERGED → マージ完了処理へ（セクション 7 の「完了後」）
```

**新規 PR 作成:**
```bash
curl -X POST "http://localhost:$VIBE_PORT/api/task-attempts/{id}/pr"
```

### 6. CI 監視

```bash
# 6分間隔、最大30分
gh pr checks <PR番号> --repo <owner/repo>
```

- CI 成功 → マージへ
- CI 失敗 → NO-GO と同じ処理（検証削除、follow-up）
- タイムアウト → エスカレーション

### 7. マージ

```bash
# dev にマージ
gh pr merge <PR番号> --repo <owner/repo> --merge
```

**完了後:**

1. **実装タスクを done に移動:**
   ```bash
   curl -X PUT "http://localhost:$VIBE_PORT/api/tasks/<implementation_task_uuid>" \
     -H "Content-Type: application/json" \
     -d '{"status": "done"}'
   ```

2. **検証タスクを done に移動:**
   ```bash
   # 各検証タスク（Claude-1, Codex-1, Codex-2）を done に移動
   for task_uuid in "<claude1_uuid>" "<codex1_uuid>" "<codex2_uuid>"; do
     curl -X PUT "http://localhost:$VIBE_PORT/api/tasks/$task_uuid" \
       -H "Content-Type: application/json" \
       -d '{"status": "done"}'
   done
   ```

**注意:** status は小文字で指定（"done", "todo", "inprogress", "inreview"）

### 8. エスカレーション

`scripts/notify_slack.py` で通知:

```bash
python3 scripts/notify_slack.py "Task failed: {title}" "Escalation Required"
```

### 9. 全作業完了
TODO, in_review, in_progress にタスクが0件になり全てDONEになったらユーザーに

`scripts/notify_slack.py` で通知:

```bash
python3 scripts/notify_slack.py "全作業完了" "全作業完了"
```


## タイトルマーカー

| マーカー | 意味 |
|---------|------|
| `[R1]` `[R2]` `[R3]` | リトライ回数 |
| `[V2]` | 2回目の実装 |
| `[V2-R1]` | V2の1回目リトライ |
| `[Claude-1]` | Claude検証1回目 |
| `[Codex-2]` | Codex検証2回目 |
| `[ESCALATED]` | エスカレーション済み |
| `[CI-TIMEOUT]` | CIタイムアウト |

## 実行例

```
# 状況確認
「dify2lg プロジェクトのタスクを進めて」

# 特定タスク指定
「タスク 1.1 の検証結果を確認して」

# PR作成
「検証OKのタスクのPRを作成して」
```


</details>

約300行はちょっとでかいですね、そもそも責務が多すぎる。
skill-creator でやるにはちょっと無理があったみたいなんで、分割していきます。

## オーケストレーションskillを分割


```
.claude/skills/
├── vk-router/
│   ├── SKILL.md
│   └── scripts/
│       └── get_project_state.py
├── vk-start-task/
│   └── SKILL.md
├── vk-create-verify/
│   └── SKILL.md
├── vk-judge-verdict/
│   └── SKILL.md
├── vk-create-pr/
│   └── SKILL.md
├── vk-monitor-ci/
│   └── SKILL.md
├── vk-merge-pr/
│   └── SKILL.md
├── vk-handle-nogo/
    └── SKILL.md

vk-router/scripts/
├── get_project_state.py   # 状態取得 + 状態判定
├── get_tasks_status.py    # タスク一覧取得
├── get_verdict.py         # GO/NO-GO 取得
├── get_repo_id.py         # repo_id 取得
├── get_db_path.py         # DB パス取得
└── api_helpers.py         # Vibe Kanban API ヘルパー

```
責務を洗い出して、上の構成に分割しました。、かつスクリプトを用意して確実性を高めてみました。

おかしいな、思ったよりめんどくさいぞこれ...
skill-creator とかあるからもっと簡単に終わると思ってたんだが...
かなり...泥臭い...

## VibeKanbanの状態監視スクリプト
ClaudeCodeを起動し続けるのは安定性が悪いので、もう少し信頼性の高いやり方としてスクリプトから何時間でも終わるまで立ち上げ直し続けてもらいます。
トリガーとして、SQLiteを監視して ステータスに変化があったら立ち上げることにします。
初回起動時にセッションをとって、セッションIDで resumeしていきます

SQLiteのテーブルを監視して変化があったら Claudeを起動する、という仕掛けにします。
またgitのCIに変化があったときもClaudeを起動するようにします



### 監視スクリプトデバッグ

![](/images/2026/sdd_kanban_claudecode/debug.png)
うーん？やってみるとわかりますが、やはりスキルを確実に実行しないのでこの手一気通貫365/24で動かすやつは挙動が難しいですね。
まあ最悪の敵は rate_limitですけど。

### Structured Decomposition for LLM Reasoning: Cross-Domain Validation and Semantic Web Integration
https://arxiv.org/abs/2601.01609
2026年1月4日なのでこれ書いてる3日前ですが、ちょうどいい感じの事かいてる論文出てたので適用します。

問題定義として
- ルール適用が必要な分野では説明可能性と一貫性が重要。
- LLMは柔軟だがルール適用の保証が弱い。記号推論は保証が強いが入力が構造化前提。
なので今の状況にバチハマりですね。



1. LLMは「状態」を作る（State Builder）

VibeKanbanやgitの現状を読み込ませてJSONで状態を吐き出すようにする
LLMに「次に何をするべきか」を決めさせない。“事実＋根拠”だけ出させる。
つまり get_project_state.py から得たい最小の factsを決める
LLMは「次どうする？」を言わない。上の facts を埋めるだけ。

2.  状態定義
```
IDLE
RUNNING
IMPL_IN_REVIEW
VERIFYING
VERIFY_IN_REVIEW
VERIFIED_GO
NO_GO
RETRY
PR_OPEN
CI_PASS
DONE
```


3. 遷移ルール（決定論 if/then）
ルールで「アクション」を決める（Policy / Rules）
LLMの出してきたJSONを使った状態決定論（if/then）を構築する

```
A. IDLE → RUNNING（vk-start-task）
Guard
todo_count > 0
かつ in_progress_count == 0
かつ impl_in_review_count == 0
かつ verify_in_progress_count == 0
かつ verify_in_review_count == 0
かつ pr_state in {none, merged}（PRが開いてない）
```
こんな感じで決めていきます。

ただ今回は実装が並列で動いて、検証も同時に走るため
並列稼働の条件も含めていきます。
一つのタスクにたいして2つの実装、4つの検証が走りますが枝としては実装を親と考えて
1タスク2ブランチみたいに枝葉でのつながりを考えます。

よって
get_project_state.py は
```
children = workspaces where parent_workspace_id == ROOT
    impl_candidates_in_progress
    impl_candidates_in_review
verify_jobs = workspaces where parent_workspace_id in children.workspace_id
    verify_in_progress_total
    verify_in_review_total

さらに 候補ごとの内訳

verify_count_by_impl[impl_ws_id] = {in_progress, in_review, done, verdicts[]}
```

………
---

こんな感じでごりっごりにルールを決めていきます。

### 監視スクリプト実行
と、いうわけで完成。

![](/images/2026/sdd_kanban_claudecode/監視.png)
![](/images/2026/sdd_kanban_claudecode/実行.png)


実行方法はcronとかサービス化とかいろいろ検討しましたが
一つの機能開発が終わるまで回したいだけなので

`tmux new -d -s watcher "python3 /workspace/scripts/watch_tasks.py {機能名}"`
こうやって VibeKanbanへのタスク登録が終わったら tmux で動かしておくのが一番楽だわって結論に達しました。


## 結果
agent core runtimeにS3経由でファイルを送りつけて invocation する機能のために8個のタスクが作られ（実装、検証、リトライで最低48個に分岐）一気通貫でタスク完了するまでにかかった時間は14時間くらいでした。

初めに書いた通り、僕は設計に数日かけるので14時間でおわられちゃうと、寝て起きたら終わってるので、次の仕事がまだできてないんだけど...ってなりますね。

まだ試せてはいませんが
監視スクリプト＋Skill＋VibeKanbanのこのシステムなら
一週間かかる仕事でも1ヶ月かかる仕事でも最後まで一気通貫してくれることでしょう





### 余談
オーケストレーションをCodexのskillでやることも検討したんですが、内部で git 操作とかcurl連発とかSQLiteに繋ぐとかするの、CodexがあまりにもガチガチでやりたがらないのでCodexは実装のみにしてます。
あとClaudeCodeのSkillやっててわかりましたがMaxの$100プランじゃrate limit がすぐきますね
2時間くらいでrate limitに到達しました。
はじめの計画だとClaudeCodeとCodexの二つで実装、検証していましたが、オーケストレーションやらせるならClaudeCodeの仕事減らさないと無理でした




