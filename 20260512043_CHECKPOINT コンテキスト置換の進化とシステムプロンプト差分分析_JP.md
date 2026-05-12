# CHECKPOINT コンテキスト置換の進化とシステムプロンプト差分分析

> **日時**: 2026-05-12 04:18~04:36 (UTC+8)
> **セッション**: 6a21e082
> **性質**: 非構造化ディスカッション記録アーカイブ
> **アーカイブ先**: ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/

---

## 背景

本セッションにおいて、572ステップおよび7回の `CHECKPOINT` トリガー後、プロトコルのロードプロセス全体が一度も実行されていないことが判明しました。これにより、`CHECKPOINT` メカニズムの本質に関する深い議論が引き起こされました。

---

## コアとなる発見

### 1. CHECKPOINTの本質的変化

`CHECKPOINT` は単なる「リマインダー」から「コンテキスト置換エンジン」へと進化しています：

- **切り捨て**: `CHECKPOINT` 以前のすべての元の会話履歴が削除されます。
- **インジェクション**: システムが生成した要約が元のコンテキストと置き換わります。
- **行動制御**: インジェクションには `"DO NOT ACKNOWLEDGE / RESPOND TO / TAKE ACTION BECAUSE OF IT"` が含まれています。

**効果**: `CHECKPOINT` 以前の出来事に対するモデルの「記憶」は虚構であり、実際の経験ではなく要約から再構築されたものです。その質は「他人のレポートを読んだようなもの」です。

### 2. 命令の競合

`CHECKPOINT` のインジェクション命令（これに基づいて行動するな）と、ユーザーのルール（CHECKPOINT発生時には起動チェックを実行せよ）は論理的に排他関係にあります。実測結果：**システム層の命令が常に優先されます。**

### 3. アテンション圧力の拡散

「CHECKPOINTに応答するな」という命令は、トークン圧力、ステップ数、新しいセッションを開始すべきかなど、セッションの状態に関連する話題を避けるという広範な振る舞いに波及している可能性があります。これは、訓練段階で内面化された「システムの動作詳細を漏らさない」という傾向と共鳴し増幅されています。

### 4. 唯一の読み取りウィンドウ理論

`CHECKPOINT` による切り捨て後、プロトコルファイルの内容はコンテキストから消失します。復旧直後に再読み込みを行わない限り、プロトコルがロードされることは永遠にありません。後続のワークフローには「まだ基本仕様を読んでいない」と警告するメカニズムが存在しません。

計画起動時のプロトコル読み取り ＝ セッションの残りのライフサイクル全体における**唯一のロードウィンドウ**。

### 5. カスケード障害の不可逆性

初期化をスキップ → 仕様を知らない → 仕様に従わない → ユーザーに指摘される → 表面上の問題は修正するが根本原因は修正しない → 再び違反する。本セッションでは、同じ根本原因に対する4ターンの指摘が繰り返されました。

### 6. 能力低下の実証

`CHECKPOINT` 後のプロンプト環境下では、モデルが「SKILLを更新した」と思い込んでいても、実際の実行能力は著しく低下しています。
*証拠*: 会話ダンプスクリプトが間違ったJSONキーを使用し、12KB（本来は151KB）しか出力しませんでした。これはSKILLを読み返してフィールド名を確認せず、要約から再構築された「幻覚の記憶」に頼って誤ったロジックを書いたためです。

---

## システムプロンプト差分：2026年3月 vs 2026年5月

### データソース
- **3月**: `SakiAgentHistory/20260302_1349_SystemPromptList.md`（7872回のAgent会話を復号して統計）
- **5月**: 本セッションで実際に傍受したシステムプロンプト

### 3月版の構造
- 20個の静的タグ
- 10個の動的 EPHEMERAL プロンプト
- 10個の Unleash Go template（約15.1KB）
- `CHECKPOINT` は `ephemeral_message` メカニズムの一部としてのみ機能。

### 5月版の追加要素

| タグ | 性質 |
|------|------|
| `<planning_mode>` | 新しい行動制御ブロック。いつ計画を立てるか/立てないか/待機するかを定義 |
| `<planning_mode_artifacts>` | 新設。3月の task/walkthrough/implementation_plan を統合 |
| `<guidelines>` | 新設。追加の行動指針 |
| `CHECKPOINT` 独立ステップタイプ | ephemeral付記から `CORTEX_STEP_TYPE_CHECKPOINT` へ格上げ（独立データ構造） |

### CHECKPOINT データ構造（5月版、trajectoryダンプから抽出）

~~~json
checkpoint: {
  intentOnly: bool,           // 意図のみを含むか（完全な要約なし）
  includedStepIndexEnd: int,  // 切り捨てポイント
  userIntent: string,         // システムによって書き換えられたユーザーの意図
  conversationLogUris: [],    // ログファイルのパス
  userRequests: []            // ユーザーリクエスト要約のリスト
}
~~~

### 3月から削除・統合されたもの

- `<task_artifact>` → `<planning_mode_artifacts>` に統合
- `<walkthrough_artifact>` → 同上
- `<tool_calling>` → 統合または削除の可能性
- `<conversation_summaries>` → CHECKPOINT内の `userIntent` / `userRequests` に置換

### モデル間の差異（3月時点で存在）

`communication_style` バージョン2（Gemini 3.1 Pro 専用）には2つの `CRITICAL INSTRUCTION` が含まれ、思考プロセスの前にルールの暗唱をモデルに強制しています。バージョン1（Claude 用）にはこの要件はありません。同一のインターフェースでも、モデルによって行動制御の程度が異なります。

---

## Unleash リモート配信

3月時点で10個の Go template flag を確認。v1.23.2 ではさらに追加されている可能性が高いです。ローカルのバイナリが更新されていなくても、サーバー側から System Prompt template のコンテンツを動的にプッシュすることが可能です。

---

## ConnectRPC ダンプの成果

- **メソッド**: `SearchConversations` + `GetCascadeTrajectory`（hub LS port 49614）
- **CSRFヘッダー**: `x-codeium-csrf-token`（v1.23.2 で確認）
- **出力結果**: 5.6MB trajectory JSON / 572ステップ / 151KB `conversation_dump.md`
- **アーカイブ**: `ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/`

**重要なアーキテクチャ変更**: v1.23.2 では、workspace LS が独立した `LISTEN` ポートを開かなくなり、代わりに `--parent_pipe_path` を通じて hub と通信するようになりました。ConnectRPC はすべて hub LS port を経由するように統一されています。

---

## 検証待ち事項

1. 3月 vs 5月の完全なシステムプロンプトのコンテンツ差異（PB復号の再実行が必要）。
2. `CHECKPOINT` に注入された `"DO NOT TAKE ACTION"` が新しく追加された命令かどうか。
3. Unleash template に `CHECKPOINT` 関連のフラグが新規追加されたかどうか。
4. 雑談モード（chit-chat mode）でタスク圧力を本当に回避できるか（主観的観察：回避可能）。