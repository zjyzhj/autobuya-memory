---
date: 2026-10-21
area: functional
project: "DozeClock"
platform: ios
status: fixed
title: 设置里点协议链接没有反应
symptom: 设置或同意页里点隐私协议、用户协议、服务商政策，没有任何页面出现
fix: 协议页的 sheet 挂在当前已经弹出的设置页或同意页上，不要挂在正在展示这张页的父视图上
error_signatures:
  - "sheet(item:)"
  - "sheet(isPresented:)"
  - "legalDocument"
tags:
  - 协议
  - 隐私
  - 点击
  - 没反应
  - 弹窗
  - sheet
  - 设置
---

## 背景

iOS SwiftUI 应用的设置页和同意页都是从根视图 `.sheet` 弹出的。隐私协议、用户协议、服务商政策本来也是一张页面，用根视图上的另一个 `.sheet(item:)` 打开。

## 现象

1. 用户打开设置，点 Privacy、Terms 或服务商政策。
2. 同意页上的协议链接同样点了没有页面。
3. 没有报错、没有跳转、按钮看起来点过了但界面不变。
4. 协议页的加载代码是好的，单独从没有其他 sheet 的根视图打开时可以出现。

## 定位

根视图已经在 present 设置或同意这张 sheet。协议点击只是把 `legalDocument` 设上，再让同一个根视图 present 第二张 sheet。SwiftUI 一个 presenter 同时只能有一张 sheet，后一次 present 被丢掉，状态变了，用户看不到页面。

这和「sheet 内容刚 `onAppear` 时再弹嵌套同意页」不是同一条触发。那条见 `cases/2026-10-21-ai-consent-nested-sheet-skipped.md`。那条的修复是改成同层覆盖，并且不要在看到说明前 `grant()`。把协议页改挂回「还没 present 的父级」会正好踩中本案例。

## 修复

协议链接用设置页、同意页自己的 `@State` 记录要打开的文档，`.sheet(item:)` 挂在这张已经弹出的页面上。用户点击时由当前页再 present 协议页。根视图不再为协议挂第二张 sheet。

## 适用边界

适用：

- 设置、同意或其他已经是 sheet / fullScreenCover 的页面里，有一个链接、按钮要点开另一页。
- 目标页的 `.sheet` 写在弹出当前页的那个父视图上，点击后状态变了但没有界面。

不适用：

- 当前页刚出现就在 `onAppear` 里再弹一张 sheet（用同意弹窗那条案例，嵌套 sheet 在出现动画期间会被跳过）。
- 按钮被遮罩吃掉点击，或按钮根本没进可点区域。
- 页面出现了但网页加载失败。那是链接或 WebView 的问题，不是 present 被丢掉。

## 复盘（可迁移的规则）

已经弹出的页面里，再打开的页面要由这张页面自己 present。不要把第二张 sheet 挂在正在展示第一张 sheet 的父视图上。父视图上的第二次 present 会静默失败。

## 提交前检查

- 根因已定位，不是猜测。
- 标题或 symptom 能用 2 字通用中文词搜到（协议、点击、没反应）。
- `fix` 是一句话可复用修复。
- tags 带通用词和同义词。
- 已查重：与「打开 AI 没有同意弹窗」相邻但触发和修复不同，未合并。
- 已脱敏。
