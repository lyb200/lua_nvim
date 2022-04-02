# 从 init.vim 到 init.lua

## 为什么是 lua？

Neovim 内嵌了 lua 5.1 运行时，用于创建你喜欢的编辑器更快和更强大的扩展。在 [Neovim 章节](https://neovim.io/charter/) 中，它列出了其目标之一是开发替代 VimL 的第一类 lua 脚本。这样做的原因之一是 VimL 是较慢的解释性语言，几乎没有优化。大部分时间花费在 vim 启动和会阻止编辑器主循环的用于解析和执行 vimscript 的插件动作中。这在 Neovim 主要维护者 Justin M.Keyes 的演讲中可以找到很好的解释，[我们可以拥有美好的东西](https://www.youtube.com/watch?v=Bt-vmPC_-Ho)。

随着最近在使用 lua 编写的主分支中引入内置的 LSP 客户端，我对 lua 所提供的可用性更感兴趣，并开始尝试在 Neovim 中使用 lua。我以前从来没有编写过 lua 代码，也没有看过很多关于如何在 Neovim 中利用 lua 运行时的使用指南，所以我想告知我是如何利用 Neovim 运行时提供的强大脚本功能的过程。鉴于我的经验还很基础，且提供的这些例子也很少，但我希望对于那些有兴趣使用 lua 扩展 Neovim 的人来说，这可以成为一个很好的起点。

## 准备开始

第一件让我困惑的事件是如何在 vim 和 vimscript 中使用 lua 代码。幸运的是，命令行输入：`:h lua` 得到的文档给出了一些在编辑器中如何使用 lua 的示例。我建议好好阅读它，以深入理解 Neovim 如何处理 lua 代码和 lua 源文件。下面是在编辑器中执行 lua 代码的不同方法的概述：

- 你可以在命令行运行 `:lua <你的代码>` 。这对键绑定、命令和其他一次性执行情况都有用。
- 在 VimL 文件中，你可以使用以下两行代码包裹 lua 代码：

  ```lua
  lua << EOF
    -- your lua cede here, 你的lua代码放在这里
  EOF
  ```

- 在 VimL 文件中，你可以使用 `lua` 关键字来执行与第一个示例相近的命令。（例如，`lua <你的代码>` )。

这里需要注意的一点是，Neovim 将在你设置的 `runtimepath` 中查找 lua 代码。另外，它会把`/lua/?.lua` 和 `/lua/?/init.lua` 追加到你的 `runtimepath` 中，因此，到 `nvim` 中查看 `/lua` 子目录是常见的做法。有关 Neovim 到哪里去寻找 lua 代码的详细信息，请参见 `:h lua-require` 。

## 第一个函数

> **免责声明**<br>
> 下面的一些使用 `vim.bo` API 调用的代码，目前只能在 Neovim 主分支中有效。

把你的 `init.vim` 迁徙到 lua 会是一件大工程，因此最好是从小处着手。第一个示例，我们将创建一个函数，而该函数会创建一个临时缓冲区。

该函数存在于我们称为 `tools` 文件中，因此在你的 nvim 配置文件位置创建它： `~/.config/nvim/lua/tools.lua` 。一旦我们创建了这个文件，我们会用一些模板代码来填充：

```lua
-- in tools.lua
local M = {}
function M.makeScratch()
end
return M
```

这里用该表 M ，其目的是允许我们不在全局作用域内进行操作，且在 Nvim 调用表内函数时只使用我们需要的成员。我们将使用 Neovim API 来创建一个临时缓冲区，所以，让我们在 `tools.lua` 文件中为它创建一个简写：

```lua
-- in tools.lua
local api = vim.api
local M = {}
function M.makeScratch()
end

return M
```

我们可以使用 `enew` 命令创建一个新的缓冲区，且 Neovim API 提供了一种在 lua 中调用 nvim 命令的方法：

```lua
-- in tools.lua
local api = vim.api
local M = {}
function M.makeScratch()
  api.nvim_command('enew')  -- equivalent to :enew ，等同于 :enew
end

return M
```

接下来，我们需要设置一些缓冲区选项，这样临时缓冲区就不会在缓冲区列表中出现，也不会为它创建一个 swapfile(交换文件)：

```lua
-- in tools.lua
local api = vim.api
local M = {}
function M.makeScratch()
  api.nvim_command('enew') -- 等同于：enew
  --set the current buffer's (buffer 0) buftype to nofile, 把当前缓冲区的类型设置为nofile
  vim.bo[0].buftype=nofile
  vim.bo[0].bufhidden=hide
  vim.bo[0]swapfile=false
end

return M
```

这就是我们创建临时缓冲区所需的全部内容！现在让我们在 `init.vim` 中使用它：

```vim
" in init.vim

command! Scratch lua require'tools'.makeScratch()
```

现在通过运行命令 `Scratch` ，就可以创建出一个临时缓冲区。

你可以将 `init.vim` 一次性移植到一个函数中，如果遇到问题，可以总是可以使用 `vim.api.nvim_command` ！在寻求帮助时，请确保检查 `:h api` 和 `:h lua` 。

## 使用 v:lua

变量 `v:lua` 可用在 vimscript 中调用 lua 函数。这种使用的一个很好的用例是访问 LSP 客户端的 ominifunc 函数。如果你想在 Rust 中使用 LSP 补全功能，你可以在你的配置中编写如下代码：

```vim
" in init.vim
lua << EOF
  local nvim_lsp = require 'nvim_lsp'
  nvim_lsp.rust_analyzer.setup =({})
EOF
```

```vim
" in ftplugin/rust.vim

set ominifunc=v:lua.vim.lsp.ominifunc
```

## 与 vim.fn 交互

在 lua 内部可以访问 vimscript 函数这非常有用，特别是在与自动加载函数或者有插件提供的函数交互时。在下面的示例中，我们有一个能执行 vimscript 函数的 autocmd 命令，即当 `VimEnter` 事件发生时，执行`tools#loadCscope` 函数。

```vim
" in autoload/tools.vim

function! tools#loadCscope() abort
  try
    silent cscope add cscope.out
  catch /^Vim\%((\a\+)\)\=:E/
  endtry
endfunction
```

```lua
-- in file that you source, such as init.lua
function sourceCscope()
  vim.fn['tools#loadCscope']() -- no auguments needed, 不需要参数
end

function nvim_create_augroups(definitions)
  for group_name, definition in pairs(definitions) do
    vim.api.nvim_command('augroup ' .. group_name)
    vim.api.nvim_command('autocmd!')
    for _, def in pairs(definition) do
      local command = table.concat(vim.tbl_flatten('autocmd', def), ' ')
      vim.api.nvim_command(command)
    end
    vim.api.nvim_command('augroup END')
  end
end

local autocmds = {
  startup = {
    {"VimEnter",        "*",      [[lua sourceCScope()]]};
  }
}

nvim_create_augroups(autocmds)
```

---

原文件 URL：

[From init.vim to init.lua](https://teukka.tech/luanvim.html)
