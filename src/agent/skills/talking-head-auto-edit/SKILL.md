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
- **不遮挡**：截帧确认演讲者脸和手的位置，MG 放置绝对不能覆盖脸部、手部、**字幕安全区**
- **效果不重复**：连续两个效果不使用同一种形式（例如两张数字卡），穿插不同类型
- **配图必须使用**：每段视频至少要有 3 张来自 Unsplash 或 GPT 生图的配图，不允许只有 MG 文字动效

### 字幕安全区（所有 MG 必须遵守）

竖屏（9:16）视频字幕固定在底部居中，占据画面底部约 20% 的区域。所有叠加效果的 `y` 坐标必须保证效果的底边不超过画面高度的 80%（即从顶部算 80% 以上）。

```
可安全放置 MG 的区域：
  - 画面顶部 0–75%（演讲者脸上方）
  - 画面左侧 / 右侧条状区域（不覆盖脸）

绝对禁止区域：
  - 底部 20%（字幕区）
  - 演讲者脸部中心 ±100px 范围
```

在开始生成之前，把方案用简洁文字列给用户确认（仅在不是完全自动运行时）：
> "我计划在 0:03 加标题大字，0:12 加数字卡片，0:28 加列表条，0:45 加 Unsplash 配图…方向对吗？"

---

## 第四步：A-roll 先行（必须）

**在放任何 MG 或配图之前，必须先完成 A-roll 处理。** A-roll 的时间线决定了所有后续效果的帧位置；跳过这步会导致效果和讲话错位。

```
transcribe_track  → 转录主视频得到逐词时间戳
clean_script      → 压缩明显的停顿（>0.8s 压到 0.3s），去除 um/uh/呃/额
```

A-roll 处理完毕后，用 `read_timeline` 重新读取时间线，获取最终的帧时间戳，再进入第五步。

---

## 第五步：效果生成与放置

按决策表逐项执行，参考 references/visual-decisions.md 的生成规则。

### MG 生成原则

**每个效果都基于内容生成，不是套模板。**

`submit_motion_graphic` 的 prompt 必须包含实际内容——把讲话里的关键词、数字、主题写进去。

```
✅ 好的 prompt：
  "大字动效：从左向右扫光揭示关键词'每天复利1%'，白色粗体大字，
  深色半透明背景条，字体大小约画布高度的 25%，放置在画面左侧，
  文字必须自动换行不溢出，3秒"

❌ 坏的 prompt：
  "a text animation motion graphic"
```

**所有 MG 生成的代码必须包含以下防溢出约束**（在 prompt 里明确要求）：

```
- 所有文字容器必须设置 maxWidth: '90%' 或具体像素值
- 文字设置 wordBreak: 'break-word', overflowWrap: 'break-word'
- 绝对定位元素必须设置 right 或 maxWidth 防止超出父容器
- 禁止使用固定 px 宽度大于画布宽度的值
- 最外层容器用 overflow: 'hidden'
```

在 prompt 末尾总是加上：`"确保所有文字在画布内完整显示，不能被裁切或溢出边界"`

### 图片/配图——全屏切出，演讲者完全消失

**配图不叠加在演讲者身上。** 参考视频里所有配图都是让演讲者完全消失，整个画面切换成图片（截图/新闻/图表），字幕仍在底部。

操作方式：
1. 用 `search_photos` 搜索对应内容图片（英文关键词），或 `submit_image` 生成定制图
2. 用 `edit_item` adds 把图片放在 V1 轨道，覆盖对应时间段的演讲者片段
3. `transform.scale=1.0`（全屏），`fadeInSeconds=0.3, fadeOutSeconds=0.3`

只有在有屏录素材时，才做演讲者圆形 PiP 模式（见 references/visual-decisions.md）。

### 大字动效——叠加在演讲者侧面，不在中央

大字放置在画面**左侧或右侧**，极大字体（画面高度的 30-40%），演讲者脸保持可见：
- 演讲者居中时：大字放左侧（`transform.x = -0.3`），或分两侧（左侧中文、右侧英文）
- 字体颜色：红色（强调/冲击感）或白色（轻一些的提示）
- 可叠加小图标贴纸（灯泡、箭头等）放在演讲者头部附近
- 时长 2-4 秒，与讲话关键词同步

大字 MG 生成时：`submit_motion_graphic` 的 prompt 里要指明"放置在画面左侧 1/3 区域"和字体大小占画面高度的比例。

---

## 第六步：字幕（强制固定格式）

字幕必须按以下顺序执行，不允许跳过任何一步：

```
edit_captions action=enable
edit_captions action=template templatePreset=netflix
edit_captions action=style json={"textAlign":"center"}
edit_captions action=layout json={"preset":"bottom-center","offsetYRatio":0}
```

**严禁之后再调字幕位置**——字幕一旦设置好就不要动，MG 效果需要绕开字幕区，而不是让字幕去适应 MG。

英文内容用 `studio` 模板替代 `netflix`，其余步骤相同。

---

## 第七步：背景音乐

所有效果和字幕放置完毕后才处理 BGM。

```
list_audio                    → 选匹配内容情绪的曲目（商务/知识类选轻快但不躁动的）
add_audio                     → 放入 A2 轨道，startFrame=0
edit_track (music track)      → role=follower，fadeIn=1.5，fadeOut=2.5
edit_track (speech track)     → role=anchor
```

BGM clip 的 `durationInFrames` 必须等于主视频时长，不允许超出。

---

## 第八步：QA 核查（强制）

**这步不可省略。** 用 `view_timeline_frames` 在以下帧截图并逐一确认：

检查清单：
- 每个 MG 效果的帧：没有遮挡演讲者的脸和手
- 字幕区域（底部 20%）没有任何 MG 内容压入
- 字幕对齐是居中（`textAlign: center`），不是左对齐
- 相邻两个效果形式不同（不连续出现两张相同类型的卡片）
- 配图（B-roll）至少出现了 3 次

发现任何遮挡或错位：用 `edit_item` 的 `transform.y` 调整，无需重新生成 MG。
发现字幕左对齐：重新执行 `edit_captions action=style json={"textAlign":"center"}` 和 `edit_captions action=layout json={"preset":"bottom-center"}`。

---

## 参考文件

- [内容分析规则](references/content-analysis.md) — 如何从文字稿提取视觉触发点
- [视觉决策规则](references/visual-decisions.md) — 不同触发类型对应什么效果、如何生成 prompt
