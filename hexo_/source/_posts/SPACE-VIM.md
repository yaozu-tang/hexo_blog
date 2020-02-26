---
title: SPACE-VIM 安装及配置
date: 2020-02-27 04:21:03
tags: ["neovim", "space-vim", "tabnine"]
categories: Linux日常折腾
description: 受不了vim带了一大堆插件的龟速了 想起之前听到邓佬说有SPACE-VIM这个玩意 心血来潮撸来试试

---

## 安装 NEOVIM

`neovim`是基于vim的一款编辑器 操作与vim大致相同 手感更好

### PPA安装

1. 加入PPA源

   `apt install software-properties-common && add-apt-repository ppa:neovim-ppa/unstable`

2. 安装依赖

   `apt install -y python-dev python-pip python3-dev python3-pip`

3. 安装neovim

   `apt update && apt install -y sudo apt install python3-neovim`

### 源码编译(推荐)

1. 安装依赖

   `apt install ninja-build gettext libtool libtool-bin autoconf automake cmake g++ pkg-config unzip`

   `apt install gperf libluajit-5.1-dev libunibilium-dev libmsgpack-dev libtermkey-dev libvterm-dev libjemalloc-dev lua5.1 lua-lpeg lua-mpack lua-bitop`

   其中`libvterm-dev`可能找不到

   可以在[libvterm](https://launchpad.net/ubuntu/+source/libvterm)找到 下载`.deb`文件安装

   `pip install neovim && pip3 install neovim`

2. 克隆源码

   源码包较大 如果速度慢建议开代理

   `git clone https://github.com/neovim/neovim.git`

3. 编译安装

   `make -j8 && make install`



## 安装 SPACE-VIM

**首先备份.vimrc ( 好吧他会自动给你备份的 ) **

备份好后使用以下命令一键完成

`curl -sLf https://spacevim.org/cn/install.sh | bash`

注意在国内可能需要代理 `export all_proxy="socks:127.0.0.1:xxxx"`

安装好后启动vim/neovim就会自动安装插件



## 基础配置

### 安装字体

SPACE-VIM这尿性 字体还非得是Nerd的

在[nerd](https://github.com/ryanoasis/nerd-fonts)里挑选你中意的字体安装

在终端中配置

![终端配置首选项](https://res.cloudinary.com/v1mecn/image/upload/v1582738047/blog/WUKT_GXNKVJ5B_PGK_Y8_V_skjksy.png)

![设置首选字体](https://res.cloudinary.com/v1mecn/image/upload/v1582738133/blog/4_E2_5_9EXN1_SBF_QAXM_srdwxt.png)

### 设置别名

因为有了neovim不想再用vim了

所以`$ nvim ~/.zshrc`

在最后一行写入`alias vim="nvim"`

`$ source ~/.zshrc`

### 剪切板互通

`apt install xsel`

然后就可以以`"+y`的形式把vim的内容复制到`+`寄存器中

`+`寄存器就是系统剪切板 

### 安装TabNine

TabNine是一款补全神器

也是目前做的最好的AI补全的软件

用在vim/neovim上面 TabNine提供了两个插件: `codota/tabnine-vim`和`tbodt/deoplete-tabnine`

这里推荐`tbodt/deoplete-tabnine`

因为它更快

#### 安装`deoplete.nvim`

编辑`~/.SpaceVim.d/init.toml`

在结尾加入

```
[[custom_plugins]]
    name = "Shougo/deoplete.nvim"
    merged = false
```

 在`[options]`标签下加入

```
# [options]
	bootstrap_before = "myspacevim#before"
```

编辑`~/.SpaceVim.d/autoload/myspacevim.vim`

```function! myspacevim#before() abort
function! myspacevim#before() abort
    let g:deoplete#enable_at_startup = 1
endfunction
```

#### 安装`tbodt/deoplete-tabnine`

编辑`~/.SpaceVim.d/init.toml`

在结尾加入

```
[[custom_plugins]]
    name = "tbodt/deoplete-tabnine"
    merged = false
```

重新进入nvim发现插件就安装好了

但是现在还不能运行TabNine

执行`cd ~/.cache/vimfiles/repos/github.com/tbodt/deoplete-tabnine && bash install.sh`

之后就能使用TabNine了



## 其他功能

### 插件更新

更新名为"plugin_name"的插件

`:SPUpdate <plugin_name>`

更新所有插件 包括SPACE-VIM自身

`:SPUpdate`

### 添加插件

添加 github 上的插件 

在 SpaceVim 配置文件中添加 `[[custom_plugins]]` 片段：

```
[[custom_plugins]]
    name = "lilydjwg/colorizer"
    on_cmd = ["ColorHighlight", "ColorToggle"]
    merged = false
```

### 禁用插件

```
[options]
    # 请注意，该值为一个 List，每一个选项为插件的名称，而非 github 仓库地址。
    disabled_plugins = ["clighter", "clighter8"]
```

### 指令速查

#### 界面元素切换

所有的界面元素切换快捷键都以 `[SPC] t` 或 `[SPC] T` 开头，你可以在快捷键导航中查阅所有快捷键。

| 快捷键      | 功能描述                                  |
| :---------- | :---------------------------------------- |
| `SPC t 8`   | 高亮所有超过 80 列的字符                  |
| `SPC t f`   | 高亮临界列，默认 `max_column` 是第 120 列 |
| `SPC t h h` | 高亮当前行                                |
| `SPC t h i` | 高亮代码对齐线                            |
| `SPC t h c` | 高亮光标所在列                            |
| `SPC t h s` | 启用/禁用语法高亮                         |
| `SPC t i`   | 切换显示当前对齐(TODO)                    |
| `SPC t n`   | 显示/隐藏行号                             |
| `SPC t b`   | 切换背景色                                |
| `SPC t t`   | 打开 Tab 管理器                           |
| `SPC T ~`   | 显示/隐藏 Buffer 结尾空行行首的 `~`       |
| `SPC T F`   | 切换全屏(TODO)                            |
| `SPC T f`   | 显示/隐藏 Vim 边框(GUI)                   |
| `SPC T m`   | 显示/隐藏菜单栏                           |
| `SPC T t`   | 显示/隐藏工具栏                           |

#### 状态栏

| 快捷键      | 功能描述           |
| :---------- | :----------------- |
| `SPC [1-9]` | 跳至指定序号的窗口 |

| 快捷键      | 功能描述                                                     |
| :---------- | :----------------------------------------------------------- |
| `SPC t m b` | 显示/隐藏电池状态 (需要安装 acpi)                            |
| `SPC t m c` | toggle the org task clock (available in org layer)(TODO)     |
| `SPC t m i` | 显示/隐藏输入法                                              |
| `SPC t m m` | 显示/隐藏 SpaceVim 已启用功能                                |
| `SPC t m M` | 显示/隐藏文件类型                                            |
| `SPC t m n` | toggle the cat! (if colors layer is declared in your dotfile)(TODO) |
| `SPC t m p` | 显示/隐藏光标位置信息                                        |
| `SPC t m t` | 显示/隐藏时间                                                |
| `SPC t m d` | 显示/隐藏日期                                                |
| `SPC t m T` | 显示/隐藏状态栏                                              |
| `SPC t m v` | 显示/隐藏版本控制信息                                        |

| 快捷键    | Unicode | ASCII | 功能                 |
| :-------- | :------ | :---- | :------------------- |
| `SPC t 8` | ⑧       | 8     | 高亮指定列后所有字符 |
| `SPC t f` | ⓕ       | f     | 高亮指定列字符       |
| `SPC t s` | ⓢ       | s     | 语法检查             |
| `SPC t S` | Ⓢ       | S     | 拼写检查             |
| `SPC t w` | ⓦ       | w     | 行尾空格检查         |

#### 文件树中的常用操作

文件树中主要以 `hjkl` 为核心，这类似于 [vifm](https://github.com/vifm) 中常用的快捷键：

| 快捷键               | 功能描述                       |
| :------------------- | :----------------------------- |
| `<F3>`               | 切换文件树                     |
| **文件树内的快捷键** |                                |
| `h`                  | 移至父目录，并关闭文件夹       |
| `j`                  | 向下移动光标                   |
| `k`                  | 向上移动光标                   |
| `l`                  | 展开目录，或打开文件           |
| `N`                  | 在光标位置新建文件             |
| `y y`                | 复制光标下文件路径至系统剪切板 |
| `y Y`                | 复制光标下文件至系统剪切板     |
| `P`                  | 在光标位置黏贴文件             |
| `.`                  | 切换显示隐藏文件               |
| `s v`                | 分屏编辑该文件                 |
| `s g`                | 垂直分屏编辑该文件             |
| `p`                  | 预览文件                       |
| `i`                  | 切换至文件夹历史               |
| `v`                  | 快速查看                       |
| `>`                  | 放大文件树窗口宽度             |
| `<`                  | 缩小文件树窗口宽度             |
| `g x`                | 使用相关程序执行该文件         |
| `'`                  | 标记光标下的文件（夹）         |
| `V`                  | 清除所有标记                   |
| `Ctrl`+`r`           | 刷新页面                       |

#### 文本操作命令

文本相关的命令 (以 `x` 开头)：

| 快捷键        | 功能描述                                                     |
| :------------ | :----------------------------------------------------------- |
| `SPC x a #`   | 基于分隔符 # 进行文本对齐                                    |
| `SPC x a %`   | 基于分隔符 % 进行文本对齐                                    |
| `SPC x a &`   | 基于分隔符 & 进行文本对齐                                    |
| `SPC x a (`   | 基于分隔符 ( 进行文本对齐                                    |
| `SPC x a )`   | 基于分隔符 ) 进行文本对齐                                    |
| `SPC x a [`   | 基于分隔符 [ 进行文本对齐                                    |
| `SPC x a ]`   | 基于分隔符 ] 进行文本对齐                                    |
| `SPC x a {`   | 基于分隔符 { 进行文本对齐                                    |
| `SPC x a }`   | 基于分隔符 } 进行文本对齐                                    |
| `SPC x a ,`   | 基于分隔符 , 进行文本对齐                                    |
| `SPC x a .`   | 基于分隔符 . 进行文本对齐(for numeric tables)                |
| `SPC x a :`   | 基于分隔符 : 进行文本对齐                                    |
| `SPC x a ;`   | 基于分隔符 ; 进行文本对齐                                    |
| `SPC x a =`   | 基于分隔符 = 进行文本对齐                                    |
| `SPC x a ¦`   | 基于分隔符 ¦ 进行文本对齐                                    |
| `SPC x a |`   | 基于分隔符 \| 进行文本对齐                                   |
| `SPC x a SPC` | 基于分隔符 进行文本对齐                                      |
| `SPC x a a`   | align region (or guessed section) using default rules (TODO) |
| `SPC x a c`   | align current indentation region using default rules (TODO)  |
| `SPC x a l`   | left-align with evil-lion (TODO)                             |
| `SPC x a L`   | right-align with evil-lion (TODO)                            |
| `SPC x a r`   | 基于用户自定义正则表达式进行文本对齐                         |
| `SPC x a o`   | 对齐算术运算符 `+-*/`                                        |
| `SPC x c`     | 统计选中区域的字符/单词/行数                                 |
| `SPC x d w`   | 删除行尾空白字符                                             |
| `SPC x d SPC` | Delete all spaces and tabs around point, leaving one space   |
| `SPC x g l`   | set lanuages used by translate commands (TODO)               |
| `SPC x g t`   | 使用 Google Translate 翻译当前单词                           |
| `SPC x g T`   | reverse source and target languages (TODO)                   |
| `SPC x i c`   | change symbol style to `lowerCamelCase`                      |
| `SPC x i C`   | change symbol style to `UpperCamelCase`                      |
| `SPC x i i`   | cycle symbol naming styles (i to keep cycling)               |
| `SPC x i -`   | change symbol style to `kebab-case`                          |
| `SPC x i k`   | change symbol style to `kebab-case`                          |
| `SPC x i _`   | change symbol style to `under_score`                         |
| `SPC x i u`   | change symbol style to `under_score`                         |
| `SPC x i U`   | change symbol style to `UP_CASE`                             |
| `SPC x j c`   | 居中对齐当前段落                                             |
| `SPC x j f`   | set the justification to full (TODO)                         |
| `SPC x j l`   | 左对齐当前段落                                               |
| `SPC x j n`   | set the justification to none (TODO)                         |
| `SPC x j r`   | 右对齐当前段落                                               |
| `SPC x J`     | 将当前行向下移动一行并进入临时快捷键状态                     |
| `SPC x K`     | 将当前行向上移动一行并进入临时快捷键状态                     |
| `SPC x l d`   | duplicate line or region (TODO)                              |
| `SPC x l s`   | sort lines (TODO)                                            |
| `SPC x l u`   | uniquify lines (TODO)                                        |
| `SPC x o`     | use avy to select a link in the frame and open it (TODO)     |
| `SPC x O`     | use avy to select multiple links in the frame and open them (TODO) |
| `SPC x t c`   | 交换当前字符和前一个字符的位置                               |
| `SPC x t C`   | 交换当前字符和后一个字符的位置                               |
| `SPC x t w`   | 交换当前单词和前一个单词的位置                               |
| `SPC x t W`   | 交换当前单词和后一个单词的位置                               |
| `SPC x t l`   | 交换当前行和前一行的位置                                     |
| `SPC x t L`   | 交换当前行和后一行的位置                                     |
| `SPC x u`     | 将选中字符串转为小写                                         |
| `SPC x U`     | 将选中字符串转为大写                                         |
| `SPC x w c`   | 统计选中区域的单词数                                         |
| `SPC x w d`   | show dictionary entry of word from wordnik.com (TODO)        |
| `SPC x `      | indent or dedent a region rigidly (TODO)                     |

#### 增删注释

注释的增删是通过插件 [nerdcommenter](https://github.com/preservim/nerdcommenter) 来实现的， 以下为注释相关的常用快捷键：

| 快捷键    | 功能描述                  |
| :-------- | :------------------------ |
| `SPC ;`   | 进入注释操作模式          |
| `SPC c h` | 隐藏/显示注释             |
| `SPC c l` | 注释/反注释当前行         |
| `SPC c L` | 注释行                    |
| `SPC c u` | 反注释行                  |
| `SPC c p` | 注释/反注释段落           |
| `SPC c P` | 注释段落                  |
| `SPC c s` | 使用完美格式注释          |
| `SPC c t` | 注释/反注释到行           |
| `SPC c T` | 注释到行                  |
| `SPC c y` | 注释/反注释同时复制(TODO) |
| `SPC c Y` | 复制到未命名寄存器后注释  |
| `SPC c $` | 从光标位置开始注释当前行  |

小提示：

用 `SPC ;` 可以启动一个注释操作符模式，在该模式下，可以使用移动命令确认注释的范围， 比如 `SPC ; 4 j`，这个组合键会注释当前行以及下方的 4 行。这个数字即为相对行号，可在左侧看到。



## 完整配置

`~/.SpaceVim.d/init.toml`

```
#=============================================================================
# dark_powered.toml --- dark powered configuration example for SpaceVim
# Copyright (c) 2016-2017 Wang Shidong & Contributors
# Author: Wang Shidong < wsdjeg at 163.com >
# URL: https://spacevim.org
# License: GPLv3
#=============================================================================

# All SpaceVim option below [option] section
[options]
    # set spacevim theme. by default colorscheme layer is not loaded,
    # if you want to use more colorscheme, please load the colorscheme
    # layer
    colorscheme = "molokai"
    colorscheme_bg = "dark"
    # Disable guicolors in basic mode, many terminal do not support 24bit
    # true colors
    enable_guicolors = true
    # Disable statusline separator, if you want to use other value, please
    # install nerd fonts
    statusline_separator = "arrow"
    statusline_inactive_separator = "arrow"
    buffer_index_type = 4
    enable_tabline_filetype_icon = true
    enable_statusline_mode = true
    filemanager = "defx"
    filetree_direction = "left"
    bootstrap_before = "myspacevim#before"

# Enable autocomplete layer
[[layers]]
name = 'autocomplete'
auto-completion-return-key-behavior = "complete"
auto-completion-tab-key-behavior = "smart"

[[layers]]
name = 'shell'
default_position = 'bottom'
default_height = 30

[[layers]]
name = "colorscheme"

[[layers]]
name = "denite"

[[custom_plugins]]
    name = "Shougo/deoplete.nvim"
    merged = false

[[custom_plugins]]
    name = "tbodt/deoplete-tabnine"
    merged = false
```



`~/.SpaceVim.d/autoload/myspacevim.vim`

```
function! myspacevim#before() abort
    let g:deoplete#enable_at_startup = 1
endfunction
```

以后有看到什么好的插件再更新