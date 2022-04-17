# 从零开始构建一个 vim 状态栏

![statusline.jpg](https://github.com/lyb200/lua_nvim/blob/main/pic/statusline.jpg)

从我使用 Neovim 开始，我一直使用 [vim-airline](https://github.com/vim-airline/vim-airline) 来定制我的状态栏。它运行得很好。最近，我开始了一个名为 `minimal_vim` 的 [一个仓库](https://github.com/jdhao/minimal_vim) ，在没有使用额外插件的前提下，创建一个最小化的适用于 Vim 和 Neovim 的配置。在这过程中，我学到了如何从零开始构建一个自定义的状态栏。

我参考的主要资料是 Vim 帮助中有关 `statusline` 的介绍（参见 `:h 'ststatusline'`）。它写得很好，应该能给你足够的知识来构建一个简单的状态栏。

Vim 的状态栏由许多具有特殊格式的项目组成，这些项目表明了我们想要显示的那些内容以及它们会以什么样式出现，例如，宽度、颜色、对齐方式和填充等。

## 状态栏中的项目格式

通常，状态栏由许多 printf 样式 `%` 的项目组成，以显示有关当前文件的各种信息，例如，`%F` 用于显示当前文件的完整路径。项目的完整格式如下：

```lua
%-0{minWidth}.{maxWidth}{item}
```

- `-` 表示项目左对齐，而不是默认的右对齐。
- `0` 表示项目是数值、以数字表示的前导零，且被 `-` 覆盖。
- `minWidth` 和 `maxWidth` 决定被显示的项目的最小和最大长度。

除了 `{item}` 外，所有字段都是可选的。

## 良好的惯例：一点一点地构建状态栏

既然我们将使用几个项目来构建我们的状态栏，最好的实践是通过将各种组件拼接起来从而构建状态栏（也请参见 `:h set+=` )，而不是使用一个长的字符串来设置状态栏。逐步构建状态栏使之更容易理解和编辑。

例如，我们使用以下代码：

```vim
" 状态栏置空
set statusline=
" 显示完整的文件路径
set statusline+=%F
" 显示当前的行号
set statusline+=%l
```

## 显示函数或表达式的结果

Vim 提供了一些默认的项目用于显示有关当前文件的信息，但是，它们可能不够充分。我们想要显示函数执行的结果，或者显示基于一些条件得到的字符串。Vim 使用 `%{}` 计算函数或表达式，且在状态栏中显示返回的字符串。

想要在状态栏中显示当前模式，可以使用以下设置：

```vim
let g:currentmode={
  \ 'n'     : 'NORMAL ',
  \ 'v'     : 'VISUAL ',
  \ 'V'     : 'V.Line ',
  \"\<C-v>" : 'V.Block ',
  \ 'i'     : 'INSERT ',
  \ 'R'     : 'R ',
  \ 'Rv'    : 'V.Replace ',
  \ 'c'     : 'Command ',
  \}

set statusline=
set statusline+=\ %{toupper(g:currentmode[mode()])}
```

状态栏就会根据目前处于何种模式自动改变状态栏模式提示。

想要在当前文件修改后显示已修改符号 `[+]` ，我们可以使用以下设置：

```vim
set statusline+=%{&modified?'[+]':''}
```

## 使用组

组用于设置作为整体的几个元素，例如，宽度和对齐方式。一个组以 `(` 开始，且以 `%)` 结束，例如，`(group_element%)` 。举个示例，想要显示文件格式，我们可以使用以下的设置：

```vim
set statusline+=%-7([%{&fileformat}]%)
```

以上的组最小宽度是 7，且左对齐。

## 使用高亮组

想要知道如何设置状态栏中的不同部分如何显示不同的颜色？很简单：使用高亮组。

使用高亮显示有两种略有不同的方式。

### 使用高亮组名

第一种方法是使用高亮组的名称。使用 `:hightlight` 命令，我们可以检查有哪些高亮组可用。例如，想要使用高亮组 `Warnings` ，我们可以使用以下的设置：

```vim
set statusline+=%#Warnings#
set statusline+=%{&bomb?'[BOM]':''}
```

高亮组将在状态栏中使用的位置开始起效。想要切换高亮，我们只需在想要的位置提供另一个高亮组（这意味着，如果你不想切换它，状态栏中任何后续元素都将使用该高亮组）。

### 使用用户定义的特殊高亮组

Vim 也允许我们定义名为 `User1` 至 `User9` 一共 9 个特殊的高亮组，像下面的那样，使用特殊的语法在状态栏中引用它们：

```vim
" 使用高亮组 User2
set statusline+=%2*
```

首先，我们需要定义所需的用户高亮组：

```vim
hi User1 ctermfg=0 ctermbg=114
hi User2 ctermfg=114 ctermfg=0
```

然后我们可以在状态栏的设置中使用它们。想要回到默认的状态栏颜色，我们可以使用：

```vim
set statusline+=%*
```

### 避免常见的警告

最常见的错误是未能转义一些特殊字符。既然空格和 `|` 有特殊含义（也请参见：`:h option-backslash` ），我们需要使用反斜杠来转义它们以得到其字面值。例如：

```vim
set statusline+=%{&filetype!=#''?&filetype.'\ ':'none\ '}
```

上面的设置将会显示当前文件类型及跟一个空格，或者如果文件类型是空的，显示 `none` 及跟一个空格。

`%` 也是一个特殊字符。想要包含一个字面的 `%` 字符，我们需要使用 `%%` 。例如，想要显示当前光标位置在缓冲区中的百分比，使用：

```vim
set statusline+=%2p%%
```

## 参考资料

- [https://gist.github.com/suderman/1243665](https://gist.github.com/suderman/1243665)
- [https://gist.github.com/autrimpo/f40e4eda233977dd3a619c6083d9bebd](https://gist.github.com/autrimpo/f40e4eda233977dd3a619c6083d9bebd)
- [Building your own statusline](https://shapeshed.com/vim-statuslines/)
- [Scrooloose’s vimrc](https://github.com/scrooloose/vimfiles/blob/master/vimrc#L161)
- [A better vim statusline?](https://stackoverflow.com/q/5375240/6064933)
- [http://got-ravings.blogspot.com/search/label/statuslines](http://got-ravings.blogspot.com/search/label/statuslines)

作者 ：jdhao <br>
最后修改：2021-12-12 <br>
许可 ：[https://creativecommons.org/licenses/by-nc-nd/4.0/](https://creativecommons.org/licenses/by-nc-nd/4.0/)<br>

---

原文档网址：[Building A Vim Statusline from Scratch](https://jdhao.github.io/2019/11/03/vim_custom_statusline/) 。
