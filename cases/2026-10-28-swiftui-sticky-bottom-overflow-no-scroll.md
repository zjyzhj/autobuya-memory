---
date: 2026-10-28
area: layout
project: "Swingtempo"
platform: ios
status: fixed
title: SwiftUI 粘性底栏下满高不可滚动内容溢出遮挡且无法滑动
symptom: 底部粘性栏盖住主操作与次要控件，页面无法滑动露出被挡内容
fix: 满高内容改 ScrollView，粘性底栏用 overlay，内容显式 padding.bottom 预留底栏高度
error_signatures:
  - "safeAreaInset(edge: .bottom"
  - "GeometryReader"
  - "ScrollView"
  - ".overlay(alignment: .bottom)"
  - "padding(.bottom"
  - ".toolbar(.hidden, for: .tabBar)"
tags: [遮挡, 重叠, 底部, 无法滑动, 滚动, ScrollView, overlay, safeAreaInset, NavigationStack, GeometryReader, 粘性底栏, occlusion]
---

## 背景

练习页、教练条一类 UI 常在 `NavigationStack` 子页挂粘性底栏（Coach / Tab / CTA）。内容区若用满高 `GeometryReader` + `VStack` + `Spacer` 排布，且没有滚动容器，控件一多就会和底栏抢同一段垂直空间。

与 `cases/2026-10-20-swiftui-safeareainset-navstack-bottom-occlusion.md`（仅靠 `safeAreaInsets.bottom` clearance）互补：那条解决「inset 未上报」；本条解决「内容过高且不可滚动，clearance 不够用」。

## 现象

1. 粘性底栏（如 Rhythm coach）浮在屏幕底部。
2. 底栏上方的主操作、语音入口、「结束并保存」等半截或整块被盖住。
3. 即使用户想滑动，页面也不能滚，被挡内容永久不可达。
4. 仅隐藏系统 TabBar、或只给 `GeometryReader` 加 `bottomClearance` 后，遮挡仍在。

## 定位

主根因是 **满高不可滚动布局在粘性底栏上方溢出**：

- 子页：`NavigationStack` → `GeometryReader` → `VStack`（顶部信息 + 大环 + 多行控件）+ 少量 `padding(.bottom)`。
- 同时用 `safeAreaInset(edge: .bottom)` 挂 CoachBar：栏会画出来，但在部分嵌套下**并不可靠地缩小可排布/可滚动区域**。
- 内容高度超过「底栏以上」可用高度时，`Spacer` 压到 `minLength` 后仍溢出，视觉落在底栏背后。
- 没有 `ScrollView` 时，用户无法把被挡控件滑上来。

因此「只加 clearance」在内容本就过高时不够；必须让内容可滚动，并显式为粘性栏留白。

## 修复

```swift
ZStack {
    Background().ignoresSafeArea()
    ScrollView(.vertical, showsIndicators: false) {
        VStack(spacing: 0) {
            // top / hero / controls …
            controlDeck
                .padding(.bottom, 108) // ≈ 粘性底栏视觉高度
        }
        .frame(maxWidth: .infinity)
    }
}
.overlay(alignment: .bottom) {
    StickyBottomBar() // 如 CoachBarView
}
.toolbar(.hidden, for: .tabBar) // 专注页可去掉系统 Tab 再抢一层
```

要点：

1. 主列用 `ScrollView`，保证内容过高时可滑到 End / 次要控件。
2. 粘性栏优先 `overlay(alignment: .bottom)`，不要假设 `safeAreaInset` 一定让出滚动区。
3. 滚动内容底部用固定 `padding` 预留栏高（按实测，约 88–120），避免滑到底仍被盖。
4. 若同屏还有系统 TabBar，专注流程页可 `.toolbar(.hidden, for: .tabBar)`，减少双底栏。

## 适用边界

适用：

- SwiftUI 子页有粘性底栏（Coach / 自定义 CTA），内容为满高 `GeometryReader` / 固定 `VStack`。
- 验收出现「底部遮挡」且「无法滑动 / 滑不动」。
- 已按 clearance 案例加过 inset 判断，遮挡仍在。

不适用：

- 内容本就在 `ScrollView` 内且 `safeAreaInset` 已正确抬高 content inset、可滑到全部控件。
- 仅键盘顶起、sheet / fullScreenCover 内遮挡。
- 只缺一小段 gap、内容高度明显小于可用区：可先试 clearance 案例，不必强行改滚动结构。

## 复盘（可迁移的规则）

**粘性底栏 + 满高不可滚动列 = 溢出时必遮挡且用户无自救。** 看见「底部遮挡 + 无法滑动」时：先确认有没有 `ScrollView`；没有就改滚动，并用 `overlay` + 显式底 padding 给粘性栏留位。不要在 clearance 失败后反复调魔法数字而不改滚动模型。与「inset 未上报」案例分工：先问能不能滑，再问 inset 数字够不够。

## 提交前检查

- 根因已定位，不是猜测。
- 标题或 symptom 能用 2 字通用中文词搜到（遮挡 / 滑动 / 底部）。
- `fix` 是一句话可复用修复。
- tags 带通用词和同义词。
- 已查重：与 2026-10-20 clearance 案例相关但主根因不同（不可滚动溢出），故新建。
- 已脱敏。
