---
date: 2026-10-28
area: layout
project: "Swingtempo"
platform: ios
status: fixed
title: SwiftUI 粘性底栏用 overlay 不占高导致内容仍被遮挡
symptom: 底部粘性栏盖住主操作与次要控件；加 ScrollView 与底 padding 后首屏仍遮挡、难滑出
fix: 粘性底栏放进 VStack 布局流（上 ScrollView、下底栏），禁止用 overlay/不可靠 inset 盖在全屏内容上
error_signatures:
  - ".overlay(alignment: .bottom)"
  - "safeAreaInset(edge: .bottom"
  - "ScrollView"
  - "GeometryReader"
  - "CoachBarView"
  - ".toolbar(.hidden, for: .tabBar)"
tags: [遮挡, 重叠, 底部, 无法滑动, 滚动, ScrollView, overlay, 布局流, VStack, safeAreaInset, 粘性底栏, occlusion]
---

## 背景

练习页等场景常在内容上方保留粘性底栏（Coach / CTA）。若用 `.overlay(alignment: .bottom)` 或不可靠的 `safeAreaInset` 挂栏，栏会画出来，但**不缩小**上方 `ScrollView` / `GeometryReader` 的排布高度，内容仍按全屏高度排布，底部落在栏背后。

与 `cases/2026-10-20-swiftui-safeareainset-navstack-bottom-occlusion.md`（inset 未上报 → clearance）互补：那条解决「没收到 inset 数字」；本条解决「栏不占布局高度，clearance / overlay padding 治标不治本」。

曾错误沉淀「ScrollView + overlay + padding.bottom」为终态；实测首屏 voice cue 仍被 Rhythm coach 压住。已关闭对应 PR，以本条为准。

## 现象

1. 粘性底栏浮在屏幕底部，盖住「Play recorded cue」「End session」等。
2. 仅 `bottomClearance(safeAreaInsets)` 不够；内容过高时仍溢出。
3. 改成 `ScrollView` + `.overlay(alignment: .bottom) { Bar }` + 大 `padding.bottom` 后，**首屏仍遮挡**；用户感觉无法滑出或滑不动。
4. 隐藏系统 TabBar 后只剩一层底栏，遮挡依旧。

## 定位

主根因是 **粘性底栏未进入布局流，不占垂直高度**：

- `overlay` / 失效的 `safeAreaInset`：栏在 Z 轴盖住内容，ScrollView 视口仍是全屏。
- 内容底部控件落在视口下沿 = 栏的矩形内；靠多加 padding 指望「滑一下」不可靠（首屏观感坏、小屏可滚距离不足、命中区仍在栏下）。
- 满高 `GeometryReader` + 不可滚动 `VStack` 会加重溢出，但换成 ScrollView 后若栏仍是 overlay，遮挡主因未消。

## 修复

把粘性底栏放进 **正常布局流**，让滚动区高度 = 栏以上剩余空间：

```swift
ZStack {
    Background().ignoresSafeArea()

    VStack(spacing: 0) {
        ScrollView(.vertical, showsIndicators: true) {
            VStack(spacing: 0) {
                // top / hero / controls …
            }
            .frame(maxWidth: .infinity)
            .padding(.bottom, 20)
        }

        StickyBottomBar() // CoachBar 等：参与布局，占高度
    }
}
.toolbar(.hidden, for: .tabBar) // 专注页可去掉系统 Tab，避免双底栏
```

要点：

1. **上 ScrollView、下底栏** 同一 `VStack`，禁止用 overlay 盖全屏内容。
2. 内容过高时只在栏**上方**区域内滚动，控件不会画进栏的矩形。
3. 不要假设 `safeAreaInset` 在 `NavigationStack` 下一定让出滚动区；占位以布局流为准。
4. 专注流程页可隐藏系统 TabBar，减少再叠一层 chrome。

## 适用边界

适用：

- SwiftUI 粘性底栏 + 主列内容出现「底部遮挡」。
- 已试过 clearance / overlay+padding，首屏仍挡或无法滑出。
- 验收要求底栏上方控件首屏可读、可点。

不适用：

- 底栏本就在 `VStack`/`safeAreaInset` 中且内容区高度已正确缩小、无遮挡。
- 仅键盘顶起、sheet 内遮挡。
- 只要微调 gap、内容明显矮于可用区：可先试 2026-10-20 clearance 案例。

## 复盘（可迁移的规则）

**粘性底栏必须占布局高度；overlay 只负责画，不负责让位。** 见「底部遮挡」时先问：栏是在 `VStack` 里还是 `overlay`/`inset` 盖上的？盖上的就改布局流（上滚下栏）。不要在错误结构上反复加 `padding.bottom` 魔法数字。与 clearance 案例分工：先保证栏占位，再必要时补 inset 数字兜底。

## 提交前检查

- 根因已定位，不是猜测。
- 标题或 symptom 能用 2 字通用中文词搜到（遮挡 / 底部 / 滑动）。
- `fix` 是一句话可复用修复。
- tags 带通用词和同义词。
- 已查重：与 2026-10-20 不同根因；此前错误 overlay 案例 PR 已关闭，不保留错误 fix。
- 已脱敏。
