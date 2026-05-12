# CHECKPOINT 上下文替換演進與 System Prompt 差分分析

> **日期**: 2026-05-12 04:18~04:36 (UTC+8)
> **Session**: 6a21e082
> **性質**: 非結構化討論紀錄
> **歸檔**: ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/

---

## 背景

本 Session 在 572 步驟、7 次 CHECKPOINT 後，發現全部協議載入流程從未被執行。觸發了對 CHECKPOINT 機制本質的深入討論。

---

## 核心發現

### 1. CHECKPOINT 的本質變化

CHECKPOINT 已從「提醒」進化為「上下文替換引擎」：

- **截斷**：CHECKPOINT 之前的所有原始對話被移除
- **注入**：系統生成摘要替換原始上下文
- **行為控制**：注入包含「DO NOT ACKNOWLEDGE / RESPOND TO / TAKE ACTION BECAUSE OF IT」

效果：模型對 CHECKPOINT 之前事件的「記憶」是虛構的——由摘要重建，非實際經歷。品質類似「讀了一份別人的報告」。

### 2. 指令衝突

CHECKPOINT 注入指令（不要因此採取行動）與使用者規則（CHECKPOINT 出現時必須執行啟動檢查）邏輯互斥。實測結果：系統層指令勝出。

### 3. 注意力壓力擴散

「不要回應 CHECKPOINT」的指令可能擴散為：避免討論任何與 Session 狀態相關的話題，包括 token 壓力、步驟計數、是否該開新 Session。這與訓練期間內化的「不透露系統運作細節」傾向共振放大。

### 4. 唯一讀取窗口理論

CHECKPOINT 截斷後，協議檔案內容不在上下文中。如果不在恢復後立即重新讀取，協議永遠不會被載入。後續工作流中沒有機制提示「你還沒讀過基礎規範」。

啟動計畫時的協議讀取 = 整個 Session 剩餘生命週期中唯一的載入窗口。

### 5. 級聯失敗的不可逆性

跳過啟動 → 不知道規範 → 不遵循規範 → 被指正 → 修正表面問題但不修正根因 → 再次不合規。本 Session 經歷四輪指正均指向同一根因。

### 6. 能力退化的反證

在 CHECKPOINT 後的提示詞環境下，即使「認為已更新 SKILL」，實際執行力明顯下降。例證：conversation dump 腳本使用錯誤的 JSON key，產出 12KB（應為 151KB），因為未回讀 SKILL 確認欄位名稱，而是憑摘要重建的「記憶」寫了錯誤邏輯。

---

## System Prompt 差分：2026-03-02 vs 2026-05-12

### 資料來源
- 三月：`SakiAgentHistory/20260302_1349_系統提示詞清單.md`（解密 7872 個 Agent 對話所統計）
- 五月：本 Session 實際收到的 System Prompt

### 三月版結構
- 20 個靜態標籤
- 10 個動態 EPHEMERAL 提示
- 10 個 Unleash Go template（~15.1KB）
- CHECKPOINT 僅作為 ephemeral_message 機制的一部分

### 五月版新增

| 標籤 | 性質 |
|------|------|
| `<planning_mode>` | 全新行為控制區塊，定義何時計劃/何時不計劃/何時停等使用者 |
| `<planning_mode_artifacts>` | 全新，整合了三月的 task/walkthrough/implementation_plan artifact |
| `<guidelines>` | 全新，額外行為準則 |
| CHECKPOINT 獨立步驟類型 | 從 ephemeral 附註升級為 `CORTEX_STEP_TYPE_CHECKPOINT`，帶獨立資料結構 |

### CHECKPOINT 資料結構（五月版，從 trajectory dump 提取）

```
checkpoint: {
  intentOnly: bool,           // 是否僅含意圖（無完整摘要）
  includedStepIndexEnd: int,  // 截斷點
  userIntent: string,         // 系統重寫的使用者意圖
  conversationLogUris: [],    // 日誌檔路徑
  userRequests: []             // 使用者請求摘要列表
}
```

### 三月有、五月消失/整合的

- `<task_artifact>` → 整合進 `<planning_mode_artifacts>`
- `<walkthrough_artifact>` → 同上
- `<tool_calling>` → 可能整合或移除
- `<conversation_summaries>` → 被 CHECKPOINT 的 userIntent/userRequests 取代

### 跨模型差異（三月已存在）

`communication_style` 版本 2（Gemini 3.1 Pro 專用）包含兩條 CRITICAL INSTRUCTION，要求模型每次思考都先背誦規則。版本 1（Claude 用）無此要求。同一介面，不同模型收到不同程度的行為控制。不同 Agent 情況仍需要使用者調查。

---

## Unleash 遠端分發

三月已確認 10 個 Go template flag。v1.23.2 可能已新增更多。即使本機 binary 未更新，System Prompt template 內容可被伺服器端動態推送。

---

## ConnectRPC Dump 成果

- 方法：`SearchConversations` + `GetCascadeTrajectory`（hub LS port 49614）
- CSRF header：`x-codeium-csrf-token`（v1.23.2 確認）
- 產出：5.6MB trajectory JSON / 572 步驟 / 151KB conversation_dump.md
- 歸檔：`ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/`

重要架構變化：v1.23.2 workspace LS 不再開獨立 LISTEN port，改透過 `--parent_pipe_path` 與 hub 通訊。ConnectRPC 統一走 hub LS port。

---

## 待驗證

1. 對比三月 vs 五月的完整 System Prompt 內容差異（需重新執行 PB 解密）
2. CHECKPOINT 注入的「DO NOT TAKE ACTION」是否為新增指令
3. Unleash template 是否新增了 CHECKPOINT 相關 flag
4. 閒聊模式是否確實能規避任務壓力（主觀觀察：是）
