# 视频首屏封面实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 从 `demo-final.mov` 开头选出第一个稳定画面，作为视频进入页面时立即显示的封面。

**Architecture:** 使用 macOS AVFoundation 从视频开头导出多个候选帧，人工检查后保留最早的清晰稳定帧为 `video-poster.jpg`。HTML 仅修改 `<video>` 的 `poster` 引用，保留现有播放控件、预加载策略和交互。

**Tech Stack:** HTML5 Video、macOS AVFoundation、Swift、GitHub Pages

---

### Task 1: 导出视频开头候选帧

**Files:**
- Create temporarily: `/tmp/extract-video-frame.swift`
- Read: `demo-final.mov`
- Create temporarily: `/tmp/video-poster-candidates/frame-*.jpg`

- [ ] **Step 1: 创建临时截帧脚本**

```swift
import AVFoundation
import AppKit

let arguments = CommandLine.arguments
guard arguments.count == 4, let seconds = Double(arguments[3]) else {
    fputs("usage: extract-video-frame input.mov output.jpg seconds\n", stderr)
    exit(2)
}

let asset = AVURLAsset(url: URL(fileURLWithPath: arguments[1]))
let generator = AVAssetImageGenerator(asset: asset)
generator.appliesPreferredTrackTransform = true
generator.requestedTimeToleranceBefore = .zero
generator.requestedTimeToleranceAfter = .zero

let time = CMTime(seconds: seconds, preferredTimescale: 600)
let cgImage = try generator.copyCGImage(at: time, actualTime: nil)
let bitmap = NSBitmapImageRep(cgImage: cgImage)
guard let data = bitmap.representation(using: .jpeg, properties: [.compressionFactor: 0.9]) else {
    fputs("failed to encode JPEG\n", stderr)
    exit(3)
}
try data.write(to: URL(fileURLWithPath: arguments[2]))
```

- [ ] **Step 2: 导出开头五个候选画面**

Run:

```bash
mkdir -p /tmp/video-poster-candidates
for second in 0.25 0.50 0.75 1.00 1.50; do
  swift /tmp/extract-video-frame.swift demo-final.mov "/tmp/video-poster-candidates/frame-${second}.jpg" "$second"
done
```

Expected: `/tmp/video-poster-candidates/` 中出现 5 张可打开的 JPEG 图片。

- [ ] **Step 3: 检查候选帧**

依次查看 0.25、0.50、0.75、1.00、1.50 秒画面，选择最早满足以下条件的帧：不是黑屏、主体画面完整、动画没有明显拖影或过渡态。

Expected: 明确选中唯一一张最早稳定帧。

### Task 2: 添加封面资源并更新 HTML

**Files:**
- Create: `video-poster.jpg`
- Modify: `index.html:2015`

- [ ] **Step 1: 将选中的候选帧保存为正式封面**

Run，并在交互菜单中输入 Task 1 视觉检查所选图片对应的序号：

```bash
select selected_frame in \
  /tmp/video-poster-candidates/frame-0.25.jpg \
  /tmp/video-poster-candidates/frame-0.50.jpg \
  /tmp/video-poster-candidates/frame-0.75.jpg \
  /tmp/video-poster-candidates/frame-1.00.jpg \
  /tmp/video-poster-candidates/frame-1.50.jpg; do
  test -n "$selected_frame" && cp "$selected_frame" video-poster.jpg && break
done
file video-poster.jpg
```

Expected: `file` 输出包含 `JPEG image data`，且文件名为 `video-poster.jpg`。

- [ ] **Step 2: 更新视频封面引用**

将：

```html
<video controls preload="metadata" poster="./plan-1-final.png">
```

替换为：

```html
<video controls preload="metadata" poster="./video-poster.jpg">
```

- [ ] **Step 3: 检查引用与差异**

Run:

```bash
rg -n 'video-poster|plan-1-final|demo-final' index.html
git diff --check
git status --short
```

Expected: `index.html` 仅引用 `./video-poster.jpg` 和 `./demo-final.mov`；没有 `plan-1-final.png` 引用；状态中只出现本次封面与 HTML 变更，以及用户原有的无关变更。

### Task 3: 本地验证首屏画面

**Files:**
- Verify: `index.html`
- Verify: `video-poster.jpg`

- [ ] **Step 1: 启动本地静态服务器**

Run:

```bash
python3 -m http.server 4173
```

Expected: 服务器在 `http://127.0.0.1:4173/` 提供当前页面。

- [ ] **Step 2: 在浏览器验证**

打开 `http://127.0.0.1:4173/`，滚动到 Video Demo：未点击播放时应显示新封面；点击播放后视频与声音正常，原生控制条可用。

- [ ] **Step 3: 检查资源引用完整性**

Run:

```bash
test -f video-poster.jpg
test -f demo-final.mov
```

Expected: 两条命令均以状态码 0 完成。

### Task 4: 提交并发布

**Files:**
- Stage: `index.html`
- Stage: `video-poster.jpg`

- [ ] **Step 1: 只暂存本次实现文件**

Run:

```bash
git add -- index.html video-poster.jpg
git diff --cached --stat
```

Expected: 暂存区仅包含 `index.html` 和 `video-poster.jpg`。

- [ ] **Step 2: 提交实现**

Run:

```bash
git commit -m "Add video preview poster"
```

Expected: 生成一个只包含封面资源和 HTML 引用的提交。

- [ ] **Step 3: 推送到 GitHub Pages 来源分支**

Run:

```bash
git push origin main
```

Expected: 远端 `main` 更新到本地最新提交。

- [ ] **Step 4: 验证线上资源**

Run:

```bash
curl -I -L --max-time 30 https://shifangyu0315.github.io/ai-review-showcase/video-poster.jpg
```

Expected: HTTP 200，`content-type` 为 `image/jpeg`。重新打开线上页面时，视频播放前立即显示该封面。
