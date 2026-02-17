# 原神语音（Genshin6.2_CN.7z）批处理：text + vq0206(`<audio_X>`) 成对数据

这份 README 记录了把 `Genshin6.2_CN.7z` 里的 `.lab/.wav` 批量整理成 **训练用 text + vq0206 token_str** 的流程，以及过程中踩过的坑和能显著提速的优化点（macOS / Apple Silicon / MPS 场景）。

> 目标输出：每条样本一行 JSONL：`{"id","text","tokens","rel_wav","rel_lab","sr"}`  
> 其中 `tokens` 是形如 `<audio_123><audio_456>...` 的 **vq0206 token_str**（用于后续模型训练）。

---

## 1. 输入数据结构（经验）

- `Genshin6.2_CN.7z` 是 **solid 7z**（`Solid = +`）
  - 经验教训：**不要逐文件抽取**（`7z e archive file`），solid 的随机访问很慢；应当 **一次性解压** 再批处理。
- 常见目录结构类似：`中文 - Chinese/...`
- 语音标注与音频通过同名 stem 配对：
  - `xxx.lab`：文本（一般 UTF-8；少量可能有 BOM 或 GB18030）
  - `xxx.wav`：音频（常见 48kHz；单/多声道都有）

---

## 2. 输出格式（训练对）

脚本输出 JSONL（可 `.gz`），每行一个样本，例如：

```json
{"id":"中文 - Chinese/xxx/vo_...","text":"……","tokens":"<audio_958><audio_958>...","rel_wav":"中文 - Chinese/xxx/vo_....wav","rel_lab":"中文 - Chinese/xxx/vo_....lab","sr":48000}
```

关键点：
- **tokens 使用“未 shift”的 `<audio_X>`**（训练用）
  - 经验教训：`wav2token()` 返回的 `speech_tokens` 里有 `+65536` 的 shift（推理内部用），**不要拿它直接拼 `<audio_>`**。
  - 正确做法：拿 `vq02_ori` 与 `vq06_ori`，用 `merge_vq0206_to_token_str(vq02_ori, vq06_ori)` 生成 `<audio_X>` 串。

---

## 3. 一次性解压（强烈推荐）

### 3.1 为什么必须一次性解压

因为 solid 7z 对随机抽取不友好：即使只解一个文件，也可能需要解压大量前置块，吞吐会非常差。

### 3.2 只解压 `.wav/.lab`（更省空间）

脚本内部解压默认只提取 `*.wav` 和 `*.lab`，并使用多线程：

- `7z x -mmt=on ... -ir!*.wav -ir!*.lab`

经验教训：
- `7z` 的 include 过滤在这个场景下要用 **`-ir!`**（递归匹配），而不是 `-i!`  
  之前用错会导致 **0 files**。

---

## 4. Spotlight / mdworker 影响（macOS 必看）

现象：
- 解压/生成海量文件时出现大量 `mdworker* / mdworker_shared`，占 CPU + 磁盘 I/O，解压明显变慢。

处理建议（无需 sudo）：
- 在解压目录创建：`.metadata_never_index`
  - 已在脚本的解压步骤里自动 `touch <extract_dir>/.metadata_never_index`
  - 也可以在系统设置里把目录加入 Spotlight Privacy

不建议：
- 直接杀 `mdworker`（会自动重启）
- `mdutil -i off` 需要 root，且影响面更大

---

## 5. vq0206 Tokenizer 性能结论（核心经验）

### 5.1 vq02（FunASR encoder）是 CPU 主要瓶颈

- 在这台机器上（Apple Silicon + MPS 可用）：
  - **vq02 上 MPS 可以带来数量级提升**（实测同一条音频可 ~20x，且输出一致）
- 经验教训：别执着于 vq06 上 GPU/加速，vq06 本身不慢，vq02 才是大头。

### 5.2 vq06（ONNX）走 CPU 更稳

- 经验教训：ONNX GPU provider 在这里不一定更快，且容易引入额外复杂性。
- 当前实现：`onnxruntime` 默认只启用 `CPUExecutionProvider`，并将 `intra_op_num_threads=1`
  - 目的：避免 ONNX 在每个进程里开多线程导致 **过度抢占 CPU**（我们用进程级并行来吃满 CPU）。

### 5.3 并行重叠：vq06 与 vq02 并发

脚本默认对每条样本做一件简单但有效的事：
- 同一条音频上：`vq06` 用 1 个线程池 worker（CPU）并发执行，同时主线程跑 `vq02`（MPS/CPU）
- 经验教训：这能把 CPU “空洞时间”填起来，整体吞吐更好。

### 5.4 音频读取：`soundfile` >>> `torchaudio.load`

经验教训：
- 批处理里 `torchaudio.load` 相比 `soundfile.read` 明显慢（实测可达几十倍差距）
- 因此脚本使用 `soundfile` 读 wav，并手动转单声道

### 5.5 `torch.conv1d` / MPS

结论（简测）：
- `torch.nn.functional.conv1d` 在 MPS 上可加速（小 benchmark 显示 MPS 快于 CPU）
- 但对整体吞吐的影响主要还是取决于 **vq02 是否上 MPS**，conv1d 只是其中一个算子层面的体现。

---

## 6. 批处理脚本：prepare_genshin_pairs.py

文件：`try_genshin/prepare_genshin_pairs.py`

功能：
- （可选）解压 `.7z`
- 遍历 `.lab/.wav` 配对（同名 stem）
- 读取文本（UTF-8/UTF-8-SIG/GB18030 自动兜底）
- 读取 wav → 转单声道 → 重采样到 16k →（可选）trim silence
- 产出 `text + tokens` JSONL（支持 `.gz`）
- 使用 `tqdm` 显示整体进度

常用参数：
- `--no-extract`：跳过解压（对已解压目录做处理）
- `--extract-dir`：已解压根目录
- `--out xxx.jsonl.gz`：输出（建议 gzip）
- `--exclude-placeholder`：过滤 `带变量语音 - Placeholder`（建议开启）
- `--funasr-device auto|cpu|mps`：vq02 设备（建议 `mps` 或 `auto`）
- `--torch-threads N`：每进程的 torch 线程数（避免 oversubscribe）
- `--trim`：开启 librosa trim（更慢；默认关闭）
- `--max-duration`：跳过超长音频（可选）
- `--shard-count/--shard-index`：分片处理（配合 Makefile 并行）
- `--no-done`：不写/不读 `<out>.done`（给 Makefile 用 `.ok` 哨兵）
- `--resume`：从已有 `--out` 继续（会扫描 `--out` 已写入的 `id` 去重）

---

## 7. 并行跑满 CPU/MPS：Makefile（推荐用法）

文件：`try_genshin/Makefile`

设计目标：
- 用 `make -j` 表达并行度（你控制并发）
- 每个 shard 单独输出一个 `shard-*.jsonl.gz`
- shard 完成后写一个 `.ok` 哨兵（防止半成品文件被当成完成）
- 最后 `cat` 合并为一个总 `.gz`

### 7.1 一条命令跑起来

在 repo 根目录：

```bash
make -C try_genshin -j6 merge PROCS=6
```

说明：
- `PROCS=6` 表示切成 6 个 shard，并行跑 6 个 Python 进程
- `-j6` 控制 Make 并发（通常与 `PROCS` 一致）
- Makefile 会根据 `hw.ncpu` 自动算 `TORCH_THREADS`，让 **总 CPU 线程数 ≈ 12**（本机）

### 7.2 GPU/MPS 不够满怎么办

经验教训：
- MPS 利用率不满通常是因为并发不足；但并发太高会导致 **MPS 显存/内存压力**（每个进程会各自加载 FunASR 模型到 MPS）。

可尝试（从小到大）：

```bash
make -C try_genshin -j8  merge PROCS=8
make -C try_genshin -j12 merge PROCS=12 TORCH_THREADS=1
```

建议：
- 先从 `PROCS=6~8` 试起，观察 MPS 内存与吞吐再调

### 7.3 断点续跑

Makefile 默认 `RESUME=1`：
- 未完成的 shard 没有 `.ok`，下次 `make -j ... shards` 会重新跑该 shard
- `RESUME=1` 会让脚本带 `--resume`：从已有 shard 输出中去重继续写

如果想强制重建：

```bash
make -C try_genshin clean
make -C try_genshin -j6 merge PROCS=6 RESUME=0
```

### 7.4 进度查看

每个 shard 内部有 `tqdm`（整体进度 / ETA / rate）。

查看 shard 完成度：

```bash
make -C try_genshin progress PROCS=6
```

---

## 8. 常见坑 & 排查

### 8.1 `ModuleNotFoundError: tokenizer`

原因：
- `try_genshin` 在本环境是一个 **symlink**，从该目录直接运行脚本可能找不到 repo root。

解决：
- 脚本已做自动 root 探测；仍失败时手动指定：

```bash
export STEP_AUDIO_EDITX_ROOT=/Users/zsc/Downloads/Step-Audio-EditX
```

### 8.2 zsh 下 `7z ... -i!*.lab` 报 `no matches found`

原因：
- zsh 会把 `*` 当 glob 展开

解决：
- 手动命令行时记得加引号：`' -ir!*.lab '`
- 脚本内部用 `subprocess.run(list)` 不受此影响

### 8.3 解压后找不到配对（pairs=0）

排查：
- 看解压目录是否真的有 `*.lab/*.wav`：

```bash
find extracted/Genshin6.2_CN -name '*.lab' | wc -l
find extracted/Genshin6.2_CN -name '*.wav' | wc -l
```

经验教训：
- 7z include 过滤用错（`-i!` vs `-ir!`）会导致压根没解出来

### 8.4 输出 `.gz` 合并问题

Makefile 用 `cat shard-*.jsonl.gz > final.jsonl.gz` 会生成 **multi-member gzip**：
- 多数工具（`gzip -dc`、Python `gzip.open`）都能正常读取
- 如果你的下游工具不支持 multi-member，再考虑解压重打包

---

## 9. 推荐的运行组合（本机 12C + MPS）

保守稳妥（通常足够快）：

```bash
make -C try_genshin -j6 merge PROCS=6
```

想榨 MPS（更激进）：

```bash
make -C try_genshin -j12 merge PROCS=12 TORCH_THREADS=1
```

---

## 10. 备注

- 请自行确认语音数据的使用权限与合规性（训练/分发/商用可能受限制）。
- 若需要进一步提速：优先考虑 **提高 MPS 并行度（PROCS）** 与 **避免 CPU 过度线程化（TORCH_THREADS）**，而不是让单个进程开很多线程。

