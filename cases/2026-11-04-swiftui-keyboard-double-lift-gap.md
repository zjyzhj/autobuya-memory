---
date: 2026-11-04
area: layout
project: "Barwise"
platform: ios
status: fixed
title: SwiftUI 输入框被键盘顶高中间留出空隙
symptom: 键盘弹出后输入框悬在键盘上方，中间露出一截页面内容
fix: 底栏已在 safeAreaInset 里时不要再加键盘高度 padding；键盘出现只把 Tab 栏高度收成 0，让输入条贴住键盘
error_signatures:
  - "safeAreaInset(edge: .bottom"
  - "keyboardWillChangeFrameNotification"
  - "liftAboveHomeIndicator"
tags: [键盘, 输入框, 空隙, 顶起, 贴键盘, safeAreaInset, 底部栏, 遮挡]
---

## 背景

自定义底部输入条加 Tab 栏时，常见做法是把整条 chrome 放进 `safeAreaInset(edge: .bottom)`，再自己监听键盘高度，给 chrome 加一段 `padding(.bottom, keyboardHeight - homeIndicator)`，想让输入框贴住键盘。

## 现象

1. 点输入框后键盘正常弹出，底部 Tab 栏会收起。
2. 输入框没有贴在键盘顶边，中间空出一条，能看到底下的页面内容。
3. 收起键盘后，输入框和 Tab 栏的静止位置是对的。

## 定位

根因是 **键盘避让做了两次**：

- `safeAreaInset(edge: .bottom)` 的内容本来就布局在当前底部安全区上。键盘弹出后，系统会把这块 inset 抬到键盘上方。
- 同一块 chrome 又加了大约「键盘高度 − Home Indicator」的手动 padding。
- 第二次抬升把输入框再往上推一截，键盘和输入框之间就露出页面。

这和「底栏完全没避让、输入框被键盘盖住」不是同一条根因。那次是键盘只顶起了外层 Tab 栏、内层输入条没跟着动。本条只覆盖：**inset 已经贴着键盘，又手动加了一段键盘高度**。

## 修复

键盘可见时只把 Tab 栏高度收成 0，不要再给 chrome 加键盘高度 padding：

```swift
VStack(spacing: 0) {
    inputBar
    tabBar
        .frame(height: keyboard.isVisible ? 0 : tabBarHeight)
        .clipped()
}
// 不要再写 .padding(.bottom, keyboardHeight - homeIndicator)
```

把这段放在页面 `NavigationStack` 内的 `safeAreaInset(edge: .bottom)` 里，让系统只做一次键盘避让。

## 适用边界

适用：

- 自定义输入条已经挂在 `safeAreaInset(edge: .bottom)`，键盘弹出后输入框离键盘还有一条空隙。
- 同一条 chrome 里还有一段 Tab 栏，希望键盘出现时只留输入条。

不适用：

- 输入框还在屏幕底部、被键盘直接盖住：那时 inset 没有跟着键盘走，需要先把输入条放进底部 inset，而不是删掉抬升。
- 父级 `safeAreaInset` 进不了 `NavigationStack` + `GeometryReader`，导致按钮被底栏盖住。那是另一条根因，见 `cases/2026-10-20-swiftui-safeareainset-navstack-bottom-occlusion.md`。
- 系统 `TextField` 在普通 `ScrollView` 里被键盘挡住，且没有自定义底栏。

## 复盘（可迁移的规则）

**已经活在底部安全区里的输入条，不要再按键盘高度手动垫一层。** 先确认系统有没有把 inset 抬到键盘上；已经抬了就只收起不该跟着走的 Tab 栏。看到「输入框和键盘中间露内容」，优先查是不是键盘高度被加了两次。

## 提交前检查

- 根因已定位，不是猜测。
- 标题或 symptom 能用 2 字通用中文词搜到（键盘 / 输入框 / 空隙）。
- `fix` 是一句话可复用修复。
- tags 带通用词和同义词。
- 已查重：2026-10-20 案例是底栏遮挡内容，根因不同，未合并。
- 已脱敏。
