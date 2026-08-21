# A-simple-GUI-for-video-data-calibration
A simple GUI which is used for video data calibration to train VLA model. Based on colaboration with intern colleagues from DeepRobotics.

---

# Robot Manipulation Video Segment Annotation GUI (demo) [subtask_editor_gui_prefill]

A PyQt5-based video segment annotation tool for semantic subtask labeling of
robot manipulation datasets. The core workflow is **template + boundary
marking**: load an action template, press Space on the video timeline to place
boundaries, and split a video into named semantic segments.

The example dataset is `lerobot_v3_4to1_vase_force` ("wipe the vase."): seven
base actions (拿板擦 / 移板擦 / 抓瓶沿 / 上下擦 / 转压瓶 / 收板擦 / 移瓶收臂),
each reviewed segment by segment on a 3Hz synthetic three-view preview video
(base + left/right wrist).

## Features

Three independent workflow sections (bottom dock):

| Section | Description |
|---|---|
| **Template** | Load / edit / save action templates; usable standalone or together with either workflow |
| **Coarse-mark refinement** | Sequential boundary marking: Space advances to the next segment, P inserts a new segment, Alt+Number marks and names; includes auto tail cleanup (adjustable threshold), auto tail naming, and Alt+Number mapping configuration |
| **Out-of-order repeat annotation** | For repeated out-of-order actions: Space places a boundary (the break = end of the previous segment + start of the next) → playback pauses → a dialog names the segment that just ended via Alt+Number → playback resumes; marking the last segment auto-inserts a new one; at the end of the video the tail is finished automatically (silent naming when "Auto tail naming" is on, otherwise a dialog) |

The two annotation modes are mutually exclusive: while either is active, both
buttons act as stop buttons.

Other features:

- Common area: playback controls (±1s / ±5s, 0.5×–4× speed), manual marking
  row (mark boundary from segment 1 / record boundary / undo)
- Template "Save current, and apply proportionally to the next": suited to
  data whose frame lengths are close within a group; scales all boundaries
  proportionally to the total video duration
- Shortcuts: `Space` mark / advance, `P` insert new segment, `Alt+Number` mark
  and write the mapped state, `Ctrl+S` save, `Ctrl+Enter` save and next,
  `Ctrl+D` save and previous
- Every mark is written to disk automatically (the current episode's result
  JSON; a `.bak` of the previous state is maintained inside the GUI)

## Directory layout

```
guiproject/
├── run_guiproject.sh            # Launcher script
├── subtask_editor_gui_prefill.py# Main program (three-section panel + all workflows)
├── subtask_editor_gui.py        # Base editor (video / timeline / segment rows)
├── install_gui_deps.sh          # One-time dependency install
├── replace_char_keys.py         # Companion tool: short-character labels → English descriptions
├── build_gui_fill_videos.py     # Companion tool: build fill videos (optional)
├── prefill_custom_template.json # Template library (created on demand: after finishing an episode, use "Save as template")
└── prefill_alt_map.json         # Alt+Number mapping config (can be re-configured inside the GUI if missing)
```

## Installation

If the dependencies are already installed on your machine, you can skip this
section. The simplest way is to run, in the current directory:

```bash
bash install_gui_deps.sh   # installs PyQt5, opencv-python, imageio-ffmpeg (optional)
```

Dependencies: Python 3.10+, PyQt5, opencv-python (`cv2`), ffmpeg (used to
auto-build preview videos when the synthetic videos are missing).

It checks / attempts to install:

```text
python3
ffmpeg
PyQt5
opencv / cv2
```

If the automatic installation fails, on Ubuntu you can install manually:

```bash
sudo apt update
sudo apt install -y ffmpeg python3-pyqt5 python3-opencv
```

## Data layout

The GUI ships without data. Point it at your data root via the `ATOM_ROOT`
environment variable:

```
<ATOM_ROOT>/
├── classified_lerobot_v1/
│   └── <dataset>/
│       ├── videos/                        # Raw camera videos (base_0_rgb / left_wrist_0_rgb / right_wrist_0_rgb)
│       └── sidecars/submem/v11_final/
│           └── episode_000000/result.v11_final_memory.json   # Annotation data
└── audit_tools/
    └── gui_videos_3hz_review/             # Pre-built 3Hz three-view synthetic videos (auto-built with ffmpeg if missing)
```

`ATOM_ROOT` defaults to the parent directory of the repository (same behavior
as the original `audit_tools/run_subtask_editor_gui_prefill.sh`), or you can
set it explicitly:

```bash
./run_guiproject.sh                                # default: parent directory
ATOM_ROOT=/path/to/data ./run_guiproject.sh        # explicit
```

## Usage

1. Start with `./run_guiproject.sh`; select the dataset and episode on the left
2. Load a template in the "Template" section (or annotate one episode first, then "Save as template")
3. Choose a workflow:
   - **Coarse-mark refinement**: enable → press `Space` during playback to
     place a boundary and advance (in template order); `Alt+Number` marks and
     writes the mapped state
   - **Out-of-order repeat annotation**: enable → playback starts
     automatically; `Space` places a boundary and opens a dialog, `Alt+Number`
     names the segment; the tail is finished automatically at the end of the
     video
4. Marks are written to disk automatically; after finishing, "Save as
   template" lets you reuse it as a new template

### Coarse-mark refinement

This workflow targets pre-processed data with repeated actions, or manual
proofreading scenarios. With the shortcut keys you can mark or calibrate
quickly, making the annotation time close to the length of the preview
material and speeding up the work. In the packaged example, the "wipe
up-and-down" and "rotate and press" actions alternate several times, which
fits this workflow.

### Out-of-order repeat annotation

This workflow suits datasets with multiple repeated actions in out-of-order
sequences. When repeated actions are numerous, it avoids the human errors and
awkward operations the previous workflow is prone to.

### Batch replacement after short-label annotation

The GUI writes the short-character labels from the template (e.g. `转压瓶`).
After all annotation is done, expand them to English descriptions with the
companion tool:

```bash
python3 replace_char_keys.py --all            # Replace the whole library (auto-creates .bak, idempotent)
python3 replace_char_keys.py 000005 000042    # Replace only the specified episodes
python3 replace_char_keys.py --no-write --all # Preview without writing
```

The mapping table lives in the `MAPPING` dict inside the script (the
`REPLACE_CHAR_KEYS_RESULT_DIR` environment variable overrides the result
directory).

## Known conventions

- Boundary times are rounded to 1/3 second (3Hz); each segment is at least
  1/3 second long
- The two annotation modes are mutually exclusive; enabling one turns both
  buttons into stop buttons
- Segment editing: change the action name via the per-row dropdown; `Up` /
  `Down` moves to the adjacent segment
- Episode JSON structure: `parsed.subtasks` list with fields `start_s` /
  `end_s` / `current_subtask` / `manipulated_objects` / `target_location` /
  `confidence` / `boundary_reason` / `evidence`

## Future outlook

Based on this GUI, integrating a multimodal LLM could enable batch coarse
pre-labeling, and combined with the coarse-mark refinement workflow achieve
fast labeling.

This project is kept as a personal summary of hands-on learning experience,
built mainly around personal working habits, so it inevitably has plenty of
shortcomings.

---

# 机器人操作视频分段标注 GUI（demo）[subtask_editor_gui_prefill]

基于 PyQt5 的视频分段标注工具，用于机器人操作（robot manipulation）数据集的
语义分段（subtask/semantic segment）标注。以「模板 + 边界打点」为核心：
加载动作模板后，按空格在视频时间轴上打点，把一段视频切成若干语义片段并命名。

数据示例为 `lerobot_v3_4to1_vase_force`（wipe the vase.）：七个基础动作
（拿板擦 / 移板擦 / 抓瓶沿 / 上下擦 / 转压瓶 / 收板擦 / 移瓶收臂），
以 3Hz 合成三视角预览视频（base + 左右 wrist）逐段校对。

## 功能

三个独立工作区分区（底部 dock）：

| 分区 | 说明 |
|---|---|
| **模板** | 加载 / 编辑 / 保存动作模板，可单独使用或配合任一工作流 |
| **粗标数据手动细化** | 顺序打点细化：Space 推进到下一段、P 插入新段、Alt+数字 打点并命名；含自动清理片尾（阈值可调）、自动收尾命名、Alt+数字映射配置 |
| **乱序重复标注** | 乱序重复动作：Space 打点（断点 = 前段终点 + 后段起点）→ 播放暂停 → 弹窗 Alt+数字 命名刚结束的段 → 继续；打到最后一个段自动插入新段；视频到结尾自动收尾（开启「自动收尾命名」时静默命名，否则弹窗选择） |

两个标注模式互斥（任一激活时两个按钮都是停止键）。

其它：

- 通用区：播放控制（±1s/±5s、倍速 0.5×–4×）、手动打点行（从第 1 段打边界 / 记录边界 / 撤销）
- 模板「保存当前，并按相同比例套用到下一条」：适合同组 frame 接近的数据，按视频总时长等比例缩放全部边界
- 快捷键：`Space` 打点 / 推进，`P` 插入新段，`Alt+数字` 打点并写入映射状态，`Ctrl+S` 保存，`Ctrl+Enter` 保存并下一条，`Ctrl+D` 保存并上一条
- 每次打点自动写盘（写回当前 episode 的 result JSON，含 `.bak` 之前的状态由 GUI 内维护）

## 目录结构

```
guiproject/
├── run_guiproject.sh            # 启动脚本
├── subtask_editor_gui_prefill.py# 主程序（三分区面板 + 全部工作流）
├── subtask_editor_gui.py        # 基础编辑器（视频/时间轴/分段行）
├── install_gui_deps.sh          # 首次安装依赖
├── replace_char_keys.py         # 配套工具：短字符标签 → 英文描述批量替换
├── build_gui_fill_videos.py     # 配套工具：构建填充视频（可选）
├── prefill_custom_template.json # 模板库（缺失可新建：完成一个 episode 后「保存为模板」）
└── prefill_alt_map.json         # Alt+数字 映射配置（缺失可在 GUI 内重新配置）
```

## 安装

如果机器已经装好依赖，可以跳过。最简单的方式是在当前目录运行：

```bash
bash install_gui_deps.sh   # 安装 PyQt5、opencv-python、imageio-ffmpeg（可选）
```

依赖：Python 3.10+、PyQt5、opencv-python（`cv2`）、ffmpeg（合成视频缺失时自动构建预览）。

它会检查/尝试安装：

```text
python3
ffmpeg
PyQt5
opencv / cv2
```

如果自动安装失败，Ubuntu 可以手动执行：

```bash
sudo apt update
sudo apt install -y ffmpeg python3-pyqt5 python3-opencv
```

## 数据布局

GUI 不包含数据。通过环境变量 `ATOM_ROOT` 指向数据根：

```
<ATOM_ROOT>/
├── classified_lerobot_v1/
│   └── <dataset>/
│       ├── videos/                        # 原始摄像头视频（base_0_rgb / left_wrist_0_rgb / right_wrist_0_rgb）
│       └── sidecars/submem/v11_final/
│           └── episode_000000/result.v11_final_memory.json   # 标注数据
└── audit_tools/
    └── gui_videos_3hz_review/             # 预构建 3Hz 三视角合成视频（缺失时自动用 ffmpeg 构建）
```

`ATOM_ROOT` 默认取仓库的上一级目录（与原版 `audit_tools/run_subtask_editor_gui_prefill.sh`
行为一致），也可显式指定：

```bash
./run_guiproject.sh                                # 默认上一级
ATOM_ROOT=/path/to/data ./run_guiproject.sh        # 显式
```

## 使用

1. `./run_guiproject.sh` 启动，左侧选择数据集与 episode
2. 「模板」分区加载模板（或先标注一个 episode 后「保存为模板」）
3. 选工作流：
   - 粗标数据手动细化：开启 → 播放中按 `Space` 打点推进（模板顺序），`Alt+数字` 打点并写入映射状态
   - 乱序重复标注：开启 → 自动播放，`Space` 打点并弹窗 `Alt+数字` 命名，视频到结尾自动收尾
4. 打点结果自动写盘；收尾后「保存为模板」可作为新模板


### 粗标数据手动细化
这一功能基于多重复动作的预处理数据或手动校对的工作场景 通过快捷键的使用实现快速打点或校准 可以使标注工作时长近似与预览素材的时长 加快工作效率 在打包的案例中 上下擦和压转瓶的动作多次交替 适配当前工作流

### 乱序重复标注
该功能适用于乱序进行多个重复动作数据标注当重复动作过多时 可解决上一工作流易出现人为失误与操作不便的问题

### 简短描述后统一映射替换

GUI 写入的是模板中的短字符标签（如 `转压瓶`）。全部标注完成后，用配套工具展开为英文描述：

```bash
python3 replace_char_keys.py --all            # 全库替换（自动生成 .bak，幂等）
python3 replace_char_keys.py 000005 000042    # 只替换指定 episode
python3 replace_char_keys.py --no-write --all # 预览不写入
```

映射表见脚本内 `MAPPING`（`数据根`/`REPLACE_CHAR_KEYS_RESULT_DIR` 环境变量可覆盖结果目录）。

## 已知约定

- 边界时间按 1/3 秒（3Hz）取整；每个分段最短 1/3 秒
- 两个标注模式互斥，开启任一后另一按钮变为停止键
- 段落编辑：行内下拉框改动作名，`Up/Down` 切相邻段
- episode JSON 结构：`parsed.subtasks` 列表，字段 `start_s` / `end_s` / `current_subtask` /
  `manipulated_objects` / `target_location` / `confidence` / `boundary_reason` / `evidence`

## 未来展望
基于该GUI 通过接入多模态大模型实现批量粗略标定预处理 再结合粗标数据手动细化的工作流实现快速标定的功能

该项目仅作为个人学习实践经验的总结留档 主要基于个人操作习惯 必然存在大量不足 
