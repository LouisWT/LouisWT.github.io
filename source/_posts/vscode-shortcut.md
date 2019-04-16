---
layout: simple-article
title: VSCode 快捷键与配置
categories:
    - VSCode
tags:
    - VSCode
---
工欲善其事必先利其器, 这篇博文主要总结了VSCode 的常用快捷键和插件
<!-- more -->
# 一. 快捷键

## 1. 界面相关
| 用处 | 快捷键 |
| --- | --- |
| 显示控制台 | ctrl + shift + p 或 F1 |
| 快速打开文件 | ctrl + p |
| 关闭当前 tab | ctrl + w |
| 切换当前显示的 tab | alt + 1/2/3...
| 上一个编辑的 tab | alt + left
| 下一个编辑的 tab | alt + right
| 全屏 | F11 (全屏模式下点击 alt 显示最上方的菜单栏)
| 侧边栏是否可见 | ctrl + b
| 打开终端,如果没有会新建 | ctrl + `
| 新建终端 | ctrl + shift + `
| 是否显示下方面板 | ctrl + j


## 2. 编辑相关
| 用处 | 快捷键 |
| --- | --- |
| 块注释（需要手动设置） | ctrl + shift + /
| 行注释 | ctrl + /
| 删除一行 | ctrl + d
| 剪切当前行（选中情况下剪切选中内容） | ctrl + x
| 复制当前行（选中情况下复制选中内容） | ctrl + c
| 粘贴 | ctrl + v
| 向上/下移动行 | alt + up/down
| 向上/下复制行 | alt + shift + up/down
| 在当前行的下一行添加一行 | ctrl + enter
| 在当前行的上一行添加一行 | ctrl + shift + enter
| 折叠一块代码区域 | ctrl + shift + [
| 展开一块代码区域 | ctrl + shift + ]
| 行向前/向后缩进 | ctrl + [ 和 ctrl + ]
| 查找 | ctrl + f
| 查找下一个 | F3
| 查找上一个 | shift + F3
| 替换 | ctrl + h
| 查看文件中对变量的初始引用 | shift + F12
| 弹窗查看变量定义 | alt + F12
| 跳转到变量定义 | F12
| 变量名替换 | 光标位于变量名 + F2


## 3. 光标位置
| 用处 | 快捷键 |
| --- | --- |
| 在上/下一行添加光标（windows 下需要关闭显卡快捷键） | ctrl + alt + up/down
| 光标转到括号 | ctrl + shift + \
| 光标转到第多少行 | ctrl + g
| 选中当前行 | ctrl + l
| 光标到行首 | home
| 光标到行尾 | end
| 光标到文件首 | ctrl + home
| 光标到文件尾 | ctrl + end
| 光标左/右移一个词 | ctrl + left/right

## 4. 插件相关
### 1. Git Project Manager
| 用处 | 快捷键 |
| --- | --- |
| 打开某个 Git 仓库 | ctrl + alt + p
| 新窗口打开某个 Git 仓库 | ctrl + alt + n

### 2. npm
| 用处 | 快捷键 |
| --- | --- |
| 终端运行 npm script 命令 | ctrl + r shift + r

### 3. open in browser
| 用处 | 快捷键 |
| --- | --- |
| 在浏览器打开当前文件 | alt + b

## 5. emmet 入门用法
- E/N 代表一个标签,比如 `div`
- CLASS 代表一个类名,比如 container
- ID 代表一个ID
- ARRT 代表一个元素属性,比如 width
- TEXT 代表文字内容
- NUM 代表一个数字

| 用处 | 快捷写法 |
| --- | --- |
| html:5 | 表示生成一个 HTML5的文档骨架
| E>N | 表示E是N的父元素
| E+N | 表示E是N的同级元素
| E^N | 表示E是N的子元素
| E.CLASS | class 为 CLASS 的E元素
| E#ID | id 为 ID 的E元素
| E[ATTR=foo] | 指定ATTR 属性的 E 元素
| E{TEXT} | 内容是TEXT 的E元素
| E*NUM | 连续生成 NUM 个 E 元素
| E.item$$@STA*NUM | 自动计数,NUM 个E元素的类名分别是item01 item02... (这里假设 STA 是1了)


# 二. 插件列表
## 1. 主题
- One Dark Pro: 颜色主题
- Material Icon Theme: 文件图标主题
- Bracket Pair Colorizer
- Indent-Rainbow: 使用颜色表示缩进的多少，更直观
- TODO Highlight: 高亮 TODO 之类的注释
- Trailing Spaces: 高亮多余的空格

## 2. 代码风格
- Beautify
- ESLint

## 3. 调试
- Debugger for Chrome
- open in browser

## 4. Git
- GitLens
- Git Project Manager
- Git History

## 5. HTML & CSS
- HTML Snippets
- HTML CSS Support
- IntelliSense for CSS class names in HTML: 根据引用的 CSS文件，对HTML的类名进行补全

## 6. 图片
- SVG viewer: 查看 SVG 图片

## 7. Vue
- Vetur
- Vue 2 Snippets

## 8. React
- ES7 React/Redux/GraphQL/React-Native snippets

## 9. MarkDown
- Markdown All in One

## 7. 其他
- npm
- npm Intellisense
- filesize
- Import Cost
- path Intellisense
- RegExp Preview and Editor

# 三. setting.json
```
{
    "editor.tabSize": 2,
    "emmet.includeLanguages": {
        "javascript": "javascriptreact"
    },
    "emmet.triggerExpansionOnTab": true,
    "workbench.iconTheme": "material-icon-theme",
    "window.zoomLevel": 2,
    "git.autofetch": true,
    "terminal.integrated.shell.windows": "C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
    "files.autoSave": "afterDelay",
    "workbench.statusBar.feedback.visible": false,
    "workbench.colorTheme": "One Dark Pro",
    "editor.lineHeight": 22,
    "indentRainbow.colors": [
        "#4040401a",
        "#8080801a",
        "#C0C0C01a",
        "#ffffff1a",
        "#C0C0C01a",
        "#8080801a"
    ],
    "gitProjectManager.baseProjectsFolders": [
        "D:\\WorkSpace\\GitRepo"
    ],
    "editor.suggestSelection": "first",
    "vsintellicode.modify.editor.suggestSelection": "automaticallyOverrodeDefaultValue"
}
```