# AGENTS.md · 项目规则

> 📌 **文档基线**：2026-08-26（首次发布，commit 见本仓库首次提交）
> **更新文档/代码后，请更新此行**（日期 + 新 commit hash），并在 CHANGELOG 追加版本。

只写代码/配置里看不出的信息；核心约束控制在短篇幅内，细节放 DEVELOPMENT.md。

## 技术栈（精确版本）

- ComfyUI **Windows portable · NVIDIA v0.33.1**（自带 `python_embeded` 与 FFmpeg）
- 节点：`numz/ComfyUI-SeedVR2_VideoUpscaler`（版卦 hash `690cc39379c1481159ddd451368dbf2295930fc6`）
- 权重（`ComfyUI/models/SEEDVR2/`）：
  - `seedvr2_ema_3b_fp8_e4m3fn.safetensors`（3.16GB，首选）
  - `ema_vae_fp16.safetensors`（478MB）
- 关键依赖：`diffusers 0.40`、`peft 0.20`、`gguf`、`matplotlib`、`opencv-python`、`omegaconf`、`rotary-embedding-torch`

## 关键坑（4070S 12GB 实测）

- **VAE 不要 offload 到 CPU**：会慢到 ~8 分钟/batch。VAE 用 `cuda:0`，DiT 用 CPU offload。
- **大 batch 反而慢 / 爆显存**：用 `batch_size=9`（或 5）。长视频别设成与帧数相等的那种"优化提示"。
- **`resolution` 填整数**，别填字符串 `"fixed"`。
- **需代理时**：本机 `git`/`curl` 不走系统代理（`127.0.0.1:7890`），要显式 `-x`；克隆节点用 zip 而非 git 协议。
- **多段视频连续修**：Loader 节点 `cache_model` 开 `true`，否则每段都重新加载、每次都像第一次一样慢。

## 约定

- 文档用中文；工作流 JSON 命名带规格后缀（`_1080p` / `_res1024_batch9`）。
- 提示词类效果（Wan / LTX / MiniMax H3）属其他项目，此仓库不含推断/生成工作流，只做 SeedVR2 修复。
- 部署目录默认 `C:\seedvr2-deploy\ComfyUI_windows_portable\`；仓库里保留工作流与说明，不放二进制权重。

## 常用命令

- 启动：`run_nvidia_gpu.bat`
- 节点装依赖：`...\python_embeded\python.exe -m pip install -r ComfyUI-SeedVR2_VideoUpscaler\requirements.txt`
- 重载工作流时用文件**拖进画布**，避免浏览器缓存旧版（Workflow 菜单可能拉旧 JSON）。

## 详细规则（按需 @引用）

- @DEVELOPMENT.md（架构 + 问题记录一坑一篇）