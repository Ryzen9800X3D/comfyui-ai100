---
id: ai.procedure.comfyui-ai100-deployment
type: procedure
domain:
  - ai
  - infrastructure
version: 1
status: active
confidence: 0.95
tags:
  - comfyui
  - deployment
  - amd-rocm
  - r9700
  - minimax-h3
  - qwen-image-2.1
  - custom-nodes
  - ai-server-100
sources: []
relations:
  - ai.procedure.comfyui-installation-guide
  - ai.troubleshooting.comfyui-minimax-h3-troubleshooting
  - infrastructure.procedure.amd-rocm-llama-cpp-sop
created: 2026-10-14
updated: 2026-10-14
last_verified: 2026-10-14
title: ComfyUI AI-100 完整部署流程（v0.40.0 + MiniMax H3 導演台 + Qwen Image 2.1 T8 加速）
---

# ComfyUI AI-100 完整部署流程

> 本頁記錄 **AI Server 100（192.168.10.100）** 上 ComfyUI 的完整部署與更新流程，以 2026-10-14 實測為準。
> 涵蓋：基礎安裝/升級 → v0.40.0、MiniMax H3 導演台（無限制版）、Qwen Image 2.1 T8 加速（PE 無限制）、全部模型清單、驗證與坑。
> 一般安裝教學見 [[comfyui-installation-guide]]；H3 疑難排解見 [[comfyui-minimax-h3-troubleshooting]]。

## 0. 環境總覽（2026-10-14 實測）

| 項目 | 值 |
|------|-----|
| Host | `ryzen@192.168.10.100`（ssh 密碼見 master 帳密表） |
| GPU | **AMD Radeon AI PRO R9700**（Navi 48，lspci 確認） |
| 路徑 | `~/comfy/ComfyUI`（**系統 python3，無 venv**） |
| 版本 | v0.40.0（commit `3541246a`，tag `v0.40.0`） |
| PyTorch | `2.14.1+rocm7.2` |
| Port | `8188`（**必須 `--listen 0.0.0.0`**，127.0.0.1 會 LAN 連不到） |
| 磁碟 | `/dev/sda2` 491G（2026-10-14 growpart + resize2fs 線上擴充 398.9G→497G） |
| llama.cpp | `~/llama.cpp/build/bin/llama-server`（PE 節點依賴，ROCm build） |

---

## 1. 基礎安裝 / 升級

```bash
# 首次安裝
git clone https://github.com/comfyanonymous/ComfyUI.git ~/comfy/ComfyUI
cd ~/comfy/ComfyUI
python3 -m pip install -r requirements.txt

# 升級到特定 tag（本次：v0.37.0 → v0.40.0）
git fetch --tags
git checkout v0.40.0
python3 -m pip install -r requirements.txt
# 注意：pip 可能覆蓋 ROCm 版 torch，升級後檢查
python3 -c "import torch; print(torch.__version__)"   # 期望 2.1x.x+rocm7.x
```

升級後所有 custom_nodes 要 `git pull`（或用 Manager 更新），再重啟。

## 2. 啟動命令（定稿）

```bash
cd ~/comfy/ComfyUI
setsid nohup env QWEN_PE_LLAMA_SERVER=/home/ryzen/llama.cpp/build/bin/llama-server \
  python3 main.py --listen 0.0.0.0 --port 8188 --preview-method auto \
  > /tmp/comfyui.log 2>&1 < /dev/null &
```

- `QWEN_PE_LLAMA_SERVER` — Qwen PE 節點（Prompt-Rewrite-T8）需要 llama-server 路徑跑 gguf PE 模型
- `--listen 0.0.0.0` — 否則 LAN 其他機器連不到（曾踩坑）
- 重啟：`fuser -k 8188/tcp` 再執行上面命令（**不要用 pkill**，會連 shell 的 python 一起殺）

## 3. 自訂節點（custom_nodes/，2026-10-14 已裝）

| 節點 | Repo | 用途 |
|------|------|------|
| `ComfyUI_MiniMaxH3_Director` | — | MiniMax H3 導演台（9 工作流母節點）@ `4c6245e` |
| `Comfyui-Qwen-Image-Prompt-Rewrite-T8` | `T8mars/Comfyui-Qwen-Image-Prompt-Rewrite-T8` | Qwen PE 無限制改寫（`QwenPERewriteT8` 節點） |
| `Comfyui-Qwen-Image-2.1-BlockCache-T8` | `T8mars/Comfyui-Qwen-Image-2.1-BlockCache-T8` | Qwen Image 2.1 T8 加速核心（**原 repo 名 `Comfyui-Qwen-Image-2` 已改名，舊名 clone 會 404**） |
| `ComfyUI-Easy-Use` | — | 輔助節點 |
| `ComfyUI_Comfyroll_CustomNodes` | — | 輔助節點 |

PE 節點 pip 依賴（2026-10-14 補裝）：

```bash
python3 -m pip install --user --break-system-packages clip_interrogator lark onnxruntime peft
```

## 4. MiniMax H3 導演台配置（無限制版）

### 模型配置（定稿）

| 角色 | 檔案（`models/` 下） | 大小 | 來源 |
|------|---------------------|------|------|
| UNET | `diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 20.97 GB | Comfy-Org/MiniMax-H3 |
| UNET | `diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors` | 20.97 GB | Comfy-Org/MiniMax-H3 |
| UNET 二采加速 | `diffusion_models/minimax_h3_ref2va_pruned_w4a8_mixed.safetensors` | 11.77 GB | Kijai/MiniMax-H3-experimental |
| **CLIP（無限制）** | `text_encoders/qwen3vl_32b_heretic_minimax_h3_nvfp4.safetensors` | 15.68 GB | Momoking/Qwen3-VL-32B-Heretic-MiniMax-H3-NVFP4 |
| VAE | `vae/minimax_h3_video_vae_fp16.safetensors` | 5.21 GB | Comfy-Org/MiniMax-H3 |
| VAE | `vae/minimax_h3_audio_vae_fp32.safetensors` | 0.61 GB | Comfy-Org/MiniMax-H3 |

**關鍵決策**：「無限制」的問題出在 **CLIP 文字編碼器**（不是 UNET）。官方版 `qwen3vl_32b_minimax_h3_nvfp4_awq`（censored）已刪除，改用 Momoking 的 Heretic（abliterated）版。

**使用**：CLIPLoader 選 `qwen3vl_32b_heretic_minimax_h3_nvfp4`，type 維持 `minimax`。

### 9 個導演台工作流（`user/default/workflows/`）

`minimax_h3_director_t2v` / `fl2v` / `r2v` / `v2v` / `rv2v` / `external_groups_i2v` / `external_groups_r2v` / `加速版` / `二采_加速`（二采_加速 用 w4a8_mixed UNET）。

## 5. Qwen Image 2.1 T8 加速配置（PE 無限制）

來源包：`t8star/Qwen-Image-2.1-Turbo-Hifi-Comfy`（最強 T8 版加速 v2）+ `t8star/qwen-image-2.1-comfy`（PE gguf）。

### 模型

| 角色 | 檔案 | 大小 | 來源 |
|------|------|------|------|
| 主模型 UNET | `diffusion_models/qwen_image_2.1_int8_convrot.safetensors` | 7.26 GB | Comfy-Org/Qwen-Image-2.1 |
| **T8 加速 LoRA** | `loras/qwen_image_2.1_turbo_lora_hifi_T8_v001.safetensors` | 19.56 GB | t8star |
| CLIP | `text_encoders/qwen3vl_8b_int8_convrot.safetensors` | 9.35 GB | Comfy-Org/Qwen-Image-2.1 |
| PE t2i（官方 int8） | `text_encoders/qwen3.5_9b_qwen_image_2.1_pe_t2i.int8_convrot.safetensors` | 9.47 GB | Comfy-Org/Qwen-Image-2.1 |
| VAE | `vae/qwen_image_2.1_vae_bf16.safetensors` | 0.68 GB | Comfy-Org/Qwen-Image-2.1 |
| **PE 無限制 gguf** | `custom_nodes/Comfyui-Qwen-Image-Prompt-Rewrite-T8/models/llm/qwenimage-pe/pe_t2i_heretic-Q4_K_M.gguf` | 5.89 GB | t8star |
| **PE 無限制 gguf** | `.../pe_i2i_heretic-Q4_K_M.gguf` | 5.89 GB | t8star |
| PE 視覺組件 | `.../pe_i2i_heretic.mmproj-bf16.gguf` | 0.92 GB | t8star |

### 工作流（`user/default/workflows/`）

- `（最強T8版加速）Qwen+Image+2.1+PE(Un)+文生圖v2.json`
- `（最強T8版加速）Qwen+Image+2.1+PE（Un）+圖像編輯v2.json`
- 另有官方 `Qwen_Image_2.1_T2I_official.workflow` / `Qwen_Image_2.1_Edit_official.workflow`

### 使用參數

LoRA 強度 1、Euler、CFG 1、8 步；sigmas 已內建在工作流（ManualSigmas 節點）**不要動**；PE 節點自動改寫 prompt（Un = 無限制 Heretic 版）。

⚠️ 工作流 CLIPLoader 預設值寫的是 `qwen3vl_8b_fp8_scaled`（HF 上沒這檔）→ 打開後手動改選 `qwen3vl_8b_int8_convrot`。

## 6. 完整模型清單（2026-10-14 實測）

| 目錄 | 檔案 | GB |
|------|------|-----|
| checkpoints | v1-5-pruned-emaonly.safetensors | 4.27 |
| diffusion_models | minimax_h3_fl2va_pruned_int8_convrot.safetensors | 20.97 |
| diffusion_models | minimax_h3_ref2va_pruned_int8_convrot.safetensors | 20.97 |
| diffusion_models | minimax_h3_ref2va_pruned_w4a8_mixed.safetensors | 11.77 |
| diffusion_models | qwen_image_2.1_int8_convrot.safetensors | 7.26 |
| loras | qwen_image_2.1_turbo_lora_hifi_T8_v001.safetensors | 19.56 |
| text_encoders | qwen3.5_9b_qwen_image_2.1_pe_t2i.int8_convrot.safetensors | 9.47 |
| text_encoders | qwen3vl_32b_heretic_minimax_h3_nvfp4.safetensors | 15.68 |
| text_encoders | qwen3vl_8b_int8_convrot.safetensors | 9.35 |
| vae | minimax_h3_audio_vae_fp32.safetensors | 0.61 |
| vae | minimax_h3_video_vae_fp16.safetensors | 5.21 |
| vae | qwen_image_2.1_vae_bf16.safetensors | 0.68 |

## 7. 驗證步驟（重啟後必做）

```bash
# 1. 模型選單（CLIPLoader / UNETLoader / LoraLoader 是否出現新模型）
curl -s http://localhost:8188/object_info/CLIPLoader   | python3 -m json.tool | grep -E "heretic|qwen3vl"
curl -s http://localhost:8188/object_info/UNETLoader   | python3 -m json.tool | grep -E "minimax|w4a8"
curl -s http://localhost:8188/object_info/LoraLoader   | python3 -m json.tool | grep -iE "t8|turbo"

# 2. PE 節點是否載入（QwenPERewriteT8 節點 + gguf 偵測）
grep -E "QwenPERewriteT8|QwenImage21|missing" /tmp/comfyui.log

# 3. 端口
ss -ltnp | grep 8188
```

注意：`object_info` 的 `inputs` 欄位直接回傳 list（非 dict），解析時要 `for` 迴圈，不要 `.get("required")`。

## 8. 坑與對策（2026-10-14 部署實錄）

| 坑 | 對策 |
|----|------|
| `--listen 127.0.0.1`（預設）→ LAN 連不到 | 一律 `--listen 0.0.0.0` |
| `pkill -f main.py` 連 shell 的 python 一起殺 | `fuser -k 8188/tcp` |
| H3 無限制版：問題在 CLIP 不在 UNET | 換 Momoking Heretic CLIP；官方 censored awq 版直接刪 |
| `T8mars/Comfyui-Qwen-Image-2` clone 404 | repo 已改名 → `Comfyui-Qwen-Image-2.1-BlockCache-T8`（GitHub API `users/T8mars/repos` 可查現行名） |
| Kijai HF API 回空 | 加 `?recursive=true` |
| 磁碟 500G 認到但分區 392G | `growpart /dev/sda 2` + `resize2fs /dev/sda2`（ext4 線上擴充，不用重啟） |
| T8 工作流 CLIPLoader 預設 `qwen3vl_8b_fp8_scaled` 不存在 | 手動改選 `qwen3vl_8b_int8_convrot` |
| PE 節點缺 pip 套件（clip_interrogator/lark/onnxruntime/peft） | `pip install --user --break-system-packages`；onnxruntime/clip_interrogator 很大，背景安裝耐心 poll |
| 下載驗證 | curl `-fL --retry 5 --retry-delay 3 -C -` nohup 背景；完成後 **位元組數 vs HF API `size` 比對** |

## 9. 下載記錄（2026-10-14，全部位元組驗證 ✅）

| 檔案 | 位元組數 | 速度/時間 |
|------|---------|-----------|
| minimax_h3_ref2va_pruned_w4a8_mixed.safetensors | 11,770,657,048 | ~104 MB/s, ~1:47 |
| qwen_image_2.1_turbo_lora_hifi_T8_v001.safetensors | 19,558,578,400 | 背景 nohup |
| qwen3vl_8b_int8_convrot.safetensors | 9,350,798,360 | — |
| pe_t2i / pe_i2i heretic gguf + mmproj | 5.89G + 5.89G + 0.92G | — |

## 🔗 關聯性索引

- **相關實體**: [[Dify]]（同 101/100 叢生態系）、[[GitHub-PAT-Ryzen9800X3D]]
- **相關概念**: [[comfyui-minimax-h3-troubleshooting]]（H3 排解）、[[amd-rocm-llama-cpp-sop]]（llama.cpp ROCm build，PE 節點依賴）
- **原始文件**: [[comfyui-installation-guide]]（一般安裝教學）、https://huggingface.co/t8star/Qwen-Image-2.1-Turbo-Hifi-Comfy、https://huggingface.co/Momoking/Qwen3-VL-32B-Heretic-MiniMax-H3-NVFP4
- **綜合分析**: —
