---
title: Xcode
categories: 工具
---

## 1、整行上下移动

Xcode 自带的配置文件路径：/Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Versions/A/Resources/IDETextKeyBindingSet.plist，用文本编辑 IDETextKeyBindingSet.plist，并添加以下代码：

```oc
	<key>GDI Commands</key>
	<dict>
        <key>GDI Duplicate Current Line</key>
        <string>selectLine:, copy:, moveToEndOfLine:,insertNewline:, paste:, deleteBackward:</string>
        <key>GDI Delete Current Line</key>
        <string>moveToEndOfLine:, deleteToBeginningOfLine:,deleteBackward:,moveDown:,moveToEndOfLine:</string>
        <key>GDI Move Current Line Up</key>
        <string>selectLine:, cut:, moveUp:, moveToBeginningOfLine:, insertNewLine:, paste:, moveBackward:</string>
        <key>GDI Move Current Line Down</key>
        <string>selectLine:, cut:, moveDown:, moveToBeginningOfLine:, insertNewLine:, paste:, moveBackward:</string>
        <key>GDI Insert Line Above</key>
        <string>moveUp:, moveToEndOfLine:, insertNewline:</string>
        <key>GDI Insert Line Below</key>
        <string>moveToEndOfLine:, insertNewline:</string>
    </dict>
```

<font color=#cc0000>注意</font>：Xcode.app 需要换成实际的应用名，如 Xcode10.1.app。

详细文章：[xcode 设置快捷键 整行上下移动](https://www.cnblogs.com/goodboy-heyang/p/4732365.html)