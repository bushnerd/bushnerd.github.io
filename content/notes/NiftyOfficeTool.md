---
publish: true
created: 2025-06-08T12:56:01.000Z
modified: 2026-10-16T18:10:48.953Z
---

在日常中经常要用到office三件套，但是有些功能并不完善，因此添加了一些小功能，增强易用性。

# Req

## 重复上一步的动作

似乎这样会更好用

## 查找替换颜色

针对一段话中的重点文本，批量替换成某些颜色
设计一个查找替换窗口，增加按字体，颜色查找替换

## 提示词生成图标PPT

# formatCopy

PPT中对象的格式Copy

## 作用

### 文本框

复制位置，复制大小的话有时候导致文本无空间显示

### 形状

复制大小和位置

## 快捷键设计

复制：`Alt + Shift + Q`
粘贴：`Alt + Shift + W`

### 已有格式复制

复制格式：`Ctrl + Shift + C`
粘贴格式：`Ctrl + Shift + V`

#### 作用

文本格式：字体类型、大小、颜色、粗体、斜体、下划线等。
段落格式：对齐方式、行距、段前段后间距、项目符号和编号等。
形状格式：填充颜色、边框颜色、线条样式、阴影效果等。
图片格式：边框、阴影、亮度、对比度等调整。
动画效果：进入、退出、强调和路径动画等。

### bug

快捷键设计很复杂，还没搞定。

# Install

## 安装 reg 注册表文件

双击 utils 文件夹内的两个 reg 文件，选择安装。

```
32.reg
64.reg
```

规避 Windows 的安全审查机制。因为没有购买微软开发者证书，编译的软件不能直接安装，导入上述两个 reg 注册表项之后，将可自由安装。

## 安装 vsto 文件

下载最新版 release，然后解压压缩包。双击其中的`.vsto`文件即可安装。

> 注意：此步骤必须保证软件处于关闭状态。

# Ref

## LittleNewton/Equation\_and\_Codebox

这个项目中也遇到了分发的问题，并且参考他的已经解决。
[GitHub - LittleNewton/Equation\_and\_Codebox: Microsoft Word VSTO Add-In，可以插入带编号的公式和代码](https://github.com/LittleNewton/Equation_and_Codebox)
[不是有效的加载项 · Issue #1 · LittleNewton/Equation\_and\_Codebox · GitHub](https://github.com/LittleNewton/Equation_and_Codebox/issues/1)
