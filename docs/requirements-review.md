# StoryLens 開發需求評估報告

- 評估對象：`docs/requirements.md`（原 v0.2，評估後修正為 v0.3）
- 日期：2026-08-17
- 結論：**需求文件整體品質高，範圍切割合理，修正本報告所列問題後即可進入 Gate 0 需求凍結。**

## 1. 總體評估

原 v0.2 文件已經完成多數需求文件常見的困難工作：

- 明確區分「必須／應／可／待決策」，且把原始討論中的矛盾逐條列出並給出決議（第 3.2 節），這是最有價值的部分。
- 成功指標可量測（Top-1 辨識率、誤播率、延遲 P50/P95、離線驗收、空間預算），而不是「快速、穩定」這類無法驗收的形容詞。
- 技術選型有比較表與退場路徑（第 9 節），並以 Spike Gate 把最高風險項（頁面辨識、TTS 品質）擋在大量 UI 開發之前，這是正確的風險排序。
- 資料模型引入 ReadingUnit 解決單頁／跨頁的核心矛盾；contentHash 冪等生成避免重複計費。
- Atomic Task Breakdown（T00A–T15）有明確的檔案白名單、依賴與驗收，適合交給 Coding Agent 逐項執行。
- Android 章節採「共用契約、不共用 runtime」策略，避免 MVP 前過早抽象。

本專案風險最高的兩個技術假設仍未驗證（見第 4 節），文件已正確地將其設為 Gate 1 Spike 而非直接承諾，這個處理是對的。

## 2. 發現的問題與 v0.3 修正

以下問題已直接修入 `docs/requirements.md`（v0.3）：

| # | 問題 | 嚴重度 | v0.3 修正 |
| --- | --- | --- | --- |
| 1 | **AudioAsset 與 FR-TTS-004 矛盾**：FR-TTS-004 允許一個 TextBlock 分段生成，但 AudioAsset 只有 textBlockID，是 1:1 模型，分段後無法決定播放順序 | 高 | AudioAsset 新增 `segmentIndex` |
| 2 | **ReadingUnit.speechStatus 無聚合規則**：speechStatus 與 AudioAsset.status 兩處狀態並存，未定義由誰推導，容易出現不一致 | 高 | 明定 speechStatus 為衍生欄位，寫出聚合優先序（failed > stale > generating > ready / notNeeded），並要求 unit test |
| 3 | **Book.status = failed 與 12.1「單位級失敗不整書 failed」規則衝突**：沒有說明何時允許整書 failed | 中 | 明定 failed 保留給整書層級不可恢復錯誤（如 migration 失敗且無法 rebuild） |
| 4 | **contentHash 輸入不完整**：FR-TTS-005 未包含 generationFormatVersion 與分段規則版本；分段規則改變時舊 hash 會被誤重用 | 中 | FR-TTS-005 補齊輸入並指向 23.5 canonicalization |
| 5 | **RecognitionTarget 模型重複定義**：11.7 與 23.6 各有一版欄位，交給 Coding Agent 時會做出兩種實作 | 中 | 23.6 的 platform / engineID / engineVersion / rebuildable 併入 11.7，23.6 改為引用 |
| 6 | **「確認事件」無定義**：誤播率以確認事件為分母，但全文未定義何謂一次確認事件，指標無法客觀量測 | 中 | 定義為 recognition log 中一次 stabilizing → confirmed 轉移，並要求 benchmark runner 與 App 內 metrics 同一定義（4.2、16.4） |
| 7 | **整本重生成的成本防護缺失**：更換 voice/model 會讓全書音訊 stale，下一次生成就是整本重新計費，原文只有「不重複計費」的冪等保護，沒有「範圍確認」保護 | 中 | 新增 FR-TTS-011：生成前顯示影響範圍與字元總數，整本重生成需二次確認 |
| 8 | **建書長任務與 App 生命週期未規範**：32 單位的 TTS 生成可能耗時數分鐘，App 退背景時佇列行為未定義 | 中 | 新增 NFR-PERF-005：背景時安全暫停、可續作；背景續跑列為「可」 |
| 9 | **繁中 TTS 破音字／人名發音無對策**：兒童繪本朗讀對「長大」「重複」等破音字錯讀很敏感 | 低 | 新增 FR-EDIT-008（可）與 TextBlock.pronunciationHint 欄位，實作方式併入 T07 Spike 評估 |
| 10 | **備份策略未提**：Application Support 預設會進 iCloud/裝置備份，一本書上百 MB 會吃使用者備份額度；但私人自用場景又可能希望備份 | 低 | 新增 FR-STO-008 與待決策第 10 項：預設不排除，但資產集中單一根目錄以便日後單點切換 |
| 11 | **缺電量指標**：4.2 有 thermal 但無電量，20 分鐘連續相機＋辨識是高耗電場景 | 低 | 新增電量量測項，目標值於基準裝置凍結（初始建議 ≤ 10%） |

## 3. 已評估、判定不需修改的事項

- **150 MB 空間預算合理性**：32 單位 × 處理後影像（約 1–2 MB）＋ 縮圖 ＋ 索引 ＋ 音訊（每單位約 30–60 秒 MP3）粗估 60–120 MB，150 MB 預算合理，且文件已要求超標顯示實際占用而非靜默膨脹。
- **iOS 17 deployment target**：私人自用、SwiftData 簡化 migration 的理由成立，且 8.1 已保留「實機無法升級則重評」的退路。
- **MVP 先選書再辨識**：大幅縮小搜尋空間、直接提升 Top-1 準確率與延遲，是正確的範圍決議。
- **狀態機（第 12 節）**：「Recognizer 只發 candidate event、只有 ReaderSession 能觸發播放」的單一決策點設計正確，能防住重播與單幀誤播這兩個最傷體驗的 bug。
- **Keychain 私用版 + 對外散布必須後端代理**：安全邊界劃分正確。

## 4. 未修改、但必須在 Gate 1 前正視的風險

這些是技術假設風險，不是文件缺陷，文件已安排 Spike，此處列出以便排優先序：

1. **Vision Feature Print 對同書相似插圖頁的鑑別力**（最高風險）。同一畫風、同一主角的相鄰頁全域特徵可能非常接近，Top-K + homography 驗證是否足以拉開 runner-up margin，只能靠 T09 實測。若失敗，fallback（局部特徵／OpenCV）成本顯著。
   - 2026-08-17 產品決策：相似頁不列為 MVP 阻擋條件，測試時記錄即可；歧義時可用頁碼區 OCR 輔助（多數書籍有頁碼）。此風險由「阻擋」降為「觀察」。
2. **VNHomographicImageRegistrationRequest 的單幀成本**。對 Top-3 候選逐一做 homography，在較舊裝置上能否守住 5–10 fps 分析頻率與 1.5 秒 P95，需要 T09 一起量測；必要時第二階段只對第一名做驗證。
3. **MiniMax 繁中童書聲音品質與條款**。若盲測不過，整個 TTS 選型回到比較階段，會直接影響時程；建議 T07 與 T09 並行，不要串行。
   - 2026-08-17 更新：已決策改用本地端 TTS，MiniMax 退場；風險轉為「本地繁中語音自然度」（見第 6 節第 7 項），T07 改為比較 Apple TTS 與裝置端神經模型。
4. **SwiftData 成熟度**。SwiftData 在複雜 migration 與大量關聯下仍有已知穩定性風險；T03 驗收已含 migration 與 cascade 測試，建議在 T03 就把「退回 Core Data」的觸發條件寫進決策紀錄。
5. **complete file protection 與未來鎖屏播放互斥**（FR-STO-006 已註記）。若 21 節第 7 項決策改為「要鎖屏續播」，檔案保護等級要在建書時就選對，事後遷移成本高。
   - 2026-08-17 更新：因建書 job 改為背景（含鎖屏）續跑，建書相關檔案已明定不可用 complete protection（見 FR-STO-006 v0.4），此風險已有明確處理方向。

## 5. 建議的下一步

1. **完成 Gate 0**：第 1（基準機型）、3（TTS 成本）、5（換頁行為）、10（備份）已決策；剩餘的關鍵輸入是第 2（預設掃描模式：單頁或跨頁為主）。
2. **啟動 T00A（contracts）**：schema、contentHash test vectors、座標契約與 ReaderSession scenario fixtures 不依賴任何 Apple API，可立即在本 repo 開工。
3. **T07 與 T09 兩個 Spike 並行**：兩者是互不依賴的最高風險項，串行會白白拉長時程。T07 的 Mac 端服務與模型評測（server/）不依賴 iOS 程式碼，可立即在本 repo 開工。
4. **建立測試資料集前先確認 16.4 的「測試集／調參集分離」**，避免 Spike 報告過擬合。

## 6. 產品決策更新（2026-08-17）

需求文件 v0.4 已併入下列產品決策：

1. **建書 job 改為背景任務續跑**（對應第 2 節問題 8）：TTS 網路請求使用 URLSession background session，OCR 與索引使用 BGProcessingTask 排程。iOS 不保證背景執行時間，因此「可安全暫停、可續作」仍是正確性底線。連帶修正 FR-STO-006：背景寫檔不可使用 complete protection。
2. **MVP 不處理備份策略**（對應問題 10）：維持系統預設行為，FR-STO-008 與待決策第 10 項標記為已決策。
3. **閱讀時相機不常開**（修正原問題 11 的錯誤假設）：相機只在建書掃描與閱讀定位時短暫使用，定位確認後立即關閉並播放。原「閱讀 20 分鐘電量指標」建立在相機常開的假設上，已改為只記錄建書階段電量；第 2、3.2、4.2、5.2、6.7、12.2、22 節的閱讀流程與狀態機已同步改寫（waitingForChange 改為 waitingForNextScan，新增 FR-READ-016）。
4. **相似頁先不考慮**（對應第 4 節風險 1）：不列為 MVP 阻擋條件；多數書籍有頁碼，歧義時可用頁碼區 OCR 作為可選輔助訊號（9.1）。

需求文件 v0.5 再併入下列產品決策：

5. **互動模型 = 點讀筆**：掃一頁播一頁，想聽下一頁就掃下一頁；重掃同一頁即從頭重播。原「同一單位不重播」規則是為相機常開模式防誤觸發而設，現在每次掃描都是使用者的明確動作，已移除。待決策第 5 項（換頁即中斷）同時定案為「是」。
6. **基準裝置 = iPhone 15 Pro Max**：目前唯一實機；所有效能門檻在此機 benchmark 並凍結，未來支援較舊機型必須重新 benchmark。待決策第 1 項定案，deployment target 維持 iOS 17。
7. **TTS 改為本地端模型運算**：MVP 不使用第三方雲端 TTS——無 API 費用、無 Key 管理、無資料出境；Provider 抽象層保留未來加入雲端的空間。原問題 7 的「整本重新計費」防護改為「整本重新生成」防護。
   - 2026-08-17 補充（v0.6）：「本地端」明確為**自架在使用者 M2 MacBook Air 16GB 上的 TTS 服務**，iPhone 建書時經區域網路呼叫，閱讀仍完全離線。此架構讓品質風險大幅下降——Mac 可跑完整開源模型（候選：CosyVoice 2、GPT-SoVITS、fish-speech／OpenAudio、MeloTTS，繁中表現以 T07 盲測定案），Apple 內建 TTS 改為 Mac 不在線時的保底。新增的代價：建書時 Mac 必須在線且同一網段、iOS 需 Local Network 權限（FR-SET-007）、repo 多一個 server/ 元件要維護（6.5.1）。
8. **音檔以頁（ReadingUnit）為單位回傳與保存**（2026-08-17，v0.7）：Mac 端把整頁區塊依閱讀順序合成單一 MP3（含停頓控制，可選每區塊時間戳），iPhone 不做音訊拼接。資料模型與播放器同步簡化：AudioAsset 由「每區塊一檔（＋分段編號）」改為「每頁一檔」，第 2 節問題 1 的 segmentIndex 方案作廢；contentHash 改為頁級（區塊依 readingOrder 以固定分隔規則串接）。代價是改一個字會重生成整頁音檔——在 Mac 上運算成本低，可接受。

## 7. 修訂紀錄

- v0.2 → v0.3：套用第 2 節共 11 項修正（涉及 4.2、6.4、6.5、6.9、7.1、11.1–11.7、16.4、21、23.6 節）。原始 v0.2 內容未被刪除，僅補強與消歧。
- v0.3 → v0.4：併入第 6 節前四項產品決策（涉及 2、3.2、4.2、5.2、6.7、6.9、7.1、9.1、12.2、16.4、20、21、22 節）。
- v0.4 → v0.5：定案點讀筆互動模型、基準機 iPhone 15 Pro Max 與 TTS 本地端模型運算（涉及 2、3.1、3.2、4.2、5.2、6.5、6.7、6.10、7.1、7.3、8、9.2、12.2、14.1、18.1、19、20、21、22 節）。
- v0.5 → v0.6：TTS 本地端明確為自架 Mac 服務——新增 6.5.1（SelfHostedSpeechServer SRV-001–006）、FR-SET-007（Local Network 權限）、repo 結構加入 server/、NFR-SEC-005 放寬區網 HTTP、T07/T08 改為 Mac 模型評測與 HTTP adapter、待決策新增第 13 項。
- v0.6 → v0.7：音檔改以頁（ReadingUnit）為單位（涉及 6.4、6.5、6.5.1、6.8、11.2、11.6、12、16.2、20、23.5 節）；AudioAsset 改掛 readingUnitID，segmentIndex 移除，新增可選 blockTimestamps。
