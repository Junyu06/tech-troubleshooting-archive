plex ios ass subtitles render as square (tofu characters)
# Plex iOS ASS subtitles render as squares (tofu characters)

## TL;DR (Final Conclusion)

**This is NOT a font, encoding, Docker, or fontconfig problem.**

The real root cause is:

> **Plex does not burn ASS subtitles by default, and iOS Plex cannot properly render ASS subtitles when they are Direct Streamed (`subtitleDecision="copy"`).**

As a result:
- ASS subtitles are sent raw to the iOS client
- iOS cannot render them correctly
- Chinese characters become □□□ (tofu)
- Subtitle outlines remain sharp because they are client-rendered overlays

---

## Symptoms

- iOS Plex shows ASS subtitles as □□□
- Outlines / colors / positioning look correct
- Text itself is unreadable
- Happens even when:
  - Fonts exist (Noto CJK installed)
  - `fontconfig` is correctly configured
  - ASS `Encoding` is UTF-8 (`1`)
  - ASS `Style` font is valid
- Clearing Plex transcoder cache does **not** help

---

## Misleading Observations (Important)

### Why subtitles look *too sharp*
If subtitles were burned into video, they would blur together with low-bitrate video.

Instead:
- Video looks blurry
- Subtitle edges remain extremely sharp

This proves:
> **Subtitles are NOT burned into video. They are rendered client-side.**

---

## The Actual Cause

From Plex Transcoder logs:

```
subtitleDecision="copy"
```

You can confirm this quickly from the server logs:

```bash
LOG_DIR="/config/Library/Application Support/Plex Media Server/Logs"
grep -RIn --line-number 'subtitleDecision=' "$LOG_DIR"/Plex\ Transcoder\ Statistics*.log | tail -n 20
```

This means:
- Plex decided to *Direct Stream* ASS subtitles
- No server-side rendering (libass) occurred
- iOS client attempted to render ASS → failed → tofu characters

iOS Plex **does not support ASS rendering**, even though Plex may incorrectly assume it does.

---

## Why All Server-Side Fixes Appeared Useless

All of the following only apply **if subtitles are burned server-side**:

- Installing Noto CJK fonts
- Fixing `fontconfig`
- Correcting ASS `Encoding`
- Cleaning caches
- Editing `Style` font names

Because Plex never entered the burn-in pipeline, **none of these changes were actually used**.

---

## The Real Fix (Verified Working)

### ✅ Enable subtitle burn-in on the **iOS Plex client**

On iOS Plex:

```
Settings → Player → Subtitles → Burn Subtitles → Always
```

After enabling this:
- Plex is forced to render ASS subtitles server-side
- `subtitleDecision` becomes `burn`
- libass + fontconfig + Noto CJK are used
- Chinese subtitles render correctly
- Even ASS files without explicit fonts work correctly

### Optional: Quick CLI checks (Docker / Bash)

#### 1) Verify ASS is actually being burned (look for `subtitleDecision="burn"`)
```bash
LOG_DIR="/config/Library/Application Support/Plex Media Server/Logs"
grep -RIn --line-number 'subtitleDecision=' "$LOG_DIR"/Plex\ Transcoder\ Statistics*.log | tail -n 50
```

Expected:
- Before the fix: `subtitleDecision="copy"`
- After enabling **Burn Subtitles = Always** on iOS: `subtitleDecision="burn"`

#### 2) Verify the Plex user can see CJK fonts (inside the container)
```bash
id plex
su -s /bin/sh plex -c 'fc-list | grep -i -E "noto.*cjk|source han" | head -n 20'
```

If you see entries like `Noto Sans CJK ...`, fonts are visible to Plex.

#### 3) (Optional) Clear transcoder cache (if you want a clean test run)
```bash
rm -rf "/config/Library/Application Support/Plex Media Server/Cache/Transcode"/*
rm -rf "/config/Library/Application Support/Plex Media Server/Cache/PhotoTranscoder"/*
```

Then restart the Plex container and retry playback.

---

## Why the Server UI Does NOT Have a “Force Burn ASS” Option

Older Plex versions had global burn controls.

Modern Plex behavior:
- Server no longer forces burn for text subtitles
- Client declares subtitle capability
- Server trusts client (incorrectly for ASS on iOS)

Therefore:
> **The burn decision must be made on the client.**

---

## Recommended Final Setup

### Server
- Hardware transcoding enabled
- Noto CJK fonts installed
- fontconfig correctly configured
- No special subtitle hacks required

### iOS Client
- **Burn Subtitles = Always**

### Other Clients
- Desktop Plex: Direct Play ASS (works fine)
- Web Plex: Works or burns as needed

---

## Key Takeaway

> **Plex + iOS + ASS subtitles do NOT fail because of fonts.  
They fail because ASS must be burned, and Plex will not do so unless the iOS client explicitly requests it.**

This issue is resolved entirely by forcing burn-in on iOS.

---


## 中文版总结（完整翻译）

### 最终结论（TL;DR）

**这不是字体问题，也不是编码、Docker、fontconfig 或 Plex 转码配置的问题。**

真正的根因是：

> **Plex 默认不会对 ASS 字幕进行烧录（burn-in），而 iOS 版 Plex 客户端本身无法正确渲染 ASS 字幕。当字幕被 Direct Stream（`subtitleDecision="copy"`）时，就一定会出问题。**

结果就是：

- ASS 字幕被“原样”发送到 iOS 客户端
- iOS 无法正确渲染 ASS
- 中文字符显示为方块（□□□）
- 字幕描边、颜色、位置依然正确（因为是客户端叠加渲染）

---

### 典型现象

- 仅在 **iOS Plex** 上出现 ASS 字幕方块字
- 字幕样式（描边、颜色、位置）正常
- 字幕文字内容不可读
- 即使已经：
  - 正确安装了 Noto CJK 字体
  - 正确配置了 fontconfig
  - ASS 文件是 UTF-8 编码（Encoding = 1）
  - 清空了 Plex 转码缓存  
  **问题仍然存在**

---

### 一个非常具有误导性的现象

#### 为什么字幕“看起来特别清晰”？

如果字幕是被真正烧录进视频里的：

- 视频码率低 → 视频会糊
- 字幕也应该一起变糊

但实际看到的是：

- 视频画面很糊
- 字幕边缘却异常清晰

这恰恰说明：

> **字幕并没有被烧进视频，而是由客户端单独渲染的叠加层。**

---

### 真正的原因是什么？

在 Plex 的转码统计日志中可以看到：

```
subtitleDecision="copy"
```

这代表：

- Plex 选择了 **Direct Stream** 字幕
- 没有在服务器端使用 libass 渲染字幕
- ASS 字幕被直接交给 iOS 客户端
- iOS 客户端无法渲染 ASS → 中文变成方块

**iOS Plex 实际上并不支持 ASS 字幕渲染，但 Plex 错误地认为可以。**

---

### 为什么之前所有“服务器端修复”都看起来没有效果？

以下操作 **只有在字幕被服务器端烧录时才会生效**：

- 安装 Noto CJK 字体
- 修复 fontconfig
- 修正 ASS 的 Encoding
- 清理缓存
- 修改字幕 Style 中的字体名

但由于 Plex **从未进入 burn-in 管线**，  
所以这些修复 **在之前根本没有被用到**。

---

### 真正有效的解决方案（已验证）

#### ✅ 在 iOS Plex 客户端中强制字幕烧录

在 iOS Plex 客户端中：

```
设置 → 播放器 → 字幕 → Burn Subtitles → Always
```

开启后：

- iOS 客户端会明确告知服务器：**不接受 copy 的字幕**
- Plex 被迫在服务器端烧录 ASS
- `subtitleDecision` 变为 `burn`
- libass + fontconfig + Noto CJK 全部生效
- 中文字幕正常显示
- **即使 ASS 文件中没有显式指定字体，也可以正常显示中文**

---

### 为什么服务器端 UI 里没有“强制烧录 ASS”的选项？

这是 Plex 的设计变化：

- 早期 Plex 服务器端存在全局字幕烧录选项
- 新版本中：
  - 服务器不再强制决定是否烧录文本字幕
  - 客户端声明自己的字幕能力
  - 服务器“信任”客户端（但对 iOS + ASS 来说是错误的）

因此：

> **是否烧录 ASS 字幕，必须由客户端来触发。**

---

### 推荐的最终配置

#### 服务器端
- 开启硬件转码
- 安装 Noto CJK 字体
- 正确配置 fontconfig
- 无需额外字幕 Hack

#### iOS 客户端
- **Burn Subtitles = Always**

#### 其他客户端
- 桌面版 Plex：可 Direct Play ASS（正常）
- Web Plex：按需 burn 或 direct

---

### 最重要的一句话总结

> **Plex + iOS + ASS 字幕出问题，并不是因为“字体不行”，  
而是因为 ASS 必须被烧录，而 Plex 只有在 iOS 客户端明确要求时才会这么做。**

**在 iOS 上强制开启字幕烧录，即可彻底解决该问题。**