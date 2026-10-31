# colab-talking-avatar

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/dawang-sakura/colab-talking-avatar/blob/main/talking_avatar.ipynb)

一张角色图 + 一段文稿 → 会说话的主播视频。全程在 Google Colab（L4 GPU）上运行，你的电脑不用安装任何东西。

> **第一次用 Colab、不会写程序？请直接看 → [新手操作手册](docs/新手操作手册.md)**（一步一个动作，照着点就行）

## 快速开始（3 步）

1. 点上方 **Open in Colab** 徽章，在 Colab 菜单「代码执行程序 → 更改运行时类型」（英文界面：Runtime → Change runtime type）选 **L4 GPU**。
2. 「代码执行程序 → 全部运行」（英文界面：Runtime → Run all），弹出「此笔记本不是由 Google 创作」时点「仍然运行」。不改任何设置会用示例图和示例文稿。
3. 等浏览器自动下载 `talking_avatar.mp4`，然后「代码执行程序 → 断开连接并删除运行时」（英文界面：Runtime → Disconnect and delete runtime）。

## 流程

```
① 设置（文稿、音色描述、角色图）
        │
        ▼
③ 配音：VoxCPM2
   音色设计（用文字描述造一把声音，只做一次，存成参考音）
   → 逐句可控克隆（每句 ≤80 字，超过自动切开）
   → 句间插入停顿，接成一条音轨 voice.wav
   → 删除模型、释放显存
        │
        ▼
④ 对嘴：ComfyUI v0.43.0 + InfiniteTalk（Wan2.1 I2V 14B 480P）
   角色图裁成 480×832 → 每段 81 帧、重叠 9 帧往后延伸 → 长度跟配音走
        │
        ▼
⑤ 合成：画面 + 原始配音（AAC）→ /content/output/talking_avatar.mp4 → 自动下载
```

（② 是环境检查：显示 GPU 型号、取得示例文件、确认输入文件存在。）

## 设置说明

只需要改 notebook 的 **① 设置** 格：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SCRIPT` | 3 句示例文稿 | 一行一句；每句最多 80 字，超过会在标点处自动切开（VoxCPM2 单次太长容易啸叫或音色漂移）。句首可以加括号写语气，例如 `(轻松地)大家好` |
| `VOICE_DESC` | 年轻女性、清亮自然… | 音色设计用的文字描述（性别、年龄、音色、语速、语气） |
| `PAUSE_SEC` | `0.4` | 句与句之间的停顿秒数 |
| `SEED` | `42` | 随机种子。音色设计每次结果不同，对声音不满意就换个数字重跑 |
| `IMAGE_PATH` | `examples/host.png` | 角色图。换成自己的图：上传到 Colab 左侧「文件」面板（会放在 `/content/`），改成 `/content/你的图.png` |
| `PROMPT` | 通用英文描述 | 对嘴画面的英文描述，可补充角色外观（发型、服装），减少配件消失 |
| `LOW_VRAM` | `False` | 对嘴阶段爆显存时改成 `True`（ComfyUI `--novram`，更慢） |
| `ENABLE_VOICE_CLONE` 等 | 关闭 | 见下方「声音克隆」 |

角色图建议：竖图、正面、闭嘴、脸够大、手不入镜。图会先裁成 480:832 比例（保留上方、裁掉下方），再缩成 480×832。

示例文稿也放在 [`examples/script.txt`](examples/script.txt)。

## 声音克隆（默认关闭）

> **只能用你本人、或已取得当事人同意的录音。**

1. 把参考录音上传到 Colab 左侧「文件」面板（建议 5～15 秒、单人说话、没有背景音乐）。
2. 在 ① 设置格里：
   - `ENABLE_VOICE_CLONE = True`
   - `REF_WAV = "/content/你的录音.wav"`
   - `REF_TEXT`：
     - **留空** → 可控克隆：声音接近参考录音，句首括号的语气可以用。
     - **填参考录音的逐字稿**（必须一字不差）→ 极致克隆：最像本人，但**不能**加语气括号（VoxCPM 官方不允许语气指令和 `prompt_text` 一起用），文稿里有语气括号会直接报错。
   - `I_HAVE_CONSENT = True`：表示你确认录音是你本人、或已取得当事人同意。**没改成 True 就开启克隆，notebook 会直接报错停止。**
3. 开启克隆后 `VOICE_DESC` 不再使用。

## 使用规范

- 只使用 **你本人**、**已取得当事人同意者**、或 **AI 生成的虚构角色** 的脸和声音。
- **禁止冒充他人**，禁止用来误导、诈骗、诽谤，或制作未经同意的他人影像与声音。
- Google Colab 的使用限制明文列出禁止 *creating deepfakes*，见 [Colab FAQ](https://research.google.com/colaboratory/faq.html)。违反可能导致账号受限。
- 发布成品时，建议标明「AI 生成」。

## 耗时与费用

以下是**作者实测**（L4，单次运行），会随配音长度变化，仅供参考：

| 项目 | 实测 |
|---|---|
| L4 计费 | 约 **1.48 计算单元／小时** |
| 开机、安装 VoxCPM 并载入 VoxCPM2 | 约 3～4 分钟 |
| VoxCPM2 配音 | 生成耗时 ≈ 音频长度（例：10 秒配音约 10 秒） |
| 安装 ComfyUI＋下载约 30GB 模型＋启动 | 约 2.5 分钟（一次实测：28＋92＋27 秒） |
| 对嘴（21.5 秒成品，8 段） | 约 **36 分钟**，约 **0.9 计算单元** |

对嘴耗时大致跟段数成正比：第 1 段 81 帧（约 3.24 秒），之后每段多 72 帧（约 2.88 秒）。作者实测过的最长成品是 21.5 秒，更长的文稿耗时和系统内存是否足够未验证。

**跑完一定要「断开连接并删除运行时」，闲置也会扣计算单元。**

## 常见问题

**选了 L4，结果是 T4？**
浏览器同时登录多个 Google 账号时，Colab 默认用**第一个**账号；那个账号计算单元为 0 时会被降成 T4。到 <https://colab.research.google.com/signup> 确认当前账号的余额。T4（15GB 显存）未测试，很可能爆显存或非常慢。

**报错 `OutOfMemory` / `OOM`（爆显存）？**
「代码执行程序 → 重启会话」→ ① 设置格改 `LOW_VRAM = True` → 全部运行。已装的 ComfyUI 和已下载的模型会跳过，配音会重新生成。

**报错「ComfyUI 进程结束了」？**
多半是系统内存或显存不够，文稿越长越容易遇到。先缩短文稿，或开 `LOW_VRAM = True`；详细原因看左侧文件面板的 `/content/comfyui.log`。

**声音不满意？**
换 `SEED` 或改 `VOICE_DESC` 重跑。音色设计本来就每次不同。

**用完了怎么关？**
「代码执行程序 → 断开连接并删除运行时」。第一次点可能先弹出警告，确认后要再点一次删除。

## 第三方模型与授权

本仓库的代码以 MIT 发布；下列组件**不包含在本仓库里**，是在 Colab 运行时从原始来源下载的，各自适用其授权。以下为作者 2026-10-31 查证的结果，使用前请以各来源页面为准。

| 组件 | 用途 | 运行时下载来源（commit 已固定） | 授权 | 查证出处 |
|---|---|---|---|---|
| VoxCPM（代码） | 配音程序 | GitHub `OpenBMB/VoxCPM` @ `cce58a5b` | Apache-2.0 | <https://github.com/OpenBMB/VoxCPM/blob/main/LICENSE> |
| VoxCPM2（权重） | 配音模型 | HF `openbmb/VoxCPM2` | Apache-2.0 | <https://huggingface.co/openbmb/VoxCPM2> |
| ComfyUI v0.43.0 | 对嘴的运行框架 | GitHub `comfyanonymous/ComfyUI`（现为 `Comfy-Org/ComfyUI`） | GPL-3.0 | <https://github.com/Comfy-Org/ComfyUI/blob/master/LICENSE> |
| Wan2.1 I2V 14B 480P fp8 | 视频底模 | HF `Kijai/WanVideo_comfy_fp8_scaled` | 托管仓库：Apache-2.0；上游 Wan2.1：Apache-2.0 | <https://huggingface.co/Kijai/WanVideo_comfy_fp8_scaled>、<https://huggingface.co/Wan-AI/Wan2.1-I2V-14B-480P> |
| umt5-xxl fp8 | 文本编码器 | HF `Comfy-Org/Wan-2.1` | 托管仓库：Apache-2.0；上游 umt5-xxl：Apache-2.0 | <https://huggingface.co/Comfy-Org/Wan-2.1>、<https://huggingface.co/google/umt5-xxl> |
| InfiniteTalk 单人 fp16 | 对嘴模型 | HF `Comfy-Org/Wan-2.1` | 托管仓库：Apache-2.0；上游 InfiniteTalk：Apache-2.0 | <https://huggingface.co/Comfy-Org/Wan-2.1>、<https://huggingface.co/MeiGen-AI/InfiniteTalk>、<https://github.com/MeiGen-AI/InfiniteTalk/blob/main/LICENSE.txt> |
| chinese-wav2vec2-base fp16 | 音频特征 | HF `Kijai/wav2vec2_safetensors` | 托管仓库：MIT；上游：MIT | <https://huggingface.co/Kijai/wav2vec2_safetensors>、<https://huggingface.co/TencentGameMate/chinese-wav2vec2-base> |
| Wan2.1 VAE bf16 | 视频编解码 | HF `Kijai/WanVideo_comfy` | 托管仓库：**未确认**（模型卡未标授权）；上游 Wan2.1：Apache-2.0 | <https://huggingface.co/Kijai/WanVideo_comfy>、<https://github.com/Wan-Video/Wan2.1/blob/main/LICENSE.txt> |
| lightx2v I2V 480P 步数/CFG 蒸馏 LoRA rank64 | 加速（6 步） | HF `Kijai/WanVideo_comfy` | 托管仓库：**未确认**（模型卡未标授权）；上游 lightx2v 模型：Apache-2.0（按文件名对应，未逐字节核对） | <https://huggingface.co/Kijai/WanVideo_comfy>、<https://huggingface.co/lightx2v/Wan2.1-I2V-14B-480P-StepDistill-CfgDistill-Lightx2v>、<https://github.com/ModelTC/LightX2V/blob/main/LICENSE> |

示例图 `examples/host.png` **由 AI 生成，为虚构人物，非真实人物**（说明见 [`examples/README.md`](examples/README.md)）。

## 致谢

- [OpenBMB / VoxCPM](https://github.com/OpenBMB/VoxCPM)
- [MeiGen-AI / InfiniteTalk](https://github.com/MeiGen-AI/InfiniteTalk)
- [Wan-Video / Wan2.1](https://github.com/Wan-Video/Wan2.1)
- [ComfyUI](https://github.com/Comfy-Org/ComfyUI) 及其官方 InfiniteTalk 工作流模板
- [Kijai](https://huggingface.co/Kijai)（模型格式转换与托管）、[Comfy-Org](https://huggingface.co/Comfy-Org)
- [LightX2V](https://github.com/ModelTC/LightX2V)
- [TencentGameMate / chinese-wav2vec2-base](https://huggingface.co/TencentGameMate/chinese-wav2vec2-base)

## 授权

[MIT](LICENSE) © 2026 dawang-sakura
