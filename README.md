# colab-talking-avatar

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/dawang-sakura/colab-talking-avatar/blob/main/talking_avatar.ipynb)

一张角色图 + 一段文稿 → 会说话的主播视频。全程在 Google Colab（L4 GPU）上运行，你的电脑不用安装任何东西。

> **第一次用 Colab、不会写程序？请直接看 → [新手操作手册](docs/新手操作手册.md)**（一步一个动作，照着点就行）

## 快速开始（3 步）

1. 点上方 **Open in Colab** 徽章，在 Colab 菜单「代码执行程序 → 更改运行时类型」（英文界面：Runtime → Change runtime type）选 **L4 GPU**。
2. 「代码执行程序 → 全部运行」（英文界面：Runtime → Run all），弹出「此笔记本不是由 Google 创作」时点「仍然运行」。不改任何设置会用示例图和示例文稿。
3. 等浏览器自动下载 `talking_avatar_outputs.zip`（一次流程只自动下载这一个文件），解压后是 `talking_avatar.mp4`（默认带字幕）和字幕文件 `talking_avatar.srt`、不带字幕的 `talking_avatar_nosub.mp4`（开了去背时还有透明背景的 `talking_avatar_alpha.webm`，选「绿幕」时还有绿幕图 `input_greenscreen.png`；字幕和去背都关时直接下载 `talking_avatar.mp4`），然后「代码执行程序 → 断开连接并删除运行时」（英文界面：Runtime → Disconnect and delete runtime）。

## 流程

```
① 设置（文稿、声音、角色图）
        │
        ▼
③ 配音：VoxCPM2
   声音包参考音（默认）；或音色设计（用文字描述造一把声音，只做一次，存成参考音）
   → 逐句可控克隆（每句 ≤80 字，超过自动切开）
   → 句间插入停顿，接成一条音轨 voice.wav（同时记下每句的起止时间＝字幕时间轴）
   → 删除模型、释放显存
③-2 字幕准备：下载字体、检查缺字（有问题在对嘴之前就报错）
        │
        ▼
④ 对嘴：ComfyUI v0.43.0 + InfiniteTalk（Wan2.1 I2V 14B 480P）
   角色图按 ASPECT／CROP_POS 裁切缩放（默认 9:16 的 480×832）
   （BG_REMOVE＝绿幕时：先用 BiRefNet 把背景换成纯绿（发丝先做前景色估计），释放显存后才启动 ComfyUI）
   → 每段 81 帧、重叠 9 帧往后延伸 → 长度跟配音走
        │
        ▼
⑤ 合成：画面 + 原始配音（AAC）→ 烧进字幕（ASS＋ffmpeg）→ /content/output/talking_avatar.mp4 ＋ talking_avatar.srt ＋ 不带字幕的 talking_avatar_nosub.mp4 → 去背关闭时在这里打包下载
        │
        ▼
⑤-2 去背输出（BG_REMOVE 不是「关」时）：绿幕＝ffmpeg chromakey＋despill；逐帧＝每帧 BiRefNet
   → talking_avatar_alpha.webm（VP9＋透明通道，不烧字幕）＋ 绿幕时的绿幕图 input_greenscreen.png
   → 连同 ⑤ 的文件打包成 talking_avatar_outputs.zip 下载（一次流程只自动下载一次）
        │
        ▼
⑥ 只重做字幕（换字幕样式时才用；「全部运行」时会自动跳过）
```

（② 是环境检查：显示 GPU 型号、取得示例文件、确认输入文件存在。）

## 设置说明

只需要改 notebook 的 **① 设置** 格：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SCRIPT` | 3 句示例文稿 | 一行一句；每句最多 80 字，超过会在标点处自动切开（VoxCPM2 单次太长容易啸叫或音色漂移）。句首可以加括号写语气，例如 `(轻松地)大家好` |
| `VOICE` | `新闻女主播` | 下拉选单选声音包里的声音（见下方「声音包」，可先试听）；选「自定义（用 VOICE_DESC）」改用文字描述做音色设计。开启声音克隆时不使用 |
| `VOICE_DESC` | 年轻女性、清亮自然… | 音色设计用的文字描述（性别、年龄、音色、语速、语气）；`VOICE` 选「自定义」时才使用 |
| `PAUSE_SEC` | `0.4` | 句与句之间的停顿秒数 |
| `SEED` | `42` | 随机种子。音色设计每次结果不同，对声音不满意就换个数字重跑；用声音包时音色由参考音固定，`SEED` 只影响每句的细微差异和对嘴画面 |
| `IMAGE_PATH` | `examples/host.png` | 角色图。换成自己的图：上传到 Colab 左侧「文件」面板（会放在 `/content/`），改成 `/content/你的图.png` |
| `PROMPT` | 通用英文描述 | 对嘴画面的英文描述，可补充角色外观（发型、服装），减少配件消失 |
| `ASPECT` | `9:16 竖屏短视频（480×832）` | 画面比例下拉选单，见下方「画面比例」 |
| `CROP_POS` | `偏上（保留头部）` | 角色图比例和画面不同时裁掉哪一边：偏上、置中、偏左、偏右，见下方「画面比例」 |
| `BG_REMOVE` | `关` | 去背：关、绿幕（推荐）、逐帧，见下方「去背」 |
| `LOW_VRAM` | `False` | 对嘴阶段爆显存时改成 `True`（ComfyUI `--novram`，更慢） |
| `SUBTITLE` 等 | 开启 | 字幕开关、字体、字号、效果、位置、描边、颜色，见下方「字幕」 |
| `ENABLE_VOICE_CLONE` 等 | 关闭 | 见下方「声音克隆」 |

角色图建议：比例接近 `ASPECT` 选的画面（默认竖图）、正面、闭嘴、脸够大、手不入镜。图会先按 `CROP_POS` 裁成画面的比例（默认保留上方、裁掉下方），再缩成画面尺寸（默认 480×832）。

示例文稿也放在 [`examples/script.txt`](examples/script.txt)。

## 声音包

`VOICE` 下拉选单里的 10 把声音。**全部是用 VoxCPM2 音色设计（文字描述）生成的虚构 AI 声音，不对应任何真实人物。** 参考音放在 [`voices/`](voices/)，描述原文见 [`voices/voices.json`](voices/voices.json)。选了声音包后，句首括号的语气照样可以用。

| 声音 | 简介 | 试听 |
|---|---|---|
| 新闻女主播 | 女，30 岁左右。字正腔圆、端庄亲切，像电视台晚间新闻主播 | [试听](voices/samples/f1_news.mp3) |
| 温柔知性 | 女，28 岁左右。柔和温润、带一点气声，像深夜电台主持人 | [试听](voices/samples/f2_gentle.mp3) |
| 元气少女 | 女，19 岁左右。明亮清甜、活泼带笑，像有朝气的校园学妹 | [试听](voices/samples/f3_genki.mp3) |
| 甜美邻家 | 女，20 岁出头。甜美软糯、带点撒娇，像邻家女孩跟朋友聊天 | [试听](voices/samples/f4_sweet.mp3) |
| 成熟御姐 | 女，30 岁出头。低沉有磁性、自信慵懒，气场强但不凶 | [试听](voices/samples/f5_queen.mp3) |
| 热情带货 | 女，26 岁左右。清亮有穿透力、兴奋急切，像直播间带货主播 | [试听](voices/samples/f6_live.mp3) |
| 沉稳旁白 | 男，40 岁左右。浑厚低沉、沉着有故事感，像纪录片旁白 | [试听](voices/samples/m1_narrator.mp3) |
| 阳光青年 | 男，23 岁左右。清爽干净、阳光真诚，像人缘很好的学长 | [试听](voices/samples/m2_sunny.mp3) |
| 亲切大叔 | 男，50 岁左右。温厚略沙哑、随和幽默，像楼下爱聊天的老大哥 | [试听](voices/samples/m3_uncle.mp3) |
| 激情解说 | 男，30 岁左右。高亢有力、激昂兴奋，像体育或电竞比赛解说 | [试听](voices/samples/m4_hype.mp3) |

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
3. 开启克隆后 `VOICE` 和 `VOICE_DESC` 都不再使用。

## 字幕（默认开启）

成品默认把字幕烧进画面，另外给一份 `talking_avatar.srt`（可导入剪辑软件或上传到平台）。

- **文字**：直接用文稿原文（不含句首语气括号），不做语音识别，所以不会有错字。
- **时间**：按每句配音的实际长度加 `PAUSE_SEC` 推算。句间停顿时上一句字幕继续留着，直到下一句出现。
- **长句**：一行放不下时自动切成几段轮流显示（优先在标点后切；只在词和词之间切，用 [jieba](https://github.com/fxsjy/jieba) 分词，「了、的、吗」这类助词不放行首；jieba 装不上时退回按字切），每段时间按字数比例分配。
- 字号、边距、一行放几个字都按画面尺寸（notebook 的 `WIDTH`、`HEIGHT`）推算，不写死像素。

在 ① 设置格「字幕」那一段调整（Colab 右侧表单有开关、滑杆和下拉选单）：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SUBTITLE` | `True` | 字幕开关。关掉时成品不带字幕 |
| `SUB_FONT` | `思源黑体（粗）` | 字体下拉选单，见下方字体表；选「自定义（上传字体文件）」改用 `SUB_FONT_FILE` |
| `SUB_FONT_FILE` | `/content/my_font.ttf` | 自定义字体：把 `.ttf`／`.otf` 上传到左侧「文件」面板后填路径。请自行确认该字体允许你的用途（例如商用） |
| `SUB_SIZE` | `100` | 字号（%）。100＝画面短边的 1/12（480×832 画面约 40 像素）；滑杆 50～200 |
| `SUB_EFFECT` | `淡入` | 出现效果：无、淡入、淡入淡出、弹出、滑入、逐字（见下表） |
| `SUB_POSITION` | `底部` | 底部、中间、顶部 |
| `SUB_MARGIN` | `12` | 离画面下边缘（选「顶部」时是上边缘）的距离，占画面高度的 %；选「中间」时不用 |
| `SUB_OUTLINE` | `8` | 描边粗细，占字号的 %；0＝不描边。默认是单层黑色描边 |
| `SUB_COLOR` | `#FFFFFF` | 字的颜色，写成 `#RRGGBB` |
| `SUB_OUTLINE_COLOR` | `#000000` | 描边颜色，写成 `#RRGGBB` |

效果（都是 ASS 字幕的标准特效，由 ffmpeg 的 libass 画上去）：

| 效果 | 看起来 |
|---|---|
| 无 | 直接出现、直接消失 |
| 淡入 | 每段字幕约 0.2 秒淡淡出现（默认） |
| 淡入淡出 | 出现和消失都淡入淡出 |
| 弹出 | 从小放大、略超过一点再回到原大小 |
| 滑入 | 从下方往上滑到定位，同时淡入 |
| 逐字 | 按念的速度一个字一个字出现（按字数平均分配时间，是估算） |

### 只重做字幕（换样式不用重跑）

跑完一次后想换字幕样式，**在删除运行时之前**：改 ① 的字幕设置 → 运行 ① → 运行 **⑥ 只重做字幕** 格。它拿 `/content/output/` 里不带字幕的版本 `talking_avatar_nosub.mp4` 和字幕时间轴 `subtitle_timeline.json` 重新烧字幕，不重跑配音和对嘴，新的成品和字幕打包成 `talking_avatar_resub.zip` 下载。中途「重启会话」过的话，先依次运行 ①、②、③-2，再运行 ⑥（不要运行 ③-1，会重新配音）。

### 字体

都是开源字体（除「霞鹜新晰黑」是 IPA Font License 1.0 外，其余均为 SIL Open Font License 1.1），**不包含在本仓库里**，运行时从下列官方发布处下载。网址都固定到具体版本（或固定分支上的文件），下载网址已于 2026-11-02 实际下载验证过；授权链接已于同日用 GitHub API 核对过各仓库里确实有对应的 LICENSE／OFL 文件。

| 下拉选项 | 字体 | 运行时下载来源 | 授权 |
|---|---|---|---|
| 思源黑体（粗） | Source Han Sans SC Bold | GitHub `adobe-fonts/source-han-sans`（release 分支） | [SIL OFL 1.1](https://github.com/adobe-fonts/source-han-sans/blob/master/LICENSE.txt) |
| 站酷快乐体 | ZCOOL KuaiLe | GitHub `google/fonts`（Google Fonts 官方仓库） | [SIL OFL 1.1](https://github.com/google/fonts/blob/main/ofl/zcoolkuaile/OFL.txt) |
| 未来荧黑 Heavy | Glow Sans SC Normal Heavy | GitHub `welai/glow-sans` Release v0.93（压缩包约 64MB，只取其中的 Heavy 字重） | [SIL OFL 1.1](https://github.com/welai/glow-sans/blob/master/OFL.txt)（该仓库的程序代码是 MIT，字体文件是 OFL） |
| 得意黑 | Smiley Sans Oblique | GitHub `atelier-anchor/smiley-sans` Release v2.0.1 | [SIL OFL 1.1](https://github.com/atelier-anchor/smiley-sans/blob/main/LICENSE) |
| 寒蝉圆黑体 | ChillRoundM | GitHub `Warren2060/ChillRound` Release v1.805 | [SIL OFL 1.1](https://github.com/Warren2060/ChillRound/blob/main/LICENSE) |
| 霞鹜新晰黑 | LXGW Neo XiHei | GitHub `lxgw/LxgwNeoXiHei` Release v1.308 | [IPA Font License 1.0](https://github.com/lxgw/LxgwNeoXiHei/blob/main/LICENSE.md)（**不是 OFL**） |
| 霞鹜文楷 | LXGW WenKai Medium | GitHub `lxgw/LxgwWenKai` Release v1.530 | [SIL OFL 1.1](https://github.com/lxgw/LxgwWenKai/blob/main/OFL.txt) |

③-2 会检查字体有没有收录文稿里的每个字，缺字时会列出来（烧进画面时那些字会被换成别的字体或显示成方框）。思源黑体收的字最全；寒蝉圆黑体只收 GB2312 的字，文稿里有生僻字时容易缺字。

## 画面比例

在 ① 设置格用 `ASPECT` 下拉选单选。像素总量都接近 480×832（对嘴底模是 480P），长宽都是 16 的倍数。**作者实测过 9:16 和 1:1（624×624），都一次跑通；16:9、4:5、3:4 未实测。**

| `ASPECT` | 画面尺寸 | 常见用途 |
|---|---|---|
| 9:16 竖屏短视频（默认） | 480×832 | 短视频、限时动态 |
| 16:9 横屏视频 | 832×480 | 横屏视频 |
| 1:1 方形帖子 | 624×624 | 方形帖子 |
| 4:5 Instagram 竖版帖子 | 576×720 | Instagram 竖版帖子 |
| 3:4 小红书竖版 | 576×768 | 小红书 |

角色图比例和画面不同时，先裁切再缩放。裁切只会裁一个方向（图太宽裁左右、太高裁上下），`CROP_POS` 决定保留哪里：

| `CROP_POS` | 图太高（裁上下） | 图太宽（裁左右） |
|---|---|---|
| 偏上（保留头部，默认） | 保留上方 | 保留中间 |
| 置中 | 保留中间 | 保留中间 |
| 偏左 | 保留上方 | 保留左边 |
| 偏右 | 保留上方 | 保留右边 |

字幕的字号、边距、一行放几个字都按实际画面尺寸推算，换比例不用另外调。对嘴做完之后再改 `ASPECT` 不会生效（⑥ 只重做字幕，照现有视频的尺寸），要换比例得重新全部运行。

## 去背（默认关闭）

想把主播放进别的模板合成（例如直播版面）时用。① 设置格 `BG_REMOVE` 选「绿幕（推荐）」或「逐帧」后，除了一般的 `talking_avatar.mp4`，⑤-2 会再输出一份透明背景的 **`talking_avatar_alpha.webm`**：

- VP9 视频＋透明通道（`yuva420p`）＋原始配音（Opus）。
- **不烧字幕**（需要字幕可以在模板里另外加，或用 `talking_avatar.srt`）。字幕照常烧在 `talking_avatar.mp4` 里，⑥ 只重做字幕也照常能用。
- 透明效果要用支持透明 WebM 的软件才看得到；有些播放器会显示成黑底。

去背工具是 [BiRefNet](https://github.com/ZhengPeng7/BiRefNet)（MIT），权重运行时从 Hugging Face 官方仓库 `ZhengPeng7/BiRefNet` 下载（commit 已固定）。BiRefNet 和对嘴模型不会同时占显存。

| `BG_REMOVE` | 做法 | 优缺点 |
|---|---|---|
| 关（默认） | 不去背 | — |
| 绿幕（推荐） | ④-3 对嘴前先用 BiRefNet 把角色图的背景换成纯绿色（用完马上释放显存；合成到绿底前用 [pymatting](https://github.com/pymatting/pymatting) 做前景色估计，去掉发丝里混进来的原背景颜色）→ 对嘴 → ⑤-2 用 ffmpeg `chromakey` 扣掉绿色，再用 `despill` 压掉边缘的绿光 | 只多处理一张图。**这时 `talking_avatar.mp4` 的背景是绿色的。** 已知风险见「常见问题」 |
| 逐帧 | 角色图照原样对嘴 → ⑤-2 先关掉 ComfyUI 释放显存，再对成品的每一帧跑 BiRefNet 得到透明遮罩 | `talking_avatar.mp4` 保留原背景；比绿幕慢（每一帧都要算），前后帧的边缘可能轻微闪动 |

绿幕路线的扣色参数（`KEY_SIMILARITY`、`KEY_BLEND`、`DESPILL_MIX`、`DESPILL_BLUE`）写在 ⑤-2 格开头，一般不用改。绿幕的实际色值会从第一帧的背景自动取。⑤-2 下载的 zip 里会附上对嘴用的绿幕图 `input_greenscreen.png`：边缘不满意时拿它对照，绿幕图上就有毛边＝去背那一步的问题，绿幕图干净＝扣绿这一步的问题。

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
| 安装 ComfyUI＋下载约 30GB 模型＋启动 | 约 2.5 分钟（两次实测：28＋92＋27 秒、25＋92＋18 秒） |
| 对嘴（21.5 秒成品，8 段） | 约 **36 分钟**，约 **0.9 计算单元** |
| 示例文稿全程（10.88 秒成品，4 段） | 对嘴 966 秒（约 16 分钟）；从 ② 到 ⑤ 跑完共 22.2 分钟 |
| 1:1（624×624）、14.16 秒配音，5 段 | 对嘴约 19 分钟；一次完整流程（含装环境）连线约 31～37 分钟，约 1 个计算单元 |
| 去背：绿幕 ⑤-2 | 约 40 秒 |
| 去背：逐帧 ⑤-2 | 354 帧约 92 秒（整格约 141 秒） |
| ⑥ 只重做字幕 | 每次约 2～5 秒 |

16:9、4:5、3:4 三种比例的耗时还没实测。

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
先在「声音包」试听，`VOICE` 换一把重跑；都不合适就选「自定义（用 VOICE_DESC）」，改 `VOICE_DESC` 或换 `SEED` 重跑（音色设计本来就每次不同）。

**字幕和声音有点对不上？**
字幕时间是按每句配音长度推算的；一句被切成几段时，每段时间按字数比例估，念得快慢不均时会有些偏差。把长句拆短（一行一句）最有效。

**字幕想换样式？**
不用重跑：改 ① 的字幕设置 → 运行 ① → 运行 ⑥（见「字幕 → 只重做字幕」）。删除运行时后就不能重做了。

**去背选了「绿幕」，扣出来有绿边，或背景有一块没扣掉？**
作者实测过绿幕（1:1，一次跑通）：对嘴后绿色背景没有明显色偏，扣除色自动取到接近纯绿；绿底合成前已做前景色估计，发丝外围的深色描边实测消失。逆光、头发蓬乱飞毛多的图，发丝外围仍可能留一圈淡毛边，见下一条「去背后头发外围有一圈毛边？」。
绿幕路线出问题（背景扣不干净、边缘带绿）时，在 ① 把 `BG_REMOVE` 改成「逐帧」，然后**重新全部运行**。不要只运行 ⑤-2：拿绿底的成品再跑逐帧，发丝和缝隙边缘实测会带绿。

**去背后头发外围有一圈毛边？**
多半是角色图本身的问题：逆光（人物背后有强光、发丝边缘发亮）或头发蓬乱、飞毛多的图，去背后发丝外围容易留下一圈毛边（原图背景的颜色混在发丝里，换参数也去不干净）。主播图尽量选不逆光、头发整齐的；有这种图时建议换图。

**透明 WebM 打开是黑底？**
不是所有播放器都支持透明 WebM。放进支持透明视频的剪辑或合成软件里才看得到透明效果。

**浏览器没有自动下载？**
一次流程只会自动下载一次（文件多时打包成一个 zip），但 Chrome 还是可能拦截。网址栏右边出现下载被拦截的图标时，点它选允许；或到左侧「文件」面板的 `output` 文件夹，右键文件 →「下载」。

**用完了怎么关？**
「代码执行程序 → 断开连接并删除运行时」。第一次点可能先弹出警告，确认后要再点一次删除。

## 第三方模型与授权

本仓库的代码以 MIT 发布；下列组件（以及「字幕」一节的字体）**不包含在本仓库里**，是在 Colab 运行时从原始来源下载的，各自适用其授权。以下为作者 2026-10-31 查证的结果，使用前请以各来源页面为准。

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
| BiRefNet（代码＋权重） | 去背（`BG_REMOVE` 不是「关」时才下载） | HF `ZhengPeng7/BiRefNet` @ `e2bf8e44`（`trust_remote_code`，代码和权重都来自这个仓库） | MIT（2026-11-02 查证） | <https://huggingface.co/ZhengPeng7/BiRefNet>、<https://github.com/ZhengPeng7/BiRefNet/blob/main/LICENSE> |
| jieba 0.42.1 | 字幕切段时中文分词（开字幕且有长句要切时才安装） | PyPI `jieba==0.42.1` | MIT（2026-11-02 查证） | <https://github.com/fxsjy/jieba/blob/master/LICENSE> |
| pymatting 1.1.16 | 绿幕路线的前景色估计（`BG_REMOVE`＝绿幕时才安装） | PyPI `pymatting==1.1.16` | MIT（2026-11-02 查证） | <https://github.com/pymatting/pymatting/blob/master/LICENSE.md> |
| lightx2v I2V 480P 步数/CFG 蒸馏 LoRA rank64 | 加速（6 步） | HF `Kijai/WanVideo_comfy` | 托管仓库：**未确认**（模型卡未标授权）；上游 lightx2v 模型：Apache-2.0（按文件名对应，未逐字节核对） | <https://huggingface.co/Kijai/WanVideo_comfy>、<https://huggingface.co/lightx2v/Wan2.1-I2V-14B-480P-StepDistill-CfgDistill-Lightx2v>、<https://github.com/ModelTC/LightX2V/blob/main/LICENSE> |

示例图 `examples/host.png` **由 AI 生成，为虚构人物，非真实人物**（说明见 [`examples/README.md`](examples/README.md)）。声音包 `voices/` 的 10 把声音由 VoxCPM2 音色设计生成，同样是虚构声音。

## 致谢

- [OpenBMB / VoxCPM](https://github.com/OpenBMB/VoxCPM)
- [MeiGen-AI / InfiniteTalk](https://github.com/MeiGen-AI/InfiniteTalk)
- [Wan-Video / Wan2.1](https://github.com/Wan-Video/Wan2.1)
- [ComfyUI](https://github.com/Comfy-Org/ComfyUI) 及其官方 InfiniteTalk 工作流模板
- [Kijai](https://huggingface.co/Kijai)（模型格式转换与托管）、[Comfy-Org](https://huggingface.co/Comfy-Org)
- [LightX2V](https://github.com/ModelTC/LightX2V)
- [TencentGameMate / chinese-wav2vec2-base](https://huggingface.co/TencentGameMate/chinese-wav2vec2-base)
- [ZhengPeng7 / BiRefNet](https://github.com/ZhengPeng7/BiRefNet)

## 授权

[MIT](LICENSE) © 2026 dawang-sakura
