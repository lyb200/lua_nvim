# 使用 Lua 配置 Neovim 及把 init.vim 转换成 init.lua

<center>发表时间：2021-02-07</center>
<br><br>

如果你像我一样，从未真正地理解 Vimscript，且对这门语言深恶痛绝，那你就来对了地方！现在你可以完全放弃 Vimscript，进而使用简单、快速和更优雅的语言 —— Lua 来取代它！但是，这只有在 Neovim 0.5 及以上 [版本](https://github.com/neovim/neovim/pull/12235) 才有可能，到现在为止，需要你从 HEAD 中安装 Neovim（译者：现在版本已经没有问题了）。如何安装该软件就留给读者当成一个练习。也请记住，Lua API 现在几乎处于测试状态，且许多 Vim 的功能还没有直接的接口。

因此，假设你正在运行 Neovim master 分支，请转到 `~/.config/nvim` 目录中，并创建你的 `init.lua` 文件。为什么，是的，我们现在正在移植你的 `init.vim` 到 `init.lua`！在你的日程安排表上留出几个小时————为你的编辑器进行配置是最优先要考虑的事件！

我也建议在开始之前，先通读一下 [nanotee/nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide) 和 [Learn Lua in Y minutes](https://learnxinyminutes.com/docs/lua/) 。

## 目录结构

通常，Lua 文件都是放置在 `~/.config/nvim/lua` 目录中，且被当成是 Lua 模块予以加载。这是难以置信的强大 —— 你可以随心所欲地组织自己的配置。

```lua
$ tree .config/nvim
.
|-- ftplugin
|   `-- ...
|-- init.lua
|-- lua
|   |-- maps.lua
|   |-- settings.lua
|   |-- statusline.lua
|   `-- utils.lua
`-- plugin
    `-- ...
```

常规方法是在 `Lua/` 目录下，存放使用 Lua 编写的不同的配置文件，并且在 `init.lua` 中使用 `require` 引入它们，就像下面这样：

```lua
-- init.lua

require('settings')       -- lua/settings.lua
require('maps')           -- lua/maps.lua
require('statusline')     -- lua/statusline.lua
```

## 基础部分：设置选项

Vim 有三种选项类型 —— 全局、缓冲区-本地和窗口-本地。在 Vimscript 中，你仅仅设置它们就可以了。而在 Lua 中，你得使用其中之一（译者：现在已经有了一个通用的接口，无需区别作用域，即使用 vim.opt）：

- vim.api.nvim_set_option() -- 全局选项
- vim.api.nvim_buf_set_option() -- 缓冲区-本地选项
- vim.api.nvim_win_set_option() -- 窗口-本地选项

这些都相当冗长和笨重，但幸运的是，我们有其对应的“元访问器”：`vim.{o, wo, bo}` 。以下是从我的`settings.lua` 文件中摘录出来的部分示例：

```lua
local o = vim.o
local wo = vim.wo
local bo = vim.bo

-- 全局选项
o.swapfile = true
o.dir = '/tmp'
o.smartcase = true
o.hlsearch = true
o.incsearch = true
o.ignorecase = true
o.scrolloff = 12
-- ... snip ...

-- 窗口-本地选项
wo.number = false
wo.wrap = false

-- 缓冲区-本地选项
bo.expandtab = true
```

如果你不确定某个选项是全局的、缓冲区或窗口-本地的，则查询 Vim 帮助！例如，`:h 'number'` ：

```lua
'number' 'nu'              boolean (default off)
                           local to window
```

另外注意，你不要设置一个否定选项为 `true` ，例如，`wo.nonumber = true` ，应该使用 `wo.number = false` 。

## 定义自动命令

不幸的是，Lua 中还没有 Vim 的自动命令的接口 —— 现在正在 [开发](https://github.com/neovim/neovim/pull/12378) 中。在开发完成之前，你不得不使用 `vim.api.nvim_command()` 或者，短点的 `vim.cmd()` 命令。我已经定义了一个简单的函数，它能把一个包含自动命令的 Lua 表作为参数，且为你创建一个 `augroup` 。

```lua
-- utils.lua ，工具文件

local M = {}
local cmd = vim.cmd

function M.create_augroup(autocmds, name)
  cmd('augroup ' .. name)
  cmd('autocmd!')
  for _, autocmd in ipairs(autocmds) do
    cmd('autocmd ' .. table.concat(autocmd, ' '))
  end
  cmd('augroup END')
end

return M
```

```lua
-- settings.lua
local cmd = vim.cmd
local u = require('utils')

u.create_augroup({
  { 'BufRead,BufNewFile', '/tmp/nail-*', 'setlocal', 'ft=mail' },
  { 'BufRead,BufNewFile', '*s-nail-*', 'setlocal', 'ft=mail' },
})

cmd('au BufNewFile,BufRead * if &ft == "" | set ft=text | endif')
```

## 定义键映射

键映射是通过`vim.api.nvim_set_keymap` 进行设置的。它需要传递过来四个参数：映射将在那个模式下起效，键序列，要执行的命令和一个包含选项的表（`:h :map-arguments` ）。

```lua
-- maps.lua

local map = vim.api.nvim_set_keymap

-- map the leader key，映射 leader 键
map('n', '<Space>', '', {})
vim.g.mapleader = ' '       -- 'vim.g' 设置全局变量

options = { noremap = true }
map('n', '<leader><esc>', ':nohlsearch<cr>', options)
map('n', '<leader>n', ':bnext<cr>', options)
map('n', '<leader>p', ':bprev<cr>', options)
```

对于用户自定义的命令，你得使用 `vim.cmd` 接口：

```lua
local cmd = vim.cmd

cmd(':command! WQ wq')
cmd(':command! WQ wq')
cmd(':command! Wq wq')
cmd(':command! Wqa wqa')
cmd(':command! W w')
cmd(':command! Q q')
```

## 包管理

当然，你不能再使用你最喜欢的 Vimscript 包管理器，或者至少不能在没有 `vim.api.nvim_execing` 执行一大堆 Vimscript （呕！）的前提下使用。

幸运的是，已经有几个使用纯 Lua 编写的插件管理器可用（也可参见：[packer.nvim](https://github.com/wbthomason/packer.nvim/)）—— 我个人在使用，也推荐 `paq` 包管理插件。该管理器很轻，且利用了 `vim.loop API` 实现异步 `I/O` 。`paq` 的文档丰富，因此我就不谈论如何设置它。

## 额外奖励：编写你自己的状态栏

想象一下，当你能编写你自己的状态栏时，再使用一个臃肿的、第三方的是什么感觉。它其实非常简单！为每个模式定义一个表：

```lua
-- statusline.lua

local mode_map = {
  ['n'] = 'normal ',
  ['no'] = 'n·operator pending ',
  ['v'] = 'visual ',
  ['V'] = 'v·line ',
  ['^V'] = 'v·block ',
  ['s'] = 'select ',
  ['S'] = 's·line ',
  ['^S'] = 's·block ',
  ['i'] = 'insert ',
  ['R'] = 'replace ',
  ['Rv'] = 'v·replace ',
  ['c'] = 'command ',
  ['cv'] = 'vim ex ',
  ['ce'] = 'ex ',
  ['r'] = 'prompt ',
  ['rm'] = 'more ',
  ['r?'] = 'confirm ',
  ['!'] = 'shell ',
  ['t'] = 'terminal '
}
```

其想法是通过 `vim.api.nvim_get_mode()` 函数得到当前的模式，然后把它映射到我们设计的文本上。让我们把该函数包裹在一个小的 `mode()` 函数中：

```lua
-- statusline.lua

local function mode()
  local m = vim.api.nvim_get_mode().mode
  if mode_map[m] == nil then return m end
  return mode_map[m]
end
```

现在，设置你的高亮。同样，还没有高亮的任何接口，因此，请使用该函数 `vim.api.nvim_exec()` 。

```lua
-- statusline.lua

vim.api.nvim_exec(
[[
  hi PrimaryBlock    ctermfg=06  ctermbg=00
  hi SecondaryBlock  ctermfg=08  ctermbg=00
  hi Blanks          ctermfg=07  ctermbg=00
]], false)
```

创建一个新表来代表整个状态栏自身。你可以添加任何其他你想要的函数（例如，能返回当前 git 分支的函数）。如果你不能理解以下发生了什么，请参见 `:h 'statusline'` 。

```lua
-- statusline.lua

local stl = {
  '%#PrimaryBlock#',
  mode(),
  '%#SecondaryBlock#',
  '%#Blanks#',
  '%f',
  '%m',
  '%=',
  '%#SecondaryBlock#',
  '%l,%c ',
  '%#PrimaryBlock#',
  '%{&filetype}',
}
```

最后，使用 `table.concat()` 的强大功能，设置你的状态栏。这类似于执行一系列的字符串拼接，但是速度更快。

```lua
-- statusline.lua

vim.o.statusline = table.concat(stl)
```

图示如下：
![statusline.png](https://github.com/lyb200/lua_nvim/blob/main/pic/statusline.png)

## 这就是当 tpope 的感觉

现在你可以编写你一直希望的那个插件！我住下来编写了用于 [fzy](https://github.com/jhawthorn/fzy) ( 一个比较清瘦的用于替代 fzf 的插件，是用 C 编写的) 的插件，那里你可以在 [这里](https://git.icyphox.sh/dotfiles/tree/config/nvim/lua/fzy) 找到它，还有我整个 Neovim 配置文件(`GitHub link`—— 如果你想进入这类领域)。我计划不久把我的 `plugin/` 目录迁移到 Lua 中。

而且，只有当 Lua API 开发完成后，它才会变得更好。我们现在都可以成为 Vim 插件的艺术家。

如果有什么问题或疑问，发送[邮件](x@icyphox.sh)。

---

原文档网址：[Configuring Neovim using Lua
And switching from init.vim to init.lua](https://icyphox.sh/blog/nvim-lua/#fn:3) 。
