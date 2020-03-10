---
title: SPACE-VIM一些模块的安装
date: 2020-02-27 20:12:03
tags: ["neovim", "space-vim"]
categories: Linux日常折腾
description: 昨天安装好了能用了 今天尝试让space-vim变得顺手
---



## VersionControl

这个好用 最主要的是可以显示当前代码和已经commit的代码的区别

例如

![](https://res.cloudinary.com/v1mecn/image/upload/v1582791030/blog/N2_G71QN67G4C_Q_REF_XK_vaqwwv.png)

左边亮眼的原谅色显示了代码加了这几行

安装很简单

```toml
[[layers]]
  name = "VersionControl"
```

## git

git和VersionControl会有点冲突

看喜欢用哪个了

![](https://res.cloudinary.com/v1mecn/image/upload/v1582794411/blog/VXJ_UU_M0_L_XV__5Q_X24_qmt3n0.png)

这默认的加号颜色也太......

不过这样看起来比较没有违和感倒是真的

同样的

```toml
[[layers]]
  name = "git"
```

| 快捷键      | 功能描述             |
| :---------- | :------------------- |
| `SPC g s`   | 打开 git status 窗口 |
| `SPC g S`   | stage 当前文件       |
| `SPC g U`   | unstage 当前文件     |
| `SPC g c`   | 打开 git commit 窗口 |
| `SPC g p`   | 执行 git push        |
| `SPC g d`   | 打开 git diff 窗口   |
| `SPC g A`   | git add 所有文件     |
| `SPC g b`   | 打开 git blame 窗口  |
| `SPC g h a` | stage current hunk   |
| `SPC g h r` | undo cursor hunk     |
| `SPC g h v` | preview cursor hunk  |



## Chinese

中文插件

```toml
# [options]
    vim_help_language = "cn"
    
[[layers]]
 name = "chinese"
```



## Cscope

主要用来看函数定义

安装依赖

`apt install cscope cscope-el`

neovim命令模式下

`:echo has("cscope")`

如果输出`1`就是已经有了支持

```toml
[[layers]]
  name = "cscope"
```

| 快捷键      | 功能描述                        |
| :---------- | :------------------------------ |
| `SPC m c =` | Find assignments to this symbol |
| `SPC m c i` | 建立当前项目 cscope 索引        |
| `SPC m c u` | 更新所有项目 cscope 索引        |
| `SPC m c c` | 列出某个方法调用的所有函数      |
| `SPC m c C` | 列出某个方法被哪些函数调用      |
| `SPC m c d` | 查询 symbol 的定义处            |
| `SPC m c r` | 查询 symbol 的引用              |
| `SPC m c f` | 搜索文件                        |
| `SPC m c F` | 列出 include 某个文件的所有文件 |
| `SPC m c e` | 搜索正则表达式                  |
| `SPC m c t` | 搜索文本                        |



## C/C++ Python 配置

参考: 

- [https://spacevim.org/cn/layers/lang/python/#%E6%95%B4%E7%90%86-imports](https://spacevim.org/cn/layers/lang/python/#整理-imports)

- https://spacevim.org/cn/layers/lang/c/
- https://spacevim.org/cn/use-vim-as-a-python-ide/
- https://spacevim.org/cn/use-vim-as-a-c-cpp-ide/

安装依赖:

`apt install -y clang libclang-dev clang-tools-8 && update-alternatives --install /usr/bin/clangd clangd /usr/bin/clangd-8 100`

`apt install clang-format`

`pip install flake8 isort yapf autoflake coverage `

看看自己的clang是什么版本

`clang -v`

然后pip安装对应版本的clang

`pip install clang==x.x.x`

配置文件

```toml
[[layers]]
  name = "lang#c"
  enable_clang_syntax_highlight = true
  clang_executable = "/usr/bin/clang"
  libclang_path = "/usr/lib/llvm-3.8/lib/"
  [layer.clang_std]
    c = "c11"
    cpp = "c++1z"
    objc = "c11"
    objcpp = "c++1z"

[[layers]]
  name = "format"

[[layers]]
  name = "lsp"
  filetypes = [
    "c",
    "cpp"
  ]
  [layers.override_cmd]
    c = ["clangd"]

[[layers]]
  name = "lang#python"
  format_on_save = 1
  python_file_head = [
      '#!/usr/bin/env python',
      '# -*- coding: utf-8 -*-',
      '',
      ''
  ]
```

重启neovim自动安装

`cd ~/.cache/vimfiles/repos/github.com/autozimu/LanguageClient-neovim`

`bash install.sh`

## 更新后的配置文件

`~/.SpaceVim.d/init.toml`

```toml
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
    vim_help_language = "cn"

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

# [[layers]]
#   name = "VersionControl"

[[layers]]
  name = "chinese"

[[layers]]
  name = "cscope"

[[layers]]
  name = "git"

[[layers]]
  name = "lang#c"
  enable_clang_syntax_highlight = true
  clang_executable = "/usr/bin/clang"
  libclang_path = "/usr/lib/llvm-3.8/lib/"
  [layer.clang_std]
    c = "c11"
    cpp = "c++1z"
    objc = "c11"
    objcpp = "c++1z"

[[layers]]
  name = "format"

[[layers]]
  name = "lsp"
  filetypes = [
    "c",
    "cpp"
  ]
  [layers.override_cmd]
    c = ["clangd"]

[[layers]]
  name = "lang#python"
  format_on_save = 0
  python_file_head = [
      '#!/usr/bin/env python',
      '# -*- coding: utf-8 -*-',
      '',
      ''
  ]

[[custom_plugins]]
    name = "Shougo/deoplete.nvim"
    merged = false

[[custom_plugins]]
    name = "tbodt/deoplete-tabnine"
    merged = false

[[custom_plugins]]
    name = "vim-python/python-syntax"
    merged = false
```

`~/.SpaceVim.d/autoload/myspacevim.vim`

```vim
function! myspacevim#before() abort
    let g:deoplete#enable_at_startup = 1
    " CTRL-X and SHIFT-Del are Cut
    vnoremap <C-X> "+x
    vnoremap <S-Del> "+x
    " CTRL-C and CTRL-Insert are Copy
    vnoremap <C-C> "+y
    vnoremap <C-Insert> "+y
    let g:python_highlight_all = 1

    augroup fmt
        autocmd!
        autocmd BufWritePre * undojoin | Neoformat
    augroup END
endfunction
```

`~/.clang-format`

```yaml
BasedOnStyle:  Google
Language: Cpp
ColumnLimit: 80
IndentWidth: 4
TabWidth: 4
UseTab: Never
IndentCaseLabels: true
BreakBeforeBraces: Attach
AlignEscapedNewlinesLeft: false
AllowShortFunctionsOnASingleLine: false
AlignTrailingComments: true
SpacesBeforeTrailingComments: 2
PenaltyReturnTypeOnItsOwnLine: 200
AllowAllParametersOfDeclarationOnNextLine: false
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
BinPackParameters: false
BreakBeforeBinaryOperators: true
BreakBeforeTernaryOperators: true
ContinuationIndentWidth: 4
```

