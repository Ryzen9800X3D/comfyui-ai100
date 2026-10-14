---
id: ai.troubleshooting.comfyui-minimax-h3-troubleshooting
type: troubleshooting
title: ComfyUI + MiniMax H3 疑難排解完整記錄
domain: ai
status: active
confidence: 0.95
tags:
  - comfyui
  - minimax-h3
  - amd-rocm
  - troubleshooting
  - quantops
  - clip-loader
  - generation-tail
  - pytorch
sources:
  - type: local
    reference: "D:/ComfyUI_Win_portable_RX9070 TRFv0.5-2/ComfyUI_windows_portable/"
  - type: incident
    reference: "CLIPLoader meta tensor error"
  - type: incident
    reference: "QuantOps register_layout_op failure"
relations:
  - ai.entity.minimax-h3-comfyui-setup
  - ai.knowledge.comfyui-quantops
  - ai.procedure.comfyui-ai100-deployment
created: 2026-08-08
updated: 2026-10-14
last_verified: 2026-08-08
---


# ComfyUI + MiniMax H3 疑難排解完整記錄

> 📌 **AI-100 實戰部署紀錄**（v0.40.0 + H3 導演台 + Qwen Image 2.1 T8）見 [[ComfyUI-AI100-Deployment]]。

## 🖥️ 環境資訊

| 項目 | 規格 |
|------|------|
| **平台** | ComfyUI Portable (Windows) |
| **GPU** | AMD RX 9070 XT |
| **ROCm** | 7.15 |
| **路徑** | `D:\ComfyUI_Win_portable_RX9070 TRFv0.5-2\ComfyUI_windows_portable\` |
| **ComfyUI** | v0.31.0-3-gdd79c643a |
| **torch** | 2.12.0+rocm7.15.0a20260727 |
| **torchvision** | 0.27.0+rocm7.15.0a20260727 |
| **comfy-kitchen** | 0.2.28 |
| **comfy-aimdo** | 0.4.13 |
| **comfyui-frontend** | 1.48.7 |

---

## ❌ 問題一：CLIP 載入時 `Cannot copy out of meta tensor; no data!`

### 錯誤節點
`MiniMaxH3AudioConditioningT8`（`NotImplementedError`）

### 原因分析
**CLIPLoader 選用的檔案錯誤：**
- 使用了：`qwen3vl_32b_h3_generation_tail_50_63_int8_convrot.safetensors`
- 這是 Qwen3-VL-32B 的**第 50~63 層「尾段（tail）」檔案**，不是完整文字編碼器
- 必須搭配「MiniMax H3 Generation Tail Loader」節點使用
- 借用已連接的完整 CLIP 的第 0~49 層 + embedding + vision tower

**直接載入的問題：**
- 第 0~11 層、embedding、position_embedding 等權重停留在「meta tensor」狀態
- 只有形狀、沒有實際資料
- 推論時嘗試複製到 GPU 便報錯

### ✅ 解決方法
**CLIPLoader 的 `clip_name` 換成完整的 MiniMax H3 文字編碼器：**

| 檔案 | 特點 |
|------|------|
| `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 檔案最小（AWQ 量化） |
| `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | int8 量化 |
| `qwen3vl_32b_minimax_h3_bf16.safetensors` | 最完整（bf16 精確） |

**放置位置：** `ComfyUI/models/text_encoders/`
**type 設定：** 維持 `minimax`

---

## ❌ 問題二：UNETLoader 載入時 `'NoneType' object has no attribute 'Params'`

### 錯誤節點
`UNETLoader`（`AttributeError`）

### 錯誤訊息
```python
[ERROR] ComfyUI-QuantOps: Failed to register layouts: cannot import name 'register_layout_op' from 'comfy.quant_ops'
[ERROR] Failed to import comfy_kitchen, Error: cannot import name 'AsymW4A8Int8Layout' from 'comfy_kitchen.tensor'
```

### 原因分析
**根本原因：版本不匹配**
- 核心 ComfyUI 是最新 nightly commit
- 周邊依賴（`comfy-kitchen` 0.2.22、`comfy-aimdo` 0.4.10）版本過舊
- API 對不上 → 註冊失敗 → 查詢量化 layout 類別回傳 `None`
- 呼叫 `None.Params()` 崩潰

**負責的模組：**
- `ComfyUI-QuantOps`（自訂節點）→ 註冊 int8_convrot 等量化格式
- `comfy-kitchen`（依賴套件）→ 提供量化 layout 類別

### ✅ 解決方法（完整更新流程）

#### Step 1: 更新 ComfyUI 核心
```bash
cd "D:\ComfyUI_Win_portable_RX9070 TRFv0.5-2\ComfyUI_windows_portable\"
git pull
# 若出現 tracking 錯誤，先執行：
# git branch --set-upstream-to=origin/master master
```

#### Step 2: 更新 Python 依賴套件
```bash
"...\python_embeded\python.exe" -m pip install -r "...\ComfyUI\requirements.txt" --upgrade
```

**版本變更：**
- `comfy-kitchen` 0.2.22 → **0.2.28** ✅
- `comfy-aimdo` 0.4.10 → **0.4.13** ✅

#### ⚠️ 副作用：ROCm PyTorch 被覆蓋
pip upgrade 把 ROCm 專用的 torch 覆蓋成通用版：
- `torch` 2.12.0+rocm7.15.0 → **2.13.0（無 ROCm）** ❌
- `torchvision` 同樣被換掉 ❌

#### Step 3: 修復 ROCm 版 PyTorch
執行客製腳本：
```bash
Update ROCm.bat
# （位於 ComfyUI_windows_portable 根目錄）
```

**修復後確認：**
- `torch` = **2.12.0+rocm7.15.0a20260727** ✅
- `torchvision` = **0.27.0+rocm7.15.0a20260727** ✅

#### Step 4: 確認 QuantOps 最新
```bash
# 在 ComfyUI-QuantOps 目錄執行
git pull  # 顯示 Already up to date
```

### ✅ 驗證結果
- 啟動 log 確認 QuantOps 成功註冊 layouts
- 量化模型可正常載入
- 完整 workflow 跑完 **318 秒生成成功** 🎉

### 📝 小插曲：Windows fatal exception: access violation
- `comfy_aimdo\control.py` 的 `init_devices` 在開機探測階段連續崩潰 **8 次**
- 屬於**良性重試**（benign retry）
- 之後仍成功抓到 GPU 並完成初始化
- `DynamicVRAM support detected and enabled` ✅
- **不影響後續使用**

---

## ❌ 問題三：Prompt Enhancer 報錯 `Connected CLIP uses SD1Tokenizer`

### 錯誤節點
`MiniMaxH3PromptEnhancer`（`RuntimeError`）

### 錯誤訊息
```python
Prompt Enhancer formats Qwen chat and vision tokens, but the connected CLIP uses
comfy.sd1_clip.SD1Tokenizer. Connect a complete Qwen3-VL/Qwen3.5 CLIP,
or MiniMax H3's CLIP with its Generation Tail Loader.
```

### 原因分析
**Prompt Enhancer 需求：**
- 需要 Qwen 的「對話格式」(chat template)
- 需要圖片 token 支援

**實際連接的 CLIP：**
- 標準 MiniMax H3 conditioning-only CLIP
- 只有第 0~49 層、簡化 tokenizer
- 格式不支援 chat template

### ⏳ 解決方法（擇一）

#### 方法 A（建議，省資源）：加掛 Generation Tail Loader
1. 新增節點：`MiniMax H3 Generation Tail Loader`
2. 選擇檔案：`qwen3vl_32b_minimax_h3_generation_tail_50_63_int8_convrot.safetensors`
   - 放置位置：`ComfyUI/models/text_encoders/MiniMax-H3/`
3. 連接輸出：
   - `clip_tail` → Prompt Enhancer 的 `clip_tail`（optional）輸入
4. **執行時暫時補滿 14 層（50~63 層），跑完自動卸載**
5. ✅ 不影響主要 conditioning CLIP

#### 方法 B：直接換完整生成式 CLIP
- 使用 `qwen3vl_32b_minimax_h3_bf16.safetensors` 或 Qwen3.5-4B
- 接到 Prompt Enhancer 的 `clip` 輸入
- ⚠️ 較吃 VRAM/硬碟

### 狀態：⏳ 待處理（尚未回報結果）

---

## 📊 MiniMax H3 CLIP 檔案分類

| 用途 | 層數 | 接法 | 檔案範例 |
|------|------|------|----------|
| **Conditioning**（影片生成文字引導） | 完整 0~49 層 | CLIPLoader | `qwen3vl_32b_minimax_h3_bf16.safetensors` |
| **Generation Tail**（Prompt Enhancer 用） | 50~63 層 | Generation Tail Loader | `qwen3vl_32b_h3_generation_tail_50_63_int8_convrot.safetensors` |