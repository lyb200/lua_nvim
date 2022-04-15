# Neovim 0.5 特性和切换到 init.lua

作者：Olivier Roques

- 2022-03-10 更新：小心，该指南现在已经过时了！
- 2021-07-18 更新：用 pylsp 取代 pyls
- 2021-07-02 更新：Neovim 0.5 发布！
- 2021-06-01 更新：切换到 vim.opt 来设置选项和更新 tree-sitter 这节
- 2021-04-18 更新：增加链接到 nvim-compe 和 awesome-neovim，新增一节提及其他插件。

Neovim 0.5 提供了一些主要的功能。该新版本的主要特性有：

- 内置 `LSP` 客户端。
- 在代码中具有更好语法高亮的 `tree-sitter` 。
- 改进 `Lua API` 和特别是支持把 `init.lua` 作为用户的配置。

这篇文章将帮助你编写非常基本的 `init.lua` 文件，其中包含了所有这些新特性。这里的设置当然是个人的选择，所以你可以随意忽略它们或者使用推荐的替代品。

我们将完全避免使用 `Vimscript` ，而是广泛地使用 `Lua` 和 `Neovim Lua API` 。关于这个问题的最好参考资料是 [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide) 。还可以查看 [Learn Lua in Y minutes](https://learnxinyminutes.com/docs/lua/) 以获得有关 Lua 语言快速概述。

## 目录

- 别名
- 插件
- 设置选项
- 键映射
- 配置 LSP 客户端
- 配置 Tree-sitter
- 命令和自动命令
- 其他插件
- 结论

## 别名

这篇文章后面部分，我们将使用这些别名：

```lua
local cmd = vim.cmd  -- 执行vim命令，例如：cmd('pwd')
local fn = vim.fn    -- 调用vim函数，例如：fn.bufnr()
local g = vim.g      -- 一个访问全局变量的表
local opt = vim.opt  -- 设置选项
```

## 插件

管理你的插件可以有几种选择：

- 继续使用你喜欢的插件管理器，例如：`vim-plug` 。这意味着需要一些 `Vimscript` 代码，因此我们将跳过该选项。
- 一个流行的选择是 `packer.nvim` 。它是用 Lua 编写的，绝对是个不错的选择。但它有点冗长，所以我们不会在这里使用它。
- 我们将使用 `paq-nvim` ，一个最小化的包管理器。像这样安装它：

```lua
$ git clone https://github.com/savq/paq-nvim.git \
    "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/pack/paqs/opt/paq-nvim
```

我们的 `init.lua` 文件像下面这样开始：

```lua
cmd 'packadd paq-nvim'               -- 加载包管理器
local paq = require('paq-nvim').paq  -- 一个方便的别名
paq {'savq/paq-nvim', opt = true}    -- paq-nvim 管理自己
paq {'shougo/deoplete-lsp'}
paq {'shougo/deoplete.nvim', run = fn['remote#host#UpdateRemotePlugins']}
paq {'nvim-treesitter/nvim-treesitter'}
paq {'neovim/nvim-lspconfig'}
paq {'junegunn/fzf', run = fn['fzf#install']}
paq {'junegunn/fzf.vim'}
paq {'ojroques/nvim-lspfuzzy'}
g['deoplete#enable_at_startup'] = 1  -- 启动时启用 deoplete
```

现在你可以运行 `:PaqInstall` 来安装所有插件，`:PaqUpdate` 来更新它们，而 `:PaqClean` 用于删除不再使用的插件。

有关这些插件的说明：

- `deoplete-lsp` 和 `deoplete.nvim` ：这两个插件提供自动补全功能。完全用 Lua 编写的两个流行的替代方案是 `nvim-compe` 和 `completion.nvim` 。这里我使用 deoplete 完全是个人的选择。
- `nvim-treesitter` ：tree-sitter 已整合到 Neovim 0.5 中，但是语言模块没有整合。使用该插件，你可以进行配置和对它们进行安装。
- `nvim-lspconfig` ：Neovim 0.5 提供了一个本地的 LSP 客户端，但是你仍然需要对你想编写的编程语言配置一个服务器。该插件就是为了方便语言服务器的配置而提供的。
- `fzf, fzf.vim` 和 `lspfuzzy` ：FZF 是一个非常流行的模糊查找器，lspfuzzy 是我开发的一个插件，它使 Neovim LSP 客户端使用 FZF 而不是 quickfix （快速修复）列表。同样，这也是我个人的选择：用 Lua 编写的最受欢迎的模糊查找器目前是 `telescope.nvim` 。

这里是在寻找符号引用：LSP 和 FZF 时，LSP 和 FZF 是如何相互作用的。

你也可以在这里：[awesome-neovim](https://github.com/rockerBOO/awesome-neovim) 找到一个专门为开发 Neovim 的插件完整列表。

## 设置选项

想要使用 Lua 进行选项设置，请使用 `vim.opt` 表，该表的行为与 Vimscript 中的`set` 函数完全相同。该表应该覆盖了大部分用法。

否则，如果需要在本地（只在缓冲区或窗口中）或者全局设置选项，Neovim Lua API 提供了 3 个表：

- `vim.o` 用于设置全局选项：`vim.o.hidden = true`
- `vim.bo` 用于设置缓冲区作用域内的选项：`vim.bo.expandtab = true`
- `vim.wo` 用于设置窗口作用域内的选项：`vim.wo.number = true`

想要了解某个特殊的选项可用于那个作用域，请查看 Vim 帮助页。

要得到有关 `vim.opt` 更进一步的文档，请查看帮助 `:h lua-vim-options` 。这里是一个用于演示`vim.opt` （这里使用别名 `opt` ）使用的一些有用的设置列表：

```lua
cmd 'colorscheme desert'            -- 这里放置你喜欢的颜色主题
opt.completeopt = {'menuone', 'noinsert', 'noselect'}  -- 为deoplete设置补全选项
opt.expandtab = true                -- 使用空格替代tab
opt.hidden = true                   -- 启用缓冲区隐藏功能
opt.ignorecase = true               -- 忽略大小写
opt.joinspaces = false              -- 进行行连接时不使用双空格
opt.list = true                     -- 显示一些可视的字符
opt.number = true                   -- 显示行号
opt.relativenumber = true           -- 设置相对行号
opt.scrolloff = 4                   -- 滚动屏幕时上下都会留下的行数
opt.shiftround = true               -- 四舍五入到一个缩进
opt.shiftwidth = 2                  -- 一个缩进的大小
opt.sidescrolloff = 8               -- 滚动屏幕时左右都会留下的列数
opt.smartcase = true                -- 如果有大写字母时不会忽略大小写
opt.smartindent = true              -- 自动插入缩进
opt.splitbelow = true               -- 把新打开的窗口放在当前窗口下面
opt.splitright = true               -- 把新打开的窗口放在当前窗口右边
opt.tabstop = 2                     -- 一个tab的空格数量
opt.termguicolors = true            -- 终端图形界面支持真色彩
opt.wildmode = {'list', 'longest'}  -- 命令行补全模式
opt.wrap = false                    -- 禁止自动换行
```

## 映射

`vim.api.nvim_set_keymap()` 函数允许你定义新的映射。特殊的行为例如，`noremap` 必须以一个表的格式传递给这个函数。这里有一个帮助函数用于创建默认设置 `noremap` 为 `true` 的 选项映射：

```lua
local function map(mode, lhs, rhs, opts)
  local options = {noremap = true}
  if opts then options = vim.tbl_extend('force', options, opts) end
  vim.api.nvim_set_keymap(mode, lhs, rhs, options)
end
```

以下是用于演示如何使用以上帮助函数进行映射的建议：

```lua
map('', '<leader>c', '"+y')      -- 在常规、可视、选择和操作模式下，复制到剪贴板
map('i', '<C-u>', '<C-g>u<C-u>') -- 使用<C-u>的撤销更友好
map('i', '<C-w>', '<C-g>u<C-w>') -- 使用<C-w>的撤销更友好

-- 使 <Tab> 能在自动补全菜单中进行导航
map('i', '<S-Tab>', 'pumvisible() ? "\\<C-p>" : "\\<Tab>"', {expr = true})
map('i', '<Tab>', 'pumvisible() ? "\\<C-n>" : "\\<Tab>"', {expr = true})

map('n', '<C-l>', '<cmd>noh<CR>')   -- 清除高亮显示
map('n', '<leader>o', 'm`o<Esc>``') -- 在常规模式下，在当前行下面插入一个新行，光标位置不变
```

## 配置 Tree-sitter

tree-sitter 是很容易配置的：

```lua
local ts = require 'nvim-treesitter.configs'
ts.setup {ensure_installed = 'maintained', highlight = {enable = true}}
```

这里这个 `maintained` 的值表示我们希望使用所有维护的语言模块。你还需要设置 `highlight` 为 `true`，否则，该插件将被禁用。完整的文档可以在 [Github 页面](https://github.com/nvim-treesitter/nvim-treesitter) 找到。

高亮显示不是唯一可用的模块：你还可以启用 `indent`（缩进）模块，该模块会根据文件类型和上下文提供缩进。更多关于可用模块的详细资料参考 [这里](https://github.com/nvim-treesitter/nvim-treesitter#available-modules) 。

## 配置 LSP 客户端

感谢 `lspconfig` 插件的帮助，使配置 LSP 客户端相对简单：

- 首先安装一个你所需语言的服务器：点击 [这里](https://microsoft.github.io/language-server-protocol/implementors/servers/) 查看可用的实现。
- 然后，调用 `setup()` 以启用该服务器。点击 [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) 查看有关高级配置的文档。

以下是一个为 Python 和 C/C++ 设置服务器的配置示例（假设对应的 `pylsp` 和 `ccls` 已经安装好）。我们也为大多数 LSP 命令创建映射。

```lua
local lsp = require 'lspconfig'
local lspfuzzy = require 'lspfuzzy'

-- 我们为 ccls和pylsp 使用默认的设置：选项表保持空
lsp.ccls.setup {}
lsp.pylsp.setup {}
lspfuzzy.setup {}  -- 使 LSP 客户端使用 FZF ，而不是快速修复列表

map('n', '<space>,', '<cmd>lua vim.lsp.diagnostic.goto_prev()<CR>')
map('n', '<space>;', '<cmd>lua vim.lsp.diagnostic.goto_next()<CR>')
map('n', '<space>a', '<cmd>lua vim.lsp.buf.code_action()<CR>')
map('n', '<space>d', '<cmd>lua vim.lsp.buf.definition()<CR>')
map('n', '<space>f', '<cmd>lua vim.lsp.buf.formatting()<CR>')
map('n', '<space>h', '<cmd>lua vim.lsp.buf.hover()<CR>')
map('n', '<space>m', '<cmd>lua vim.lsp.buf.rename()<CR>')
map('n', '<space>r', '<cmd>lua vim.lsp.buf.references()<CR>')
map('n', '<space>s', '<cmd>lua vim.lsp.buf.document_symbol()<CR>')
```

## 命令和自动命令

很不幸，Neovim 还没有创建命令和自动命令的接口。实现这些接口的工作在进行中，有关命令开发，参见 [PR#11613](https://github.com/neovim/neovim/pull/11613) ，有关自动命令，参见 [PR#12378](https://github.com/neovim/neovim/pull/12378) 。

你仍然可以使用 Vimscript 的 `vim.cmd` 来定义命令或自动命令。例如，Neovim 0.5 引入的 [highlight on yank 在复制时高亮显示](https://github.com/neovim/neovim/pull/12279) 特性，其主要作用是高亮显示被复制的文本。你可以像下面这样作为自动命令启用它：

```lua
-- 在可视模式下禁用
cmd 'au TextYankPost * lua vim.highlight.on_yank {on_visual = false}'
```

## 其他插件

我前面已经加入了一些插件，我认为这些插件对于基本使用 Neovim 来说是最重要的。这里还有一些插件，它们利用 Neovim 的新特性，可以极大地改善你的工作流程：

- [gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim) ：在 Lua 中取代 [vim-gitgutter](https://github.com/airblade/vim-gitgutter) 。
- [indent-blankline.nvim](https://github.com/lukas-reineke/indent-blankline.nvim) ：使用 [新的虚拟文本特性](https://github.com/neovim/neovim/pull/13952) 来显示缩进行，包括在空行上。
- [nvim-dap](https://github.com/mfussenegger/nvim-dap) ：在 Neovim 中实现了 [Dedug Adapter Protocol 调试适配器协议](https://microsoft.github.io/debug-adapter-protocol/)（基本上相当于调试器的 LSP）。注意，设置它可能有点棘手，而且现在还没有多少适配器。
- [nvim-treesitter/nvim-treesitter-textobjects](https://github.com/nvim-treesitter/nvim-treesitter-textobjects) ：一个基于 treesitter 引擎创建文本对象的插件，例如，用于选择参数/函数，或者交换代码元素。
- [nvim-lightbulb](https://github.com/kosayoda/nvim-lightbulb) ：一个简单的插件，每当当前光标位置有代码操作可用时，就在标志栏中显示一个灯泡。

## 结论

以下是 init.lua 全部代码：

```lua
-------------------- HELPERS -------------------------------
local cmd = vim.cmd  -- to execute Vim commands e.g. cmd('pwd')
local fn = vim.fn    -- to call Vim functions e.g. fn.bufnr()
local g = vim.g      -- a table to access global variables
local opt = vim.opt  -- to set options

local function map(mode, lhs, rhs, opts)
  local options = {noremap = true}
  if opts then options = vim.tbl_extend('force', options, opts) end
  vim.api.nvim_set_keymap(mode, lhs, rhs, options)
end

-------------------- PLUGINS -------------------------------
cmd 'packadd paq-nvim'               -- load the package manager
local paq = require('paq-nvim').paq  -- a convenient alias
paq {'savq/paq-nvim', opt = true}    -- paq-nvim manages itself
paq {'shougo/deoplete-lsp'}
paq {'shougo/deoplete.nvim', run = fn['remote#host#UpdateRemotePlugins']}
paq {'nvim-treesitter/nvim-treesitter'}
paq {'neovim/nvim-lspconfig'}
paq {'junegunn/fzf', run = fn['fzf#install']}
paq {'junegunn/fzf.vim'}
paq {'ojroques/nvim-lspfuzzy'}
g['deoplete#enable_at_startup'] = 1  -- enable deoplete at startup

-------------------- OPTIONS -------------------------------
cmd 'colorscheme desert'            -- Put your favorite colorscheme here
opt.completeopt = {'menuone', 'noinsert', 'noselect'}  -- Completion options (for deoplete)
opt.expandtab = true                -- Use spaces instead of tabs
opt.hidden = true                   -- Enable background buffers
opt.ignorecase = true               -- Ignore case
opt.joinspaces = false              -- No double spaces with join
opt.list = true                     -- Show some invisible characters
opt.number = true                   -- Show line numbers
opt.relativenumber = true           -- Relative line numbers
opt.scrolloff = 4                   -- Lines of context
opt.shiftround = true               -- Round indent
opt.shiftwidth = 2                  -- Size of an indent
opt.sidescrolloff = 8               -- Columns of context
opt.smartcase = true                -- Do not ignore case with capitals
opt.smartindent = true              -- Insert indents automatically
opt.splitbelow = true               -- Put new windows below current
opt.splitright = true               -- Put new windows right of current
opt.tabstop = 2                     -- Number of spaces tabs count for
opt.termguicolors = true            -- True color support
opt.wildmode = {'list', 'longest'}  -- Command-line completion mode
opt.wrap = false                    -- Disable line wrap

-------------------- MAPPINGS ------------------------------
map('', '<leader>c', '"+y')       -- Copy to clipboard in normal, visual, select and operator modes
map('i', '<C-u>', '<C-g>u<C-u>')  -- Make <C-u> undo-friendly
map('i', '<C-w>', '<C-g>u<C-w>')  -- Make <C-w> undo-friendly

-- <Tab> to navigate the completion menu
map('i', '<S-Tab>', 'pumvisible() ? "\\<C-p>" : "\\<Tab>"', {expr = true})
map('i', '<Tab>', 'pumvisible() ? "\\<C-n>" : "\\<Tab>"', {expr = true})

map('n', '<C-l>', '<cmd>noh<CR>')    -- Clear highlights
map('n', '<leader>o', 'm`o<Esc>``')  -- Insert a newline in normal mode

-------------------- TREE-SITTER ---------------------------
local ts = require 'nvim-treesitter.configs'
ts.setup {ensure_installed = 'maintained', highlight = {enable = true}}

-------------------- LSP -----------------------------------
local lsp = require 'lspconfig'
local lspfuzzy = require 'lspfuzzy'

-- We use the default settings for ccls and pylsp: the option table can stay empty
lsp.ccls.setup {}
lsp.pylsp.setup {}
lspfuzzy.setup {}  -- Make the LSP client use FZF instead of the quickfix list

map('n', '<space>,', '<cmd>lua vim.lsp.diagnostic.goto_prev()<CR>')
map('n', '<space>;', '<cmd>lua vim.lsp.diagnostic.goto_next()<CR>')
map('n', '<space>a', '<cmd>lua vim.lsp.buf.code_action()<CR>')
map('n', '<space>d', '<cmd>lua vim.lsp.buf.definition()<CR>')
map('n', '<space>f', '<cmd>lua vim.lsp.buf.formatting()<CR>')
map('n', '<space>h', '<cmd>lua vim.lsp.buf.hover()<CR>')
map('n', '<space>m', '<cmd>lua vim.lsp.buf.rename()<CR>')
map('n', '<space>r', '<cmd>lua vim.lsp.buf.references()<CR>')
map('n', '<space>s', '<cmd>lua vim.lsp.buf.document_symbol()<CR>')

-------------------- COMMANDS ------------------------------
cmd 'au TextYankPost * lua vim.highlight.on_yank {on_visual = false}'  -- disabled in visual mode
```

我希望你会发现这个指南是有用的。你可以找到我的 init.lua ，以上大部分代码都是从这里获取的：[init.lua](https://github.com/ojroques/dotfiles/blob/master/nvim/.config/nvim/init.lua) 。

也许你会对我开发的 Vim/Neovim 插件感兴趣：

- [nvim-lspfuzzy](https://github.com/ojroques/nvim-lspfuzzy) ：使用 FZF 扩展 Neovim 内置的 LSP 客户端。
- [vim-oscyank](https://github.com/ojroques/vim-oscyank) ：使用 [OSC52](https://oroques.dev/2020/11/27/vim-osc52.html) 从任何地方（包括通过 SSH）拷贝文本。
- [nvim-bufdel](https://github.com/ojroques/nvim-bufdel) ：改善缓冲区的删除功能。
- [vim-scrollstatus](https://github.com/ojroques/vim-scrollstatus) ：在状态栏中显示滚动条（对于 Neovim 0.5 ，[这里](https://github.com/dstein64/nvim-scrollview) 或者 [这里](https://github.com/Xuyuanp/scrollbar.nvim) 有更好的替代品）。

---

原文件网址：[Neovim 0.5 features and the switch to init.lua](https://oroques.dev/2020/01/07/neovim-05.html) 。
