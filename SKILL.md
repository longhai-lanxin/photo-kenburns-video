---
name: photo-kenburns-video
description: 把一批静态照片做成「Ken Burns 运镜 + 叠化转场 + 烧录字幕」的文艺短视频（图片为主）。用于探店/生活/宣传片图文视频。触发词：照片做视频、图片视频、Ken Burns、图文短视频、运镜视频、照片宣传片。
agent_created: true
---

# 照片 → 文艺短视频（Ken Burns + 叠化 + 字幕）

把 HEIC/JPG 照片批量做成横版 1920×1080 短视频，含缓动运镜、叠化转场、旁白字幕、片尾 Logo。

## 环境
- Windows + Git Bash；`python`（PIL/pillow-heif/opencv-python）、`ffmpeg` 需可用。
- 中文字体：`C:\Windows\Fonts\msyh.ttc`（微软雅黑）。

## 流程

### 1. HEIC 解码（原稿保留）
```python
import pillow_heif; pillow_heif.register_heif_opener()
from PIL import Image
Image.open(src).convert("RGB").save(dst, quality=95)
```

### 2. 逐帧 Ken Burns（关键：勿用 ffmpeg zoompan，会抖）
**抖动三大根因**：
1. zoompan 的 `x_dir/y_dir` 用 `+N*on` 时每帧平移 N px，N≥10 就频闪；
2. zoompan 裁剪中心整数取整 → 亚像素漂移；
3. 线性步进无缓动 → 起停顿挫。

**解法**：Python + cv2 逐帧渲染，smoothstep 缓动 + 浮点亚像素裁剪 + INTER_CUBIC：
```python
import cv2, numpy as np
def smoothstep(t):
    t=max(0,min(1,t)); return t*t*(3-2*t)
def render(src, out, dur, zoom_start=1.0, zoom_end=1.12, pan_x=0.0, pan_y=0.0, W=1920, H=1080, fps=30):
    img=cv2.imread(src); ih,iw=img.shape[:2]
    n=int(round(dur*fps))
    w=cv2.VideoWriter(out, cv2.VideoWriter_fourcc(*'mp4v'), fps, (W,H))
    for i in range(n):
        e=smoothstep(i/max(n-1,1)); z=zoom_start+(zoom_end-zoom_start)*e
        cw,ch=W/z,H/z
        cx=iw/2+pan_x*e*(iw-cw)/2; cy=ih/2+pan_y*e*(ih-ch)/2
        x0=max(0,min(cx-cw/2,iw-cw)); y0=max(0,min(cy-ch/2,ih-ch))
        crop=img[int(round(y0)):int(round(y0+ch)), int(round(x0)):int(round(x0+cw))]
        w.write(cv2.resize(crop,(W,H),interpolation=cv2.INTER_CUBIC))
    w.release()
```
- pan_x/pan_y 用归一化偏移 ∈[-1,1]，走缓动，步长天然平滑。
- 拉远 = `zoom_start > zoom_end`。

### 3. 段内/段间 xfade 叠化
- filter_complex 输入标签必须带方括号 `[0:v]`（写 `0:v` 报 "Trailing garbage"）。
- offset 累加用 `show`（单图显示时长），不是 `show-fade`：
  ```
  [0:v][1:v]xfade=transition=fade:duration=0.6:offset=2.4[x1];[x1][2:v]xfade=...offset=5.4[x2]
  ```

### 4. drawtext 烧字幕（两大坑）
- **字体路径冒号必须转义**：`fontfile='C\:/Windows/Fonts/msyh.ttc'`（不转义会报 "No option name near '/Windows/Fonts/..."）。
- **多段字幕用逗号串联**，共享 `[0:v]` 输入，末尾 `[outv]`（链式标签容易报 unconnected）：
  ```
  [0:v]drawtext=...enable='between(t,0,5.4)',drawtext=...enable='between(t,4.9,10.4)'[outv]
  ```
- 换行用 `\N`；`enable='between(t,start,end)'` 控显隐。

### 5. 片尾黑屏 Logo + concat
```bash
ffmpeg -f lavfi -i color=c=black:s=1920x1080:d=3:r=30 -vf "drawtext=...Logo..." end.mp4
# 用 concat demuxer 拼：写 list.txt 两行 file '...'，-f concat -safe 0 -c copy
```

## 关键坑速查
| 现象 | 原因 | 修 |
|---|---|---|
| 运镜抖动/频闪 | zoompan 平移步长过大/整数取整 | 改逐帧渲染 + smoothstep |
| filter "Trailing garbage" | 标签漏方括号 | `[0:v]` |
| drawtext "No option name" | 路径 `C:` 冒号未转义 | `C\:/...` |
| xfade "unconnected output" | 标签未连/未 map | 每链都带标签 + `-map [outv]` |

## 输出
- 成品 mp4 + 抽帧预览（`ffmpeg -vf "select='eq(n,..)+..'" -vsync vfr`）。
- 原稿 HEIC 不动，产物单独目录 `_art/`。
