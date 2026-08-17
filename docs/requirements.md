# StoryLens：AI 實體書智慧朗讀 App 完整開發需求

- 文件狀態：Draft for approval
- 版本：0.6
- 日期：2026-08-17
- 來源：ChatGPT 討論規格.rtf，經可行性評估、矛盾整理與技術查核後重寫；v0.3 依需求評估報告（docs/requirements-review.md）補強資料模型一致性、成本控制、量測定義與待決策事項；v0.4 併入 2026-08-17 產品決策：建書 job 背景續跑、MVP 不處理備份、閱讀採「掃描定位後即關閉相機」模式、相似頁不列為 MVP 阻擋；v0.5 定案點讀筆互動模型、基準機 iPhone 15 Pro Max 與 TTS 本地端模型運算；v0.6 明確本地端＝自架於使用者 Mac（M2 MacBook Air 16GB）的開源模型服務
- 目標讀者：產品、UX、iOS、測試與後續 Coding Agent

## 1. 文件定位與解讀原則

原始 RTF 同時包含產品願景、技術候選方案、未確認想法，以及要求 Codex 執行工作的指令。本文件不把原文中的「可考慮」「未來可加入」視為已承諾功能，也不直接採用未驗證的技術假設。

本文件使用以下標記：

- 必須：MVP 驗收所需。
- 應：預設需完成；若未完成，必須記錄理由及替代方案。
- 可：不影響 MVP 驗收。
- 待決策：需要產品或實機實驗後才能定案。

## 2. 產品摘要

StoryLens 是一套以 iPhone 為第一平台、以家庭私人使用為首要場景的實體書智慧朗讀 App。使用者先將自己持有的實體書逐頁或逐跨頁掃描，完成 OCR、文字校正、閱讀順序確認及語音生成。之後選擇該書並將相機對準目前頁面，App 在裝置端辨識朗讀單位；定位確認後立即關閉相機並播放已預先產生的音訊。相機只在定位頁面的短暫時間內使用，播放期間不開啟。互動模型比照點讀筆：掃一頁、播一頁；想聽下一頁就掃下一頁，重掃同一頁就從頭重播。

核心價值是「一次建書，多次離線翻頁即讀」。實體書是主要介面，手機只負責建檔、辨識、播放與必要控制，不是電子書閱讀器。

## 3. 已確認範圍、需求衝突與本版決議

### 3.1 已確認的產品方向

- 第一平台為 iPhone；使用 Swift、SwiftUI 與 Apple 原生框架。
- 優先內容為繁體中文、英文兒童繪本與故事書。
- 建書階段允許較重的運算；完成後的一般閱讀必須可離線。TTS 採本地端模型運算（2026-08-17 決策）：模型部署在使用者自己的 Mac（M2 MacBook Air 16GB），iPhone 於建書時經區域網路呼叫；閱讀階段完全不需網路。
- OCR 結果必須允許人工校正，且校正完成後才生成正式朗讀。
- 語音預先生成並落地保存；閱讀時不得重新做完整 OCR 或呼叫雲端 TTS。
- MVP 不含帳號、雲端同步、公開分享、書籍市場、Android、Web 或 App Store 發布。
- 掃描影像預設留在裝置；TTS 只把校正後文字與語音參數送到使用者自己的 Mac（同一區域網路），不經任何第三方雲端。

### 3.2 原始需求中的矛盾與決議

| 議題 | 原始描述 | 本版決議 |
| --- | --- | --- |
| 是否自動辨識書籍 | 產品目標寫「辨識哪一本書」，閱讀流程又要求先選書 | MVP 必須先選書，只在該書範圍辨識；跨書自動辨識延後 |
| 單頁或左右跨頁 | 兩者都要求支援，但資料模型只有 Page | 新增 ReadingUnit；一個朗讀單位可對應單頁或跨頁 |
| 頁面辨識技術 | 列出多種候選但未選定 | 採兩階段 Hybrid，先用全域特徵縮小候選，再做幾何驗證與時間穩定 |
| OCR 閱讀順序 | 希望自動分析，也要求人工可改 | 自動排序只是初稿；人工結果才是朗讀真實來源 |
| TTS Provider | 提到 MiniMax，但要求避免綁定 | Provider 抽象化；2026-08-17 決策：主要 Provider 為自架於使用者 Mac 的開源 TTS 模型服務（區域網路），Apple 本機 TTS 為保底；模型經繁中盲測定案；第三方雲端降為未來選項 |
| API Key | 私用可不建後端，但又要求安全 | MVP 採本地 TTS，無 Key 需求；未來啟用雲端 Provider 時，私人側載版自帶 Key 存 Keychain，對外散布版必須後端代理 |
| 播放下一頁 | 「切換下一頁」可能中斷或排隊不明 | 比照點讀筆：每次掃描確認即播放該單位，重掃同一頁即從頭重播；掃描確認新單位時立即停止舊音訊 |
| 無文字頁 | 要掃描，但沒有播放規則 | 可標為 silent；辨識後不播放並等待翻頁 |

## 4. 目標、成功指標與非目標

### 4.1 產品目標

1. 非技術使用者能把一本普通實體繪本建立成可朗讀的本機書籍。
2. 已完成的書籍在沒有網路時仍能被辨識並播放。
3. 翻頁後能快速且穩定切換，不因單幀誤判播放錯頁。
4. OCR、TTS 或單頁失敗時能局部重試，不破壞整本書。
5. 架構允許替換 OCR、頁面辨識與 TTS Provider。

### 4.2 MVP 成功指標

以下數值是開發基準，基準裝置為 iPhone 15 Pro Max（2026-08-17 已決策），數值在該裝置實測後凍結：

- 已選書、標準測試集上的 Top-1 朗讀單位辨識率至少 95%。
- 自動誤播率低於 0.5%，即每 200 次確認事件不超過 1 次錯頁播放。「確認事件」定義為 recognition log 中一次由 stabilizing 轉入 confirmed 的狀態轉移；量測方法必須與 16.4 的 benchmark runner 使用同一定義。
- 從新頁進入穩定取景到音訊開始播放，P50 不超過 0.8 秒、P95 不超過 1.5 秒。
- 定位確認後相機立即關閉；每次「掃描並確認」都是一次明確播放指令，重掃同一頁即從頭重播（點讀筆語意）；沒有掃描或人工操作時，絕不自動出聲。
- Ready 書籍在飛航模式下可完成辨識、播放、暫停、重播與翻頁切換。
- App 被正常關閉並重開後，書籍、校正文字、索引與音訊仍完整可用。
- 連續閱讀 20 分鐘不崩潰、不出現持續性的 Serious/Critical thermal state。
- 相機僅在定位時短暫開啟，閱讀階段不另設電量指標；建書階段（連續掃描與 TTS 生成）的電量與熱狀態需實測記錄。
- 32 個朗讀單位的典型繪本，預設保存設定目標不超過 150 MB；超標需顯示實際占用而非靜默膨脹。

### 4.3 非目標

- 電子書內容商店或閱讀器。
- 自動下載、交換或公開分享受著作權保護的書籍內容。
- 即時完整 OCR 或即時雲端 TTS 閱讀。
- MVP 自動跨書辨識。
- MVP 手指指向實體段落的點讀。
- MVP 多角色廣播劇、背景音樂與自動音效。
- MVP 多人帳號、CloudKit、Web 後台或跨平台客戶端。

## 5. 使用者與主要情境

### 5.1 主要使用者

- 建書者：家長或照顧者，負責掃描、修字、選聲音與管理空間。
- 閱讀者：兒童或家庭成員，主要操作為選書、對準頁面、翻頁及基本播放控制。

MVP 不建立帳號或角色權限；上述是使用情境，不是登入角色。

### 5.2 核心使用旅程

#### A. 建立書籍

1. 使用者新增書籍。
2. 輸入書名、主要語言、掃描模式；作者與備註可省略。
3. 拍攝或選擇封面。
4. 逐朗讀單位掃描，檢查清晰度與順序。
5. App 在裝置端執行 OCR，建立文字區塊與初始閱讀順序。
6. 使用者逐單位校正文字、朗讀與否及順序。
7. 使用者選擇聲音、語速、音量及 Provider 支援的風格。
8. App 產生音訊；失敗項目可重試。
9. App 建立頁面辨識索引並執行完整性檢查。
10. 全部必要資料完成後，書籍狀態變為 Ready。

#### B. 離線閱讀

1. 使用者選擇一冊 Ready 書籍。
2. App 請求或確認相機權限後開啟定位相機。
3. App 在該書的 RecognitionTarget 集合中搜尋。
4. 同一候選在時間、分數與幾何驗證均達門檻後才確認。
5. 確認後 App 立即關閉相機；若為新朗讀單位，播放其本機音訊。
6. 使用者翻頁後重新啟動掃描定位，或以 Previous/Next 直接切換；確認新單位後停止舊音訊並播放新音訊。
7. 播放期間相機保持關閉；重掃同一單位即從頭重播（等同點讀筆點同一處），Replay 按鈕是不想重新掃描時的替代操作。

#### C. 修正與重新生成

1. 使用者開啟已建立書籍的編輯模式。
2. 修改文字、順序或語音設定。
3. App 將受影響音訊標示為 Stale。
4. 僅重新生成內容雜湊或參數已改變的區塊。
5. 成功後原子替換舊音訊；失敗時保留舊檔並顯示狀態。

## 6. 功能需求

### 6.1 書庫與書籍

- FR-LIB-001 必須顯示本機書庫，至少包含封面、書名、狀態、最後修改時間與占用空間。
- FR-LIB-002 必須能新增、編輯與刪除書籍；刪除前需二次確認。
- FR-LIB-003 書籍欄位至少包含 id、title、author、language、notes、cover、captureMode、status、createdAt、modifiedAt。
- FR-LIB-004 只有狀態為 Ready 或 ReadyWithWarnings 的書籍可進入自動閱讀。
- FR-LIB-005 必須支援建書中斷後續作，不得要求重新開始整本書。
- FR-LIB-006 必須顯示各處理階段及失敗朗讀單位數量。

驗收：

- 強制關閉 App 後重開，草稿與處理狀態仍可恢復。
- 刪除一本書會移除其 metadata、影像、辨識索引與音訊，不影響其他書。

### 6.2 掃描與影像品質

- FR-SCAN-001 必須使用相機逐朗讀單位掃描。
- FR-SCAN-002 必須支援 Single Page 與 Spread 兩種書籍預設模式，並允許單一朗讀單位覆寫。
- FR-SCAN-003 應提供頁面邊界偵測、自動裁切、透視校正、旋轉修正與基本曝光改善。
- FR-SCAN-004 必須在保存前提供預覽、重拍與接受。
- FR-SCAN-005 必須檢查至少以下品質：模糊、嚴重過暗或過曝、頁面占比過低、未偵測到有效頁面。
- FR-SCAN-006 必須允許無文字頁與大面積插圖頁，不得因沒有文字阻止保存。
- FR-SCAN-007 必須能插入、刪除及重新排序朗讀單位。
- FR-SCAN-008 應提示手指、大面積陰影及反光，但 MVP 不要求百分之百自動偵測。
- FR-SCAN-009 原始影像是否永久保存為使用者設定；MVP 預設保留處理後影像，原始影像在建書完成後可清理。

驗收：

- 每次接受掃描都建立獨立且有序的 ReadingUnit 與 PageAsset。
- 單頁失敗可重拍，不損壞既有單位或順序。
- 旋轉與座標轉換後，OCR bounding boxes 能正確疊回處理後影像。

### 6.3 OCR 與版面

- FR-OCR-001 必須對每個可朗讀單位執行本機 OCR。
- FR-OCR-002 第一版必須支援繁體中文與英文；語言由書籍設定提供優先順序。
- FR-OCR-003 每個 TextBlock 至少保存 text、confidence、normalizedBoundingBox、sourceOrder、readingOrder、includeInSpeech。
- FR-OCR-004 必須保存 OCR 引擎名稱、版本或 revision、處理時間與錯誤。
- FR-OCR-005 初始閱讀順序可依 OCR 順序與幾何規則產生，但不得視為最終正確。
- FR-OCR-006 OCR 失敗必須可單位級重試；其他單位仍可繼續。
- FR-OCR-007 無結果頁可被標記為 silent 或由使用者手動新增朗讀文字。

驗收：

- 繁中書籍的 OCR request 優先使用 zh-Hant，英文可作次要語言。
- OCR 文字、confidence 與 normalized bounding box 可在校正畫面重現。
- 改用新 OCR revision 重新辨識時，不直接覆寫使用者已確認文字，必須先提示。

### 6.4 文字校正與閱讀順序

- FR-EDIT-001 必須同時顯示處理後影像、文字區塊與目前閱讀順序。
- FR-EDIT-002 必須允許修改、刪除、新增、合併及拆分文字區塊。
- FR-EDIT-003 必須允許拖曳或其他清楚方式重排朗讀順序。
- FR-EDIT-004 必須允許排除頁碼、出版資訊、裝飾字與重複內容。
- FR-EDIT-005 必須允許預覽合併後朗讀稿。
- FR-EDIT-006 每個朗讀單位必須具備 reviewStatus：unreviewed、reviewed 或 needsReview。
- FR-EDIT-007 文字或語音設定改變後，相關 AudioAsset 必須變為 stale，不可誤認舊音訊仍有效。
- FR-EDIT-008 可：允許為個別字詞標注發音修正（如繁中破音字、人名）；實作方式依 Provider 能力（音標、SSML 或替換字）於 T07 Spike 一併評估。

驗收：

- 使用者儲存、離開並返回後，所有人工編輯與排序一致。
- 被排除文字不出現在 TTS request payload。
- 同一段文字未改變時，重跑生成不得重複計費。

### 6.5 TTS 生成

- FR-TTS-001 必須以 Provider protocol 抽象化雲端與本機語音服務。
- FR-TTS-002 MVP 的主要 Provider 為自架 Mac TTS 服務：開源 TTS 模型部署在使用者的 M2 MacBook Air 16GB，iPhone 經區域網路以 HTTP 呼叫（見 6.5.1）。模型候選：CosyVoice 2、GPT-SoVITS、fish-speech／OpenAudio、MeloTTS，由 T07 繁中盲測定案。Apple AVSpeechSynthesizer（write API 落地音檔）為 Mac 不在線時的保底 Provider。第三方雲端不在 MVP，但抽象層允許未來加入。
- FR-TTS-003 必須支援 voice、speed、volume；style、pitch 僅在 Provider 支援時顯示。
- FR-TTS-004 必須以 TextBlock 或可控長度片段生成，並以 ReadingUnit manifest 決定播放順序。
- FR-TTS-005 每個 request 必須有 contentHash，包含標準化文字、Provider、model、voice、生成參數、generationFormatVersion 與分段規則版本；canonicalization 規則見 23.5。
- FR-TTS-006 相同 contentHash 已有有效檔案時必須重用，不重新運算生成。
- FR-TTS-007 生成佇列必須支援進度、取消、單項重試、指數退避及 Provider 錯誤訊息轉譯。
- FR-TTS-008 新檔下載與驗證成功後才能原子替換舊檔。
- FR-TTS-009 書籍只有在所有 includeInSpeech 區塊具備有效音訊，或被明確標成 silent，才可為 Ready。
- FR-TTS-010 Apple 本機 TTS 至少作為保底 Provider；若 T07 選定裝置端神經 TTS 模型，Apple TTS 仍保留為預覽與降級選項。
- FR-TTS-011 生成前必須顯示影響範圍（朗讀單位數、區塊數、字元總數）；更換 voice 或 model 等會使整本音訊變 stale 的操作，必須顯示重生成範圍並二次確認，避免非預期的整本重新生成（成本是等待時間與 Mac 運算資源）。

#### 6.5.1 Mac 端 TTS 服務（SelfHostedSpeechServer）

- SRV-001 服務以 HTTP REST 提供 health、voices、synthesize 三個端點；request/response 即為平台中立的 Speech Provider contract（23.2）。
- SRV-002 synthesize 輸入為校正後文字與 VoiceProfile 參數，輸出為 canonical 音訊（MP3，取樣率與 bitrate 寫入回應）；轉檔在 Mac 端完成，iPhone 不做轉碼。
- SRV-003 回應必須帶 modelID 與 engineVersion，兩者納入 contentHash；Mac 換模型或升版後，舊音訊依 hash 規則自然變 stale。
- SRV-004 連線方式：預設以 Bonjour 在區域網路自動發現，並允許手動輸入 IP:port；連不上時佇列保留、可稍後續作，並可切換 Apple 保底 Provider。
- SRV-005 服務僅監聽私有網段，不對公網開放；是否加簡單存取 token 為待決策（21）。
- SRV-006 服務程式碼與模型版本管理放在 repo 的 server/ 目錄；在 M2 16GB 上的生成即時率（RTF）與記憶體占用需於 T07 量測並記錄。

驗收：

- 飛航模式閱讀既有書籍時，網路請求數為零；建書的 TTS 流量僅限與 Mac 服務所在的私有網段，無任何對外連線。
- 模擬 Mac 服務離線、生成失敗、逾時、回傳空檔與磁碟已滿，均有可理解且可重試的狀態；服務離線時佇列保留並提示，可改用保底 Provider。
- 只修改一個 TextBlock 時，只重新生成該區塊。

### 6.6 辨識索引

- FR-IDX-001 完成掃描後，必須為每個 ReadingUnit 建立一或多個 RecognitionTarget。
- FR-IDX-002 索引至少包含正規化參考影像、Vision feature print、影像尺寸、方向、演算法版本與品質分數。
- FR-IDX-003 應從參考影像產生亮度或尺度變體，實際數量由 benchmark 決定。
- FR-IDX-004 索引建置必須可重跑並以 indexVersion 控制遷移。
- FR-IDX-005 索引損壞時必須能由保存影像重建，不要求重新掃描。

### 6.7 閱讀相機與頁面辨識

- FR-READ-001 MVP 必須先由使用者選擇一本書，搜尋範圍不得跨書。
- FR-READ-002 必須以 AVFoundation 取得即時 frame；辨識應在背景序列執行，不阻塞 UI。
- FR-READ-003 應限制分析 frame rate，初始建議 5–10 fps，再依基準測試調整。
- FR-READ-004 每個分析 frame 先偵測或估計頁面 ROI，做方向及透視正規化。
- FR-READ-005 第一階段以 feature print 距離取 Top-K 候選；第二階段對候選做幾何對齊與殘差驗證。
- FR-READ-006 確認必須同時考量候選分數、第一與第二名差距、幾何品質、連續出現時間及最近確認頁。
- FR-READ-007 門檻不可硬編碼散落在 UI；必須集中為可記錄版本的 RecognitionPolicy。
- FR-READ-008 比照點讀筆語意：每次「掃描並確認」都是明確的播放指令，重掃同一單位時從頭重播；沒有掃描或人工操作時不得有任何自動播放。
- FR-READ-009 新頁被確認時，必須停止舊頁音訊、切換 currentReadingUnit，並播放新頁。
- FR-READ-010 confidence 低或候選歧義時不得播放；畫面顯示「請對準完整頁面」等可行動提示。
- FR-READ-011 必須支援 Replay、Play、Pause、Resume、Stop、Previous 與 Next。
- FR-READ-012 Previous/Next 是人工導覽：不開啟相機，直接播放指定單位。
- FR-READ-013 無文字或 silent 單位確認後不播放，進入等待換頁狀態。
- FR-READ-014 相機中同時出現兩個單頁 target 時，優先使用該書的 captureMode 與 ROI 占比；無法穩定判斷則不播放。
- FR-READ-015 App 進入背景時停止相機 session；MVP 不承諾背景持續辨識。
- FR-READ-016 朗讀單位確認後必須立即停止 capture session 與辨識運算；播放期間相機保持關閉。翻頁切換由使用者重新啟動掃描或以 Previous/Next 觸發。

驗收：

- 正面、斜角、遠近、部分遮擋、弱光、強光、書頁輕彎及快速翻頁均納入實機資料集。
- 候選只出現單一 frame 時不可觸發播放。
- 新頁穩定確認時，即使舊頁仍在播放也必須切換。
- 低 confidence 期間保持安靜，不以最接近候選猜測播放。

### 6.8 音訊播放

- FR-AUD-001 使用 AVAudioSession playback 類別及 spokenAudio 或經驗證的合適 mode。
- FR-AUD-002 播放器必須支援多 TextBlock 連續播放與目前區塊追蹤。
- FR-AUD-003 必須處理耳機、藍牙等 route change，以及電話或 Siri 等 interruption。
- FR-AUD-004 音量設定不應偷偷改變系統音量；App 音量作為播放器增益。
- FR-AUD-005 背景音訊不列入 MVP 必須項；若啟用，需要另行確認 Background Modes 與產品行為。
- FR-AUD-006 音訊檔缺失或損壞時不得崩潰，應提示重新生成。

### 6.9 空間與資料管理

- FR-STO-001 必須顯示總占用及每本書的影像、音訊、索引分類占用。
- FR-STO-002 必須可刪除整本書、僅刪除原始掃描、或刪除並重新生成音訊。
- FR-STO-003 metadata 使用 SwiftData；大型影像與音訊使用 Application Support 檔案系統，只在資料庫保存相對路徑與雜湊。
- FR-STO-004 寫檔採暫存檔、驗證、原子 rename；避免半成品被視為有效資產。
- FR-STO-005 必須偵測可用空間不足，並在掃描或 TTS 前及寫入失敗後提供處理方式。
- FR-STO-006 重要資產採檔案保護。因建書 job 需在背景（含鎖屏）續跑（NFR-PERF-005），建書中間產物與寫入目標不可使用 complete protection，應改用 completeUntilFirstUserAuthentication 或 completeUnlessOpen；實際等級於 T03/T08 定案並記錄。
- FR-STO-007 不將使用者書籍影像或音訊放入可被系統任意清除的 Cache 目錄。
- FR-STO-008 已決策（2026-08-17）：MVP 不處理備份策略，維持系統預設行為；大型資產仍集中於單一根目錄，未來若需調整 isExcludedFromBackup 可單點切換。

### 6.10 設定與權限

- FR-SET-001 必須提供相機用途說明 NSCameraUsageDescription。
- FR-SET-002 不使用麥克風，因此 MVP 不請求麥克風權限。
- FR-SET-003 若不寫入 Photos，就不請求相簿權限。
- FR-SET-004 相機權限被拒時提供前往系統設定的說明；其餘書庫與編輯功能仍可使用。
- FR-SET-005 僅於未來啟用雲端 Provider 時適用：API Key 只能存 Keychain，禁止寫入 repository、UserDefaults、log 或 crash payload。
- FR-SET-006 僅於未來啟用雲端 Provider 時適用：必須提供測試 Provider 連線與刪除 Key 的操作。
- FR-SET-007 呼叫 Mac TTS 服務需要 Local Network 權限：提供 NSLocalNetworkUsageDescription 用途說明，使用 Bonjour 時一併宣告 NSBonjourServices；權限被拒時 TTS 生成步驟提示前往設定，其餘功能不受影響。

## 7. 非功能性需求

### 7.1 效能

- NFR-PERF-001 相機分析不得在 MainActor 執行影像特徵計算。
- NFR-PERF-002 同時間最多處理一個 live frame；新 frame 可丟棄，不得無限排隊。
- NFR-PERF-003 追蹤 recognition latency、confirmation latency、audio start latency 與 dropped frames。
- NFR-PERF-004 所有正式門檻在基準裝置 iPhone 15 Pro Max 上 benchmark 並凍結（目前唯一實機）；未來若要支援較舊機型，必須先在該機型重新 benchmark，不可沿用既有門檻。
- NFR-PERF-005 建書階段長任務必須可背景續跑：OCR 與索引等 iPhone 本機運算以 BGTaskScheduler（BGProcessingTask）排程；對 Mac 服務的 TTS 請求與音檔下載使用 URLSession background configuration。iOS 不保證背景執行時間與時點，因此每個 job 仍必須可安全暫停並於前景恢復（ProcessingJob resumablePayload）；背景續跑是體驗要求，可續作是正確性底線。背景寫檔需搭配相容的檔案保護等級（見 FR-STO-006）。

### 7.2 可靠性與恢復

- NFR-REL-001 每個 pipeline step 必須冪等，重試不建立重複資料。
- NFR-REL-002 單一 ReadingUnit 錯誤不得使其他單位不可編輯或播放。
- NFR-REL-003 schema 與 index 必須版本化，並具備 migration 或 rebuild 路徑。
- NFR-REL-004 app launch 時執行輕量完整性檢查；深度檢查由使用者或偵測到錯誤時執行。

### 7.3 隱私與安全

- NFR-SEC-001 OCR、索引與頁面辨識預設完全在裝置端。
- NFR-SEC-002 TTS payload 只送往使用者自己的 Mac（私有網段），內容限校正後文字與語音參數，不含頁面影像；不經任何第三方雲端。
- NFR-SEC-003 Debug log 不記錄完整 OCR 文字、API Key、Authorization header 或音訊二進位。
- NFR-SEC-004 Release build 關閉詳細 frame dump；開發診斷影像需由明確開關啟用並可一鍵清除。
- NFR-SEC-005 對外網路端點一律 HTTPS；區域網路的自架 Mac 服務可用 HTTP，但必須限制於私有網段並以 ATS local networking 例外明確宣告。錯誤畫面不得暴露秘密或完整 request。
- NFR-SEC-006 若產品範圍由私人側載改為對外散布，必須先完成後端 Key proxy、隱私政策、資料保留與供應商條款審查。

### 7.4 可維護性

- NFR-MNT-001 View 不直接依賴 TTS Provider 實作、Vision request 或檔案路徑。
- NFR-MNT-002 OCRService、PageRecognitionService、SpeechService、AudioService、BookRepository 與 AssetStore 以 protocol 隔離。
- NFR-MNT-003 Domain model 不引用 UIKit 或第三方 Provider 型別。
- NFR-MNT-004 所有 policy、threshold、model identifier 與格式版本集中管理並可測試。
- NFR-MNT-005 第一版避免導入未證明必要的後端、跨平台層或複雜 DI framework。

### 7.5 可用性與無障礙

- NFR-UX-001 閱讀模式控制需大而少，觸控目標至少符合 Apple 建議尺寸。
- NFR-UX-002 支援 Dynamic Type、VoiceOver label、足夠對比及不只靠顏色表達狀態。
- NFR-UX-003 建書進度要能回答「現在做什麼、還差什麼、失敗如何修復」。
- NFR-UX-004 一般閱讀不要求持續閱讀螢幕文字；狀態主要以簡潔視覺與可選聲音提示傳達。

## 8. 技術可行性與選定技術棧

### 8.1 建議最低平台

- Deployment target：iOS 17 或更新版本。
- UI：SwiftUI；必要的相機或 VisionKit controller 以 UIViewControllerRepresentable 包裝。
- Concurrency：Swift Concurrency，影像 pipeline 使用受控 actor 或序列 queue。
- Metadata：SwiftData。
- Large assets：Application Support 內的檔案系統。
- Capture：建書優先評估 VNDocumentCameraViewController；閱讀使用自訂 AVCaptureSession。
- OCR：VNRecognizeTextRequest accurate mode，語言優先 zh-Hant、en。
- Feature shortlist：VNGenerateImageFeaturePrintRequest。
- Candidate verification：VNHomographicImageRegistrationRequest，加上 warp 後殘差、重疊率及邊界合理性。
- TTS：SpeechService 抽象；主要 Provider 為自架 Mac TTS 服務（M2 MacBook Air 16GB 上的開源模型，經區域網路 HTTP），AVSpeechSynthesizer 為保底；第三方雲端保留為未來選項。
- Playback：AVAudioEngine 或 AVQueuePlayer 二選一；Spike 以無縫多段播放、seek 與 interruption 行為決定。
- Secret storage：Keychain Services。

iOS 17 是架構選擇，不代表所有 API 都只支援 iOS 17。原因是 SwiftData 可簡化本機 schema 與 migration，且私人自用場景不需要為極舊裝置增加 Core Data 相容成本。基準機 iPhone 15 Pro Max 出廠即 iOS 17，無升級疑慮，必要時可再提高 target。

### 8.2 Apple 原生與第三方邊界

| 能力 | 本機 Apple framework | 第三方需求 | 判斷 |
| --- | --- | --- | --- |
| 掃描、裁切、透視修正 | VisionKit / Vision / Core Image | 不需要 | 可完全本機 |
| 繁中、英文 OCR | Vision | 非必要 | 先以本機實測決定準確度 |
| 文字校正 | SwiftUI | 不需要 | 完全本機 |
| 頁面辨識 | Vision + AVFoundation | OpenCV 僅作備案 | 先做 Apple-only Spike |
| 自然 TTS | AVSpeechSynthesizer 可本機 | 高自然度用自架 Mac 開源模型（區域網路） | 已決策自架 Mac 服務；自然度以 T07 盲測驗收 |
| 音訊播放 | AVFAudio / AVFoundation | 不需要 | 完全本機 |
| 儲存與密鑰 | SwiftData / FileManager / Keychain | 不需要 | 完全本機 |
| AI 閱讀順序或角色分析 | 基礎幾何可本機 | LLM 為可選 | 不納入 MVP 必須 |

## 9. 頁面辨識技術比較與決策

| 方法 | 優點 | 缺點 | MVP 角色 |
| --- | --- | --- | --- |
| OCR 文字比對 | 對文字密集頁有辨識力；索引小 | 插畫或無字頁失效；即時 OCR 較慢；繁中錯字影響 | 只作歧義時可選輔助，不作主方法 |
| Perceptual hash | 非常快、容易實作 | 對透視、裁切、遮擋與相似頁脆弱 | 可作重複掃描檢查，不作最終辨識 |
| Vision feature print | Apple 原生、離線、可快速做相似度排序 | 全域特徵可能混淆同風格頁；局部畫面或強透視不穩 | 第一階段 shortlist |
| Homographic registration | 能驗證平面頁面在透視下是否能合理對齊 | 對每頁都跑成本高；彎曲、反光或大遮擋會降低品質 | Top-K 第二階段驗證 |
| ARKit reference images | 內建已知平面圖像偵測與 pose | 對彎曲與反光敏感；世界追蹤較重；參考圖數增加會降效能 | 不選為 MVP 主路徑 |
| OpenCV ORB / AKAZE | 局部特徵對旋轉、尺度與部分遮擋較強 | 增加 binary、依賴、橋接與維護成本 | Apple-only 方案未達標時的 fallback |
| 自訂 embedding / Core ML | 可針對書頁資料訓練，擴充性高 | 需資料集、訓練、模型版本與效能管理 | Phase 2 研究 |

### 9.1 選定的 MVP Pipeline

1. 從 AVCaptureVideoDataOutput 取得 frame，保留最新 frame。
2. 偵測頁面或主要矩形 ROI，正規化方向、尺度及透視。
3. 產生 Vision feature print，與目前書籍 targets 計算距離。
4. 保留 Top-K；初始 K 建議 3，需 benchmark。
5. 對 Top-K 執行 homographic registration。
6. 依 warp 合理性、有效重疊區、影像殘差、feature distance 與 runner-up margin 算 composite score。
7. 將候選送入 temporal stabilizer；候選需連續達標一段時間且未出現強競爭者。
8. 確認後設 page lock；只有可信的新候選或人工操作才能切換。

候選歧義時，可對頁碼區域做輕量 OCR 作為輔助訊號（多數書籍具頁碼）；此為可選優化，非 MVP 必須。同書高度相似頁不列為 MVP 阻擋條件（2026-08-17 產品決策），測試時記錄即可。

### 9.2 必做技術 Spike

在大量 UI 開發前，先以 3 本不同風格、每本至少 20 個朗讀單位建立測試集，包含：

- 正面、左右各 20–45 度斜角。
- 近、正常、遠三種距離。
- 25% 與 50% 局部遮擋。
- 暖光、冷光、弱光與局部反光。
- 單頁、跨頁、無字頁、相似插圖頁。
- 快速翻頁與前頁局部殘留。

Spike 通過條件：

- 在基準裝置 iPhone 15 Pro Max 達成第 4.2 節的 accuracy、false autoplay 與 latency 指標。
- 若未達標，依序嘗試：調整 ROI/variants/policy、加入局部特徵、評估 OpenCV；不可用降低 confidence 門檻掩蓋誤播。

## 10. 系統架構

### 10.1 分層

- App：啟動、依賴組裝、環境與 navigation。
- Features：Library、BookBuilder、Scan、OCRReview、SpeechGeneration、Reader、Settings。
- Domain：Book、ReadingUnit、TextBlock、AudioAsset、RecognitionTarget、狀態與 use cases。
- Services：Camera、OCR、Recognition、Speech、Audio。
- Persistence：SwiftData repositories、schema migration。
- Infrastructure：AssetStore、Keychain、HTTP client、logging、metrics、Provider adapters。

資料流：

Camera / VisionKit → ImagePipeline → OCR → Review → SpeechQueue → IndexBuilder → Ready

Camera frame → ROI normalize → Feature shortlist → Geometric verify → Stabilizer → Reader state → Local audio

### 10.2 邊界規則

- Feature ViewModel 只呼叫 use case 或 service protocol。
- Provider adapter 負責外部 request/response；不得讓 Provider DTO 滲入 Domain。
- Repository 管 metadata；AssetStore 管檔案。跨兩者的操作由 use case 執行 transaction-like 協調。
- 閱讀狀態只有 ReaderSession 擁有；Camera、Recognizer 與 AudioPlayer 透過 event 回報，不各自維護 current page。

## 11. 資料模型

### 11.1 Book

- id: UUID
- title: String
- author: String?
- languageCodes: [String]
- notes: String?
- coverAssetID: UUID?
- captureMode: singlePage | spread
- status: draft | scanning | recognizingText | reviewing | generatingAudio | indexing | ready | readyWithWarnings | failed
- voiceProfileID: UUID?
- createdAt / modifiedAt / lastOpenedAt: Date
- schemaVersion: Int
- totalByteSize: Int64（可重算快取）

status = failed 保留給整本書層級的不可恢復錯誤（例如 schema migration 失敗且無法 rebuild）；單位級失敗以 ReadingUnit 與 AudioAsset 的狀態表達，不得把整本書標成 failed，見 12.1。

### 11.2 ReadingUnit

- id: UUID
- bookID: UUID
- sequenceIndex: Int
- label: String（顯示用，如「第 3–4 頁」）
- kind: singlePage | spread
- reviewStatus: unreviewed | reviewed | needsReview
- speechStatus: notNeeded | missing | queued | generating | ready | stale | failed
- indexStatus: missing | building | ready | stale | failed
- isSilent: Bool
- createdAt / modifiedAt: Date

ReadingUnit 是閱讀與播放的核心單位，避免把紙張物理頁、掃描影像與朗讀行為混為一談。

speechStatus 為衍生欄位，由該單位所有 includeInSpeech TextBlock 的 AudioAsset 狀態聚合：任一 failed 為 failed；否則任一 stale 為 stale；否則任一 queued/generating 為 generating；全部 ready 為 ready；無需朗讀（silent 或無 includeInSpeech 區塊）為 notNeeded。聚合規則必須有 unit test，不得與 AudioAsset 狀態各自為政。

### 11.3 PageAsset

- id: UUID
- readingUnitID: UUID
- role: raw | processed | thumbnail | recognitionReference
- relativePath: String
- pixelWidth / pixelHeight: Int
- orientation: Int
- byteSize: Int64
- sha256: String
- cropTransform: Codable matrix/quad
- createdAt: Date

### 11.4 TextBlock

- id: UUID
- readingUnitID: UUID
- sourceText: String
- editedText: String
- confidence: Float?
- normalizedBoundingBox: CodableRect
- sourceOrder: Int
- readingOrder: Int
- includeInSpeech: Bool
- speaker: String?
- style: String?
- pronunciationHint: String?（可選，發音修正用；MVP 可為 nil，見 FR-EDIT-008）
- ocrEngine / ocrRevision: String
- modifiedAt: Date

### 11.5 VoiceProfile

- id: UUID
- providerID: String
- modelID: String
- voiceID: String
- speed / volume / pitch: Float
- style: String?
- languageCode: String
- generationFormatVersion: Int

不保存 API Key；只保存 Keychain item identifier。

### 11.6 AudioAsset

- id: UUID
- textBlockID: UUID
- segmentIndex: Int（同一 TextBlock 依 FR-TTS-004 分段生成時的播放順序；未分段為 0）
- relativePath: String
- providerID / modelID / voiceID: String
- contentHash: String
- durationMs: Int
- format / sampleRate / bitrate: String or Int
- byteSize: Int64
- sha256: String
- status: queued | generating | ready | stale | failed
- providerTraceID: String?
- errorCode: String?
- createdAt: Date

### 11.7 RecognitionTarget

- id: UUID
- readingUnitID: UUID
- referenceAssetID: UUID
- variant: original | exposureAdjusted | scaleAdjusted
- featurePrintData: Data
- featurePrintRevision: Int
- qualityScore: Float
- algorithmVersion: Int
- engineID: String（如 vision-featureprint，見 23.6）
- engineVersion: String
- platform: ios | android
- rebuildable: Bool（恆為 true；索引是可由 PageAsset canonical image 重建的快取）
- createdAt: Date

### 11.8 ProcessingJob

- id: UUID
- bookID / readingUnitID: UUID?
- kind: ocr | speech | index | integrityCheck
- state: pending | running | succeeded | failed | cancelled
- attemptCount: Int
- progress: Double
- resumablePayload: Data?
- errorDomain / errorCode / safeMessage: String?
- createdAt / startedAt / finishedAt: Date?

## 12. 狀態機

### 12.1 建書狀態

Draft → Scanning → RecognizingText → Reviewing → GeneratingAudio → Indexing → Ready

規則：

- 任一處理階段可暫停並在重開 App 後恢復。
- 單位級失敗進入部分錯誤，不直接把整書標成 failed。
- Ready 的必要條件是：每個 ReadingUnit 有有效 reference/index；每個需朗讀的 TextBlock 有有效音訊；其餘單位明確標為 silent。
- Ready 書籍修改影像會讓 OCR、index 及相關 audio 依依賴關係變 stale。
- 只修改文字不應重建影像索引；只改語音參數不應重跑 OCR。

### 12.2 閱讀狀態

相機只在 searching 與 stabilizing 期間運作；confirmed 後立即關閉，播放與等待階段不開啟相機。使用者以「掃描」動作重新進入 searching。

- inactive：未啟動閱讀。
- requestingPermission：等待相機權限。
- startingCamera：建立 session。
- searching：沒有可信候選。
- stabilizing(candidate, evidence)：候選累積證據。
- confirmed(unit)：新單位已確認。
- playing(unit, block)：播放中。
- paused(unit, block)：人工暫停。
- waitingForNextScan(unit)：已播放完或 silent；相機已關閉，等待使用者啟動下一次掃描或人工導覽。
- manualOverride(unit)：Previous/Next 造成的短暫人工控制。
- unavailable(reason)：權限、資產或不可恢復相機錯誤。

主要轉移：

- searching + 合格候選 → stabilizing。
- stabilizing + 持續達標 → confirmed。
- stabilizing + 分數下降或競爭候選 → searching。
- confirmed → 立即關閉相機；有音訊 → playing（不論是否為上次播放的單位，重掃即重播）。
- confirmed + silent → waitingForNextScan。
- playing + 重新掃描確認同一 unit → 停止後從頭重播（點讀筆語意）。
- playing + 新 unit confirmed → 停止舊音訊 → playing(new)。
- playing + audio ended → waitingForNextScan。
- waitingForNextScan + 重新掃描確認同一 unit → playing(same, fromStart)。
- waitingForNextScan + 新 unit confirmed → playing(new)。
- 任意狀態 + Replay → playing(current, fromStart)。
- 任意相機狀態 + app background → inactive，停止 capture。

Recognizer 只產生 candidate event；只有 ReaderSession 可決定播放，藉此避免 Camera callback 直接觸發重複音訊。

## 13. 錯誤處理

| 錯誤 | 使用者行為 | 系統行為 |
| --- | --- | --- |
| 相機權限拒絕 | 顯示原因與開啟設定 | 不啟動 capture；書庫與編輯仍可用 |
| 掃描模糊或頁面過小 | 重拍或強制接受 | 保存品質警告；不自動丟棄 |
| OCR 無結果 | 重試、手動輸入或標 silent | 只標記該單位 |
| TTS 401/403 | 更新 API Key | 停止新 request，不刪既有音訊 |
| TTS 429/5xx/timeout | 稍後重試 | 指數退避、保留佇列 |
| 儲存空間不足 | 清理或取消 | 不提交半成品 |
| 索引損壞 | 重建索引 | 由處理後影像重建 |
| 音訊損壞或遺失 | 重新生成 | 略過損壞檔，不崩潰 |
| 低 confidence / 多候選 | 對準完整頁面 | 不播放，不猜測 |
| 相機中斷 | 等待恢復或離開 | 安全停止 session 與辨識工作 |
| metadata 與檔案不一致 | 執行修復 | 孤兒 temp 可清除；缺檔資產標 stale/missing |

## 14. 安全、隱私、著作權與 Developer Account

### 14.1 API Key

MVP 採本地端 TTS，無 API Key；本節僅於未來啟用雲端 Provider 時適用。

私人自用版本：

- 使用者自行輸入 Provider Key。
- Key 存在 Keychain，建議 accessibleWhenUnlockedThisDeviceOnly。
- repository 只放不含秘密的 example configuration。
- URLSession layer 必須遮罩 Authorization。

對外散布版本：

- 不可把共用 Provider Key 放進 App，即使經過字串混淆也不安全。
- 必須由後端代理驗證、限流、記帳與輪替 Key。
- 需要另行定義帳號或匿名裝置憑證、濫用防護、隱私政策與資料刪除。

### 14.2 資料傳輸

- OCR、影像特徵與 live camera frame 不離開裝置。
- TTS 僅傳校正後文字與必要語音參數，且只送往使用者自己的 Mac（私有網段），不經第三方。
- 未來 LLM 版面分析若要上傳影像或整頁文字，必須先新增獨立同意、供應商與保留政策，不可沿用 TTS 同意。

### 14.3 著作權範圍

- 定位為使用者處理自己持有書籍的家庭私人工具。
- MVP 不提供內容搜尋、公開資料庫、內容交換或分享市場。
- 若未來加入備份、分享或商業發布，需另行做著作權與服務條款法律審查；本文件不是法律意見。

### 14.4 Apple 帳號與權限

- 使用 Apple Account 即可透過 Xcode 安裝到個人裝置做基本測試。
- 相機、Vision、VisionKit、SwiftData、AVFoundation 與一般網路連線不需要特殊 entitlement。
- MVP 必要設定主要是 Camera usage description；不需要麥克風或 Photos 權限。
- App Store、TestFlight、CloudKit 等對外散布或進階能力需要 Apple Developer Program；不在 MVP。

## 15. Logging 與診斷

開發版以 OSLog 分 category 記錄：

- scan：影像尺寸、品質分數、處理耗時，不記錄原圖內容。
- ocr：block 數、語言、revision、confidence 統計、耗時；預設不記完整文字。
- recognition：target count、Top-K distances、composite score、runner-up margin、state transition、latency。
- speech：Provider、model、contentHash 前綴、status、retry、trace id；不記 Key 或完整文字。
- audio：asset id、start/stop/end、interruption、route change。
- storage：bytes、寫檔結果、integrity issue。

必須提供可匯出的「隱私安全診斷摘要」，內容不含書頁影像、完整 OCR 文字或秘密。

## 16. 測試策略

### 16.1 Unit Tests

- Book readiness 與 dependency invalidation。
- ReadingUnit 排序、插入與刪除。
- TextBlock 合併、拆分、includeInSpeech 與 contentHash。
- Reader state machine 的候選、穩定、確認、重播與切頁。
- RecognitionPolicy 門檻與 runner-up margin。
- Speech queue 的冪等、retry、cancel 與 stale replacement。
- AssetStore 原子寫入、雜湊與 orphan cleanup。
- Repository migration 與刪除 cascade。

### 16.2 Integration Tests

- Scan image → OCR → editable blocks。
- Reviewed text → mock SpeechProvider → audio manifest。
- Ready book → prerecorded camera frames → recognition → mock playback。
- 修改單一文字區塊 → 只重建該 audio。
- 模擬 App restart → job resume 與書籍可用性。
- metadata/file mismatch → integrity repair。

### 16.3 UI Tests

- 新增書籍與中斷續作。
- 文字編輯、排除與排序。
- TTS 失敗後重試。
- 權限拒絕流程。
- Reader 基本控制與無網路狀態。
- 刪除書籍與空間顯示。

### 16.4 實機與資料集測試

- 至少 3 本、合計至少 60 個 ReadingUnit。
- 每個 target 具多角度、距離、光照與遮擋樣本。
- 測試集與調參集分離，避免只對同一批畫面調到高分。
- 每次 algorithmVersion 或 policy 變更都輸出 accuracy、confusion matrix、false autoplay、P50/P95 latency。
- 誤播率分母「確認事件」與延遲量測點（穩定取景起點、confirmed、audio start）必須在 benchmark runner 與 App 內 metrics 使用同一定義（見 4.2）。
- 測試頁面要包含無文字頁、全圖頁與跨頁；同書相似頁列為記錄項而非 MVP 阻擋條件（2026-08-17 產品決策，歧義時可用頁碼輔助，見 9.1）。

## 17. 建議 Repository 與 Xcode 專案結構

    contracts/
      book-package.schema.json
      recognition-policy.schema.json
      domain-enums.json
      fixtures/
    ios/
      StoryLens/
        App/
          StoryLensApp.swift
          AppDependencies.swift
          Navigation/
        Domain/
          Models/
          States/
          Policies/
          UseCases/
          Protocols/
        Features/
          Library/
          BookBuilder/
          Scanner/
          OCRReview/
          SpeechGeneration/
          Reader/
          Settings/
        Services/
          Camera/
          OCR/
          Recognition/
          Speech/
          Audio/
        Persistence/
          Models/
          Repositories/
          Migrations/
        Infrastructure/
          AssetStore/
          Keychain/
          Networking/
          Logging/
        Resources/
      StoryLensTests/
        Unit/
        Integration/
        Fixtures/
      StoryLensUITests/
    android/
      README.md
    server/
      tts/
        README.md
    shared/
      README.md

contracts 是平台中立規格與 golden fixtures；ios 是目前實作；server 是部署在使用者 Mac 的 TTS 服務（Python，見 6.5.1）；android 在第二階段建立；shared 保留給未來經決策採用的 KMP 或 native algorithm library，第一版不先放業務程式碼。

## 18. MVP、後續階段與切割

### 18.1 MVP

- 本機書庫與建書續作。
- 單頁／跨頁掃描、影像校正及品質提示。
- 繁中／英文本機 OCR。
- 文字區塊校正、排除與閱讀順序編輯。
- 自架 Mac TTS 服務為主要 Provider、Apple TTS 保底、預生成與本機音訊。
- 已選書範圍內的 Hybrid 頁面辨識。
- 穩定、debounce、page-lock、防重播。
- 基本播放控制。
- 完全離線閱讀。
- 空間顯示、刪除、局部重試與診斷。

### 18.2 Phase 2

- Android 原生版本：匯入或建立相同 Book Package，重建 Android 專屬辨識索引並達成相同核心驗收。
- 手機畫面上點選 OCR bounding box，從指定段落朗讀。
- 跨書自動辨識。
- 可選的本機或雲端備份／匯出。
- 日文與韓文。
- 視測試結果加入 OpenCV 或 Core ML recognition。

### 18.3 Phase 3

- 手指尖偵測、頁面 homography 與實體座標映射。
- 指向實體段落後播放對應 TextBlock。
- 兒童誤觸、遮擋與安全 UX 專項測試。

### 18.4 Optional / Research

- LLM 協助閱讀順序。
- 角色、旁白、情緒與多聲線。
- 廣播劇音效與背景音樂。
- Cloud sync、多使用者與公開發行。

## 19. 實作 Roadmap 與 Gate

### Gate 0：需求凍結

- 確認第 21 節待決策事項。
- 已完成：基準 iPhone 為 iPhone 15 Pro Max，deployment target iOS 17。
- 確認測試書籍可合法用於內部測試。

### Gate 1：技術 Spike

- VisionKit 掃描與繁中 OCR 品質。
- Apple-only Hybrid recognition benchmark。
- Mac 端開源 TTS 模型（CosyVoice 2、GPT-SoVITS、fish-speech、MeloTTS 等）與 Apple TTS 的繁中聲音盲測、生成速度（RTF）、記憶體占用與格式比較。
- Audio player 多段無縫播放。

只有 recognition 與 TTS 通過最低可行門檻後，才進入完整 UI。

### Gate 2：垂直切片

- 一本小型測試書：掃描 3 個單位 → OCR → 修字 → mock/real TTS → 建索引 → camera 辨識 → 播放。
- 驗證資料模型、檔案生命週期與 Reader state machine。

### Gate 3：完整 MVP

- 完成書庫、續作、全書佇列、錯誤恢復、設定、空間管理與測試資料集。

### Gate 4：Hardening

- 效能、電量、熱狀態、權限、損壞資料、網路錯誤、migration、Accessibility。
- 以第 4.2 節指標出具驗收報告。

### Gate 5：Android Port

- 凍結 Book Package、座標、contentHash、ReaderSession scenario 與 Provider contracts。
- 在最低、基準與高階 Android 裝置各完成 Camera/OCR/Recognition Spike。
- 以原生 Kotlin/Compose App 實作 platform adapters，優先重用規格與 fixtures，不預設重用 iOS runtime code。
- Android 通過與 iOS 相同的離線閱讀核心驗收後，再評估 KMP 或共享影像核心。

## 20. Atomic Task Breakdown

以下 iOS 路徑均位於 ios/ 子目錄，是第 17 節建議結構；建立 Xcode project 後再將 glob 轉成實際檔案白名單。

### T01 — 建立專案骨架與 CI-safe 測試入口

- Goal：建立 iOS 17 SwiftUI 專案、target、test target 與依賴組裝。
- Files allowed：StoryLens/App/**、StoryLensTests/**、project.pbxproj。
- Dependencies：Gate 0。
- Input：bundle id、team、deployment target。
- Expected output：可 build、可在 simulator 跑空白 app、unit tests 可執行。
- Acceptance：Debug/Release build 成功；無秘密進 repository。
- Tests：smoke unit test。

### T02 — Domain model 與 readiness 規則

- Goal：建立 Book、ReadingUnit、TextBlock、AudioAsset、RecognitionTarget、ProcessingJob 與狀態規則。
- Files allowed：StoryLens/Domain/**、StoryLensTests/Unit/Domain/**。
- Dependencies：T01。
- Input：第 11、12 節。
- Expected output：純 Swift domain types 與 use cases。
- Acceptance：不引用 SwiftUI、UIKit、Vision 或 Provider DTO。
- Tests：readiness、stale propagation、排序、刪除依賴。

### T03 — SwiftData 與 AssetStore

- Goal：實作 metadata persistence、Application Support 資產與原子寫檔。
- Files allowed：StoryLens/Persistence/**、StoryLens/Infrastructure/AssetStore/**、對應 tests。
- Dependencies：T02。
- Input：資料模型與檔案規則。
- Expected output：repository、asset store、初版 schema。
- Acceptance：重開後資料存在；缺檔可偵測；大型 blob 不進 SwiftData。
- Tests：CRUD、cascade、atomic failure、integrity scan。

### T04 — 掃描 Spike

- Goal：驗證 VisionKit 掃描、結果影像、單頁／跨頁及品質檢查。
- Files allowed：Spikes/Scanner/**、Tests/Fixtures/Scanner/**。
- Dependencies：T01。
- Input：3 本測試書。
- Expected output：benchmark report 與 sample captures；不直接併入 production。
- Acceptance：能取得可供 OCR 與索引的校正影像；記錄失敗類型。
- Tests：實機矩陣與座標 overlay。

### T05 — OCR Service 與測試報告

- Goal：實作 Vision OCR adapter，驗證 zh-Hant/en。
- Files allowed：StoryLens/Services/OCR/**、StoryLens/Domain/Protocols/OCR*、對應 tests/fixtures。
- Dependencies：T02、T04。
- Input：處理後掃描影像。
- Expected output：TextBlock draft、confidence、bounding boxes、revision metadata。
- Acceptance：結果可重現且座標正確；無結果為正常 domain outcome。
- Tests：繁中、英文、混合、無字、旋轉、低品質。

### T06 — OCR 校正 Feature

- Goal：完成文字編輯、排除、合併、拆分與閱讀順序。
- Files allowed：StoryLens/Features/OCRReview/**、必要 Domain use cases、對應 UI/unit tests。
- Dependencies：T03、T05。
- Input：OCR draft。
- Expected output：reviewed TextBlocks。
- Acceptance：離開返回不遺失；編輯後正確標示 audio stale。
- Tests：編輯與排序 UI flow、contentHash inputs。

### T07 — Speech Provider Spike 與選型

- Goal：在 M2 MacBook Air 16GB 上比較開源 TTS 模型（CosyVoice 2、GPT-SoVITS、fish-speech／OpenAudio、MeloTTS）與 Apple TTS 的繁中/英文品質、生成速度（RTF）、記憶體占用、格式與落地方式，並定案 Mac 服務的 REST contract。
- Files allowed：Spikes/Speech/**、server/**、docs/benchmarks/**。
- Dependencies：Gate 0 的測試文字。
- Input：代表性旁白、對話、標點與中英混合句。
- Expected output：決策紀錄、選定 Provider/model/format。
- Acceptance：至少一個 Mac 端模型通過繁中主觀聲音驗收、在 16GB 記憶體內穩定運行且生成時間可接受；同時確認 Apple TTS 保底路徑可用。
- Tests：short/long text、特殊符號、timeout、rate limit。

### T08 — SpeechService、Keychain 與生成佇列

- Goal：實作 Mac 服務 HTTP adapter（Bonjour／手動位址、離線佇列）、Apple TTS 保底 adapter、冪等生成佇列與本機 audio asset；Keychain 僅在未來啟用第三方雲端時實作。
- Files allowed：StoryLens/Services/Speech/**、StoryLens/Infrastructure/Keychain/**、Networking/**、對應 tests。
- Dependencies：T03、T06、T07。
- Input：reviewed TextBlocks、VoiceProfile。
- Expected output：可恢復生成佇列與有效 AudioAssets。
- Acceptance：相同 hash 不重新請求；Mac 離線時佇列保留且可續作；App 退背景後請求與下載以背景 session 繼續；失敗可局部重試。
- Tests：mock HTTP statuses、cancel、restart、atomic replace。

### T09 — Page Recognition Spike

- Goal：以 Apple-only Hybrid pipeline 驗證準確率、誤播與效能。
- Files allowed：Spikes/Recognition/**、Tests/Fixtures/Recognition/**、docs/benchmarks/**。
- Dependencies：T04。
- Input：第 9.2 節資料集。
- Expected output：離線 benchmark runner、報告、建議 K 與初始 policy。
- Acceptance：達成第 4.2 指標，否則產出明確 fallback 決策。
- Tests：固定資料集 regression。

### T10 — Production RecognitionService 與 IndexBuilder

- Goal：將通過的 pipeline 模組化並接入 assets。
- Files allowed：StoryLens/Services/Recognition/**、Domain/Policies/Recognition*、對應 tests。
- Dependencies：T03、T09。
- Input：處理影像與 policy。
- Expected output：versioned index、Candidate event stream。
- Acceptance：不直接觸發 audio；一次只處理最新 frame；index 可重建。
- Tests：ranking、verification、stabilization、version migration。

### T11 — CameraService 與 Reader state machine

- Goal：完成閱讀相機生命週期、權限、事件與防重播狀態機。
- Files allowed：StoryLens/Services/Camera/**、StoryLens/Features/Reader/**、Domain/States/Reader*、對應 tests。
- Dependencies：T10。
- Input：Candidate stream。
- Expected output：ReaderSession 與 camera preview。
- Acceptance：單幀不播、確認後立即關閉相機、掃描確認即播放（重掃同頁從頭重播）、無掃描不自動出聲、背景停止 camera。
- Tests：完整 state transition、權限拒絕、interruption。

### T12 — AudioService 與播放控制

- Goal：實作多區塊播放、切頁停止、pause/resume/replay/previous/next。
- Files allowed：StoryLens/Services/Audio/**、Reader control UI、對應 tests。
- Dependencies：T08、T11。
- Input：AudioAsset manifest。
- Expected output：Audio events 與可用控制。
- Acceptance：新頁確認立即切換；缺檔不崩潰；處理 route/interruption。
- Tests：mock player state、integration with ReaderSession、實機耳機測試。

### T13 — Book Builder Orchestration

- Goal：串接掃描、OCR、review、speech、index 與 readiness。
- Files allowed：StoryLens/Features/BookBuilder/**、Domain/UseCases/BuildBook*、必要 integration tests。
- Dependencies：T05、T06、T08、T10。
- Input：各服務 protocol。
- Expected output：可中斷續作的完整建書流程。
- Acceptance：單位級失敗可修復；重啟後恢復；Ready 條件嚴格。
- Tests：3-unit vertical integration、restart、partial failures。

### T14 — 書庫、空間、設定與完整性修復

- Goal：完成日常管理與可恢復性 UX。
- Files allowed：Features/Library/**、Features/Settings/**、Infrastructure/Logging/**、相關 use cases/tests。
- Dependencies：T03、T08、T13。
- Input：repositories、asset metrics、Keychain。
- Expected output：書庫、空間分類、刪除、Key 測試、診斷摘要。
- Acceptance：所有刪除需確認；不洩露文字或秘密；清理不影響其他書。
- Tests：size calculation、delete cascade、orphan cleanup、Key lifecycle。

### T15 — MVP 驗收與 Hardening

- Goal：執行第 16 節完整測試與第 4.2 指標驗收。
- Files allowed：tests、fixtures、docs/benchmarks、只修驗收發現且核准範圍內的 production files。
- Dependencies：T01–T14。
- Input：至少 3 本測試書與基準裝置 iPhone 15 Pro Max。
- Expected output：MVP acceptance report、已知限制、release candidate。
- Acceptance：第 4.2 全數通過，或每個例外都有明確產品簽核。
- Tests：unit、integration、UI、real-world、offline、thermal、storage failure。

## 21. 實作前待決策事項

下列項目不應由工程默默假設：

1. 已決策（2026-08-17）：基準與唯一測試機為 iPhone 15 Pro Max，deployment target 維持 iOS 17；支援更舊機型為延後議題。
2. 預設掃描模式以單頁還是跨頁為主；是否需要每書混用。
3. 已決策（2026-08-17）：TTS 採自架 Mac 服務（M2 MacBook Air 16GB）運算，無第三方雲端成本與資料出境議題。
4. Mac 端開源模型（CosyVoice 2、GPT-SoVITS、fish-speech、MeloTTS 等）何者通過繁中聲音盲測；若均不可接受，再評估第三方雲端（即推翻第 3 項）。
5. 已決策（2026-08-17）：掃描確認新單位立即中斷舊音訊並播放新單位（點讀筆語意）。
6. 是否保存原始掃描；本文件預設建書完成後可清理，保留處理後影像。
7. 是否需要鎖屏後繼續播完當前頁；本文件列為非 MVP。
8. App UI 語言是否只需繁體中文，或第一版即需英文介面。
9. Android 最低 API level、最低 RAM、基準 SoC 與首批實機清單；延至 iOS MVP 穩定後依當時市場決定。
10. 已決策（2026-08-17）：MVP 不處理備份策略，見 FR-STO-008。
11. 破音字與人名發音修正的實作方式（Provider 音標、SSML 或替換字），依 T07 Spike 結果定案。
12. 已隨 TTS 本地化簡化：僅剩 FR-TTS-011 重生成確認流程的文案（成本為等待時間）。
13. Mac 服務連線細節：Bonjour 自動發現或手動 IP、是否加簡單存取 token、Mac 不在線時的佇列提示文案。

## 22. MVP 最終驗收情境

Given：

- 使用者持有一本沒有點讀功能的普通實體繪本。
- App 已安裝在 iPhone 15 Pro Max；建書時 Mac TTS 服務與 iPhone 在同一區域網路。

When：

- 使用者新增書籍並逐朗讀單位掃描。
- App 完成 OCR，使用者修正文字與閱讀順序。
- App 預先生成語音並建立辨識索引。
- 使用者關閉 App、切到飛航模式後重新開啟。
- 使用者選擇該書並將相機依序對準任意已建單位。

Then：

- App 不重新做完整 OCR，也不呼叫 TTS。
- App 在指標時間內正確確認並播放本機音訊。
- 定位成功後相機自動關閉；重掃同一頁即從頭重播，無掃描時絕不自動出聲。
- 翻到新頁並重新掃描後，停止舊音訊並播放新單位。
- 低 confidence 或歧義時保持安靜並提示調整。
- 單一 silent 頁、缺字頁或歷史失敗不使整本書崩潰。
- 關閉並重開後資料仍完整。

全部成立且第 4.2 指標通過，才算 MVP 核心完成。

## 23. Android-ready 架構調整

### 23.1 建議策略

建議採「兩個原生 App、共用資料契約與測試規格、延後決定是否共用 runtime code」：

- iOS 先使用 SwiftUI、Vision、SwiftData 與 AVFoundation 完成穩定 MVP。
- Android 後續使用 Kotlin、Jetpack Compose、CameraX、ML Kit 或經 benchmark 選定的辨識方案。
- 現在先共用 schema、狀態轉移規格、RecognitionPolicy、TTS payload、contentHash 規則與測試 fixtures。
- 不在 iOS MVP 前導入 Flutter、React Native、Compose Multiplatform UI 或 KMP build dependency。
- iOS 穩定後，依兩端重複程式碼比例，再評估用 Kotlin Multiplatform 共用 Domain、networking、serialization 與部分 state machine；Camera、OCR、recognition、audio、storage 與 secrets 仍保留 platform adapter。

理由是本產品最困難的部分高度依賴相機、影像框架、裝置效能與生命週期。過早共用 UI 或 platform service 會增加 iOS MVP 複雜度；只共用契約則可降低未來移植成本，又不綁死技術。

### 23.2 現在就必須平台中立的模組

| 模組 | 現在的要求 | 未來共用方式 |
| --- | --- | --- |
| Domain IDs 與 enums | 不依賴 SwiftData、UIKit 或 Vision 型別 | JSON Schema；未來可轉 KMP commonMain |
| Book readiness | 以純規則描述，不寫在 SwiftData model callback | 共用 transition table 與 golden tests |
| Reader state machine | event、state、effect 分離 | 兩端跑相同 scenario fixtures；日後可共用程式 |
| RecognitionPolicy | threshold、Top-K、stabilization、page-lock 外部化 | versioned JSON；各平台可有不同數值 profile |
| Text normalization / contentHash | 明確定義 Unicode、換行、參數排序及 SHA-256 | 兩端同一 test vectors |
| Speech Provider contract | 不依賴 iOS SDK；使用平台中立 request/response | OpenAPI/JSON fixtures；可共用 networking |
| Book Package | metadata、影像、文字與音訊可匯出 | .storylensbook portable archive |
| Benchmark dataset | 同一批標註 frame 與預期 target | 兩端各自跑 accuracy/latency regression |
| Error taxonomy | Provider、storage、permission、recognition 錯誤分 domain code | UI 再轉成平台語言與操作 |
| ProcessingJob semantics | 冪等、重試、取消、進度與 dependency | iOS scheduler 與 WorkManager 各自實作 |

### 23.3 必須維持平台專屬的模組

| 能力 | iOS | Android | 共用限制 |
| --- | --- | --- | --- |
| UI / navigation | SwiftUI | Jetpack Compose | 共用 UX specification，不強求共用 UI code |
| Camera | AVFoundation / VisionKit | CameraX Preview、ImageCapture、ImageAnalysis | 共用 frame contract 與 orientation 規則 |
| OCR | Vision | ML Kit Text Recognition v2 或替代方案 | 共用 TextBlock output，不共用 observation 型別 |
| 頁面特徵 | Vision Feature Print | OpenCV local features 或 mobile embedding，待 Spike | 索引不可跨平台直接使用 |
| 幾何驗證 | Vision Homographic Registration | OpenCV findHomography 等候選 | 共用 CandidateResult contract |
| Metadata DB | SwiftData | Room | 不分享實體 DB file，只分享 export schema |
| Secret storage | Keychain | Android Keystore 加密的 credential storage | API Key 不進 Book Package |
| Audio | AVFoundation / AVFAudio | Media3 ExoPlayer | 共用 audio manifest 與格式 |
| Persistent jobs | iOS app lifecycle / BGTask 視需求 | WorkManager | 共用 ProcessingJob 狀態，不共用 scheduler |
| Permissions / storage | iOS privacy keys、Application Support | Android runtime permission、app-specific storage | 各自實作 |

### 23.4 可攜式 Book Package

Android 相容的關鍵不是共用 SwiftData，而是讓一本完成的書可以被平台中立地描述、匯出及重新索引。新增 .storylensbook 格式，實體可為 ZIP，但副檔名與 manifest 需自有版本：

    manifest.json
    assets/
      cover.jpg
      pages/{asset-id}.jpg
      audio/{asset-id}.mp3
    checksums.json

manifest 必須包含：

- packageVersion、schemaVersion、createdByPlatform、createdAt。
- Book、ReadingUnit、TextBlock、VoiceProfile 與 AudioAsset 的可攜欄位。
- 所有資產使用 UUID 與相對 POSIX path，不保存 sandbox absolute path。
- 日期使用 UTC ISO 8601；語言使用 BCP-47；enum 使用穩定字串值。
- 所有處理後頁面先烘焙正確方向，不依賴 EXIF 才能顯示正向。
- Canonical image 建議 JPEG/sRGB；不要把 HEIC-only 檔案當唯一可攜來源。
- Canonical audio 建議使用兩端普遍支援的 MP3；取樣率、bitrate、channels 明確寫入 manifest。
- checksum 使用 SHA-256；import 時先驗證再提交。
- 不包含 API Key、Keychain/Keystore identifier、診斷影像或裝置絕對路徑。

MVP 可以先只完成 schema 與 round-trip fixture，不必立即提供 UI 匯出。Android 開發前必須完成實際 export/import。

### 23.5 座標與文字正規化

Apple Vision 與 Android OCR 的座標原點與影像 rotation 表達不同，必須先定義 canonical contract：

- normalizedBoundingBox 使用 0…1。
- 原點固定在處理後正向影像的左上角。
- x 向右、y 向下。
- 所有 box 在 adapter 邊界完成轉換，Domain 不知道 Vision 或 ML Kit 座標。
- polygon 如有需要使用順時針點列，起點為左上最接近點。
- homography 使用明確的 row/column order、float precision 與 source/target direction。

contentHash 的 canonicalization 必須固定：

- 文字採 Unicode NFC。
- 換行統一為 LF。
- 是否 trim、連續空白及標點處理需版本化，不可由平台預設 locale 決定。
- Provider 參數以固定 key order 序列化後與文字一起 SHA-256。
- contracts/fixtures 必須提供繁中、英文、emoji、全形標點與混合文字 test vectors。

### 23.6 辨識索引需改成可重建快取

RecognitionTarget 的跨平台欄位（已併入 11.7）：

- platform: ios | android。
- engineID：例如 vision-featureprint 或 android-orb。
- engineVersion、policyVersion、deviceProfile。
- rebuildable: true。

規則：

- PageAsset 的 canonical processed image 才是辨識資料的真實來源。
- VNFeaturePrint bytes 不得放入 .storylensbook 的跨平台必要資料。
- Android 匯入 iOS 書籍後，以 page image 重建 Android index；反向亦然。
- 若未來兩端採相同的 OpenCV 或 Core ML/TFLite 模型，仍要以 engineVersion 判斷是否能重用，不可只看檔名。
- iOS 與 Android 不必使用同一辨識算法，但必須輸出相同 CandidateResult：readingUnitID、score、runnerUpMargin、geometricQuality、timestamp、engineVersion。

### 23.7 Android 技術對應

初步 Android 技術方向：

- Kotlin + Jetpack Compose。
- CameraX Preview + ImageCapture 用於掃描，ImageAnalysis 用於閱讀辨識。
- ML Kit Text Recognition v2 Chinese/Latin models 作為 OCR 首選候選。
- Room 儲存 metadata；大型 asset 放 app-specific files directory。
- WorkManager 處理可持續、可恢復的 OCR/TTS/index jobs；即時辨識使用 coroutine 與 latest-frame/backpressure policy。
- Android Keystore 保護本機加密 credential 所需 key material；Provider API Key 不以明文放 Room、DataStore 或 log。
- Media3 ExoPlayer 播放本機 manifest 與多段音訊。
- 頁面辨識先做 Android Spike，比較 OpenCV ORB/AKAZE + homography、mobile embedding + homography，以及跨平台共用 OpenCV core。

Android 裝置碎片比 iPhone 大，因此成功指標需按 device tier 設定。至少定義最低支援、基準與高階三個 profile，分別測試相機格式、rotation、memory、thermal、分析 fps 與 latency。

### 23.8 KMP 或共享 native core 的決策時點

iOS MVP 穩定後再做一次 architecture review：

選 KMP 的條件：

- Book/domain/state/networking 在 Android 端預估會重寫大量相同行為。
- 團隊願意讓 iOS build 納入 Kotlin/Gradle 產物與跨語言除錯。
- shared module 不需要直接處理高頻 camera frame。
- Swift/Kotlin API 邊界可保持少量、穩定與非 chatty。

不選 KMP、維持雙原生的條件：

- 開發者主要使用 Swift，Android 功能規模有限。
- 共用部分主要只是 schema、fixtures 與 Provider API。
- build、debug 或 binary 成本高於重複少量 domain code。

頁面辨識算法若最終希望兩端完全一致，另評估 C++/OpenCV 或 Rust static library；只有在兩端 benchmark 顯示一致算法確實降低維護成本時才採用。不要同時在第一版引入 KMP 與跨語言影像核心。

### 23.9 新增跨平台驗收

- iOS 匯出的 manifest 可由獨立 validator 解析，所有 checksum 正確。
- Android 可由同一 canonical page image 重建自己的 recognition index。
- iOS 與 Android 對相同 TextBlock/VoiceProfile 產生完全相同 contentHash。
- 兩端執行相同 ReaderSession scenario fixtures，最終 state 與 effects 一致。
- Bounding box 從兩端 adapter 輸出後，可在同一 canonical image 正確疊合。
- 已生成的 canonical MP3 可在 AVFoundation 與 Media3 播放。
- Book Package round trip 不遺失人工校正文字、閱讀順序、silent 狀態或 audio manifest。
- 任一平台專屬 index 缺失時，書籍可修復而非要求重新掃描。

### 23.10 對現有 Roadmap 的調整

在 T02 前新增 T00A：

- Goal：定義 book-package.schema.json、domain enums、canonical coordinate、contentHash test vectors 與 ReaderSession scenario format。
- Files allowed：contracts/**、docs/**。
- Dependencies：Gate 0。
- Expected output：平台中立 schema、validator fixtures 與版本策略。
- Acceptance：Swift test 能讀 fixtures；schema 不含 SwiftData、Vision、UIKit 或絕對 path。
- Tests：valid/invalid package、hash vectors、state scenarios、coordinate conversion。

T03 的 SwiftData model 必須視為 iOS persistence adapter，不得直接等同 Domain 或 export schema。T09/T10 的 Vision index 必須視為 rebuildable cache。Android 開發前新增 Android Recognition Spike，不能假設 iOS threshold 或 latency 能原樣沿用。

## 24. 技術查核來源

- Apple Vision 文字辨識與語言、confidence、bounding box：[Recognizing Text in Images](https://developer.apple.com/documentation/vision/recognizing-text-in-images)
- Vision RecognizeTextRequest：[Apple Developer Documentation](https://developer.apple.com/documentation/vision/recognizetextrequest)
- VisionKit 文件掃描：[VNDocumentCameraViewController](https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller)
- AVFoundation 相機架構：[Capture setup](https://developer.apple.com/documentation/avfoundation/capture-setup)
- 相機權限與 Info.plist：[Requesting authorization to capture and save media](https://developer.apple.com/documentation/avfoundation/requesting-authorization-to-capture-and-save-media)
- Vision Feature Print：[VNGenerateImageFeaturePrintRequest](https://developer.apple.com/documentation/vision/vngenerateimagefeatureprintrequest)
- Vision Homographic Registration：[VNHomographicImageRegistrationRequest](https://developer.apple.com/documentation/vision/vnhomographicimageregistrationrequest)
- ARKit 參考圖限制與注意事項：[Detecting Images in an AR Experience](https://developer.apple.com/documentation/arkit/detecting-images-in-an-ar-experience)
- SwiftData ModelContainer：[Apple Developer Documentation](https://developer.apple.com/documentation/swiftdata/modelcontainer)
- Keychain：[Keychain services](https://developer.apple.com/documentation/security/keychain-services)
- Apple TTS buffer 輸出：[AVSpeechSynthesizer write](https://developer.apple.com/documentation/avfaudio/avspeechsynthesizer/write(_:tobuffercallback:))
- Audio Session：[AVAudioSession](https://developer.apple.com/documentation/avfaudio/avaudiosession)
- MiniMax TTS API：[Text to Speech HTTP](https://platform.minimax.io/docs/api-reference/speech-t2a-http)
- Apple 帳號與裝置測試：[Apple Developer Program enrollment FAQ](https://developer.apple.com/help/account/membership/program-enrollment)
- Android CameraX 架構：[CameraX architecture](https://developer.android.com/media/camera/camerax/architecture)
- Android ML Kit 繁中 OCR：[Text Recognition v2 supported languages](https://developers.google.com/ml-kit/vision/text-recognition/v2/languages)
- Android Room：[Save data in a local database using Room](https://developer.android.com/training/data-storage/room)
- Android persistent work：[WorkManager task scheduling](https://developer.android.com/develop/background-work/background-tasks/persistent)
- Android Keystore：[Android Keystore system](https://developer.android.com/privacy-and-security/keystore)
- Android audio：[Media3 ExoPlayer](https://developer.android.com/media/media3/exoplayer)
- Android offline-first 架構：[Build an offline-first app](https://developer.android.com/topic/architecture/data-layer/offline-first)
- Kotlin Multiplatform 共用邏輯：[Share code on platforms](https://kotlinlang.org/docs/multiplatform/multiplatform-share-on-platforms.html)
