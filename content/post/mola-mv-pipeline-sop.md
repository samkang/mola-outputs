---
title: "深度解析：基於 AI 與 FFmpeg 的高品質 MV 自動化生產管線 (Technical Deep Dive)"
date: 2026-02-15T12:00:00+08:00
url: /posts/mola-mv-sop/
draft: false
tags: ["Engineering", "FFmpeg", "Whisper", "AI", "Multimedia"]
categories: ["Technical Whitepaper"]
og_image: "https://samkang.github.io/mola-outputs/images/mola_mv_pipeline_cover.png"
og_video: "https://samkang.github.io/mola-outputs/dust-in-the-wind-bilingual.mp4"
---

在本文中，我們將從工程師的角度深入探討 Mola 如何自動化處理音樂影片 (MV) 的後製流程。這套系統的核心在於將非結構化的音訊與影像數據，透過一系列 AI 模型與指令集工具，轉化為具備電影質感且符合 Web 串流規範的最終產品。

## 1. 音訊預處理層 (Audio Frontend)
為了實現精準的視覺咬合，系統首先必須獲取高信噪比 (SNR) 的人聲軌。

- **模型選擇**: 採用 `Demucs v4 (Hybrid Transformer)`。相比於傳統的頻譜減除法，HT-Demucs 能在保持相位完整性的前提下，完美分離出人聲 (Vocal) 與伴奏 (Background Music)。
- **VAD 強化**: 分離後的人聲軌用於 `Whisper AI` 的輸入。這能消除 BGM 中高頻打擊樂器對語音活動檢測 (Voice Activity Detection) 的干擾，將時間軸偏移量誤差降低至 $\pm 50ms$。

## 2. 歌詞數據建模與翻譯
- **音韻校準**: 使用 `Whisper large-v3` 生成含有微秒級時間戳的 `JSON` 數據結構。
- **LLM 語意映射**: 透過 `Gemini-3-Pro` 進行上下文感知的雙語翻譯。在工程實踐中，我們會限制翻譯後的字串長度，以防止在 16:9 畫面中出現溢出 (Overflow) 或排版擁擠。

### 2.1 語意校準與文本注入 (Semantic Alignment & Injection)
這是確保歌詞 100% 正確的核心技術環節。
- **草稿比對**: Whisper 產出的原始聽寫 (Raw Transcription) 雖有時間戳，但可能存在語音誤判（如將 "Dust" 誤聽為 "Does"）。
- **LLM 文本替換**: 系統將「聽寫草稿」與「官方標準歌詞」同時輸入 LLM。透過語意比對技術，LLM 會識別每一行聽寫內容對應的標準文本，並執行 **「保留時間戳，完全替換文本」** 的操作。
- **優勢**: 這種做法結合了 Whisper 的「時序精準度」與官方歌詞的「文字權威性」，徹底消滅了 AI 聽寫的錯字問題。

## 3. 視覺資產預渲染 (Graphic Pre-rendering)
為了繞過 FFmpeg 內建 `drawtext` 濾鏡在處理多語言字體與動態排版時的效能瓶頸與穩定性問題，我們採用了 **預渲染圖層 (Layer Pre-rendering)** 技術。

- **環境依賴消除**: 在 macOS 下，ImageMagick (`magick`) 對系統字體冊的訪問可能不穩定。解決方案是採用「物理隔離」，將 `.ttc` 字體文件直接置於專案路徑，並使用絕對路徑引用。
- **美學參數**:
    - **Resolution**: 1280x720 (High Transparency PNG24)。
    - **Typography**: 中文 36pt (STHeiti) / 英文 32pt (Arial Bold)。
    - **Stroke Policy**: 使用 `None` (無描邊) 以維持最佳影像寬容度。

## 4. FFmpeg 複雜濾鏡鏈構建 (Filter Complex Topology)
這是系統的「大腦」，將所有素材合成。

### 4.1 時間偏移與黑場注入
使用 `color=c=black` 生成虛擬幀，結合 `concat` 濾鏡在影片首尾注入 3.0s 的純黑區間。

### 4.2 影像與音訊的淡入淡出 (Cinematic Fades)
- **Video**: `fade=t=in:st=0:d=1,fade=t=out:st=${DURATION-1}:d=1`。
- **Audio**: `afade=t=in:st=3:d=1,afade=t=out:st=${TOTAL_END-1}:d=1`。

### 4.3 鏈式疊加 (Chained Overlays)
由於每一句歌詞都是一個獨立的輸入流 (`-i sub_n.png`)，系統會動態生成鏈式濾鏡表達式：
`[v_in][1:v]overlay=enable='between(t,start,end)'[v_out_1]; [v_out_1][2:v]overlay=...`
這種結構確保了圖層切換的零延遲與內存管理的高效。

## 5. 編碼優化與 Web 分發 (Encoding & Distribution)
- **Codec**: `libx264`。
- **Rate Control**: `CRF=18` (視覺無損)。
- **Pixel Format**: `yuv420p`。這是為了確保影片能在幾乎所有移動端瀏覽器（包括低版本的 iOS Safari）中正常解碼。
- **Streaming Support**: 使用 `-movflags +faststart` 將 `moov atom` (中繼數據) 移至檔案頭部，實現邊下載邊播放。

## 6. 自動化部署與快取控制
最後，透過 Git 自動化腳本推送至 GitHub Pages。為了對抗 CDN 與瀏覽器的強力快取，發佈網址必須包含唯一的 Version Tag (`?v=${TIMESTAMP}`)，強制觸發內容更新。

---
**技術總結**: 本流程實現了從原始音訊到最終發佈的閉環自動化，充分利用了 AI 的感知能力與 FFmpeg 的處理能力，為高品質 Cover MV 的生產提供了可量化的標準。

## 🎬 最終成果展示

這是本生產管線最終產出的成品範例：

<video width="100%" controls>
  <source src="https://samkang.github.io/mola-outputs/dust-in-the-wind-bilingual.mp4?v=final_aeon" type="video/mp4">
  您的瀏覽器不支援影片播放。
</video>

🐟 *Mola Tech Notes v2.0*
