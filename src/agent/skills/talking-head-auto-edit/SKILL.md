---
name: talking-head-auto-edit
description: |
  全自动口播视频增强——给一段口播/讲解/访谈视频，自动转录、分析内容，根据讲话内容动态决定在每个关键时刻添加什么视觉效果：大字动效、MG 动画、信息卡片、B-roll 配图、PiP 画中画、字幕、背景音乐。
  Use when the user says: 自动剪辑视频、帮我做成讲解视频、口播加特效、根据内容加效果、自动添加 MG、帮我做成和参考视频一样的效果、auto edit talking head, add motion graphics to my video, make it look like a YouTube video, 口播视频自动包装.
  Also triggers when user uploads a talking-head video and wants it produced/packaged.
user-invocable: true
---

# 口播视频全自动增强（Talking-Head Auto Edit）

把一段口播视频包装成精良的内容视频。所有视觉决策都从转录内容出发——没有固定模板，每个效果都由这段视频里讲的内容决定。

## 工作流总览

1. **环境与素材检查** — 读取项目状态，确认视频已上传
2. **内容分析** — 转录，逐句提取视觉触发点
3. **视觉方案规划** — 按时间顺序生成效果列表
4. **效果生成与放置** — 并行生成 MG + 配图，按时间线放置
5. **字幕** — 启用字幕
6. **背景音乐** — 匹配情绪
7. **QA** — 截帧核查无遮挡

---

## 第一步：环境检查

```
read_project         → 了解素材、时间线状态
read_timeline        → 确认主视频 clip 的 itemId、时长、fps
```

如果没有视频，通过 `widget-forms` 让用户上传，不要告诉他们去找上传按钮。

---

## 第二步：内容分析

```
transcribe_track     → 对主视频轨道转录，得到逐词时间戳
view_timeline_frames → 在关键时刻截帧，了解画面内容（演讲者位置、画面构图）
```

读取完整文字稿后，按「见 references/content-analysis.md」的规则提取每一个视觉触发点，生成一张按时间排序的"效果决策表"，格式如下：

```
时间点  | 讲话内容摘要              | 触发类型       | 视觉动作
--------|---------------------------|---------------|---------------------------
0:03    | "今天来聊一个很重要的话题"  | 开场钩子       | 大字动效——标题关键词
0:12    | "三个月时间增长了 400%"    | 数字/统计      | 数字卡片 MG
0:28    | "第一点：..."              | 列表项目       | 序号条 MG
1:05    | "就像这张图显示的..."       | 提到图/数据    | 生成相关配图（Unsplash/GPT）
1:40    | "对比一下旧方法和新方法"    | 对比           | 分栏对比卡 MG
2:15    | "你可以关注我的..."         | CTA            | CTA 卡片 MG
```

---

## 第三步：视觉方案规划

基于决策表，生成完整的效果方案。遵守以下原则：

- **密度**：平均每 5–8 秒一个效果，开头 15 秒可以密一些（提升留存）
- **分层**：大字动效放 V2 轨道，MG 卡片放 V3，配图/B-roll 放 V4 或替换 V1 片段
- **不遮挡**：截帧确认演讲者脸和手的位置，MG 放置不覆盖脸部、手部、字幕区域
- **效果不重复**：连续两个效果不使用同一种形式（例如两张数字卡），穿插不同类型

在开始生成之前，把方案用简洁文字列给用户确认（仅在不是完全自动运行时）：
> "我计划在 0:03 加标题大字，0:12 加数字卡片，0:28 加列表条…方向对吗？"

---

## 第四步：效果生成与放置

按决策表逐项执行，参考 references/visual-decisions.md 的生成规则。

### 关键原则

**每个效果都基于内容生成，不是套模板。**

使用 `submit_motion_graphic` 时，prompt 必须包含实际内容——把讲话里的关键词、数字、主题写进去。不要用"a motion graphic with text"这种通用描述。

```
好的 prompt 示例：
  "大字动效：白色粗体 '每天复利1%' 从左向右扫光揭示，深色背景，用于强调这句核心论点，3秒"

坏的 prompt 示例：
  "a text animation motion graphic"
```

### 图片/配图决策

当触发类型是"提到具体事物/场景/产品"时：
1. 优先用 `search_photos`（Unsplash）搜索相关图片，关键词来自讲话内容
2. 如果需要定制化（品牌图、图表、示意图），用 `submit_image`（GPT 生图）
3. 以 PiP 形式放在画面一侧（演讲者在左/右，配图在对侧）

### 全屏切出

当触发类型是"展示截图/网页/演示"且用户提供了屏录素材时：
- 把屏录片段放 V1，演讲者视频做圆形 PiP 叠加（用 `edit_item` 设置 `transform.borderRadius` + `transform.scale` + `transform.x/y`，把演讲者缩到约 25% 大小放左下角）
- 演讲者 PiP 加黄色/金色边框（用 MG 叠加或 `filters.borderRadius`）

### 大字动效（关键词揭示）

当触发类型是"核心论点/关键词"时：
- 用 `submit_motion_graphic` 生成关键词大字揭示动效
- 放在演讲者画面上方，避开脸部区域
- 时长约 2–3 秒，与讲话高度对齐

### 数字/统计卡片

当触发类型是"数字/统计/百分比"时：
- 用 `submit_motion_graphic` 生成包含实际数字的卡片动效
- 放在演讲者对侧（演讲者在左则卡片在右）
- 时长与讲话数字相关的句子对齐

### 信息列表条

当触发类型是"列表项/步骤"时：
- 为每一项生成一个列表条 MG，随讲话依次出现
- 或者生成整张列表卡，在这个话题段全程显示

---

## 第五步：字幕

```
edit_captions action=enable
edit_captions action=template templatePreset=netflix     （中文内容）
edit_captions action=layout json={"preset":"bottom-center"}
```

英文内容用 `studio` 或 `submagic` 模板。

---

## 第六步：背景音乐

在效果全部放置完毕后再处理 BGM，避免 BGM 时长与视频不一致。

- 从 `list_audio` 里选匹配内容情绪的曲目
- 用 `edit_track` 设置 music track `role=follower`，主视频轨 `role=anchor`，启用自动闪避
- BGM 音量低于主讲声 15-20dB，讲话时自动闪避
- 淡入 1-2 秒，淡出 2-3 秒

---

## 第七步：QA 核查

```
view_timeline_frames  → 在每个 MG 放置的帧截图，核查：
```
- MG 没有遮挡演讲者的脸和手
- 字幕区域（底部 15%）没有被其他内容覆盖
- 效果时机与讲话内容一致（不超前也不滞后）
- 相邻效果形式有变化（不单调）

发现问题：用 `set_item_timing` / `move_item` / `edit_item` 修正，无需重新生成。

---

## 参考文件

- [内容分析规则](references/content-analysis.md) — 如何从文字稿提取视觉触发点
- [视觉决策规则](references/visual-decisions.md) — 不同触发类型对应什么效果、如何生成 prompt
