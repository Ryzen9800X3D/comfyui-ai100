---
id: ai.procedure.comfyui-installation-guide
type: procedure
domain:
  - infrastructure
  - infrastructure
version: 1
status: active
confidence: 0.85
tags:
  - comfyui
  - installation
  - windows
  - portable
  - amd-rocm
  - custom-nodes
sources:
  - type: external
    reference: https://docs.comfy.org/installation/comfyui_portable_windows
  - type: external
    reference: https://localaimaster.com/blog/comfyui-complete-guide
  - type: external
    reference: https://rocm.blogs.amd.com/artificial-intelligence/comfyui-radeon-9000/README.html
relations:
  - type: related_to
    reference: ""
  - type: related_to
    reference: "ai.procedure.comfyui-ai100-deployment"
created: 2026-08-08
updated: 2026-10-14
last_verified: 2026-08-08
title: ComfyUI 建置教學完整指南
---


# ComfyUI 建置教學完整指南

> 📌 **本機實戰紀錄**：AI Server 100（R9700）上的實際部署流程（v0.40.0 + MiniMax H3 導演台 + Qwen Image 2.1 T8 加速）見 [[ComfyUI-AI100-Deployment]]。

## 📋 目錄

1. [安裝方式比較](#1-安裝方式比較)
2. [Portable Windows 版本安裝](#2-portable-windows-版本安裝)
3. [Desktop App 安裝](#3-desktop-app-安裝)
4. [AMD ROCm 環境設定](#4-amd-rocm-環境設定)
5. [自訂節點安裝（Custom Nodes）](#5-自訂節點安裝custom-nodes)
6. [ComfyUI-Manager 安裝與使用](#6-comfyui-manager-安裝與使用)
7. [模型下載與放置](#7-模型下載與放置)
8. [基本操作流程](#8-基本操作流程)

---

## 1. 安裝方式比較

| 方式 | 適合對象 | 優點 | 缺點 |
|------|----------|------|------|
| **Portable Windows** | 大多數用戶 | 獨立 Python、無需安裝、解壓即用 | 需手動更新 |
| **Desktop App** | 新手用戶 | GUI 安裝器、自動處理依賴 | 功能較少 |
| **Pinokio** | 喜歡瀏覽器管理 | 一键安裝多個 AI 工具 | 依賴 Pinokio 生態 |
| **原始碼安裝** | 進階開發者 | 最新功能、完整控制 | 需手動處理依賴 |

> 💡 **推薦：** 一般用戶使用 **Portable Windows**，AMD GPU 用戶使用 **ROCm 版本**。

---

## 2. Portable Windows 版本安裝

### 步驟 1：下載 Portable 包

從官方 GitHub Releases 頁面下載：
- **網址：** https://github.com/comfyanonymous/ComfyUI/releases
- **檔案：** `ComfyUI_windows_portable_xxxxxx.zip`

### 步驟 2：解壓縮

```bash
# 解壓到任意位置（建議路徑）
D:\ComfyUI_Win_portable_RX9070 TRFv0.5-2\ComfyUI_windows_portable\
```

> ⚠️ **注意：** 路徑中避免使用特殊字元（如 `#`、`$`），否則可能導致 Python 模組載入失敗。

### 步驟 3：啟動 ComfyUI

```bash
# 執行 start.bat
start.bat
```

首次啟動時會自動下載預設模型（約 2-4 GB），請耐心等待。

### 步驟 4：確認啟動成功

瀏覽器開啟：`http://127.0.0.1:8188`

看到 ComfyUI 介面即表示啟動成功！

---

## 3. Desktop App 安裝

### 步驟 1：下載 Desktop App

從官方網站下載：
- **網址：** https://comfy.org/
- **檔案：** `ComfyUI-Setup-x.x.x.exe`

### 步驟 2：執行安裝程式

1. 雙擊 `.exe` 檔案
2. 選擇安裝路徑
3. 等待自動下載與安裝
4. 安裝完成後自動啟動

### 步驟 3：基本設定

- 首次啟動會提示選擇 GPU（NVIDIA / AMD / Apple）
- 根據硬體選擇對應的 PyTorch 版本
- 確認無誤後自動下載模型

---

## 4. AMD ROCm 環境設定

### 支援的 AMD GPU

| 系列 | 型號 | 支援狀態 |
|------|------|----------|
| **RDNA 3** | RX 7900 GRE / 7900 XT / 7900 XTX / 7800 XT / 7700 XT / 7600 | ✅ 完整支援 |
| **RDNA 3.5** | RX 9070 / 9070 XT | ✅ 完整支援 |
| **CDNA 3** | Instinct MI300X | ✅ 完整支援 |

### ROCm 安裝步驟（Windows）

#### 方法 A：WSL + ROCm（官方推薦）

1. **安裝 WSL2：**
   ```powershell
   wsl --install
   ```

2. **安裝 AMD ROCm for WSL：**
   - 下載：https://rocm.amd.com/
   - 按照官方 Quickstart 指示操作

3. **重啟電腦**（註冊 Secure Boot）

4. **在 WSL 中安裝 ComfyUI：**
   ```bash
   cd ~
   git clone https://github.com/comfyanonymous/ComfyUI.git
   cd ComfyUI
   pip install -r requirements.txt
   ```

#### 方法 B：原生 Windows + ROCm（無需 WSL）

1. **安裝 AMD ROCm for Windows：**
   - 下載：https://rocm.amd.com/
   - 按照官方 Quickstart 指示操作

2. **驗證 GPU 偵測：**
   ```bash
   rocm-smi
   ```

3. **安裝 ComfyUI：**
   ```powershell
   git clone https://github.com/comfyanonymous/ComfyUI.git
   cd ComfyUI
   python -m pip install -r requirements.txt
   ```

### ROCm 版本檢查

```bash
# 確認 PyTorch ROCm 版本
python -c "import torch; print(torch.version.rocm)"

# 確認 GPU 偵測
python -c "import torch; print(torch.cuda.is_available())"
```

> ⚠️ **更新提醒：** 執行 `pip install --upgrade -r requirements.txt` 可能覆蓋 ROCm 版 PyTorch，更新後需執行 `Update ROCm.bat` 校正。

---

## 5. 自訂節點安裝（Custom Nodes）

### 安裝前注意事項

- ✅ 只從信賴的作者或社群常用節點安裝
- ✅ 了解節點功能後再安裝，避免未知來源
- ✅ 避免安裝冷門節點（可能缺乏維護）

### 方法 A：使用 ComfyUI-Manager（推薦）

1. **安裝 Manager：**
   ```bash
   cd ComfyUI_windows_portable
   .\python_embeded\python.exe -m pip install -r ComfyUI\manager_requirements.txt
   ```

2. **啟動 ComfyUI**

3. **使用 Manager 介面：**
   - 點擊右側選單的 `Manager` 按鈕
   - 瀏覽可安裝的自訂節點
   - 點擊 `Install` 即可自動安裝

### 方法 B：手動安裝

1. **克隆節點到 `custom_nodes/` 目錄：**
   ```bash
   cd ComfyUI_windows_portable\ComfyUI\custom_nodes
   git clone https://github.com/USER/REPO.git
   ```

2. **安裝節點依賴：**
   ```bash
   .\python_embeded\python.exe -m pip install -r "path/to/node/requirements.txt"
   ```

3. **重啟 ComfyUI**

---

## 6. ComfyUI-Manager 安裝與使用

### 安裝步驟

```bash
# Portable 版本
cd ComfyUI_windows_portable
.\python_embeded\python.exe -m pip install -r ComfyUI\manager_requirements.txt
```

或建立 `install_manager.bat`：
```batch
@echo off
cd /d %~dp0
.\python_embeded\python.exe -m pip install -r ComfyUI\manager_requirements.txt
pause
```

### Manager 功能

| 功能 | 說明 |
|------|------|
| **Install** | 瀏覽並安裝自訂節點 |
| **Update** | 更新所有已安裝的節點 |
| **Disable/Enable** | 啟用或停用特定節點 |
| **Reinstall** | 重新安裝節點 |
| **Settings** | Manager 設定 |

---

## 7. 模型下載與放置

### 模型目錄結構

```
ComfyUI_windows_portable\ComfyUI\
├── models/
│   ├── checkpoints/          # 主要模型（SDXL, SD1.5, FLUX 等）
│   ├── vae/                  # VAE 模型
│   ├── clip/                 # CLIP 編碼器
│   ├── text_encoders/        # 文字編碼器（Qwen, T5 等）
│   ├── unet/                 # UNET 模型
│   ├── loras/                # LoRA 模型
│   ├── controlnet/           # Contr