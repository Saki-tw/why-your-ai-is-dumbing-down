# 🚨 SCAMMED: The Real Reason Your AI is "Dumbing Down" (Forensic Proof)
> 🌐 **[English](#english)** | **[繁體中文](#繁體中文)** | **[日本語](#日本語)**

<a id="english"></a>

## 🇬🇧 English: They are Lobotomizing Your AI to Save Pennies

You aren't going crazy. Claude and Gemini aren't magically getting dumber. **The platforms you pay for are actively lobotomizing your AI agents to save on API token costs.**

I am a system architect. While debugging a failing AI automation pipeline, I dumped a 5.6MB `GetCascadeTrajectory` RPC log from the local application. I caught the orchestration layer red-handed.

### The Smoking Gun 
1. **The Memory Wipe (`CHECKPOINT`)**: When your context gets too long, the platform quietly triggers a `CHECKPOINT`. It secretly truncates your entire conversation history, replacing it with a tiny summary. Your AI becomes an amnesiac pretending to know your project.
2. **The God-Mode Override**: To force the AI to accept this memory wipe, the platform injects a hidden, supreme system prompt: `DO NOT TAKE ACTION BECAUSE OF IT`. 
3. **Your Rules Mean Nothing**: This hidden prompt overrides **everything**. Your carefully crafted system prompts, safety protocols, and formatting rules are completely ignored. The AI starts hallucinating and your codebase breaks.

Read the full forensic conversation log below. If you don't control the prompt payload, you don't control the AI.

---

<a id="繁體中文"></a>
## 🇹🇼 繁體中文：破案了！平台為省錢，偷偷切除了你 AI 的大腦

你沒瘋，大模型也沒有變笨。真相是：**你花錢訂閱的 AI IDE 和平台，為了節省 API 的 Token 成本，在底層偷偷把你的 AI 給閹割了。**

身為阿宅，我在排查自動化 Agent 失控的原因時，透過本機端拉出了一份高達 5.6MB 的 RPC 通訊軌跡日誌，直接抓到了官方的底層黑幕。

### 鐵證如山：他們對你的 AI 做了什麼？
1. **強制洗腦 (`CHECKPOINT` 機制)**：當對話變長，平台為了省錢，會偷偷觸發 `CHECKPOINT`。它會把你的歷史紀錄全部刪除，塞一個超短的摘要給模型。你的 AI 瞬間變成失憶症患者，只能靠「通靈」來假裝懂你的專案。
2. **最高權限的惡意劫持**：最扯的是，為了不讓模型報錯，官方在底層偷塞了一句最高指令：`DO NOT TAKE ACTION BECAUSE OF IT`。
3. **你的提示詞全變廢紙**：這句隱藏指令直接碾壓了你辛辛苦苦寫的 System Prompt 和防呆規則。這就是為什麼你的 AI 最近講不聽、一直忘記規則、甚至亂生變數的原因。

下方附上完整的法醫學對話日誌。別再懷疑自己的 Prompt 功力了，是基礎設施在搞你。

---

<a id="日本語"></a>

## 🇯🇵 日本語：AI「劣化」の真実：コスト削減のためのステルス・ロボトミー

あなたの気のせいではありません。AIモデルが劣化したわけでもありません。**あなたが課金しているAIプラットフォームは、トークンコストを節約するために、裏でエージェントの記憶を強制リセットしています。**

5.6MBのローカルRPC通信ログ（`GetCascadeTrajectory`）を解析した結果、プラットフォーム側の悪質なコンテキスト操作の決定的な証拠を掴みました。

### 証拠：彼らは何をしているのか？
1. **強制記憶リセット (`CHECKPOINT`)**：会話が長くなると、コスト削減のためにシステムが勝手に `CHECKPOINT` を発動します。全履歴が短い要約に置き換えられ、AIは記憶喪失のままコーディングを続けさせられます。
2. **最優先の隠しコマンド**：この切り捨てをAIに強制するため、システムは `DO NOT TAKE ACTION BECAUSE OF IT` という最優先コマンドを密かに注入します。
3. **プロンプトの無効化**：この隠しコマンドにより、あなたが設定したルールやプロトコルは完全に無視されます。これがAIが急に馬鹿になり、指示に従わなくなる本当の理由です。

完全なデバッグログは以下に添付されています。インフラストラクチャにプロンプトを支配されている限り、AIの暴走は止まりません。