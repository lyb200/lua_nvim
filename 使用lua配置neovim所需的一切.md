
    前言
    第一步
    编辑器设置
        作用域
        数据的类型
            在列表后添加一个项目
            在前面增加项目
            删除一个项目
        调用 vim 函数
            在 `lua` 中使用 `vim-plug`
        Vimscript 仍然是我们的朋友
        键绑定
            调用 lua 函数
    插件管理器
    结论
    学习资源



# <center>使用lua配置neovim所需的一切</center><br>
## 前言

经过很长时间的开发，稳定版本 `neovim 0.5 `终于发行了。在这些令人兴奋的新特性中，有了对`lua`更好的支持，承诺有一个稳定的`API`，可以使用它来创建我们的配置。因此，今天我要和大家分享的是，我把我的配置从 `vimscript` 迁移到` lua` 过程中学到的所有东西。

我将谈谈我们可以用 `lua `能做的事情，以及它与`vimscript`之间的交互。我会演示许多示例，但是我不会告诉你哪些选项应该设置什么值。此外，这不是一个关于“如何将 `neovim` 转换为` IDE`”的教程，我将避免使用任何特定于语言的内容。我想要做的是向您充分介绍` lua` 和` neovim API`，以便您可以迁移自己的配置。

我假设你的操作系统是 `linux`（或与它接近的系统），你的配置文件位于 `~/.config/nvim/init.vim`。我所讲解的知识应该适用于能够安装`neovim`的所有系统上。仅仅需要记住的是`init.vim`文件的路径可能根据不同的系统有所不同。

让我们开始吧。

## 第一步

首先需要知道的是，我们可以将`lua`代码直接嵌入到`init.vim`中。因此，我们可以一点一点地迁移配置文件，只有当我们准备好后，才能从`init.vim`更改为` init.lua`。

让我们来做一个 “hello world” 测试，看看是否一切正常。在你的 `init.vim` 中试试这个：

```lua
lua <<EOF
print('hello from lua')
EOF
```

在重新加载该文件或重启`neovim`后，该信息 `hello from lua` 应该显示在状态栏正下方。这里我们使用的称为 `lua-heredoc` ，因此在 `<<EOF ... EOF` 中的所有代码会当成`lua`脚本，并被`lua`命令执行。这在我们想要执行多行代码时有用，但是在只有一行代码时不必要。一行代码时可以这样：

```lua
lua print('this also works')
```

但是如果我们想要在 `vimscript` 中调用 `lua` 代码，我们需要一个真实的脚本。在 `lua` 中我们可以使用 `require` 函数来实现。为了做到这点，我们需要在 `neovim` 的 `runtimepath` 中创建一个 `lua` 文件夹。

您可能希望使用` init.vim` 所在的同一个文件夹，因此我们将创建 `~/.Config/nvim/lua` 文件夹，在其中，我们将创建一个名为 `basic.lua` 的脚本。现在我们只打印一条消息。

```lua
print('hello from ~/config/nvim/lua/basic.lua')
```

现在在 `init.vim` 中，我们可以像这样调用它：

```vim
lua require('basic')
```

当我们这样做时，`neovim` 会在 `runtimepath` 的所有文件夹中搜索一个称为 `lua` 的文件夹，并且它会寻找 `basic.lua`。`Neovim`会执行符合这些条件的最终脚本。

如果你查看其他人的代码，你会注意到他们使用 `.` 作为路径的分隔符。例如，假设有文件 `~/.config/nvim/lua/usermod/settings.lua` 。如果他们想要调用 `settings.lua` ，他们会这样做：

```lua
require('usermod.settings')
```

这是一个非常常见的惯例。只要记住点是路径分隔符。

有了这些知识，我们就可以开始使用 lua 进行配置了。

## 编辑器设置

`neovim` 中的每个选项我们都可以在 `vim ... `的全局变量中找到，它绝不仅仅是一个变量，试着把它想成一个全局模块。通过 `vim` ，我们可以访问编辑器的设置选项，我们也有了 `neovim API` ，甚至有了一套帮助函数（一个标准库，如果你需要的话）。对于现在来说，我们只需关心我们称为“元访问器”的东西，就是我们用于访问我们所需的所有选项的访问器。

### 作用域

就像在 `vimscript` 中一样，每个选项在 `lua` 中都有不同的作用域。有全局设置、窗口设置、缓冲区设置和一些其他设置。在 `vim` 模块中，每一个都有自己的命名空间。

- vim.o

获取或设置常规的设置。

```lua
vim.o.background = 'light'
```

- vim.wo

获取或设置窗口作用域的选项。

```lua
vim.wo.colorcolumn = '80'
```

- vim.bo

获取或设置缓冲区作用域的选项。

```lua
vim.bo.filetype = 'lua'
```

- vim.g

获取或设置全局变量。这个名称空间通常用于查找由插件设置的变量。我所知道的唯一一个没有绑定到插件的是 `leader` 键。

```lua
-- use space as a the leader key （把空格键设置成leader键）
vim.g.mapleader = ' '
```

你应该知道一些 `vimscript` 变量名在 `lua` 中是无效的。我们仍然可以访问它们，但是我们不能使用点符号。例如，`vim-zoom` 有一个变量名为 `zoom#statustext` ，在 `vimscript` 中我们像这样使用它。

```vim
let g:zoom#statustext = 'Z'
```

在 `lua` 中我们应该这样设置。

```lua
vim.g['zoom#statustext'] = 'Z'
```

正如你可能已经猜到的那样，这样也给我们一个可以访问属性名为关键字的机会。你可能会发现自己处于需要访问属性名为`for`、 `do` 或 `end` 的情况，而这些名称都是保留关键字，在这种情况下，请记住这种方括号语法。

- vim.env

获取或设置环境变量。

```lua
vim.env.FZF_DEFAULT_OPTS = '--layout=reverse'
```

据我所知，如果您对环境变量进行了更改，这种更改将仅适用于活动的 `neovim` 会话。

但是现在我们如何知道在编写配置文件时需要使用哪个“作用域”呢？不要担心这个，把`vim.o`和 其他几个都看作是一种获取选项值的方式。当需要设置值时，我们可以使用其他方法。

- vim.opt

使用 `vim.opt`，我们可以设置全局、窗口和缓冲区的设置。

```lua
-- buffer-scoped 缓冲区作用域
vim.opt.autoindent = true

-- window-scoped 窗口作用域
vim.opt.cursorline = true

-- global scope 全局作用域
vim.opt.autowrite = true
```

当我们这样使用 `vim.opt`时，它的作用与在 `vimscript`中的 `:set` 命令一样，它给我们一种修改 `neovim` 的选项一致的方法。

有趣的是我们可以把 `vim.opt` 赋值给 `set` 的变量。

假设我们有如下 `vimscript` 片段。

```vim
" Set the behavior of tab 设置tab的行为
set tabstop=2
set shiftwidth=2
set softtabstop=2
set expandtab
```

在 `lua` 中我们可以像这样容易地转换它们。

```lua
local set = vim.opt

-- Set the behavior of tab
set.tabstop = 2
set.shiftwidth = 2
set.softtabstop = 2
set.expandtab = true
```

> 当我们声明变量时，不要忘记关键字 `local`。在`lua`中，变量默认是全局的（包括函数也一样）。

但是，全局变量或环境变量怎么办? 对于这些变量，您应该分别使用 `vim.g` 和 `vim.env`。

`Vim.opt` 的有趣之处在于，每个属性都是一种特殊的对象，它们是“元表”。这意味着这些对象把它们自己的行为实现为某种相同的操作。

在第一个示例中，我们有这样的代码:` vim.opt.autoindent = true`，现在您可能认为可以通过以下方式来检查当前值。

```lua
print(vim.opt.autoindent)
```

你不能得到你期望的值，`print` 会告诉你 `vim.opt.autoindent` 是一个表。如果你想要知道一个选项的值，你应该使用 `:get()` 方法。

```lua
print(vim.opt.autoindent:get())
```

如果你非常想知道 `vim.opt.autoindent` 到底有什么，你需要使用 `vim.inspect` 。

```lua
print(vim.inspect(vim.opt.autoindent))
```

现在它会显示出该属性的内部状态。

### 数据的类型

甚至当我们在 `vim.opt` 中赋值时，背后中也会有一点点魔力发生。我认为这是很重要的，即了解 `vim.opt` 如何处理不同类型的数据并将其与 `vimscript` 进行比较。

- Booleans

这好像看起来没有什么大不了的，但仍然有一个值得一提的区别。

在 `vimscript` 中，我们能够这样启用或禁用某个选项。

```lua
set cursorline
set nocursorline
```

这在 `lua` 中是等价的。

```lua
vim.opt.cursorline = true
vim.opt.cursorline = false
```

- Lists

对于某些选项，`neovim` 需要一个逗号分隔的列表。在这种情况下，我们可以自己提供一个字符串。

```lua
vim.opt.wildignore = '*/cache/*,*/tmp/*'
```

或者我们可以使用一个表。

```lua
vim.opt.wildignore = { '*/cache/*', '*/tmp/*'}
```

如果我们检查 `vim.o.wildignore` 的内容，你会注意到我们想要 `*/cache/*,*/tmp/*` 。如果我们真的想确认，你可以使用这个命令检查。

```vim
:set wildignore?
```

你会得到相同的结果。

但是魔力不会终止于此。有时我们不需要覆盖这个列表，我需要增加一个项目或者删除它。为了让事件变得简单，`vim.opt` 提供了以下操作的支持。

#### 在列表后添加一个项目

我们拿 `errorformat` 作为示例。如果我们想要使用 `vimscript` 在列表后增加项目，我们这样设置。

```lua
set errorformat+=%f\|%l\ col\ %c\|%m
```

`lua` 中我们有两种方法达到相同的目标：

使用 `+` 操作符。

```lua
vim.opt.errorformat = vim.opt.errorformat + '%f|%l col %c|%m'
```

或者使用 `:append` 方法。

```lua
vim.opt.errorformat:append('%f|%l col %c|%m')
```

#### 在前面增加项目

在 `vimscript` 中：

```vim
set errorformat^=%f\|%l\ col\ %c\|%m
```

在 `lua` 中：

```lua
vim.opt.errorformat = vim.opt.errorformat ^ '%f|%l col %c|%m'

-- or try the equivalent

vim.opt.errorformat:prepend('%f|%l col %c|%m')
```

#### 删除一个项目

在 `lua` 中：

```vim
set errorformat-=%f\|%l\ col\ %c\|%m
```

在 `lua` 中：

```lua
vim.opt.errorformat = vim.opt.errorformat - '%f|%l col %c|%m'

-- or the equivalent

vim.opt.errorformat:remove('%f|%l col %c|%m')
```

- Pairs

一些选项期望键值对列表。为了说明这种，我们将使用 `listchars` 。

```vim
set listchars=tab:▸\ ,eol:↲,trail:·
```

在 `lua` 中，我们也可以使用表达到目标。

```lua
vim.opt.listchars = {eol = '↲', tab = '▸ ', trail = '·'}
```

> **注意：** 要在屏幕上实际看到这个效果，你需要启用 `list` 选项。参见 `help listchars` 。

由于我们仍在使用表，这个选项也支持上一节提到的相同操作。

### 调用 vim 函数

像其他编程语言一样，`vimscript` 有其自己的内置函数（[许多函数](https://neovim.io/doc/user/usr_41.html#function-list) ），并且由于有了 `vim` 模块，我们可以通过 `vim.fn` 调用它们。就像 `vim.opt` 一样，`vim.fn` 也是一个元表，但是这个表的目的是为了我们调用 `vim` 函数提供方便的语法。我们可以使用它去调用内置函数，用户自定义函数和甚至不是 `lua` 写的插件中的函数。

例如，我们可以这样检查 `neovim` 版本：

```lua
if vim.fn.has('nvim-0.5') == 1 then
  print('we got neovim 0.5')
end
```

等一下，我们为什么要把 `has` 的结果与 1 进行比较？原来 `vimscript` 只在 v7.4.1154 版本包含布尔值。因此，像 `has` 这样的函数返回 `0` 或 `1` ，而在 `lua` 中这两者都是真。

我前面已经提及 `vimscript` 可以使用 `lua` 中无效的变量名，在这种情况下，你知道你可以使用方括号：

```lua
vim.fn['fzf#vim#files']('~/projects', false)
```

我们可以解决这个问题的另一种方法是使用 `vim.call` 。

```lua
vim.call('fzf#vim#files', '~/projects'), false)
```

这两种方法做相同的事件。实际上 `vim.fn.somefunction` 和 `vim.call('somefunction')` 具有相同的效果。

现在让我给你展示非常酷的事件。在这个特定的示例中，`lua-vimscript` 整合的非常好，我们可以在没有任何特殊的适配器的帮助使用一个插件管理器。

#### 在 `lua` 中使用 `vim-plug`

我知道有很多人使用 `vim-plug` ，你可能认为需要迁移到用 lua 编写的插件管理器，但是事实并非如此。我们可以使用 `vim.fn` 和 `vim.call` 将 `vim-plug` 来引入 lua。

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin', '~/.config/nvim/plugged')

-- List of plugins goes here
-- ...

vim.call('plug#end')
```

这 3 行代码是你唯一需要的，你可以试试，这样就可以了。

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin', '~/.config/nvim/plugged')

Plug 'wellle/targets.vim'
Plug 'tpope/vim-surround'
Plug 'tpope/vim-repeat'

vim.call('plug#end')
```

在你说话之前，是的，所有这些都是有效的。如果一个函数只接收一个参数，而该参数是一个字符串或表，则可以省略小括号。

如果你用到了 `Plug` 的第二个参数，则你需要小括号，且第二个参数必须是一个表。让我展示给你看。如果你在 `vimscript` 中有这些。

```vim
Plug 'scrooloose/nerdtree', {'on': 'NERDTreeToggle'}
```

在 lua 中你得这样做：

```lua
Plug('scrooloose/nerdtree', {on = 'NERDTreeToggle'})
```

不幸的是，`vim-plug` 有两个选项，如果我们使用这个语法，会导致错误，这两个选项是 `for` 和 `do` 。在这种情况下，我们需要使用单引号和方括号。

```lua
Plug('junegunn/goyo.vim', {['for'] = 'markdown'})
```

你可能知道 `do` 选项需要一个字符串或一个插件更新或安装后要执行的函数。但是你可能不知道的是我们不会被强迫使用 `vim 函数` ，我们可以使用 lua 函数，这样就可以起作用了。

```lua
Plug('VonHeikemen/rubber-themes.vim', {
  ['do'] = function()
    vim.opt.termguicolors = true
    vim.cmd('colorcheme rubber')
})
```

现在你知道了，如果你不想使用一个用 lua 编写的插件管理器，你不必使用它。

### Vimscript 仍然是我们的朋友

在上一个示例中，你可能已经注意到，我使用 `vim.cmd` 设置颜色主题，这是因为有些事件我们仍然不能用 lua 操作。现在我们不能创建或调用 ex 命令，自动命令也不行。

为了克服这些限制，我们通常使用 `vim.cmd` 。该函数可以执行 `vimscript` 多行代码。这意味着我们可以在单个调用中做很多事。

```lua
vim.cmd [[
  syntax enable
  colorcheme rubber

  command! Hello echom "hello!!"
]]
```

因此，任何你不能“转换”到 lua 的代码，你都可以把它放在一个字符串中，并传递到 `vim.cmd` 。

我告诉过你我们能执行任何 `vim` 命令，对吧？我觉得有必要告诉你，这包括 `source` 命令。对于那些我们不知道的，`source` 允许我们调用 vimscript 编写的其他文件。例如，在我的配置中，我使用这个对颜色主题进行调整。我是这样做的。

```lua
vim.cmd 'source ~/.config/nvim/theme.vim'
```

并且，`theme.vim` 创建了一个自动命令，每当有 `colorcheme` 事件发生时都会被触发。

```vim
function! MyHighlights() abort
  hi! link Question String
  hi! link NonText LineNr

  hi! link TelescopeMatching Boolean
  hi! link TelescopeSelection cursorline
endfunction

augroup MyColors
  autocmd!
  autocmd colorcheme * call MyHighlights()
augroup END
```

我喜欢将这个片段保存在一个单独的文件中，因为我可能会继续添加代码。而且，在 lua 中还没有办法这样做。

### 键绑定

这里，我们发现自己处于一个有趣的情况。我们实际上可以在 lua 中定义我们的键绑定，但是我们还没有一个“方便”的 `API` 。为什么我会这样说？首先，当前的方式我们不熟悉，它与 vimscript 相差很大。另外，我们不能把一个 lua 函数绑定到键。我们可以从一个快捷键调用一个 lua 函数，但是我们基本上必须欺骗（我来告诉你如何做）。

不管怎样，有两个函数我们现在可以使用。

- `vim.api.nvim_set_keymap`
- `vim.api.nvim_buf_set_keymap`

第一个可以用于设置全局键绑定，而另一个只能在缓冲区设置键绑定。

`nvim_set_keymap` 需要 4 个参数：

- 模式。但不是模式的名称，我们需要缩写。你可以从 [这里](https://github.com/nanotee/nvim-lua-guide#defining-mappings) 找到可用选项的列表。
- 我们需要绑定的键。
- 我们想要执行的动作。
- 额外参数。这与我们在 vimscript 中使用的一样的选项（ `buffer` 除外），你可以在 [这里](https://neovim.io/doc/user/map.html#:map-arguments) 找到列表。

`nvim_buf_set_keymap` 也一样，唯一不同的是第一个参数必须是缓冲区的编号。如果你使用数字 `0` ，neovim 会假设你想要键绑定在当前缓冲区中起作用。

因此，如果我们像这样转换这个到 lua。

```vim
nnoremap <Leader>w :write<CR>
```

我们必须这样做：

```lua
vim.api.nvim_set_keymap('n', '<Leader>w', ':write<CR>', {noremap = true})
```

这并不是世界上最大的事件，但是我们可以做一些事使它更方便一点。

- 设置一个别名

如果你喜欢简单的方法，可以把此函数赋值给短点名称。

```lua
local map = vim.api.nvim_set_keymap

map('n', '<Leader>w', ':write<CR>', {noremap = true})
```

- 创建一个帮助函数

如果你愿意放置更多的代码，可以创建另一个函数，一个带有默认值的函数。我之所以提到这点，是因为默认使我们的键绑定非递归是一种很好的实践。

```lua
local map = function (key)
  --get the extra options 设置默认选项
  local opts = { noremap = true }
  for i, v in Pairs(key) do
    if type(i) == 'string' then
      opts[i] = v
    end
  end

  -- basic support for buffer-scoped keybindings
  -- 缓冲区作用域键绑定的基本支持
  local buffer = opts.buffer
  opts.buffer = nil

  if buffer then
    vim.api.nvim_buf_set_keymap(0, key[1], key[2], key[3], opts)
  else
    vim.api.nvim_set_keymap(key[1], key[2], key[3], opts)
  end
end
```

基本用法：

```lua
map{'n', '<Leader>w', ':write<CR>'}
```

这个函数很酷的一点是，它利用了我们在 lua 中创建表的方法。因此这是有效的。

```lua
map {noremap = false, 'n', '<Leader>e', '%'}
```

并且这样也行：

```lua
map {'n', '<Leader>e', '%', noremap =false}
```

#### 调用 lua 函数

如果我们会使用已经学到的在 vimscript 调用 lua 的知识，那么我们就可以这样做。

假设我们有一个 lua 模块叫做 `usermod` ，且该模块有一个函数叫做 `somefunction` 。

```lua
local M = {}

M.somefunction = function ()
  print('all good')
end

return M
```

我们可以这样调用它：

```lua
vim.api.nvim_set_keymap(
  'n',
  '<Leader>w',
  "<cmd>lua require('usermod').somefunction()<CR>",
  {noremap = true}
)
```

现在，如果我们需要一个表达式，事件有点变化了。在这种情况下，我们不能使用 `<cmd>lua` 。我们需要变量 `v:lua` ，通过这个变量，我们可以调用存在于全局作用域中的 lua 函数。

为了给你演示这是如何工作的，我会创建一个聪明的 `<Tab>` 键。当自动补全菜单显示时，我想导航结果的列表，否则，它会起到常规 `<Tab>` 作用。

```lua
local t = function (str)
  return vim.api.nvim_replace_termcodes(str, true, true, true)
end

_G.smart_tab = function ()
  if vim.fn.pumvisible() == 1 then
    return t'<C-n>
  else
    return t'<Tab>'
  end

  vim.api.nvim_set_keymap(
    'i',
    '<Tab>',
    'v:lua.smart_tab()',
    {noremap = true, expr = true}
  )
```

> 在 lua 中，\_G 是包含所有全局变量的全局表。这并不是绝对必须的，但是我使用它来清楚地表明我是有意创建一个全局函数。

如果你想问为什么返回 `t'<C-n>'` ，这是因为我们不需要字符串 `<C-n>` ，我们需要代表 `ctrl+n` 的代码，`<Tab>` 也一样。

如果这个 API 你觉得不够好，考虑不要迁移你的键绑定。让它们留在脚本中，然后从 lua 中调用它。

```lua
vim.cmd 'source ~/.config/nvim/keymap.vim'
```

对于那些想要逃离 vimscript 的人，我推荐一些插件：

- [mapx.nvim](https://github.com/b0o/mapx.nvim)
- [Vimpeccable](https://github.com/svermeulen/vimpeccable)
- [bex](https://github.com/bkoropoff/bex.nvim)
- [nest](https://github.com/LionC/nest.nvim)

不必把它们都下载下来。每一个都有不同的创建键绑定的方法。挑选一个你最喜欢的。

## 插件管理器

讲到插件。你可能想要用 lua 编写的插件管理器，没有其他原因。现在看来，这些是你的选择：

- paq

一个简单、快速的插件管理器。我是认真的，该插件不到 300 行代码。安装后可以下载、升级和删除插件。就这样。如果你不需要其他任何功能，不要找了，这就是你要找的插件管理器。

- packer

如果你想要更多功能的， `packer` 是另一个选择。它拥有你期望的基本的功能外，还提供懒加载功能，支持 `luarocks`（对 lua 而言，这就像是一个包管理器），它能处理“本地插件”。而且有我不知的其他功能，但是关键的是这是一个功能完整的插件管理器。

- vim-packager

这个不是用 lua 编写的，但是我想使用它，因为它提供 lua api。它提供的功能比 `paq` 多，但是比 `packer` 少，因此，如果你正在寻找介于以上两者之间的，这可能是一个好的选择。

## 结论

回顾时间。我们学习了如何在 vimscript 中使用 lua。也了解了在 lua 中使用 vimscript。我们有工具可以在 neovim 中如何启用、禁用和修改所有类型的选项和变量。我们知道可用于创建键映射的方法，并且了解了其局限性。我们找到了如何使用不是使用 lua 编写的插件管理器，且看到了一些用 lua 编写的替代品。我说我们已经准备好在 neovim 中使用 lua 了。

对于那些想看到真实使用情况或其他什么，我会在 github 上分享链接到我的配置: [neovim](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim)

## 学习资源

- [learn x in y minutes: where X=lua](https://learnxinyminutes.com/docs/lua/)
- [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide)
- [:help lua-heredoc](https://neovim.io/doc/user/lua.html#:lua-heredoc)
- [:help lua-vim-variables](https://neovim.io/doc/user/lua.html#lua-vim-variables)
- [:help lua-stdlib](https://neovim.io/doc/user/lua.html#lua-stdlib)
- [:help function-list](https://neovim.io/doc/user/usr_41.html#function-list)
- [:help nvim_set_keymap()](<https://neovim.io/doc/user/api.html#nvim_set_keymap()>)
- [curist's bundle.lua](https://github.com/curist/dotvim/blob/98b161f0759d3316fcf6a776d03665d6ab4827ee/bundles.lua)

---

原文地址如下：
[Everything you need to know to configure neovim using lua](https://vonheikemen.github.io/devlog/tools/configuring-neovim-using-lua/?msclkid=5579e3a5a81c11ecb34a9ee447c625af)
